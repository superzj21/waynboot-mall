# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this module.

## 模块定位

`waynboot-payment-channel` 是支付渠道适配层库模块（无独立 `*Application`、无 `application.yml`），只做渠道差异收敛：把微信、支付宝、易支付的支付/退款调用封装成统一策略实现。策略接口 `PayTypeInterface` / `RefundInterface` 和选择上下文 `PayTypeContext` / `RefundContext` 定义在 `waynboot-common`（`com.wayn.common.design.strategy.*`），本模块提供对应渠道实现；渠道配置类（`WeixinPayConfig` / `AlipayConfig` / `EpayConfig`）也在 `waynboot-common`。

依赖（见 `pom.xml`）：`waynboot-common`、`weixin-java-pay`（WxJava 微信支付 SDK）、`alipay-sdk-java`。**本模块没有测试依赖，也没有测试类**——渠道真实调用只能靠联调验证。

## 包结构与职责

```
pay/       支付策略实现（6 个，实现 PayTypeInterface）
refund/    退款策略实现（6 个，实现 RefundInterface）
epay/      自研易支付 SDK 包装：Epayapi（下单/退款 HTTP 调用）、请求/响应模型、URL 编码工具
```

策略按 `getType()` 返回的 `PayTypeEnum`（`waynboot-domain-api`）注册进 Context 的 `Map<type, 策略>`，调用方按订单 `payType` 取策略，**不要按类名硬编码引用策略类**。

## 支付策略（pay 包）

| 策略类 | PayTypeEnum | 渠道调用 | 返回给前端 |
|---|---|---|---|
| `WxH5PayStrategy` | `WX_H5(1)` | V3 `unifiedOrderV3(H5)` | `mwebUrl`（微信外浏览器跳转链接） |
| `WxJsapiPayStrategy` | `WX_JSAPI(3)` | V3 `createOrderV3(JSAPI)`，需 `payer.openid` | 调起参数 VO（appId/timeStamp/nonceStr/package/signType/paySign） |
| `AliH5PayStrategy` | `ALI_H5(2)` | `AlipayTradeWapPayRequest` + `pageExecute` | HTML form 字符串 |
| `EPayWxPayStrategy` | `EPAY_WECHAT(5)` | `Epayapi.submit`（type=wxpay） | 易支付跳转 URL |
| `EPayAliPayStrategy` | `EPAY_ALI(4)` | `Epayapi.submit`（type=alipay） | 易支付跳转 URL |
| `TestPayStrategy` | `TEST(99)` | 不调真实渠道 | 本地直接置订单已支付（写 payId/payTime）+ 更新虚拟销量 |

关键约定：

- 金额单位：微信支付为分，策略内做元 ×100 转换；支付宝/易支付直接用元字符串
- 微信 H5 的 `sceneInfo.payerClientIp` 目前取的是服务器本机 IP（`NetUtil.getLocalhost()`），JSAPI 取的是请求方 IP（`reqVo.getClientIp()`）——两者不一致，修改时注意核对正确口径
- 渠道 SDK 类型不要泄漏到 domain 层：JSAPI 结果被转换为内部 `WxJsapiPayResVO`，支付宝返回 form 字符串，易支付返回 URL
- 支付失败统一抛 `BusinessException`（`ORDER_CANNOT_PAY_ERROR` / `ORDER_SUBMIT_ERROR`）

## 退款策略（refund 包）

| 策略类 | PayTypeEnum | 渠道调用 | 幂等/对账字段 |
|---|---|---|---|
| `WxJsapiRefundStrategy` / `WxH5RefundStrategy` | `WX_JSAPI` / `WX_H5` | V3 `refundV3` | `transactionId` + 退款单号 `outRefundNo` |
| `AliH5RefundStrategy` | `ALI_H5(2)` | `AlipayTradeRefundRequest.execute` | `trade_no` + 退款请求号 `out_request_no` |
| `EpayWxRefundStrategy` / `EpayAliRefundStrategy` | `EPAY_WECHAT` / `EPAY_ALI` | `Epayapi.refund` | 商户订单号 `out_trade_no` |
| `TestRefundStrategy` | `TEST(99)` | 空实现直接返回 | - |

