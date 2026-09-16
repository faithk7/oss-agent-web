# 实现计划：OSS Agent 阅读网站

**分支**：`001-oss-agent-reading-site` | **日期**：2026-09-16 | **规格**：[../../spec.md](../../spec.md)

**输入**：仓库根目录 `spec.md`（中文产品需求，待评审草案）。本仓库尚未初始化为 Spec Kit 项目；本文件是首个功能（也就是整个 v1 站点）的 Spec Kit `plan.md`。

**状态**：等待批准。批准前不要写代码。

## 摘要

建设一个围绕开源软件与编程 Agent 的精选阅读网站，覆盖 Kimi Code、Codex 等主题。站长在 git 里写 Markdown。批处理 CLI 同步并渲染这些文件，离线打标签 Agent 再从受控词表提出标签建议。读者浏览文章库、按标题搜索、按主题／标签筛选，阅读免费正文与付费试读。付费全文只在服务端完成权益校验后返回。首版只提供一种有期限的全站通行证，通过微信支付、支付宝或 Stripe 国际卡购买，也可用一次性兑换码开通，权益相同。

技术方案：一个 Next.js App Router 应用、一个 PostgreSQL 数据库、一套负责同步／打标签／管理的 CLI。付费 HTML 不得打进客户端静态包。试读 HTML 与正文 HTML 在同步时分别生成，请求时按权限选择返回哪一份。

## 技术上下文

**语言／版本**：TypeScript 5.x，Node.js 22，strict 模式。

**主要依赖**：Next.js 15（App Router、React Server Components）、Auth.js v5、Drizzle ORM、Zod、带清理的 `unified`／`remark`／`rehype`、`gray-matter`、统一支付接口后的微信支付 API v3、支付宝 SDK 与 Stripe SDK、Resend（magic link 邮件）。

**存储**：PostgreSQL 16。Markdown 权威源在 `content/articles/`。生成的试读 HTML、正文 HTML、搜索字段、标签、用户、订单、权益授予、兑换码都放在 Postgres。首版不加独立搜索服务，不加 Redis。

**测试**：Vitest 覆盖单元／集成（权益、同步、打标签、搜索 AND、兑换并发）。Playwright 覆盖读者路径。各支付适配器合约测试覆盖支付回调；Stripe／支付宝使用可用测试环境，微信支付按产品可用测试能力联调并在启用前完成受控小额实付验证。兑换并发测试使用真实 Postgres（Testcontainers 或本地 compose）和 `pg` advisory lock。

**目标平台**：中国大陆云的 Linux 服务器（一家国内云，阿里云/腾讯云择一：容器或 ECS + 托管 PostgreSQL 16 + 对象存储 + 厂商 CDN）。Vercel 不作为生产选项（境内可达性差且不支持备案域名）。界面首版为中文。文章元数据与标题搜索同时支持 `zh` 和 `en`。

**项目类型**：单个 Web 应用加仓库内 CLI。不是前后端拆分、不是交易市场、不是 API 产品。

**性能目标**：文章库与搜索在最多 2k 篇已发布文章下，排除冷启动的 p95 < 300ms。不同步模型调用时，200 个 Markdown 文件同步 < 30s。打标签在离线跑，允许到分钟级。

**约束**：
- 付费 Markdown、完整正文 JSON、包含正文的 source map、正文 HTML，不得出现在公开静态资源或未认证响应中。
- 试读与正文是两份独立产物；服务端不得先把全文发给浏览器再隐藏。
- 共享／公共缓存不得存放需鉴权的正文响应。
- 文章阅读、搜索、认证、兑换按 IP 限流；登录后同时按账户限流；触发时返回 429。
- Agent 不得自行发布文章、改价格、改权限或改写正文。
- Markdown 中的原始 HTML 必须经过持续维护的清理工具；禁止 `javascript:` 链接与可执行嵌入。
- 不禁用复制、右键或键盘快捷键。

**规模／范围**：一名站长、数百篇文章、数千读者。一种通行证 SKU。六个公开页面，管理入口首版以 CLI 为主。首版不做浏览器内编辑、评论、读者投稿、语义搜索、多商户分账、独立移动应用。

## 宪章检查

仓库里还没有 `.specify/memory/constitution.md`。下列门禁取自 `spec.md`，约束本计划。设计后再核： **通过**。

| 门禁 | 规则 | 计划如何满足 |
| --- | --- | --- |
| G1 Markdown 权威 | Git 中的 Markdown 是唯一编辑源。自动标签与搜索数据独立存储。 | `content/articles/` 为源。数据库保存派生目录、HTML、标签。 |
| G2 一应用一库一批处理 | 首版不加搜索集群、队列或向量库。 | Next.js + Postgres + CLI。 |
| G3 服务端访问控制 | 每次受保护正文请求都重新校验权益。试读 ≠ 正文。 | 两列 HTML；RSC／数据层按授予记录分支。 |
| G4 Agent 隔离 | 打标签离线执行，模型输入视为不可信，不得改发布／价格／正文。 | CLI 用 JSON schema 校验，写权限仅限标签相关表。 |
| G5 权益记账 | 购买与兑换各自生成独立授予记录，只生效一次。从 `max(现在, 当前到期)` 起延长。撤销按剩余有效授予重算。 | `entitlement_grants` 用来源唯一键，并重算 `access_until`。 |
| G6 简单 | 不做浏览器编辑器、评论、行为追踪推荐、全文／语义搜索、多商户分账。 | 见「明确不做」。 |
| G7 先测钱相关路径 | 权限边界、支付幂等、兑换并发、搜索与打标签必须有自动化检查。上线前走支付沙箱。 | 见本文测试计划。 |

