# 接口与 CLI 合约

MVP 只启用桌面微信 Native 和支付宝电脑网站支付；contract 保留移动场景与 Stripe 的扩展位置，但未启用的 provider 不出现在收银台。

## PaymentProvider

```ts
type Provider = 'wechatpay' | 'alipay' | 'stripe';
type PaymentScene = 'desktop-qr' | 'desktop-web' | 'mobile-h5' | 'wechat-jsapi';

type PaymentAction =
  | { kind: 'qr'; codeUrl: string; expiresAt: string }
  | { kind: 'redirect'; url: string; expiresAt?: string }
  | { kind: 'form-post'; action: string; fields: Record<string, string>; expiresAt?: string }
  | { kind: 'jsapi'; appId: string; timeStamp: string; nonceStr: string;
      package: string; signType: 'RSA'; paySign: string };

type ServerOrder = {
  orderId: string; userId: string; merchantOrderNo: string;
  amountMinor: bigint; currency: string; durationDays: number;
  provider: Provider; scene: PaymentScene; idempotencyKey: string;
};

interface PaymentProvider {
  createPayment(input: ServerOrder & { payerOpenid?: string }):
    Promise<{ action: PaymentAction; checkoutRef?: string }>;
  verifyNotification(rawBody: Uint8Array, headers: Headers):
    Promise<NormalizedPaymentEvent[]>;
  queryOrder(order: ServerOrder): Promise<NormalizedPaymentEvent>;
}
```

`ServerOrder` 由服务端从价格配置和数据库订单构造，客户端不可覆盖金额、币种、时长或 userId。支付宝金额使用十进制定点字符串转换为 CNY 分，禁止浮点乘除；TypeScript 金额使用 `bigint` 或十进制字符串，不用浮点数。

`NormalizedPaymentEvent` 至少含 provider、eventId、可空的 orderId／merchantOrderNo／transactionId、商户／应用标识、可空的 amountMinor／currency、状态、发生时间和 payloadHash。只有适配器验签或认证查单生成的事件才能进入授予 reducer。

## 登录与身份绑定

- MVP 桌面微信网站扫码授权使用网站 AppID；回调通过 Auth.js 自定义 provider 进入同一会话，不另外建立弱会话。
- 一次性 state 绑定发起浏览器、用途（登录／绑定）、站内返回路径和短有效期；code 仅服务端交换，取消、过期和重复 code 均有可恢复状态。
- MVP 只启用微信 provider。启用 GitHub 或 magic link 前，必须实现当前会话 + 目标 provider 重新授权的绑定流程；账号冲突禁止自动合并，不生成虚假邮箱。
- 微信支付不要求用户先用微信登录。JSAPI 另取支付 AppID 下的 payer openid，不将付款人身份当成站内登录身份。

## Webhook 与付款状态

| 通道 | 验证与业务核对 | 持久化成功后的回执 |
|---|---|---|
| 微信支付 `/api/webhooks/wechatpay` | API v3 原始报文签名、可信公钥／平台证书、通知资源解密；mchid/appid、out_trade_no、金额与币种 | 按产品协议返回成功 HTTP 状态 |
| 支付宝 `/api/webhooks/alipay` | 表单参数按支付宝规则 RSA2 验签；app_id/seller_id、out_trade_no、total_amount、trade_status | HTTP 200，纯文本 `success` |
| Stripe `/api/webhooks/stripe` | SDK 验证原始报文与通知签名；订单、金额、币种 | 按平台协议返回成功响应 |

不得把微信 JSON 原始报文验签规则套到支付宝表单通知；防重放窗口遵循各通道协议，不拒绝合法延迟通知。未知事件验证后记录并确认；处理失败不得成功确认。相同事件重试由 `payment_events` reducer 处理：相同 payload 可重试，hash 变化拒绝并告警。仅 `processed/ignored` 是可直接确认的幂等终态。

通知与主动查单走相同 reducer 和来源唯一约束。`GET /api/orders/{id}` 仅所有者可读本地状态，限频轮询；扫码／回跳／JSAPI 的前端成功状态不授予权益。超时创建先查单，不盲目创建新支付。通知路由不做普通 IP 限流——验签是门槛，必须允许服务商重试。

## 内容与文章接口

文章响应只返回当前访问者允许的一份产物。`articles/[slug]` 整页动态执行权限检查并 `private, no-store`，不使用 `generateStaticParams` 或公共 HTML 缓存。付费正文先通过 `canReadPaidArticle`，再读取 `article_bodies`；搜索和推荐只返回元数据。

公开媒体访问 `/media/public/{hash}` 仅接受 `media_objects.public_since is not null`。付费媒体访问 `/media/paid/{articleId}/{hash}` 同时校验文章引用、对象未公开和用户权益；未知对象、文章不匹配和无权益均返回 404。

## CLI

- `content:sync [--dry-run] [--prune]`：校验不可变 `id`、渲染、输出差异；默认不因缺失文件归档文章。只有 `--prune` 在 content-root 哨兵存在且完整遍历成功时才允许归档；提交失败不改变已发布版本。
- `tags:run`：拿 advisory lock、按 `(article_id,content_hash,tagger_version,input_scope)` 幂等、最多重试 3 次，旧标签在失败时保留。
- `tags:review`：接受／拒绝／恢复自动标签，逐条处理词表外提议。
- `codes:issue`、`codes:disable`、`entitlements:revoke`：参数校验、受信操作员身份／权限、审计事件；明文兑换码只打印一次。
- `ops:status`／`ops:reconcile-orders`／`ops:cleanup-media`：只读诊断、订单对账和媒体 GC。

生产 CLI 只能在受信运维环境运行；破坏性命令必须带 `--actor <admin-user-id>`，该用户必须在 `ADMIN_USER_IDS` 中，审计记录不接受任意客户端字符串。CLI 的退出码、JSON／文本输出和错误分类要稳定，方便 cron 与 CI 使用。
