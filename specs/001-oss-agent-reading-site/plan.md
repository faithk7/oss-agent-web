# 实现计划：OSS Agent 阅读网站

**分支**：`001-oss-agent-reading-site` | **日期**：2026-09-16 | **规格**：[../../spec.md](../../spec.md)

**输入**：仓库根目录 `spec.md`（中文产品需求，待评审草案）。本仓库尚未初始化为 Spec Kit 项目；本文件是首个功能（也就是整个 v1 站点）的 Spec Kit `plan.md`。

**状态**：等待批准。批准前不要写代码。

## 摘要

建设一个围绕开源软件与编程 Agent 的精选阅读网站，覆盖 Kimi Code、Codex 等主题。站长在 git 里写 Markdown。批处理 CLI 同步并渲染这些文件，离线打标签 Agent 再从受控词表提出标签建议。读者浏览文章库、按标题搜索、按主题／标签筛选，阅读免费正文与付费试读。付费全文只在服务端完成权益校验后返回。MVP 只提供一种有期限的全站通行证，先证明兑换码、微信 Native 桌面支付和支付宝电脑网站支付闭环；移动支付场景与 Stripe 保留相同适配器并独立启用，权益模型不变。

技术方案：一个 Next.js App Router 应用、一个 PostgreSQL 数据库、一套负责同步／打标签／管理的 CLI。付费 HTML 不得打进客户端静态包。试读 HTML 与正文 HTML 在同步时分别生成，请求时按权限选择返回哪一份。

## MVP 成功标准（概念验证门槛）

MVP 的成功不是“把所有扩展接口都做完”，而是用一套可复现的最小部署证明核心读者闭环。以下标准全部满足后，才把 MVP 标记为成功；外部商户审核、备案或支付产品尚未完成时，只能标记为 POC 阻塞，不能宣称已上线。

| 标准 | 可观察结果 | 最小证据 |
| --- | --- | --- |
| 内容可发布、可发现 | 站长用 `content:sync` 发布一组包含免费／付费、中文／英文文章的 fixture；文章库、标题搜索、主题／标签 AND 筛选和固定推荐均可用。任一源文件出错时，上一版有效发布保持不变。 | 一次干净数据库同步 + Playwright smoke test + 同步失败回滚记录 |
| 付费边界成立 | 匿名或无权益用户只能收到摘要／试读；有有效权益的登录用户才能收到正文。正文不出现在 HTML、RSC、静态包、公共缓存或未授权媒体响应中；到期／撤销后立即回到试读。 | 匿名、无权益、有权益、到期和撤销五条访问测试；正文泄漏扫描 |
| 身份与兑换闭环成立 | 桌面微信扫码登录能处理成功、取消、过期和 state／code 重放；登录不依赖邮箱。有效兑换码只授予一次，重复或并发兑换不会重复延长。 | 微信登录联调记录 + 真实 Postgres 并发测试 + 账户页到期时间 |
| 国内桌面支付闭环成立 | 同一通行证 SKU 可选择微信 Native 或支付宝电脑网站支付；只接受服务端验签事件授予权益，通知重试／乱序／查单最终只产生一次授予。两个通道的金额、币种和时长一致。 | 两个通道各一条可用测试／受控小额实付链路、webhook 与查单记录；不能用成功回跳或兑换码替代 |
| 轻量部署可运营 | 单个 Web 进程 + PostgreSQL + 仓库 CLI 即可启动、同步、运行 smoke test 和恢复；不依赖搜索集群、队列或 Redis。核心性能守住既定门槛：2k 篇文章搜索 warm p95 < 300ms，200 个 Markdown 无模型同步 < 30s。 | 可复现启动说明、健康／readiness 检查、备份恢复演练和基准报告 |

未启用的移动支付、Stripe、GitHub、magic link 只需保留 adapter／contract fixture，不计入 MVP 成功率，也不应出现在收银台或增加首版运行依赖。

## 技术上下文

**语言／版本**：TypeScript 5.x，Node.js 22，strict 模式。

**主要依赖**：Next.js 15（App Router、React Server Components）、Auth.js v5、Drizzle ORM、Zod、带清理的 `unified`／`remark`／`rehype`、`gray-matter`、统一支付接口后的微信支付 API v3 与支付宝 SDK。Stripe 与邮件 SDK 只在对应 feature flag 启用时安装／加载，不进入核心 MVP 路径。

**存储**：PostgreSQL 16。Markdown 权威源在 `content/articles/`。生成的试读 HTML、正文 HTML、搜索字段、标签、用户、订单、权益授予、兑换码都放在 Postgres。首版不加独立搜索服务，不加 Redis。

**测试**：Vitest 覆盖单元／集成（权益、同步、打标签、搜索 AND、兑换并发）。Playwright 覆盖读者路径。所有支付适配器共用合约测试；MVP 对支付宝使用可用测试环境，对微信 Native 按产品可用测试能力联调并在启用前完成受控小额实付验证。未启用的扩展适配器只要求 fixture／contract tests，不宣称真实可用。兑换并发测试使用真实 Postgres（Testcontainers 或本地 compose）。

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