用户最新需求明确要求微信登录、微信支付和支付宝；该授权取代原规格的单支付服务商限制。仍保持单一 SKU、单应用和同一套权益账本，不增加分账或订阅计费。

## 调研结论（Phase 0，内联）

以下内容替代 Spec Kit 的 `research.md`。`spec.md` 第 12 节和技术栈未定项都在这里锁死，避免实现时再卡住。

### 决策 1 — 应用栈：Next.js App Router

- **决定**：TypeScript + Next.js 15 App Router，单包。
- **理由**：规格要求一个 Web 应用同时负责页面、账户、搜索、支付与兑换。RSC 和 Route Handlers 能把付费 HTML 留在服务端。同一仓库可以用 `tsx` 暴露 CLI，不必再起一个服务。
- **备选**：Nuxt／Nitro（可以，但这套鉴权拆分上 Auth.js 资料更熟）。Astro + API（适合内容站，付费会话会变成第二个应用）。Django／Rails（对 Markdown 目录来说偏重）。拆 `frontend/` + `backend/`（违反规格第 9 节）。

### 决策 2 — 数据库：PostgreSQL + Drizzle

- **决定**：PostgreSQL 16，Drizzle ORM，drizzle-kit 做迁移。
- **理由**：规格要求授予记录、一次性兑换码、参数化搜索都有关系完整性。Drizzle 让标题 `ILIKE` 和标签 AND 的 SQL 看得见。标题搜索用 `LOWER(title) LIKE '%' || LOWER($q) || '%'` 并限制长度——按规格不依赖中文分词。
- **备选**：SQLite（兑换并发弱）。Prisma（能用；更想直接控 SQL，所以选 Drizzle）。Meilisearch／OpenSearch（明确延后）。

### 决策 3 — 认证：微信登录 + GitHub + magic link

- **决定**：Auth.js 管理本站会话，增加微信开放平台网站应用扫码登录；保留 GitHub OAuth 和 Resend magic link。微信不是企业微信，不要求用户必须有邮箱。GitHub OAuth 与邮件 magic link 为辅助通道，境内可达性弱于微信；微信扫码是面向大陆用户的主登录方式。
- **浏览器范围**：桌面网站支持微信扫码；手机浏览器保留 GitHub／邮件登录及跨设备扫码说明，不假设手机能扫描自身屏幕。微信扫码登录只支持桌面网站二维码场景。
- **身份**：以服务端换取的 `(appid, openid)` 唯一识别微信账号；`unionid` 可空，不凭昵称／头像／邮箱自动合并账户。登录后重新验证目标身份才能绑定，权益始终绑定内部 `user.id`。
- **安全与接入**：一次性 state、限定回调域名、服务端 code 交换、取消／过期／重放处理。网站应用 AppID/Secret 与微信支付商户密钥分开配置。详见 [合约](plan/contracts.md)。

### 决策 4 — 支付：微信支付 + 支付宝 + Stripe

- **v1 必须支持**：微信支付和支付宝；保留 Stripe 国际卡适配器。一个有期限通行证 SKU，各通道授予完全相同的权限与时长。
- **微信支付**：桌面采用 Native 二维码；手机外部浏览器采用已开通的 H5 支付，微信内采用已开通的 JSAPI 支付。JSAPI 的 payer openid 必须来自支付所用 AppID 的授权，不能复用网站登录 AppID 的 openid。
- **支付宝**：桌面使用电脑网站支付，手机外部浏览器使用手机网站支付；微信内遇到限制时提示在外部浏览器打开。服务端生成支付请求，不使用个人收款码或截图确认。
- **国际卡**：Stripe 保留为独立通道，其商户准入／打款资格需上线前确认；不再是微信支付、支付宝交付的前提。
- **价格与订单**：微信支付／支付宝以 CNY 分计价，Stripe 使用单独服务端定价；不做浏览器换汇。创建订单时固化通道、金额、币种、时长。每个订单只绑定一个通道，重试幂等，切换通道建立明确的新订单。
- **回调与对账**：各适配器独立验签、解析和回执，统一到订单、支付事件与权益事务；查询付款状态也通过同一适配器。回跳、扫码和客户端 SDK 返回均不作为授予证据。首版无退款流程，渠道侧支付争议由管理员人工撤销对应授予兜底。
- **上线门禁**：网站微信登录应用审核，以及微信支付／支付宝商户、产品权限、回调域名和密钥配置分别检查。某通道未开通时明确显示不可用，其他通道和兑换码继续服务；不将降级视为该通道验收完成。

### 决策 5 — 同步时生成试读

- **决定**：同步分别写入清理后的 `preview_html` 和 `body_html`。试读来源按顺序：`preview=custom` 时用 `preview_markdown`（缺失或为空则同步报错）；源文件里若有 `<!-- preview:start -->`／`<!-- preview:end -->` 则用标记区间（标记必须成对，否则同步报错）；否则 `standard` 取正文开头约 30%（按正文纯文本字符计，CJK 计 1、非 CJK 计 0.5），在小节或段落边界收尾，同步时生成，不在请求时截。`summary` 档的试读 HTML 为空。请求路径只选择返回哪一列。搜索响应永不包含试读或正文 HTML。
- **理由**：规格 4.3 禁止把全文发给浏览器再截断。源文件里的标记让试读仍由站长控制。
- **备选**：前端截断（规格否决）。随机摘录（规格否决）。

### 决策 6 — 打标签模型：OpenAI 兼容 CLI

