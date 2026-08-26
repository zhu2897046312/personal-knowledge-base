---
title: PayPal 国区与国际版账号体系隔离避坑
tags: [paypal, payment, cross-border, oauth]
created: 2026-08-26
updated: 2026-08-26
aliases: [paypal.cn 登录失败, PayPal OAuthException]
summary: paypal.cn（贝宝中国）与 paypal.com（PayPal 国际版）账号体系彻底隔离，用国区账号登录海外结算弹窗会报密码错误/账号不存在/OAuthException
type: pitfall
---

# 问题

在开发或测试跨境电商支付（如 Medusa / Shopify 等）时，使用在 `paypal.cn` 注册的账号在海外结算页面（`paypal.com` 弹窗）登录，系统会频繁提示密码错误、账号不存在，或直接提示请求无效（`OAuthException`），无法完成支付交互。

# 原因

- **数据与体系彻底隔离**：`paypal.cn`（贝宝中国）与 `paypal.com`（PayPal 国际版）属于完全独立的业务体系。
- **定位不同**：国区账号属于境内持牌机构（贝宝支付），主要功能是跨境结算与商户提现，不具备作为买家在全球 `paypal.com` 统一收银台进行跨国网购消费的数据库注册身份。
- **重定向机制**：浏览器登录过 `paypal.cn` 后会保留 Cookie，导致访问 `paypal.com/c2` 时自动重定向并加载国区身份，引发混淆。

# 解决方案

## 1. 注销原国区账号（可选）

1. 登录 `https://www.paypal.cn/`。
2. 进入账户设置与钱包，清空余额并解绑所有银行卡/信用卡。
3. 点击「设置（齿轮）」→「注销账户（Close your account）」。
4. 静置等待 24 小时，确保系统彻底清空该邮箱与卡片的绑定记录。

## 2. 重新注册 PayPal 国际版账号

1. 打开浏览器无痕/隐身模式（避免之前的 Cookie 自动跳转 `paypal.cn`）。
2. 访问国际版注册地址：`https://www.paypal.com/c2/webapps/mpp/account-selection`。
3. 注册类型选择 Personal Account（个人账户），地区选择 China（中国）。
4. 建议使用未在 `.cn` 注册过的新邮箱（如 Gmail / Outlook，或刚解绑释放的旧邮箱）。
5. 绑定带有 Visa 或 MasterCard 标识的双币/多币种信用卡或借记卡。

# 验证

用无痕窗口访问 `paypal.com/c2` 结算弹窗，用新注册的国际版账号登录，不再出现密码错误/账号不存在/`OAuthException`，能正常完成支付授权。

# 相关链接

- [[]]
