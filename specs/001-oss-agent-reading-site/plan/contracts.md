# 接口与 CLI 合约

## PaymentProvider

```ts
type Provider = 'wechatpay' | 'alipay' | 'stripe';
type PaymentAction =
  | { kind: 'redirect'; url: string }
  | { kind: 'qr'; codeUrl: string; expiresAt: string }
  | { kind: 'jsapi'; parameters: Record<string, string> };
interface PaymentProvider {
  createPayment(input: {
    orderId: string; userId: string; amountMinor: number; currency: string;
    durationDays: number; scene: 'desktop' | 'mobile' | 'wechat';
    idempotencyKey: string; payerOpenid?: string;
  }): Promise<{ action: PaymentAction; checkoutRef?: string }>;
  verifyNotification(rawBody: Uint8Array, headers: Headers): Promise<NormalizedPaymentEvent[]>;
  queryOrder(orderId: string): Promise<NormalizedPaymentEvent>;
  queryRefund(orderId: string): Promise<NormalizedRefundState>;
}
```

类型为合约草案；NormalizedPaymentEvent 至少含 provider、eventId、orderId、transactionId、商户/应用标识、amountMinor、currency、状态与发生时间。只有适配器验签或认证查单生成的事件才能进入授予事务。退款由商户后台发起，站内通知/查询确认后撤销；v1 不提供浏览器退款 API。

金额、币种、时长由服务端价格配置固化到订单，客户端不可覆盖。支付宝金额使用十进制定点字符串与 CNY 分精确转换，禁止浮点乘除。`POST /api/checkout` 接收 provider、scene 与幂等键，不接收可信的价格或 userId；幂等键绑定当前用户及请求指纹。未支持的设备场景返回明确错误，不静默切换付款方式。

## 登录与身份绑定

- 桌面微信网站扫码授权使用网站 AppID；回调通过 Auth.js 微信适配器进入同一会话，不另外建立弱会话。
- 一次性 state 绑定发起浏览器、用途（登录/绑定）、站内返回路径和短有效期；code 仅服务端交换，取消、过期和重复 code 均有可恢复状态。
- 账号绑定要求当前登录会话及目标微信重新授权，账号冲突禁止自动合并；不生成虚假邮箱。
- 微信支付不要求用户先用微信登录；已有 GitHub/邮件会话可付款。JSAPI 另取支付 AppID 下的 payer openid，不将付款人身份当成站内登录身份。

## Webhook 与付款状态

| 通道 | 验证与业务核对 | 持久化成功后的回执 |
|---|---|---|
| 微信支付 `/api/webhooks/wechatpay` | API v3 原始报文签名、可信公钥/平台证书、通知资源解密；mchid/appid、out_trade_no、金额与币种 | 按产品协议返回成功 HTTP 状态 |
| 支付宝 `/api/webhooks/alipay` | 表单参数按支付宝规则 RSA2 验签；app_id/seller_id、out_trade_no、total_amount、trade_status | HTTP 200，纯文本 `success` |
| Stripe `/api/webhooks/stripe` | SDK 验证原始报文与通知签名；订单、金额、币种 | 按平台协议返回成功响应 |

不得把微信 JSON 原始报文验签规则套到支付宝表单通知；防重放窗口遵循各通道协议，不拒绝合法延迟通知。未知事件验证后记录并确认；处理失败不得成功确认。数据库事务回滚后，重试必须仍可处理，不能因为存在失败事件就跳过。仅 `processed/ignored` 是可直接确认的幂等终态。

通知与主动查单走相同 reducer 和来源唯一约束；退款先到、付款后到不能恢复已撤销权益。`GET /api/orders/{id}` 仅所有者可读本地状态，限频轮询；扫码/回跳/JSAPI 的前端成功状态不授予权益。超时创建先查单，不盲目创建新支付。全额退款撤销来源；部分退款记录并进入人工处理，不自动全额撤销。

## 内容与文章接口

文章响应只返回当前访问者允许的一个 HTML 字段；搜索和推荐只返回元数据。付费文章、付费媒体和所有 RSC 路径均动态执行权限检查并 `private, no-store`。媒体授权同时绑定 `article_id` 和 hash。

## CLI

- `content:sync [--dry-run]`：完整扫描、输出差异；提交失败不改变已发布版本。
- `tags:run`：加锁、幂等、最多重试 3 次，旧标签在失败时保留。
- `codes:issue`、`codes:disable`、`entitlements:revoke`：参数校验、操作员身份/权限、审计事件；明文兑换码只打印一次。
- `ops:status`、`ops:reconcile-orders`、`ops:cleanup-media`：只读诊断、订单对账和媒体 GC。

CLI 的退出码、JSON/文本输出和错误分类要稳定，方便 cron 与 CI 使用。