- **决定**：`pnpm tags:run` 调用 OpenAI 兼容的 Chat Completions（`TAGGER_BASE_URL`、`TAGGER_API_KEY`、`TAGGER_MODEL`）。默认指向 DeepSeek：`TAGGER_BASE_URL=https://api.deepseek.com`、`TAGGER_MODEL=deepseek-flash`（DeepSeek V4.1 Flash，官方当前模型 id；不依赖旧别名路由）；换其他 OpenAI 兼容服务只改环境变量。提示词把标题、摘要、正文当不可信数据，并附上词表。输出必须通过 Zod schema：`tags: string[]`（词表内，最多 8 个）与 `new_terms: string[]`（词表外新词建议，不占 8 个上限）；新词落 `taxonomy_terms` 且 `public=false`，站长接受后才成为公开筛选项。默认每天 cron 一次，手动跑同一条命令。正文允许出站到模型服务；若未来要求正文留在本地，输入降级为标题、摘要与试读。
- **理由**：规格允许联网模型调用。OpenAI 兼容地址可以指向 Moonshot／Kimi、OpenAI 或本地网关，不用改代码。
- **备选**：完全本地 llama.cpp（以后走同一组环境变量即可）。请求链路上打标签（否决：不是离线）。

### 决策 7 — 界面语言与收录

- **决定**：界面文案用中文。展示时按 Asia／Shanghai 渲染日期，存储仍用 UTC。`robots.txt` 允许公开元数据路由；付费正文页始终 `noindex`；公开页在上线 SEO 决策把 `PUBLIC_INDEXING=true` 打开之前也 `noindex`。
- **理由**：规格是中文，且收录策略要到上线前才定。默认不收录是安全门禁。

### 决策 8 — 托管形态

- **决定**：生产环境在中国大陆云：应用跑一个 Node 进程（ECS 或容器服务），数据库用托管 PostgreSQL 16（自带备份/WAL），媒体放实例持久卷或对象存储，静态产物与公开媒体走厂商 CDN（备案后启用）。定时任务用主机 cron；GitHub Action 跨境调境内数据库仅作备选。本地开发只用 `docker compose` 起 Postgres。
- **理由**：规格写明任务量未到之前不加队列。

## 架构

```text
                    git: content/articles/*.md
                                |
                         pnpm content:sync
                         （加锁、校验、
                          渲染试读／正文、
                          upsert 目录）
                                |
                                v
                         PostgreSQL 16
                          ^     ^     ^
          pnpm tags:run   |     |     |  Auth.js 会话
          （词表、         |     |     |
           模型 HTTP、     |     |     |
           标签决策）      |     |     |
                          |     |     |
   读者浏览器 --> Next.js RSC／Route Handlers
                          |           \
                     GET 文章          POST 兑换
                     GET 搜索          POST 支付通知
                     GET 文章库        POST Checkout 会话
                          |
                     权益重算：
                     按授予时间折叠 duration
                     未撤销、来源唯一
```

没有环。模型 HTTP 只从打标签 CLI 出站。支付适配器入站为验签通知，出站包括创建支付和查单。

降级：
- 打标签或模型挂了：文章仍可发布；上次成功标签保留；本次运行记 `failed`，可重跑。
- 单一支付通道故障或未开通：其他已启用通道与兑换码仍能开通权限；收银台给出明确错误，前端不得自己宣布「支付成功」。
- 邮件／Resend 挂了：微信／GitHub 登录仍可用；微信登录故障不影响其他登录方式。

回滚：应用无状态。迁移只向前、只加列。授予记录只追加（撤销用标记），错误发布不必回滚支付。

## CDN 与缓存分层

| 内容 | CDN 策略 |
| --- | --- |
| `/_next/static/*` 构建产物（构建不含付费正文） | 长缓存 immutable，按构建 hash 寻址 |
| `/media/public/*` 公开媒体 | 长缓存 immutable，按内容 hash 寻址，无需失效 |
| 公开 HTML（文章库、搜索、匿名试读页） | v1 源站动态直出，CDN 不缓存 HTML；后续可评估匿名缓存键 |
| 付费响应（有权益正文视图、`/media/paid/*`、账户、收银台、webhook） | 绝不缓存：源站 `Cache-Control: private, no-store`，CDN 规则显式绕过，仅透传 |

- 托管形态：生产环境用国内云厂商 CDN（加速域名需 ICP 备案，备案完成后启用），回源到应用或对象存储；公开媒体 CDN 化只换「按 hash 取文件」这一层，`article_media` 表不变。付费媒体即使迁对象存储，也走签名 URL 或鉴权代理，不进公共 CDN 缓存。Cloudflare 中国网络需 ICP 备案与企业合作，不采用。
- 失效：全部按内容／构建 hash 寻址，无需主动失效；HTML 不缓存，发布即时生效。
- 门禁：CDN 必须服从源站 `Cache-Control`；上线检查包含「经 CDN 域名请求付费路径确认无缓存命中」。

## 项目结构

### 本功能文档

```text
spec.md                                          # 产品需求（已有）
specs/001-oss-agent-reading-site/
├── plan.md                                      # 本文件
└── tasks.md                                     # 之后由 speckit.tasks／实现阶段生成
```

调研结论仍内联在本文件；可执行的模型、合约、安全、运维和验收细节拆到下列专题文件，避免主计划过长且方便分别评审。

为便于实现和评审，以下专题计划从本文件拆出：

- [数据模型与一致性](plan/data-model.md)
- [接口与 CLI 合约](plan/contracts.md)
- [安全、隐私与访问控制](plan/security.md)
- [运维、部署与恢复](plan/operations.md)
- [验收标准与测试矩阵](plan/acceptance.md)

### 源码（仓库根）