**规模／范围**：一名站长、数百篇文章、数千读者。一种通行证 SKU。六个公开页面，管理入口首版以 CLI 为主。MVP 是模块化单体与单进程部署，不为尚未出现的规模增加服务、队列或缓存；首版不做浏览器内编辑、评论、读者投稿、语义搜索、多商户分账、独立移动应用。

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
| G7 先测钱相关路径 | 权限边界、支付幂等、兑换并发、搜索与打标签必须有自动化检查。上线前走可用测试环境与受控真实联调。 | 见本文测试计划。 |

用户最新需求明确要求微信登录、微信支付和支付宝，同时强调 MVP 与概念验证。核心验收因此收敛为桌面微信扫码登录、微信 Native 和支付宝电脑网站支付；移动场景、Stripe、GitHub 与 magic link 通过端口／适配器保持兼容，但不与核心 MVP 同时展开。仍保持单一 SKU、单应用和同一套权益账本，不增加分账或订阅计费。

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

### 决策 3 — 认证：微信为 MVP，其他 provider 可插拔

- **决定**：Auth.js 管理本站会话，MVP 只要求微信开放平台网站应用扫码登录。认证层保留标准 provider adapter；GitHub OAuth 和 magic link 在对应配置、可达性验证及账号绑定流程完成后独立开启，不进入核心证明路径。微信不是企业微信，不要求用户必须有邮箱。
- **浏览器范围**：MVP 支持桌面网站微信扫码。手机浏览器先给出跨设备扫码／稍后再试说明；不假设手机能扫描自身屏幕。额外登录方式启用后才展示。
- **身份**：以服务端换取的 `(appid, openid)` 唯一识别微信账号；`unionid` 可空，不凭昵称／头像／邮箱自动合并账户。登录后重新验证目标身份才能绑定，权益始终绑定内部 `user.id`。
- **安全与接入**：一次性 state、限定回调域名、服务端 code 交换、取消／过期／重放处理。网站应用 AppID/Secret 与微信支付商户密钥分开配置。账号绑定 UI/API 在启用第二个 provider 前完成；MVP 不做“按相同邮箱自动合并”。详见 [合约](plan/contracts.md)。

### 决策 4 — 支付：先证明两个国内桌面通道

- **MVP 必须支持**：微信 Native 桌面二维码和支付宝电脑网站支付。一个有期限通行证 SKU，两个通道授予完全相同的权限与时长。
- **移动扩展**：微信 H5、JSAPI 与支付宝手机网站支付保留 adapter scene，但只有取得对应产品权限并逐场景通过真实设备验证后才打开。JSAPI 的 payer openid 必须来自支付所用 AppID 的授权，不能复用网站登录 AppID 的 openid。
- **国际卡扩展**：保留 Stripe provider 类型和 contract fixture，不在核心 MVP 实现／上线门禁内；以后启用时单独确认商户准入、打款资格和生产配置。
- **价格与订单**：微信支付／支付宝以 CNY 分计价，Stripe 使用单独服务端定价；不做浏览器换汇。创建订单时固化通道、金额、币种、时长。每个订单只绑定一个通道，重试幂等，切换通道建立明确的新订单。
- **回调与对账**：各适配器独立验签、解析和回执，统一到订单、支付事件与权益事务；查询付款状态也通过同一适配器。回跳、扫码和客户端 SDK 返回均不作为授予证据。首版无退款流程，渠道侧支付争议由管理员人工撤销对应授予兜底。
- **上线门禁**：网站微信登录应用审核，以及微信 Native／支付宝电脑网站支付商户、产品权限、回调域名和密钥配置分别检查。某通道未开通时明确显示不可用，其他通道和兑换码继续服务；只有已声明启用的场景进入验收矩阵。

### 决策 5 — 同步时生成试读

- **决定**：同步分别写入清理后的 `preview_html` 和独立 `article_bodies.body_html`。试读来源按顺序：`preview=custom` 时用 `preview_markdown`（缺失或为空则同步报错）；源文件里若有 `<!-- preview:start -->`／`<!-- preview:end -->` 则用标记区间（标记必须成对，否则同步报错）；否则 `standard` 取正文开头约 30%（按正文纯文本字符计，CJK 计 1、非 CJK 计 0.5），在小节或段落边界收尾。所有非空试读还受 `PREVIEW_MAX_WEIGHTED_CHARS` 绝对上限约束（默认 6000）；custom 等同完整正文或超过上限时同步失败。`summary` 档的试读 HTML 为空。请求路径只选择允许的一份产物。搜索响应永不包含试读或正文 HTML。
- **理由**：规格 4.3 禁止把全文发给浏览器再截断。源文件里的标记让试读仍由站长控制。
- **备选**：前端截断（规格否决）。随机摘录（规格否决）。

### 决策 6 — 打标签模型：可替换、最小出站

