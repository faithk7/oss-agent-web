# 安全、隐私与访问控制计划

## 内容与媒体

HTML 使用 allowlist 清理：禁止事件属性、脚本、危险 URL scheme、可执行嵌入和外部 SVG 引用。SVG/Mermaid 输出再次清理，并限制大小、渲染时间和资源引用。媒体路径只接受 hash，拒绝路径穿越、符号链接和不允许的 MIME/大小。付费媒体校验文章与权益，不以 hash 单独授权。

## 认证与请求

Auth.js cookie 使用 Secure、HttpOnly、SameSite=Lax；浏览器业务 POST 检查 CSRF token 与可信 Origin（服务端支付通知使用通道签名认证，不要求浏览器 CSRF token）。magic-link 短期、单次、哈希存储；登录失败和兑换失败使用不枚举账户的文案。账户按 IP 和用户限流；仅信任显式配置的反向代理，IP 规范化后用可轮换 HMAC 存储，不记录原始 IP。

## 付费内容边界

付费 HTML 不进入静态构建产物、公共缓存、CDN、客户端 props、RSC 非必要序列化对象、错误页、日志或 analytics。数据库错误、缺失会话和未知媒体均 fail closed。付费页面强制动态渲染和 `no-store`；设置 CSP、HSTS、Referrer-Policy、X-Content-Type-Options、frame-ancestors。

## 管理与隐私

`ADMIN_USER_IDS` 启动时严格解析且默认拒绝；破坏性 CLI 必须由受信操作员执行并写审计。定义邮箱、OAuth ID、订单、审计和限流数据的保留/删除策略；隐私政策与服务条款（声明不支持退款）上线前完成。微信登录／微信支付、支付宝、Resend、Stripe、标签模型作为处理方记录数据跨境与密钥轮换方案。用户数据境内存储；隐私政策符合《个人信息保护法》（PIPL）：个人信息最小化、境内存储、明示处理方清单。DeepSeek 为境内服务（正文不出境）；Stripe、Resend、GitHub OAuth 涉及出境，在隐私政策中明示并评估必要性。

## 安全测试

覆盖 sanitizer 回归、SVG/Mermaid、CSRF、会话固定、代理 IP、路径穿越、RSC/缓存泄漏、错误路径、媒体跨文章访问、webhook 重放和密钥泄露扫描。

## 微信与支付宝接入边界

- 网站登录 Secret、微信 API v3 密钥、商户私钥、支付宝私钥分离管理；私钥仅服务端读取。微信按公钥 id/平台证书序列号选可信验签材料并支持轮换；支付宝明确选择 RSA2 公钥或证书模式，不混用配置。
- OAuth state、防 code 重放、绑定账号冲突和开放重定向均有测试。登录与付款互不冒充授权，付款 openid 不改变订单所属用户。
- 通知验签后仍核对商户、AppID、订单、金额、币种和交易状态。支付宝按其参数规范验签，微信按 API v3 验签并解密，使用成熟密码库，不自制签名算法。
- 支付 URL/二维码不得包含站内会话密钥；订单状态端点防越权、限流和 no-store。密钥、授权 code、token、通知解密正文不进入日志。
- 支付通知路由不做 IP 限流（必须允许服务商重试），验签是门槛；签名失败率异常时告警。