```text
content/
├── articles/
│   └── kimi-code-getting-started/
│       ├── index.md                  # YAML front matter + Markdown（权威源）
│       └── assets/
│           ├── setup.png             # 配图、截图
│           └── architecture.svg      # 手绘／导出的架构图
├── taxonomy.yml                      # 受控主题与标签词表
└── redirects.yml                     # 已发布 slug 变更时的显式重定向

src/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                      # 文章库：精选 + 最新 + 搜索
│   ├── search/page.tsx
│   ├── articles/[slug]/page.tsx      # 元数据始终返回；正文或试读按授予决定
│   ├── pricing/page.tsx
│   ├── checkout/[orderId]/page.tsx   # 收银台等待／结果，只展示本地订单状态
│   ├── account/page.tsx
│   ├── login/page.tsx
│   ├── robots.ts
│   ├── media/public/[hash]/route.ts  # 试读引用过的图，可匿名
│   ├── media/paid/[hash]/route.ts    # 付费正文专用图，走权益校验
│   └── api/
│       ├── auth/[...nextauth]/route.ts
│       ├── checkout/route.ts
│       ├── orders/[id]/route.ts
│       ├── webhooks/wechatpay/route.ts
│       ├── webhooks/alipay/route.ts
│       ├── webhooks/stripe/route.ts
│       └── redeem/route.ts
├── components/                       # 文章卡片、筛选、试读区、通行证 CTA
├── lib/
│   ├── db/                           # drizzle schema 与客户端
│   ├── content/                      # 解析、校验、清理、试读抽取、同步
│   ├── search/                       # 标题子串 + 标签 AND
│   ├── recommend/
│   ├── auth/
│   ├── entitlements/                 # 授予生效、重算 access_until
│   ├── payments/                     # PaymentProvider + 微信支付／支付宝／Stripe 适配器
│   ├── redeem/
│   ├── tagging/                      # 提示词、schema、人工／自动／拒绝合并
│   └── rate-limit/                   # IP + 用户键，存在 Postgres
├── scripts/
│   ├── content-sync.ts
│   ├── tags-run.ts
│   ├── tags-review.ts
│   ├── codes-issue.ts
│   ├── codes-disable.ts
│   ├── entitlements-revoke.ts
│   ├── ops-status.ts
│   ├── ops-reconcile-orders.ts
│   └── ops-cleanup-media.ts
└── tests/
    ├── unit/
    ├── integration/                  # 连库：同步、搜索、授予、兑换
    └── e2e/

drizzle/
media/                                # 同步产物：按 hash 存放，不进 git
compose.yml                           # postgres:16
.env.example
```

**结构决定**：仓库根就是一个 Next.js 项目。CLI 脚本直接 import `src/lib/*`。不拆 `frontend/`／`backend/`。

## 原文、图片、架构图放哪里

权威源是 git，不是 Postgres，也不是对象存储。站长改文章、换图、改示意图，都是改仓库里的文件，然后跑 `pnpm content:sync`。

| 东西 | 放哪 | 不放哪 |
| --- | --- | --- |
| 文章 Markdown | `content/articles/<slug>/index.md` | 不进 Postgres，不进对象存储 |
| 配图、截图、PNG／WebP | 同目录 `assets/` | 不进 `bytea`，不进 `public/` |
| 已导出的架构图 SVG | 同目录 `assets/` | 同上 |
| Mermaid 源 | 写在 `index.md` 的围栏代码块里 | 不另存一份、不变成库字段 |
| 同步生成的试读／正文 HTML | Postgres `articles.preview_html` / `body_html` | 不回写 git |
| 媒体引用（路径、hash、是否公开） | Postgres `article_media` | 只存元数据 |

一篇文章一个目录，避免扁平 `content/articles/*.md` 把几十张图堆在同一层。旧的单文件写法不同步进库。

Markdown 里只允许相对引用：

```md
![安装界面](./assets/setup.png)

```mermaid
flowchart LR
  A[Markdown] --> B[sync]
  B --> C[Postgres]
```
```

同步时做三件事：

1. 把 `index.md` 渲成两份 HTML（试读／正文），写入 Postgres。
2. 把 `assets/` 里实际被引用的文件按 content hash 拷到应用私有目录 `media/<hash>.<ext>`（不在 `public/`，不进 Next 静态包）。
3. 把 Markdown 中的相对路径改写成媒体 URL：公开图 `/media/public/<hash>`，付费正文专用图 `/media/paid/<hash>`。

**Mermaid／架构图：** 源在 Markdown 里。同步时用服务端渲染器（如 `@mermaid-js/mermaid-cli` 或等价库）变成 SVG，再按普通资源走第 2、3 步。不要运行时在读者浏览器里跑 Mermaid——付费图会泄漏，而且首屏会抖。站长如果已经导出了 SVG／PNG，直接放 `assets/`，不必再包一层 Mermaid。

**付费图不进 `public/`。** 试读 HTML 引用过的 hash 标成公开，匿名可取。只出现在 `body_html` 的 hash 标成付费，走和正文同一套权益校验；无授予返回 404，不 401，避免用状态码探图。缓存头：公开图可短缓存；付费图 `Cache-Control: private, no-store`。

**对象存储（阿里云 OSS／腾讯云 COS）首版可不上。** 私有 `media/` 目录跟应用进程走。图量变大或要上 CDN 时，只换「按 hash 取文件」这一层，`article_media` 表不用改。

## 数据模型

规格第 9 节的逻辑记录落到表。时间戳一律 `timestamptz` UTC。

### `articles`

