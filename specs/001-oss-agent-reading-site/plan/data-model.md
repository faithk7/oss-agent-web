# 数据模型与一致性计划

本文是 `plan.md` 的规范性补充。MVP 保持关系模型小而明确；数据库迁移必须把这里的约束实现为数据库约束或事务逻辑。

## 事实来源

- Git 中带不可变 `id` 的 Markdown 和资源文件是内容事实来源。
- `articles` 保存公开元数据与试读；`article_bodies` 保存独立正文。公开查询默认不选正文表。
- `article_tag_decisions` 是文章标签决策事实来源；`tag_term_proposals` 是词表外建议事实来源；数组或缓存只能重建。
- `entitlement_grants` 是权益账本事实来源；`users.access_until` 是事务维护的快速投影，可由账本重建。
- `payment_events` 是已验签支付事件的幂等事实来源，不保存完整支付报文。

## 支付事件与 reducer

`payment_events` 至少保存 `provider`、`event_id`、可空的 `order_id`／`merchant_order_no`、`transaction_id`、`amount_minor`、`currency`、`payment_state`、`payload_hash`、`event_type`、`status`、接收／处理时间和脱敏错误。`(provider,event_id)` 唯一；非空 `(provider,transaction_id)` 也唯一。

验签／主动查单在事务外完成，得到规范事件后，在一个短事务中：

1. 用 `(provider,event_id)` upsert 事件。
2. 相同 event id 且 payload hash 相同：`processed`／`ignored` 直接幂等确认；`failed` 允许再次处理。
3. 相同 event id 但 payload hash 不同：拒绝并告警，不改变订单。
4. 锁定订单，核对 provider、商户订单号、商户／应用标识、金额、币种和合法状态迁移。
5. 成功事件将订单置为 `paid`，按 `(source_type,source_id)` 唯一约束创建授予并更新权益投影；未知或不适用事件记录后置为 `ignored`。

订单状态只允许 `pending -> paid | failed | canceled`，`paid` 不回退。若本地订单已过期／取消但服务商随后给出验签成功事件，仍按真实付款核对并授予一次，订单转为 `paid`，同时告警供人工处理；不能因本地超时静默吞掉付款。

事务回滚时 webhook 返回失败，让服务商重试；不能因为已有 `failed` 行而跳过。查单使用带 provider 命名空间的稳定业务 event id，并经过同一 reducer。

## 权益事务与读取

兑换、支付授予、撤销都使用事务：锁定用户行及相关授予，按 `(granted_at,id)` 升序折叠未撤销授予，`until := max(until, granted_at) + duration_days`，再写回 `users.access_until`。新授予的基线为 `max(transaction_timestamp(), current access_until)`。

所有正文访问调用同一个 `canReadPaidArticle(userId, articleId)`：在数据库中读取当前已提交的用户投影和文章状态；缺失会话、数据库错误、非 published 或投影不晚于当前时间均拒绝。投影是服务端数据库值，不从 session、RSC props 或客户端传入值读取。撤销提交后影响后续请求；已经在撤销提交前取得一致性快照的请求可以完成当前响应。

## 内容同步与媒体

- `articles.id` 来自 front matter 的不可变 UUID；`slug` 可变但唯一。
- 同步先扫描、校验、渲染并建立 manifest；拒绝符号链接、路径穿越和 content root 外的资源。只有检测到固定 content-root 哨兵且使用 `--prune` 的完整遍历才允许归档缺失文章。
- 只有所有文件成功后才在一个发布事务中替换文章、正文、标签决策、媒体引用和重定向。失败保留最近一次有效版本；不完整扫描禁止归档缺失文章。
- 已发布文章以 `id` 匹配；slug 变化没有 `redirects.yml` 映射就失败。`article_redirects` 保存 `from_slug -> article_id`，路由时解析文章当前 slug，避免重定向链。
- `media_objects(hash)` 保存去重后的物理对象；`article_media(article_id,source_path)` 保存引用及 `scope`。任何公开引用都会把对象的 `public_since` 固化为非空，不能降级为私有。付费媒体 URL 必须带 article id。
- `article_bodies` 与文章元数据分表；同步可写入正文，但阶段 3 前正文仓储没有公开读取路径。

## 标签与提议

- `article_tag_decisions` 的 `source` 为 `frontmatter`、`auto` 或 `reviewer`，唯一键为 `(article_id,term_id,source)`。同步只增删 front matter 决策；人工 reviewer 的明确拒绝不会被自动重跑覆盖。
- `tagging_jobs` 的跳过键为 `(article_id,content_hash,tagger_version,input_scope)`。`content_hash` 覆盖标题、摘要、试读和参与提示词的规范化内容。
- `new_terms` 先写入 `tag_term_proposals`，包含文章、任务、原文、语言和审核状态；接受提议时在同一事务创建／更新 taxonomy term 与文章自动标签。词表外建议在接受前不可公开搜索。

## Auth.js 数据

使用官方 adapter 表（users、accounts、sessions、verification_tokens），明确 UUID 映射。账号唯一键包含 provider 与 providerAccountId；微信身份额外包含 `appid:openid` 命名空间。邮箱／OAuth 账号合并必须要求已验证身份或已登录确认，禁止仅凭相同邮箱自动接管账户。magic-link token 只存哈希、单次使用、短期有效。

## 索引与约束

- `articles(status, published_at DESC, slug)`，并约束 published 文章的 `published_at <= now()` 由同步层保证（MVP 不做定时发布）。
- 标题搜索使用 `lower(title)` + `pg_trgm`；2k 篇基准记录 SQL 与 HTTP p95。
- `article_bodies.article_id`、`article_media(article_id,hash)`、`article_media.hash`、`article_tag_decisions(term_id,state)`、`tag_term_proposals(state)` 建索引。
- 所有外键、状态枚举、非负时长／金额、来源唯一约束、provider checkout reference 唯一约束写入迁移。

## 微信身份与多通道订单

- 账号身份唯一键包含 provider 与 `appid:openid`；`unionid` 可空、带开放平台命名空间，仅作服务端验证后的关联线索。昵称、头像和邮箱不参与唯一性。
- `orders.provider` 为 wechatpay/alipay/stripe；`amount_minor` 使用整数，保存不可变价格／时长快照、scene、merchant_order_no、checkout_ref、transaction_id、expires_at、created_at、paid_at。
- 创建幂等键 unique(user_id,idempotency_key)，并保存请求指纹；同 key 不同通道／参数拒绝。切换通道新建订单，旧订单仍需对账。
- 网站 AppID 的 openid 与 JSAPI 支付 AppID 的 openid 不可互换；不持久化无必要的 OAuth token 或完整付款人资料。