- **决定**：`pnpm tags:run` 调用 OpenAI 兼容接口（`TAGGER_BASE_URL`、`TAGGER_API_KEY`、`TAGGER_MODEL`），不在计划里固化易过期的模型 id。默认 `TAGGER_EGRESS_MODE=preview`，只发送标题、摘要和清理后的试读；只有显式设为 `full` 且目标 host 在 `TAGGER_ALLOWED_HOSTS` 中才发送正文。提示词把文章当不可信数据并附词表。输出经 Zod 校验：`tags: string[]`（词表内，最多 8 个）与 `new_terms: { label: string; language: 'zh' | 'en' }[]`。新词先进入逐条提议表，审核后才写入／关联公开词表。默认每天 cron 一次，手动跑同一条命令。
- **理由**：规格允许联网模型调用。OpenAI 兼容地址可以指向 Moonshot／Kimi、OpenAI 或本地网关，不用改代码。
- **备选**：完全本地 llama.cpp（以后走同一组环境变量即可）。请求链路上打标签（否决：不是离线）。

### 决策 7 — 界面语言与收录

- **决定**：界面文案用中文。展示时按 Asia／Shanghai 渲染日期，存储仍用 UTC。`robots.txt` 允许公开元数据路由；付费正文页始终 `noindex`；公开页在上线 SEO 决策把 `PUBLIC_INDEXING=true` 打开之前也 `noindex`。
- **理由**：规格是中文，且收录策略要到上线前才定。默认不收录是安全门禁。

### 决策 8 — 托管形态

- **决定**：生产环境在中国大陆云：应用先跑一个 Node 进程，数据库用托管 PostgreSQL 16（自带备份/WAL）。本地与封闭 POC 用 `LocalMediaStore`；公开生产用同接口的 OSS/COS 私有桶，避免“无状态应用 + 本地磁盘”矛盾。静态产物与已公开媒体走厂商 CDN（备案后启用）。定时任务用主机 cron；本地开发只用 `docker compose` 起 Postgres。
- **理由**：规格写明任务量未到之前不加队列。

### 决策 9 — MVP 切片与扩展边界

- **核心闭环**：Markdown 同步 → 免费／试读阅读 → 微信扫码登录 → 兑换码授予 → 微信 Native／支付宝电脑网站支付 → 服务端事件授予 → 到期／撤销。
- **扩展槽位**：GitHub、magic link、微信 H5／JSAPI、支付宝手机网站支付、Stripe 只保留接口、配置和 contract fixture；对应产品权限与端到端验证完成后单独启用。
- **不提前抽象**：不建通用工作流引擎、消息总线、插件系统或多 SKU 定价表。只对认证、支付和媒体存储保留小接口，因为这些边界已经确定会替换。

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
- 可选 provider 挂了：未启用的 provider 不影响微信登录；微信登录故障时 MVP 明确显示不可用，不建立弱会话。启用第二个 provider 后再补充独立降级路径。

回滚：应用无状态。迁移只向前、只加列。授予记录只追加（撤销用标记），错误发布不必回滚支付。

## CDN 与缓存分层

| 内容 | CDN 策略 |
| --- | --- |
| `/_next/static/*` 构建产物（构建不含付费正文） | 长缓存 immutable，按构建 hash 寻址 |
| `/media/public/*` 公开媒体 | 长缓存 immutable，按内容 hash 寻址，无需失效 |
| 公开 HTML（文章库、搜索、匿名试读页） | v1 源站动态直出，CDN 不缓存 HTML；后续可评估匿名缓存键 |
| 付费响应（有权益正文视图、`/media/paid/{articleId}/*`、账户、收银台、webhook） | 绝不缓存：源站 `Cache-Control: private, no-store`，CDN 规则显式绕过，仅透传 |

- 托管形态：生产环境用国内云厂商 CDN（加速域名需 ICP 备案，备案完成后启用），回源到应用或对象存储；公开媒体 CDN 化只换 `MediaStore` 实现，`article_media` 引用契约不变。付费媒体即使迁对象存储，也走鉴权代理，不进公共 CDN 缓存。Cloudflare 中国网络需 ICP 备案与企业合作，不采用。
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
├── .content-root                       # prune 安全哨兵，禁止误把错误目录当完整扫描
├── articles/
│   └── kimi-code-getting-started/
│       ├── index.md                  # YAML front matter + Markdown（权威源）
│       └── assets/
│           ├── setup.png             # 配图、截图
│           └── architecture.svg      # 手绘／导出的架构图
├── taxonomy.yml                      # 受控主题与标签词表
└── redirects.yml                     # from_slug -> current_slug；同步解析为 article_id

src/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                      # 文章库：精选 + 最新 + 搜索
│   ├── search/page.tsx
│   ├── articles/[slug]/page.tsx      # 整页动态 no-store；先授权再读取正文表
│   ├── pricing/page.tsx
│   ├── checkout/[orderId]/page.tsx   # 收银台等待／结果，只展示本地订单状态
│   ├── account/page.tsx
│   ├── login/page.tsx
│   ├── robots.ts
│   ├── media/public/[hash]/route.ts  # public_since 非空的对象，可匿名
│   ├── media/paid/[articleId]/[hash]/route.ts # 文章绑定 + 权益校验
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
│   ├── media/                        # MediaStore、hash 对象与文章引用授权
│   ├── search/                       # 标题子串 + 标签 AND
│   ├── recommend/
│   ├── auth/
│   ├── entitlements/                 # 授予生效、重算 access_until
│   ├── payments/                     # PaymentProvider + 微信支付／支付宝／Stripe 适配器
│   ├── redeem/
│   ├── tagging/                      # 提示词、schema、决策与词表提议审核
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
| 同步生成的试读／正文 HTML | Postgres `articles.preview_html` / `article_bodies.body_html` | 不回写 git |
| 媒体引用（文章、源路径、hash、scope） | Postgres `article_media`；对象索引在 `media_objects` | 只存元数据 |