- `id` uuid pk
- `slug` text unique not null
- `source_path` text not null
- `content_hash` text not null
- `title` text not null
- `summary` text not null
- `language` text not null check（`zh`、`en`）
- `status` text not null check（`draft`、`published`、`archived`）
- `access` text not null check（`free`、`paid`）
- `preview_mode` text not null check（`summary`、`standard`、`custom`）默认 `standard`
- `topics` text[] not null
- `human_tags` text[] not null 默认 `{}`  —— 同步时以 human/accepted 落入 `article_tag_decisions`，决策表是标签唯一事实来源
- `recommended` boolean not null 默认 false
- `related_slugs` text[] not null 默认 `{}`
- `published_at` timestamptz null
- `updated_at` timestamptz not null
- `preview_html` text not null 默认 `''`
- `body_html` text not null
- `outline_headings` text[] not null 默认 `{}`  —— 试读截断点之后的标题，公开「你将获得什么」
- `reading_minutes` int not null
- `audience` text null  —— 可选 front matter，公开
- 入站文件校验失败时，同步不得覆盖最近一次有效的已发布行。
- 扫描不完整时，不得把缺失文件当成删除。删除条件：完整扫描后文件不在，或 `status` 不是 `published`。

### `article_media`

- `hash` text pk  —— 文件内容 sha256
- `article_id` uuid not null
- `source_path` text not null  —— git 内相对路径，如 `kimi-code-getting-started/assets/setup.png`
- `kind` text not null check（`image`、`svg`、`mermaid`）
- `visibility` text not null check（`public`、`paid`）  —— 被试读 HTML 引用则为 `public`，否则 `paid`
- `content_type` text not null
- `byte_size` int not null
- 二进制不进这一行。磁盘上的对象路径由 hash 推导：`media/<hash>`。
- 同一文件被两篇文章引用可以有两行，hash 相同、文件只存一份。

### `article_redirects`

- `from_slug` text pk
- `to_slug` text not null
- 仅当已发布 slug 变更，且站长写入 `content/redirects.yml` 时才写。

### `taxonomy_terms`

- `id` text pk  —— 规范化标识符
- `kind` text not null check（`topic`、`tag`）
- `label_zh` text not null
- `label_en` text not null
- `public` boolean not null 默认 false
- 自动提出的新词条保持 `public=false`，直到管理员接受。

### `article_tag_decisions`

- `article_id` uuid
- `term_id` text
- `source` text not null check（`human`、`auto`）
- `state` text not null check（`accepted`、`rejected`）
- unique（`article_id`、`term_id`）
- 搜索用标签 = 人工接受 ∪（自动接受 − 拒绝）。拒绝状态在重跑后仍保留，直到管理员明确恢复。

### `tagging_jobs`

- `id` uuid pk
- `article_id` uuid
- `content_hash` text not null
- `tagger_version` text not null
- `status` text not null check（`pending`、`succeeded`、`failed`、`skipped`）
- `attempt_count` int not null 默认 0
- `tags` text[] not null 默认 `{}`
- `error` text null
- `finished_at` timestamptz null
- 已有相同 `(content_hash, tagger_version)` 成功记录则跳过。运行中源哈希变了，丢弃本次结果。失败时保留上一组成功决策。

### `job_locks`

- `name` text pk  —— `content-sync`、`tags-run`
- `holder` text not null
- `acquired_at` timestamptz not null
- 同时使用 `pg_try_advisory_lock`，防止两个进程跑同一任务。

### `users`

- `id` uuid pk  —— Auth.js 用户 id
- `role` text not null 默认 `reader` check（`reader`、`admin`）
- `access_until` timestamptz null  —— 缓存的重算结果；取正文时仍要查授予记录
- 付费读者没有草稿权限。管理是单独角色。

### `orders`

- `id` uuid pk
- `user_id` uuid not null
- `provider` text not null  —— `wechatpay`、`alipay`、`stripe`
- `merchant_order_no` text not null —— 本站商户订单号；unique(provider, merchant_order_no)
- `transaction_id` text null —— 通道交易 id；非空时 unique(provider, transaction_id)，与 merchant_order_no 不混用
- `checkout_ref` text null  —— 通道收银会话标识（如 Stripe Checkout Session id）
- `scene` text not null check（`desktop`、`mobile`、`wechat`）
- `amount_minor` bigint not null —— 货币最小单位，必须为正
- `currency` text not null
- `duration_days` int not null
- `idempotency_key` text not null —— unique(user_id, idempotency_key)；同 key 不同通道／参数拒绝
- `status` text not null check（`pending`、`paid`、`failed`、`canceled`）
- `expires_at` timestamptz null
- `paid_at` timestamptz null
- 跳转前在服务端创建，创建时固化通道、金额、币种、时长。浏览器成功页不是付款证据。

### `payment_events`

- `provider` text not null
- `event_id` text not null  —— 通道通知 id 或查单业务事件键；unique(provider, event_id)
- `payload_hash` text not null
- `event_type` text not null
- `status` text not null check（`received`、`processed`、`ignored`、`failed`）
- `received_at` timestamptz not null
- `processed_at` timestamptz null
- `error` text null
- webhook 验签后在同一事务插入事件、锁订单、更新订单、创建授予并写审计。规范细节见 [数据模型与一致性](plan/data-model.md)。

### `entitlement_grants`

- `id` uuid pk
- `user_id` uuid not null
- `source_type` text not null check（`order`、`redeem`）
- `source_id` text not null  —— 订单 id 或兑换码 id
- unique（`source_type`、`source_id`）  —— 每个来源只生效一次
- `duration_days` int not null
- `granted_at` timestamptz not null
- `revoked_at` timestamptz null
- 生效规则：新授予的基线是 `max(现在, 当前 access_until)`，再加时长。重算在一次事务内完成：按 `(granted_at, id)` 升序折叠所有未撤销授予，`until := max(until, 该授予 granted_at) + duration_days`，起点为空；`access_until` 取最终值。
- 管理员撤销对应授予（写审计），然后重算。其他授予不动。首版无退款流程，渠道争议也走人工撤销。

