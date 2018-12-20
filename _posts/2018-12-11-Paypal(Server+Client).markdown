---
layout:     post
title:      "关于Paypal支付"
subtitle:   "服务端和客户端"
# date:       2018-11-29 19:00:00
author:     "Jackcao"
header-img: "img/post-bg-20181211.jpg"
tags:
    - iOS
    - Objective-C
    - Paypal
    - BrainTree
    - Server
    - Service
--- 

<br>   
# 前言
***这几天都在调研服务端和客户端的支付行为,因为要使用Paypal用于跨境的支付***

<br>
# 正文

## 选择平台

打开[Paypal开发者](https://developer.paypal.com/docs/integration/mobile/ios-integration-guide/)网站,映入眼帘的是

![post-paypal-warning](/img/in-post/in-post-2018/post-paypal-warning.png)

什么意思呢？
> Paypal的意思是，如果有任何高级功能(如直接信用卡流程)不被批准用于新的集成。如果你在SDK被弃用之前已经集成，你仍然可以使用，但是PayPal不鼓励任何新的集成。Paypal建议接入BrainTree和Express checkout。

什么是[BrainTree](https://www.braintreepayments.com)?
> Paypal旗下的聚合支付方式,包含很多种支付方式(Paypal、信用卡、Apple Pay...)。

什么是[Express checkout](https://developer.paypal.com/docs/checkout/)?
> Webview中的Paypal支付方式，订单支付在客户端完成。

**我对比了一下这三种方式,最后决定使用原生的SDK,原因如下**
1. 后端已经为Web提前接入了原生的SDK，如果要切换到BrainTree，需要增加额外的代码。
2. BrainTree比较重，相对于原生的SDK。
3. Express checkout界面风格和我们的App风格差距太大，需要自定义。
4. 我们需要原生的体验,基于HTML的操作行为，体验不太好。
5. 虽然官方不推荐使用SDK，但我觉得原因主要是推广BrainTree。
6. 我们目前不使用Paypal信用卡支付。

**虽然原生SDK的文档说明很模糊，甚至可以说过时，但是有了上面的理由，并不能阻止我使用它。**

## 接入

### 支付行为选择

**Paypal的原生SDK中有两种支付行为**
* Single Payment。
* Future Payments。

**由于Single Payment的支付行为在客户端,所以被我们弃用。Future Payments的支付行为只需要用户在客户端完成授权，之后的支付行为完全由后端代理。所以最后我们选择了后者。**

### 支付流程

**需要去官网[注册](https://www.paypal.com/signin?returnUri=https%3A%2F%2Fdeveloper.paypal.com%2Fdeveloper%2Fapplications)App和沙箱账号，申请步骤和流程很清晰。主要是支付流程比较复杂，支付流程总结如下**

 → 客户端申请Future Payments权限<br>
 → 用户登录(用户手动登录Paypal)<br>
 → 用户同意协议，并授权<br>
 → 授权成功后客户端回调得到授权码<br>
 → 客户端上传授权码给后端<br>
 → 后端拿到授权码去申请token<br>
 → 后端得到token后，使用token创建订单<br>
 → 创建订单成功后得到Paypal生成的Authorization Id(字段名id)<br>
 → 使用id去捕获订单<br>
 → 扣款成功，完成支付<br>
 → 通知客户端支付结果<br>

**客户端文档 [iOS开发文档](https://github.com/paypal/PayPal-iOS-SDK/blob/master/docs/future_payments_mobile.md)、[Android开发文档](https://github.com/paypal/PayPal-Android-SDK/blob/master/docs/future_payments_mobile.md)，移动端平台的接入步骤大致相同。**
**服务端文档 [Curl示例](https://github.com/paypal/PayPal-Android-SDK/blob/master/docs/future_payments_server.md)**