一篇文章一个目录，避免扁平 `content/articles/*.md` 把几十张图堆在同一层。旧的单文件写法不同步进库。

Markdown 里只允许相对引用：

````md
![安装界面](./assets/setup.png)

```mermaid
flowchart LR
  A[Markdown] --> B[sync]
  B --> C[Postgres]
```
````

同步时做三件事：

1. 把 `index.md` 渲成试读 HTML 与独立正文 HTML，正文写入 `article_bodies`，试读与元数据写入公开文章记录。
2. 把 `assets/` 里实际被引用的文件按 content hash 写入 `MediaStore`（本地 POC 为私有目录，生产为私有 OSS/COS；不在 `public/`，不进 Next 静态包）。
3. 把 Markdown 中的相对路径改写成媒体 URL：`public_since` 非空的图 `/media/public/<hash>`，仅付费图 `/media/paid/<articleId>/<hash>`。付费路由带文章身份，避免 hash 复用时跨文章授权。

**Mermaid／架构图：** 源在 Markdown 里。同步时用服务端渲染器（如 `@mermaid-js/mermaid-cli` 或等价库）变成 SVG，再按普通资源走第 2、3 步。不要运行时在读者浏览器里跑 Mermaid——付费图会泄漏，而且首屏会抖。站长如果已经导出了 SVG／PNG，直接放 `assets/`，不必再包一层 Mermaid。

**付费图不进 `public/`。** 媒体对象按 hash 去重，但引用关系按文章保存。只要某个对象曾被公开试读或免费文章引用，就把 `public_since` 固化为非空并视为永久公开；这样不会因 CDN 已缓存而把公开对象“降级”成私有。仅付费引用且 `public_since` 为空的对象，走 `/media/paid/<articleId>/<hash>`，必须同时校验文章与权益；无授予与未知对象均返回 404。公开图可按 hash 长缓存，付费图 `Cache-Control: private, no-store`。

**媒体存储接口：** `MediaStore` 只有 `putIfAbsent(hash, bytes, contentType)`、`open(hash)` 和 `delete(hash)`。本地 POC 使用私有目录；生产切到 OSS/COS 时不改 URL、文章引用或数据库模型。GC 只删除没有引用且没有 `public_since` 的对象。

## 数据模型

规格第 9 节的逻辑记录落到表。时间戳一律 `timestamptz` UTC。

### `articles`

- `id` uuid pk —— 来自 front matter 的不可变文章身份；不是每次同步新生成
- `slug` text unique not null
- `source_path` text not null
- `content_hash` text not null —— 覆盖参与渲染／打标签的规范化 front matter 与 Markdown
- `title` text not null
- `summary` text not null
- `language` text not null check（`zh`、`en`）
- `status` text not null check（`draft`、`published`、`archived`）
- `access` text not null check（`free`、`paid`）
- `preview_mode` text not null check（`summary`、`standard`、`custom`）默认 `standard`
- `topics` text[] not null —— 同步时校验为词表中的 `topic` id
- `recommended` boolean not null 默认 false
- `related_slugs` text[] not null 默认 `{}`
- `published_at` timestamptz null
- `updated_at` timestamptz not null
- `preview_html` text not null 默认 `''`
- 正文 HTML 不放在此表，见 `article_bodies`
- `outline_headings` text[] not null 默认 `{}`  —— 试读截断点之后的标题，公开「你将获得什么」
- `reading_minutes` int not null
- `audience` text null  —— 可选 front matter，公开
- 入站文件校验失败时，同步不得覆盖最近一次有效的已发布行。
- 扫描不完整时，不得把缺失文件当成删除。删除条件：完整扫描后文件不在，或 `status` 不是 `published`。

### `article_bodies`

- `article_id` uuid pk/fk -> `articles.id`
- `body_html` text not null —— 付费正文与免费正文都放在此表，公开文章查询默认不选此列
- `content_hash` text not null
- `updated_at` timestamptz not null

正文仓储接口先完成文章状态与权益判断，再单独读取这一表。这样公开卡片、搜索或 RSC 序列化器误用文章查询时，也不会顺手带出正文列。

### `article_media`

- `article_id` uuid not null —— 与 `source_path` 组成引用主键
- `source_path` text not null  —— git 内相对路径，如 `kimi-code-getting-started/assets/setup.png`
- `hash` text not null —— 外键 -> `media_objects.hash`
- `scope` text not null check（`public`、`paid`） —— 该文章是否在试读／免费正文中引用；免费文章正文引用也必须为 public
- 主键（`article_id`、`source_path`）
- 为（`article_id`、`hash`）与 `hash` 建索引
- 二进制不进这一行。

