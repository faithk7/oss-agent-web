# 数据模型与一致性计划

本文是 `plan.md` 的规范性补充。数据库迁移必须把这里的约束实现为数据库约束或事务逻辑。

## 事实来源

- Git 中的 Markdown 和资源文件是内容事实来源。
- `article_tag_decisions` 是标签决策事实来源；front matter 人工标签在同步时以 human/accepted 落入该表；`articles.search_topics/search_tags`（如实现）只能是可重建缓存。
- `entitlement_grants` 是权益事实来源；`users.access_until` 是提示性缓存，任何正文访问都以事务内重算/核验为准。
- `payment_events` 是支付事件幂等事实来源。

## 支付事件

新增 `payment_events`：`provider`、`event_id`、`payload_hash`、`event_type`、`status(received|processed|ignored|failed)`、`received_at`、`processed_at`、`error`；`(provider,event_id)` 唯一。webhook 验签后，在同一事务中插入事件、锁定订单、更新订单、创建授予并写审计。重复已处理事件按各通道协议确认且不重复授予；未知事件记录后忽略；可重试失败保留错误状态。

## 权益事务

兑换、支付授予、撤销都使用事务：锁定用户行和相关授予（`SELECT ... FOR UPDATE`），按 `(granted_at,id)` 升序折叠未撤销授予——`until := max(until, 该授予 granted_at) + duration_days`，起点为空——更新 `access_until`。授予来源唯一约束、`duration_days > 0`、金额非负和币种白名单由数据库保证。并发购买/兑换不能丢失时长或重复授予；管理员撤销只影响对应来源，重算其余有效授予。

## 内容同步

同步先完整扫描、校验、渲染并建立 manifest；支持 `--dry-run`。只有所有文件成功后才在一个发布事务中替换文章、标签、媒体引用和重定向。失败保留最近一次有效版本。不完整扫描禁止删除；完整扫描才可将缺失的已发布文章归档。已发布文章 slug 变更而 `redirects.yml` 缺对应条目时同步失败。重定向目标必须存在、不可成环。媒体按 hash 去重，未被引用的文件由定期 GC 清理。

## Auth.js 数据

使用官方 adapter 表（users、accounts、sessions、verification_tokens），明确 UUID 映射。邮箱/ OAuth 账号合并必须要求已验证身份或已登录确认，禁止仅凭相同邮箱自动接管账户。magic-link token 只存哈希、单次使用、短期有效。

## 索引与约束

- `articles(status, published_at DESC, slug)`。
- 标题搜索使用 `lower(title)` + `pg_trgm`（或在 2k 篇基准中证明现方案足够）。
- 标签决策按 `(article_id, term_id)` 唯一，并为 `term_id/state` 建索引。
- 所有外键、状态枚举、非负时长/金额、时间顺序和唯一来源约束写入迁移。

## 微信身份与多通道订单（本次范围更新）

- 账号身份唯一键包含 provider 与 `appid:openid`，`unionid` 可空、带开放平台命名空间，仅作服务端验证后的关联线索；邮箱可空，昵称/头像不参与唯一性。绑定到现有用户需要双向身份确认。
- `orders.provider` 为 wechatpay/alipay/stripe；`amount_minor` 使用整数，微信支付/支付宝为 CNY 分。保存不可变价格/时长快照、scene、merchant_order_no、checkout_ref、transaction_id、expires_at、created_at、paid_at。`(provider, merchant_order_no)` 与非空 `(provider, transaction_id)` 唯一。
- 创建幂等键 unique(user_id, idempotency_key)，并保存请求指纹；同 key 不同通道/参数拒绝。切换通道新建订单，旧订单仍需对账；两笔真实付款分别授予，不能静默吞掉第二笔。
- `payment_events` 的 event_id 来自微信通知 id、支付宝 notify_id、Stripe 事件 id；查单使用稳定的带命名空间业务事件键。不同事件指向同一订单仍由授予来源唯一约束去重。
- 不持久化无必要的微信 OAuth token／完整付款人资料；网站 AppID 的 openid 与 JSAPI 支付 AppID 的 openid 不可互换。