关键约定：

- 退款单号统一由 `OrderSnGenUtil.generateRefundOrderSn()` 生成，不要自己拼
- 所有退款策略带 `@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 2000L, multiplier = 1.5))`（spring-retry），异常抛出即触发重试
- 微信退款校验返回 `status == "SUCCESS"`，支付宝校验 `response.isSuccess()`，失败抛 `ORDER_REFUND_ERROR`
- 退款是否真正入账由 `waynboot-domain-trade` 的退款流水/对账体系跟踪，策略类只负责渠道调用

## 易支付 SDK（epay 包）

自研的轻量包装（易支付非官方 SDK，接口见其公开文档）：

- `Epayapi.prepare`：POST `{apiurl}mapi.php` 下单（本项目支付策略未使用，走 submit）
- `Epayapi.submit`：拼接 `{apiurl}submit.php?query` GET 跳转链接
- `Epayapi.refund`：POST `{apiurl}api.php?act=refund`，响应解析为 `EpayResponse`，`isSuccess()`（`code != 0`）失败时抛 `CUSTOM_ERROR`
- 签名：`EpaySignUtil.sign`（`waynboot-common`）——参数按 key 排序拼接 + key，MD5 小写，`sign_type=MD5`
- `EpayPrepareRequest` 字段：pid / type / notify_url / return_url / out_trade_no / name / money / clientip
- `HttpBuildQuery`：UTF-8 URL 编码拼接查询串

## 渠道配置来源（waynboot-common，不在本模块）

```yaml
shop:
  wxpay:      # → WeixinPayConfig，WxPayServiceConfig 装配 WxPayService Bean
    app-id / app-secret / mch-id / mch-key / api-v3key
    key-path / private-cert-path / private-key-path / notify-url
  alipay:     # → AlipayConfig
    app-id / rsa-private-key / alipay-public_key / gateway / charset
    format / signtype / notify-url
  epay:       # → EpayConfig
    apiurl / pid / key / notify-url
```

配置文件是 `waynboot-common/src/main/resources/application-dev.yml` / `application-prod.yml`。支付回调不经过本模块：微信/支付宝/易支付回调由 `waynboot-domain-trade` 的 `PaymentCallbackSupport` 统一验签、解析、幂等处理（`PaymentNotifyChannelEnum` 区分渠道），策略类只管"发起"这一侧。

## 关键不变量（修改前必须遵守）

1. 策略类只做渠道适配（参数组装、SDK 调用、结果转换），**不写订单状态、库存、流水等业务逻辑**；渠道差异不要扩散到 domain-trade
2. 新增支付方式：先在 `PayTypeEnum` 加枚举值，再新增 pay/refund 各一个策略类并 `getType()` 返回该枚举；不要只实现支付不实现退款
3. 新增渠道 SDK 依赖统一在根 `pom.xml` 的 `dependencyManagement` 声明版本，本模块 pom 只引用
4. 金额口径：微信走"分"（×100），支付宝/易支付走"元"字符串；修改金额逻辑时先确认渠道单位
5. 渠道密钥配置一律走 `shop.wxpay.*` / `shop.alipay.*` / `shop.epay.*`，禁止在策略类里硬编码或自行读取 `@Value`

## 常见改动模式

- **换渠道 SDK 版本**：改根 pom 版本 → 跑 `mvn -pl waynboot-payment-channel -am test` 确认编译（本模块无测试，主要验证依赖解析与签名 API 兼容）
- **加渠道**：domain-api 加 `PayTypeEnum` → common 确认 Context 无需改动（自动收集）→ 本模块加 pay/refund 策略 → 在 `PaymentCallbackSupport` 补回调处理（若新渠道有回调）
- **联调**：渠道真实调用需配置 `shop.wxpay` / `shop.alipay` / `shop.epay` 并启动 `waynboot-mobile-api`，从 `POST /pay/prepay` 发起；测试支付（type=99）不依赖真实渠道，适合先验证订单主链路