### `media_objects`

- `hash` text pk —— 文件内容 sha256，物理对象路径由 hash 推导
- `kind` text not null check（`image`、`svg`、`mermaid`）
- `content_type` text not null
- `byte_size` int not null
- `public_since` timestamptz null —— 一旦被公开引用即永久保留为公开对象，避免缓存中的公开 URL 失效后变成私有
- 同一文件被多篇文章引用时只有一行对象记录，多行文章引用记录。

### `article_redirects`

- `from_slug` text pk
- `article_id` uuid not null fk -> `articles.id` —— 路由时读取文章当前 slug，避免重定向链
- 仅当已发布 slug 变更，且站长写入 `content/redirects.yml` 时才写。

### `taxonomy_terms`

- `id` text pk  —— 规范化标识符
- `kind` text not null check（`topic`、`tag`）
- `label_zh` text null
- `label_en` text null
- `public` boolean not null 默认 false
- check（`label_zh` is not null or `label_en` is not null）
- 自动提出的新词条保持 `public=false`，直到管理员接受。

### `article_tag_decisions`

- `article_id` uuid
- `term_id` text
- `source` text not null check（`frontmatter`、`auto`、`reviewer`）
- `state` text not null check（`accepted`、`rejected`）
- unique（`article_id`、`term_id`、`source`）
- front matter 同步只增删 `frontmatter/accepted`；不会误删管理员对自动标签的 reviewer 决策。搜索用标签 = frontmatter accepted ∪ auto accepted，再应用 reviewer 的明确接受／拒绝覆盖。拒绝状态在重跑后仍保留，直到管理员明确恢复。

### `tagging_jobs`

- `id` uuid pk
- `article_id` uuid
- `content_hash` text not null
- `tagger_version` text not null
- `status` text not null check（`pending`、`succeeded`、`failed`、`skipped`）
- `attempt_count` int not null 默认 0
- `tags` text[] not null 默认 `{}`
- `input_scope` text not null check（`preview`、`full`）
- `error` text null
- `finished_at` timestamptz null
- unique（`article_id`、`content_hash`、`tagger_version`、`input_scope`）
- 已有相同组合的成功记录则跳过。运行中源哈希变了，丢弃本次结果。失败时保留上一组成功决策。

### `tag_term_proposals`

- `id` uuid pk
- `job_id` uuid not null fk -> `tagging_jobs.id`
- `article_id` uuid not null fk -> `articles.id`
- `raw_label` text not null
- `language` text not null check（`zh`、`en`）
- `normalized_id` text null
- `state` text not null check（`pending`、`accepted`、`rejected`）默认 `pending`
- `term_id` text null fk -> `taxonomy_terms.id`
- `reviewed_at` timestamptz null
- `reviewer_id` uuid null
- unique（`job_id`、`raw_label`、`language`）
- accepted/rejected 状态必须有 `term_id`；pending 才允许为空

新词先落这里，不直接改变公开词表；接受时在同一事务创建／更新 `taxonomy_terms`，再写入对应文章的 `auto/accepted` 决策。

任务锁只使用 `pg_try_advisory_lock`。锁随数据库连接释放，不再维护容易过期的 `job_locks` 表；任务运行结果由 `tagging_jobs`、审计事件和结构化日志记录。

### `users`

- `id` uuid pk  —— Auth.js 用户 id
- `role` text not null 默认 `reader` check（`reader`、`admin`）
- `access_until` timestamptz null  —— 由授予事务维护的快速投影；取正文时必须重新从数据库读取，不能信任会话或 CDN 缓存
- 付费读者没有草稿权限。管理是单独角色。

### `orders`

- `id` uuid pk
- `user_id` uuid not null
- `provider` text not null  —— `wechatpay`、`alipay`、`stripe`
- `merchant_order_no` text not null —— 本站商户订单号；unique(provider, merchant_order_no)
- `transaction_id` text null —— 通道交易 id；非空时 unique(provider, transaction_id)，与 merchant_order_no 不混用
- `checkout_ref` text null  —— 通道收银会话标识（如 Stripe Checkout Session id）
- `scene` text not null check（`desktop-qr`、`desktop-web`、`mobile-h5`、`wechat-jsapi`）
- `amount_minor` bigint not null —— 货币最小单位，必须为正
- `currency` text not null
- `duration_days` int not null
- `idempotency_key` text not null —— unique(user_id, idempotency_key)；同 key 不同通道／参数拒绝
- `status` text not null check（`pending`、`paid`、`failed`、`canceled`）
- `expires_at` timestamptz null
- `paid_at` timestamptz null
- 非空（`provider`、`checkout_ref`）唯一
- 跳转前在服务端创建，创建时固化通道、金额、币种、时长。浏览器成功页不是付款证据。

### `payment_events`