### `redeem_codes`

- `id` uuid pk
- `code_hash` text unique not null  —— 完整兑换码的 sha256
- `display_hint` text not null  —— 仅前 4 + 后 4
- `duration_days` int not null
- `expires_at` timestamptz null  —— 最晚兑换时间
- `status` text not null check（`unused`、`used`、`disabled`）
- `redeemed_by` uuid null
- `redeemed_at` timestamptz null
- 明文只在签发时展示一次，不落库、不进日志、不进 URL。
- 兑换：必须登录；`UPDATE ... WHERE id=$id AND status='unused' AND (expires_at is null or expires_at > now()) RETURNING *` 与插入授予在同一事务。同一账户重试自己已用过的码，返回已有授予，不再次延长。

### `audit_events`

- `id` uuid pk
- `actor_id` uuid null
- `action` text not null
- `target_type` text not null
- `target_id` text not null
- `created_at` timestamptz not null
- `metadata` jsonb not null 默认 `{}`  —— 不存密钥、不存兑换码明文、不存完整支付报文

### `rate_limit_buckets`

- `key` text pk  —— `ip:{ip}:{route}` 或 `user:{id}:{route}`
- `window_start` timestamptz not null
- `count` int not null
- 限额走环境变量。超限返回 429 + `Retry-After`。

从规格抽出的校验：
- front matter 必填：`slug`、`title`、`summary`、`topics`、`language`、`status`、`access`。
- 发布必须有 `published_at`。
- `slug` 稳定；URL 为 `/articles/{slug}`。
- 自动标签最多 8 个。查询长度、筛选标签数、每页条数由服务端限制。

状态机：
- 文章：`draft → published → archived`。完整扫描后磁盘上已发布文件消失，则退出公开目录。
- 订单：`pending → paid | failed | canceled`，均为终态；首版无退款流程。
- 兑换码：`unused → used | disabled`。`used` 是终态；管理员撤销的是对应*授予*，那是另一项操作。

## 接口约定

### 公开页面

- `GET /` 文章库：精选最多 6 篇（站长 `recommended` 优先，再用最新补足）、最新列表、搜索表单。卡片：标题、摘要、标签、发布日期、免费／付费标记。
- `GET /search?q=&topic=&tag=&tag=&page=` AND 语义。查询词、筛选、页码都在 URL。无结果状态和清除筛选。复制 URL 可还原。结果只有已发布元数据。
- `GET /articles/{slug}` 草稿／已归档对所有角色（含付费读者）都 404。命中 `article_redirects` 的旧 slug 返回 301 至新 slug。免费：完整 `body_html`。付费且无授予：标明「试读」的试读区、「你将获得什么」、一个 CTA。付费且有授予：`body_html`。相关阅读：站长 `related` 优先，再按共同主题／标签补足到 4 篇，slug 打平，排除草稿／自身／重复。付费相关卡片必须打标，打开时仍做权限校验。
- `GET /pricing` 一种通行证：时长、价格、币种、明确声明不支持退款、支付入口、兑换码入口。
- `GET /checkout/{orderId}` 收银台等待／结果页：只展示本地订单状态并限频轮询，不构成授予证据。
- `GET /account` `access_until`、购买状态、退出登录。
- `GET /login`
- `GET /media/public/{hash}` 试读引用过的图，可匿名；短缓存。未知 hash 404。
- `GET /media/paid/{hash}` 必须有该文有效授予；无授予与未知 hash 一律 404。`Cache-Control: private, no-store`。

### JSON／POST

- `POST /api/checkout` 必须登录 → 校验 provider 与设备场景，创建幂等订单及支付请求。返回 redirect／qr／jsapi 判别联合，不再只返回 URL。
- `POST /api/webhooks/{wechatpay|alipay|stripe}`：各通道独立路由及官方协议验签／回执，核对商户、应用、订单、金额、币种和状态；`payment_events(provider,event_id)` 幂等。只认验证过的服务端事件或主动查单结果，不认浏览器回跳。
- `GET /api/orders/{id}`：仅订单所有者可读，返回本地支付状态；前端限频轮询，不直接暴露支付凭据。
- `POST /api/redeem` 必须登录，限流。Body `{ "code": "..." }`。响应：已授予、你已兑换过、无效（无效／过期／停用／已被他人使用用同一句文案，不泄露兑换者）。
- 搜索与文章 GET 走 HTML／RSC，不提供未认证的批量 JSON 导出。

### CLI

- `pnpm content:sync` 完整扫描、校验、渲染、upsert，把未发布内容移出公开目录。整次原子发布：任一文件无效则整次不发布、保留上一有效版本；已发布文章 slug 变更而 `content/redirects.yml` 缺对应条目时同步报错。
- `pnpm tags:run` 加锁、跳过未变更、临时故障最多重试 3 次，成功后刷新搜索标签。
- `pnpm tags:review` 接受／拒绝／恢复自动标签；接受词表外新词（置 `public`）。
- `pnpm codes:issue --days 30 --count 10 --expires 2026-12-31` 明文只打印一次。
- `pnpm codes:disable --hint abcd…wxyz`
- `pnpm entitlements:revoke --grant <id>`
- `pnpm ops:status`／`pnpm ops:reconcile-orders`／`pnpm ops:cleanup-media` 只读诊断、pending 订单对账、媒体孤儿 GC。

## 读者侧试读规则