- `provider` text not null
- `event_id` text not null  —— 通道通知 id 或查单业务事件键；unique(provider, event_id)
- `order_id` uuid null fk -> `orders.id`
- `merchant_order_no` text null —— 未知事件可能无法映射订单；映射成功时必须与订单一致
- `transaction_id` text null
- `amount_minor` bigint null
- `currency` text null
- `payment_state` text not null check（`pending`、`succeeded`、`failed`、`closed`、`unknown`）
- `payload_hash` text not null
- `event_type` text not null
- `status` text not null check（`received`、`processed`、`ignored`、`failed`）
- `received_at` timestamptz not null
- `processed_at` timestamptz null
- `error` text null
- webhook 验签后在同一事务 upsert 事件、锁订单、更新订单、创建授予并写审计。相同 event id 的失败记录可用同 payload 重试；payload hash 变化必须拒绝并告警。规范细节见 [数据模型与一致性](plan/data-model.md)。

### `entitlement_grants`

- `id` uuid pk
- `user_id` uuid not null
- `source_type` text not null check（`order`、`redeem`）
- `source_id` text not null  —— 订单 id 或兑换码 id
- unique（`source_type`、`source_id`）  —— 每个来源只生效一次
- `duration_days` int not null
- `granted_at` timestamptz not null
- `revoked_at` timestamptz null
- 生效规则：新授予的基线是 `max(现在, 当前 access_until)`，再加时长。写入授予与更新 `users.access_until` 在同一事务内完成；撤销后按 `(granted_at,id)` 升序折叠所有未撤销授予，`until := max(until, 该授予 granted_at) + duration_days`，起点为空。
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

- `key` text pk  —— `ip:{rotating-hmac(ip)}:{route}` 或 `user:{id}:{route}`，不存原始 IP
- `window_start` timestamptz not null
- `count` int not null
- 限额走环境变量。超限返回 429 + `Retry-After`。

从规格抽出的校验：
- front matter 必填：不可变 `id`、`slug`、`title`、`summary`、`topics`、`language`、`status`、`access`。
- 发布必须有 `published_at`。
- `published_at` 不得晚于同步时间；MVP 不做定时发布。
- `slug` 是可变公开标识，URL 为 `/articles/{slug}`；文章身份由 `id` 稳定。
- 自动标签最多 8 个。查询长度、筛选标签数、每页条数由服务端限制。
- 内容扫描拒绝符号链接、路径穿越和 content root 之外的资源。

状态机：
- 文章：`draft → published → archived`。完整扫描后磁盘上已发布文件消失，则退出公开目录。
- 订单：`pending → paid | failed | canceled`，`paid` 不回退；本地过期／取消后收到真实验签成功事件仍需核对并授予一次，再告警人工处理；首版无退款流程。
- 兑换码：`unused → used | disabled`。`used` 是终态；管理员撤销的是对应*授予*，那是另一项操作。

## 接口约定

### 公开页面

- `GET /` 文章库：精选最多 6 篇（站长 `recommended` 优先，再用最新补足）、最新列表、搜索表单。卡片：标题、摘要、标签、发布日期、免费／付费标记。
- `GET /search?q=&topic=&tag=&tag=&page=` AND 语义。查询词、筛选、页码都在 URL。无结果状态和清除筛选。复制 URL 可还原。结果只有已发布元数据。
- `GET /articles/{slug}` 草稿／已归档对所有角色（含付费读者）都 404。命中 `article_redirects` 的旧 slug 返回 301 至当前 slug。整页动态 `private, no-store`。免费：授权后读取 `article_bodies.body_html`。付费且无授予：标明「试读」的试读区、「你将获得什么」、一个 CTA。付费且有授予：授权后读取正文。相关阅读：站长 `related` 优先，再按共同主题／标签补足到 4 篇，slug 打平，排除草稿／自身／重复。付费相关卡片必须打标，打开时仍做权限校验。
- `GET /pricing` 一种通行证：时长、价格、币种、明确声明不支持退款、支付入口、兑换码入口。
- `GET /checkout/{orderId}` 收银台等待／结果页：只展示本地订单状态并限频轮询，不构成授予证据。
- `GET /account` `access_until`、购买状态、退出登录。
- `GET /login`
- `GET /media/public/{hash}` `public_since` 非空的对象，可匿名；按 hash 长缓存。未知或仅付费对象 404。
- `GET /media/paid/{articleId}/{hash}` 必须有该文有效授予，且 `article_media` 存在 paid 引用；无授予、文章不匹配与未知 hash 一律 404。`Cache-Control: private, no-store`。公开对象统一走 `/media/public/{hash}`。

### JSON／POST

- `POST /api/checkout` 必须登录 → 校验已启用 provider 与支付场景，创建幂等订单及支付请求。返回 `redirect`／`form-post`／`qr`／`jsapi` 判别联合，不再只返回 URL。
- `POST /api/webhooks/{wechatpay|alipay|stripe}`：各通道独立路由及官方协议验签／回执，核对商户、应用、订单、金额、币种和状态；`payment_events(provider,event_id)` 幂等。只认验证过的服务端事件或主动查单结果，不认浏览器回跳。
- `GET /api/orders/{id}`：仅订单所有者可读，返回本地支付状态；前端限频轮询，不直接暴露支付凭据。
- `POST /api/redeem` 必须登录，限流。Body `{ "code": "..." }`。响应：已授予、你已兑换过、无效（无效／过期／停用／已被他人使用用同一句文案，不泄露兑换者）。
- 搜索与文章 GET 走 HTML／RSC，不提供未认证的批量 JSON 导出。

### CLI

- `pnpm content:sync [--dry-run] [--prune]` 校验不可变 `id`、渲染、upsert，把未发布内容移出公开目录。默认不因缺失文件归档；只有 content-root 哨兵存在且完整遍历成功的 `--prune` 才允许归档。整次原子发布：任一文件无效则整次不发布、保留上一有效版本；已发布文章 slug 变更而 `content/redirects.yml` 缺对应条目时同步报错。
- `pnpm tags:run` 拿 advisory lock、按文章／内容哈希／tagger 版本／出站范围跳过未变更、临时故障最多重试 3 次，成功后刷新搜索标签与提议。
- `pnpm tags:review` 接受／拒绝／恢复自动标签；逐条接受词表外提议并在事务中置 `taxonomy_terms.public=true`。
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
6. 文章 GET 按 IP 和账户限流；整页响应 `no-store`，不依赖 CDN 按 cookie 分流。

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
- 匿名请求的 HTML、RSC 载荷、`/_next/static`、`/media/public/*` 中都没有付费正文或仅付费图。`/media/paid/{articleId}/*` 无授予、文章不匹配或未知 hash 时 404；同一 hash 跨文章引用不会绕过授权。
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

### 阶段 0 — 可行性与闭环 spike

启动 ICP、回调域名、微信网站应用、微信支付 Native 和支付宝电脑网站支付的准入清单；用最小 throwaway route 验证微信扫码回调、一个 Native 二维码、一个支付宝电脑网站请求和各自回执。确定生产云厂商与 `MediaStore` 实现。任何未通过的外部依赖都记录为 feature flag，不用个人收款码或 mock 冒充上线能力。

### 阶段 1 — 内容与发现

Compose 起 Postgres，Drizzle 建文章／正文／媒体／词表／重定向表，Markdown 校验与同步，安全渲染，文章库／搜索／文章页，标题／标签 AND 搜索，固定规则推荐。同步为整次原子发布：任一文件无效则整次不发布，绝不出现混合代际；一个错文件挡住其他更新，由可操作错误信息缓解。付费文件同步元数据、试读和隔离正文；文章页在权益模块存在前一律拒绝返回正文（缺授予视为无授予，失败时关死）。

### 阶段 2 — 后台打标签

`taxonomy.yml`、`tags:run`、跳过／重试／加锁、决策合并、CLI 接受／拒绝／恢复。文章不必等打标签完成就能发布。人工标签应立即可搜。

### 阶段 3 — 访问控制

Auth.js、微信扫码登录、账户、试读区 + CTA、权益授予、一次性兑换码、限流、缓存头（文章动态 `Cache-Control: private, no-store`）。管理员 CLI 负责发码和撤销。GitHub／邮件 provider 的接口可以先保留，绑定流程在 provider 启用前补齐。

### 阶段 4 — 支付与上线

微信 Native 与支付宝电脑网站支付适配器；按已启用场景展示二维码／跳转、订单状态查询、幂等通知与 pending 对账。移动支付与 Stripe 只保留接口和 fixture，取得权限后独立启用。微信登录与两个国内桌面支付是 MVP 验收项；测试环境不可用时不得以 mock 代替真实联调。

## 实现前需要的密钥与账号

| 名称 | 用途 |
| --- | --- |
| `DATABASE_URL` | Postgres |
| `AUTH_SECRET` | Auth.js 会话 |
| `AUTH_GITHUB_ID`／`AUTH_GITHUB_SECRET` | GitHub 登录 |
| `RESEND_API_KEY`／`EMAIL_FROM` | magic link |
| `STRIPE_SECRET_KEY`／`STRIPE_WEBHOOK_SECRET`／`STRIPE_PRICE_ID`／`STRIPE_MODE`（`test`｜`live`） | Stripe 扩展；MVP 不要求配置 |
| `PASS_CNY_AMOUNT_MINOR`／`STRIPE_PASS_AMOUNT_MINOR`／`STRIPE_PASS_CURRENCY`／`PASS_DURATION_DAYS` | 单一 SKU，分通道服务端价格 |
| `WECHAT_LOGIN_APP_ID`／`WECHAT_LOGIN_APP_SECRET` | 网站扫码登录 |
| `WECHAT_PAY_APP_ID`／`WECHAT_PAY_MCH_ID`／`WECHAT_PAY_API_V3_KEY`／`WECHAT_PAY_PRIVATE_KEY_PATH`／`WECHAT_PAY_SERIAL_NO`／`WECHAT_PAY_PUBLIC_KEY_ID`／`WECHAT_PAY_PUBLIC_KEY_PATH` | 微信支付签名、验签及通知解密；平台证书模式由适配器另行配置 |
| `ALIPAY_APP_ID`／`ALIPAY_PRIVATE_KEY_PATH`／`ALIPAY_PUBLIC_KEY_PATH`／`ALIPAY_GATEWAY` | 支付宝 RSA2 模式；如选择证书模式，单独配置证书链 |
| `APP_BASE_URL`／`PAYMENT_PROVIDERS_ENABLED`／`WECHAT_LOGIN_ENABLED` | 固定回调地址来源、支付与登录独立开关 |
| `PUBLIC_INDEXING` | 公开页收录开关，默认关（决策 7） |
| `PREVIEW_MAX_WEIGHTED_CHARS` | 试读绝对上限，默认 6000；同步时强制 |
| `RATE_LIMIT_*` | 各路由 IP／账户限额（见限流表） |
| `TAGGER_BASE_URL`／`TAGGER_API_KEY`／`TAGGER_MODEL`／`TAGGER_EGRESS_MODE`／`TAGGER_ALLOWED_HOSTS` | 离线打标签；默认只发送标题、摘要和试读 |
| `ADMIN_USER_IDS` | 引导管理员 |