付费文章且没有有效授予时：
1. 始终展示 `summary`。
2. 在标明「试读」的区域展示 `preview_html`。
3. 展示不泄正文的「你将获得什么」：主题、可选适用读者、预计阅读时长、截断点之后的章节标题。
4. 试读区末尾只放一次主要付费行动入口。不用遮罩、模糊、禁止复制。
5. 同一文章不轮换试读片段。
6. 文章 GET 按 IP 和账户限流。

## 搜索与推荐（v1 规则）

搜索：
- 去掉首尾空白，标题大小写不敏感子串匹配。不依赖空格分词（中文标题按原文子串即可）。
- 主题／标签用规范化词表 id 精确匹配。多标签 AND。界面文案写明 AND。
- 排序：`published_at` 降序，再 `slug` 升序。
- 只出 `status=published`。
- `q` 最长 80，标签最多 5 个，每页 20 条。

推荐：
- 首页：`recommended=true` 优先，再用最新补足，最多 6 篇。
- 文章页：有效 `related` slug 优先，再按重合数然后 `published_at` 补足，最多 4 篇。
- 排除自身、重复、未发布。可以出现付费文章，但必须打标。

## 限流（初始环境变量）

| 路由 | IP | 账户 |
| --- | --- | --- |
| `GET /articles/*` | 5 分钟 60 次 | 5 分钟 120 次 |
| `GET /search` | 5 分钟 30 次 | 5 分钟 60 次 |
| 认证 | 15 分钟 20 次 | 无 |
| 兑换 | 15 分钟 10 次 | 15 分钟 5 次 |
| 创建收银台 | 15 分钟 10 次 | 15 分钟 5 次 |

看脱敏后的 429 日志再调。只有实际滥用才加边缘验证；表单必须保持键盘可操作。webhook 路由不适用普通 IP 限流——必须允许支付服务商重试；门槛是验签，签名失败率异常时告警。

## 测试计划

主路径：
1. 有效已发布 Markdown 同步后出现在文章库；免费正文能渲染。
2. 中英文标题子串搜索都可用；URL 往返能还原筛选。
3. 精选与相关阅读顺序符合规格。
4. 已登录用户兑换新码后写入 `access_until`；付费正文能渲染。
5. 金额正确且验签通过的 各通道订单成功事件只授予一次。

错误／边界：
- front matter 无效：CLI 给出可操作错误；最近一次有效已发布行保留。
- 不完整扫描不删除目录。
- 草稿／已归档对有权益用户也 404。
- 匿名请求的 HTML、RSC 载荷、`/_next/static`、`/media/public/*` 中都没有付费正文或付费图。`/media/paid/*` 无授予时 404。
- 试读在同步时按 30% 规则于小节／段落边界截断生成，不在请求时截。
- 经 CDN 请求付费正文与付费媒体始终无缓存（`no-store` 透传、无共享缓存命中）。
- 伪造／未签名／金额不符／重放的支付事件不能额外授予。
- 只打开 `/pricing/success`、没有 webhook，不能授予。
- 同一兑换码并发兑换：一个成功、一个干净失败，时长不翻倍。
- 同一用户重试同一码：不再次延长。
- 管理员撤销对应授予后，叠加的其他授予仍有效。
- 打标签失败保留旧标签；哈希变化丢弃进行中的结果。
- 已拒绝标签重跑后仍拒绝。
- 超限返回 429。
- 日志脱敏：没有兑换码明文，没有支付服务密钥。

## 交付顺序

对齐规格第 11 节。付费**正文**在阶段 3 完成前保持不可访问。独立试读 HTML 可以更早存在。

### 阶段 1 — 内容与发现

Compose 起 Postgres，Drizzle 建文章／词表／重定向表，Markdown 校验与同步，安全渲染，文章库／搜索／文章页，标题／标签 AND 搜索，固定规则推荐。同步为整次原子发布：任一文件无效则整次不发布，绝不出现混合代际；代价是一个错文件挡住其他更新，由可操作错误信息缓解。付费文件同步元数据 + 试读；`body_html` 可以入库，但文章页在权益模块存在前一律拒绝返回（缺授予视为无授予，失败时关死）。

### 阶段 2 — 后台打标签

`taxonomy.yml`、`tags:run`、跳过／重试／加锁、决策合并、CLI 接受／拒绝／恢复。文章不必等打标签完成就能发布。人工标签应立即可搜。

### 阶段 3 — 访问控制

Auth.js、微信扫码登录与账号绑定（保留 GitHub／邮件）、账户、试读区 + CTA、权益授予、一次性兑换码、限流、缓存头（正文 `Cache-Control: private, no-store`）。管理员 CLI 负责发码和撤销。

### 阶段 4 — 支付与上线

微信支付、支付宝和 Stripe Checkout 适配器；按浏览器场景展示可用方式；二维码／跳转／JSAPI、订单状态查询、幂等通知与 pending 对账。各通道独立完成商户／产品准入和测试门禁后启用。微信登录与两种国内支付是 v1 验收项；测试环境不可用时不得以 mock 代替真实联调。

## 实现前需要的密钥与账号

| 名称 | 用途 |
| --- | --- |
| `DATABASE_URL` | Postgres |
| `AUTH_SECRET` | Auth.js 会话 |
| `AUTH_GITHUB_ID`／`AUTH_GITHUB_SECRET` | GitHub 登录 |
| `RESEND_API_KEY`／`EMAIL_FROM` | magic link |
| `STRIPE_SECRET_KEY`／`STRIPE_WEBHOOK_SECRET`／`STRIPE_PRICE_ID`／`STRIPE_MODE`（`test`｜`live`） | Stripe Checkout 与 webhook（阶段 4；此前用空适配器） |
| `PASS_CNY_AMOUNT_MINOR`／`STRIPE_PASS_AMOUNT_MINOR`／`STRIPE_PASS_CURRENCY`／`PASS_DURATION_DAYS` | 单一 SKU，分通道服务端价格 |
| `WECHAT_LOGIN_APP_ID`／`WECHAT_LOGIN_APP_SECRET` | 网站扫码登录 |
| `WECHAT_PAY_APP_ID`／`WECHAT_PAY_MCH_ID`／`WECHAT_PAY_API_V3_KEY`／`WECHAT_PAY_PRIVATE_KEY_PATH`／`WECHAT_PAY_SERIAL_NO`／`WECHAT_PAY_PUBLIC_KEY_ID`／`WECHAT_PAY_PUBLIC_KEY_PATH` | 微信支付签名、验签及通知解密；平台证书模式由适配器另行配置 |
| `ALIPAY_APP_ID`／`ALIPAY_PRIVATE_KEY_PATH`／`ALIPAY_PUBLIC_KEY_PATH`／`ALIPAY_GATEWAY` | 支付宝 RSA2 模式；如选择证书模式，单独配置证书链 |
| `APP_BASE_URL`／`PAYMENT_PROVIDERS_ENABLED`／`WECHAT_LOGIN_ENABLED` | 固定回调地址来源、支付与登录独立开关 |
| `PUBLIC_INDEXING` | 公开页收录开关，默认关（决策 7） |
| `RATE_LIMIT_*` | 各路由 IP／账户限额（见限流表） |
| `TAGGER_BASE_URL`／`TAGGER_API_KEY`／`TAGGER_MODEL` | 离线打标签（阶段 2） |
| `ADMIN_USER_IDS` | 引导管理员 |

首版支付供应商限定微信支付、支付宝、Stripe；不增加其他通道或分账。

## 明确不做

浏览器内 Markdown 编辑、读者投稿、评论、社交信息流、基于行为的个性化、全文或语义搜索、向量库、任务队列、多商户分账、独立移动应用、单篇购买、自动续费订阅、退款流程（首版声明不支持退款，渠道争议走管理员人工撤销授予）、全站共享密码、面向读者的 API Key、存储银行卡、禁用复制／右键。

## 关键决策

1. **一个 Next.js 应用 + Postgres + CLI** —— 对齐规格第 9 节，付费 HTML 留在服务端。
2. **试读与正文是同步时的两份产物** —— 这是满足 4.3、又不信任客户端的唯一做法。
3. **授予只追加、来源唯一** —— 购买、兑换、叠加共用一次重算。
4. **微信支付、支付宝、Stripe 共用领域模型** —— 国内钱包纳入 v1，国际卡保留；每个通道独立准入、验签与对账，统一授予权益。
5. **打标签不能挡住发布** —— 人工标签和标题搜索立刻可用；模型只是批处理补充。

## 前提崩塌

本计划以 Next.js + Postgres 为基础，微信登录、微信支付与支付宝为 v1 必需能力，Stripe 保留国际卡。各平台账号审核与产品开通在开发初期开始，不能等到阶段 4 才发现不具备接入条件。生产环境面向中国大陆：ICP 备案与企业主体是支付开通与公开上线的硬前提，阶段 1 即启动办理。

未开通的通道保持关闭，其他通道可独立运行；兑换码可作为临时降级，但不能替代未完成的微信／支付宝需求。若商户申请受阻，记录外部依赖，不自行替换成个人收款码或删除需求。

Auth.js 没有官方微信 provider：网站扫码登录需要自定义 provider（`appid`／`secret` 参数名、`qrconnect` 授权端点等非标准 OAuth2 细节），阶段 3 早期安排 spike 验证，不假设开箱即用。

## 明确推迟的未知项

| 事项 | 为何推迟 | 负责人 |
| --- | --- | --- |
| 通行证准确价格 | 运营事项，不是表结构。读环境变量。 | 阶段 4 前由站长定 |
| ICP 备案（含支付／回调域名、CDN 加速域名） | 硬前提，需企业主体；微信支付/支付宝商户、回调域名、国内 CDN 均依赖 | 阶段 1 启动办理；备案完成前不公网上线 |
| 微信网站应用与微信支付、支付宝产品开通 | 分别登记主体、域名、产品权限和审核状态，不能假设登录审核等于支付开通。 | 阶段 1 开始，阶段 3／4 前完成 |
| Stripe 商户准入与打款资格 | 独立于国内支付，按当前平台规则确认。 | 阶段 4 前确认 |
| 是否打开 `PUBLIC_INDEXING` | 规格第 12 节上线决策。默认关。 | 上线时由站长定 |
| 国内云厂商择一（阿里云/腾讯云） | 生产环境已定中国大陆云；厂商择一影响备案主体与资源开通 | 阶段 1 由站长定 |
| Resend 邮件境内可达性 | magic link 在国内邮箱（QQ/163 等）的到达率待验证，必要时换国内邮件服务 | 阶段 3 前验证 |
| 试读篇数日限（按 IP／账户） | 防批量收割 30% 试读的后续手段；现默认不启用，滥用迹象出现时按观测数据再加 | 出现滥用时由站长定 |

这些都不挡住阶段 1–3。

## 复杂度跟踪

> 用户明确扩展首版支付范围，原单通道限制不再适用。

| 违规 | 为何需要 | 更简单方案被否的原因 |
| --- | --- | --- |
| 原单支付服务商限制 | 用户要求微信支付与支付宝，同时保留国际卡 | 单通道不能覆盖已确认需求；统一适配器与权益账本限制复杂度 |