MVP 启用微信支付、支付宝两个国内桌面场景；Stripe 与移动支付保留 provider／scene 接口，未通过准入与真实联调前不展示，不增加其他通道或分账。

## 明确不做

浏览器内 Markdown 编辑、读者投稿、评论、社交信息流、基于行为的个性化、全文或语义搜索、向量库、任务队列、多商户分账、独立移动应用、单篇购买、自动续费订阅、自动退款流程、全站共享密码、面向读者的 API Key、存储银行卡、禁用复制／右键。移动支付场景、Stripe、GitHub 和 magic link 不是 MVP 核心验收项，但保留兼容接口。

## 关键决策

1. **一个 Next.js 应用 + Postgres + CLI** —— 对齐规格第 9 节，付费 HTML 留在服务端。
2. **试读与正文是同步时的两份产物** —— 正文单独存表，授权后才读取。
3. **授予只追加、来源唯一** —— 购买、兑换、叠加共用一次重算，`access_until` 是事务维护的投影。
4. **国内支付先做桌面闭环** —— 微信 Native 与支付宝电脑网站共用领域模型；移动场景和 Stripe 只通过小 adapter 扩展。
5. **媒体对象与文章引用分离** —— hash 去重不等于 hash 授权；公开对象永久公开，付费 URL 带 article id。
6. **打标签不能挡住发布** —— 人工标签和标题搜索立刻可用；模型只是批处理补充。

## 前提崩塌

本计划以 Next.js + Postgres 为基础，微信扫码登录、微信 Native 与支付宝电脑网站支付是 MVP 必需能力；Stripe、移动支付和额外登录 provider 保留接口。各平台账号审核与产品开通在可行性阶段开始，不能等到阶段 4 才发现不具备接入条件。生产环境面向中国大陆：ICP 备案与企业主体是支付开通与公开上线的硬前提。

未开通的场景保持关闭，其他已启用通道可独立运行；兑换码可作为支付未就绪时的 POC 降级，但不能宣称支付验收完成。若商户申请受阻，记录外部依赖，不自行替换成个人收款码或删除需求。

Auth.js 没有官方微信 provider：网站扫码登录需要自定义 provider（`appid`／`secret` 参数名、`qrconnect` 授权端点等非标准 OAuth2 细节），阶段 3 早期安排 spike 验证，不假设开箱即用。

## 明确推迟的未知项

| 事项 | 为何推迟 | 负责人 |
| --- | --- | --- |
| 通行证准确价格 | 运营事项，不是表结构。读环境变量。 | 阶段 4 前由站长定 |
| ICP 备案（含支付／回调域名、CDN 加速域名） | 硬前提，需企业主体；微信支付／支付宝商户、回调域名、国内 CDN 均依赖 | 可行性阶段启动办理；备案完成前不公网上线 |
| 微信网站应用与微信支付、支付宝产品开通 | 分别登记主体、域名、产品权限和审核状态，不能假设登录审核等于支付开通。 | 阶段 1 开始，阶段 3／4 前完成 |
| Stripe 商户准入与打款资格 | 独立于国内支付，按当前平台规则确认。 | 启用 Stripe 前确认；不挡 MVP |
| 是否打开 `PUBLIC_INDEXING` | 规格第 12 节上线决策。默认关。 | 上线时由站长定 |
| 国内云厂商择一（阿里云/腾讯云） | 生产环境已定中国大陆云；厂商择一影响备案主体与资源开通 | 阶段 1 由站长定 |
| Resend 邮件境内可达性 | magic link 是扩展 provider，在国内邮箱（QQ/163 等）的到达率待验证 | 启用 magic link 前验证 |
| 试读篇数日限（按 IP／账户） | 防批量收割 30% 试读的后续手段；现默认不启用，滥用迹象出现时按观测数据再加 | 出现滥用时由站长定 |

只有 ICP、MVP 微信登录／支付产品和生产存储方案挡住正式上线；其他扩展项不挡住内容、兑换码和核心访问控制 POC。

## 复杂度跟踪

> 用户明确扩展首版支付范围，原单通道限制不再适用。

| 违规 | 为何需要 | 更简单方案被否的原因 |
| --- | --- | --- |
| 原单支付服务商限制 | 用户要求微信支付与支付宝，同时保留国际卡 | 单通道不能覆盖已确认需求；统一适配器与权益账本限制复杂度 |
