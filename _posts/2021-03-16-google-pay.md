--- 
layout: post
title: Google Pay 踩坑之路
category: Android
tags: Android, GooglePay
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 首次接入 Google Pay 走了不少弯路，遇到很多奇奇怪怪的问题，在此将接入 Google Pay 的历程做个总结与回顾。下文主要围绕两个方面，接入流程和遇到的问题。接入流程主要参考 Google 官方文档，结合服务端校验订单。遇到的问题主要包含商品获取异常、无法完成购买、消费状态异常等等。
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section>


首次接入 Google Pay 走了不少弯路，遇到很多奇奇怪怪的问题，在此将接入 Google Pay 的历程做个总结与回顾。下文主要围绕两个方面，接入流程和遇到的问题。接入流程主要参考 Google 官方文档，结合服务端校验订单。遇到的问题主要包含商品获取异常、无法完成购买、消费状态异常等等。

## 接入流程

关于如何接入 Google Pay，在官方文档有着详尽的解释，[https://developer.android.google.cn/google/play/billing/integrate#pending](https://developer.android.google.cn/google/play/billing/integrate#pending)，按照此文档可以了解客户端需要如何使用各个 API 。总体流程见下图，

![https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/7858076991/5939/33fb/1c55/1dcdf1546a466e6e6d82174460385654.png](https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/7858076991/5939/33fb/1c55/1dcdf1546a466e6e6d82174460385654.png)

总体接入流程

1. 创建服务端订单，获取 order id 和 Google sku id
2. 创建 BillingClient，连接 Google Play startConnection
3. 调用 querySkuDetailsAsync，传入 sku id，得到 SkuDetails
4. 调用 launchBillingFlow，调起 Google Pay 界面，传入加密后的 order id
5. 监听 PurchasesUpdatedListener，BillingClient.BillingResponseCode.OK 时，对已购买的 purchase（it.purchaseState == Purchase.PurchaseState.PURCHASED）进行服务端的校验
6. 服务端校验成功后，调用 consumeAsync，消耗商品，使得商品可以再次购买

按照上述的流程可以完成大致一次订单的购买，但大概率会遇到种种困难，下文会一一阐述我们是如何解决的。

## 遇到的问题

遇到了一下几个问题：
1. 运营后台的商品和 Google 的商品是如何关联的？
2. 服务端订单和 Google 订单是如何关联的？
3. 获取商品空
4. 无法购买您要买的商品
5. 消费状态异常

最后小结一下测试的前置条件。


### 运营后台的商品和 Google 的商品的关联

后台配置的商品和 Google 的商品需要关联起来，这需要服务端在创建 Google 商品的时候将其 sku id 记录下来。一般来说，通常会接入多种支付渠道，服务端需要维护这样的对映关系。商品展示的时候，可以使用服务端的 sku id，但是在调用 Google Pay 的 querySkuDetailsAsync 的时候，需要传入 Google 的 sku id。因此，流程大概是这样，获取服务端的商品列表，用户选择付款渠道后，拿到对应渠道的 sku id，通过 Google 获取对应的 Google 商品详情，进行后续的购买流程。

### 服务端订单和 Google 订单的关联

同样的，服务端订单和 Google 的订单也需要关联起来。订单的关联会更加复杂一点，服务端需要通过 Google 的回调来得到用户订单的支付成功与否，Google 的订单中并不包含我们的订单信息。因此需要在客户端申请付款的时候，将订单 id 透传给 Google，而 Google 提供的 API 中并没有额外的透传字段，因此我们在 launchBillingFlow 的时候通过 setObfuscatedAccountId() 将订单 id 加密后传递给 Google。服务端在获取到该笔的订单的时候，通过我们约定的加密的方式来解密得到真实的服务端订单 id。

为什么不能使用和商品一样的方式，由客户端在调用验证订单接口的时候将订单 id 携带上。在大多数时候，这种方式是可以行得通的。但是如果客户端由于某种原因没有收到购买回执，服务端依据 Google 的回调来决定是否对该笔订单进行履约的时候，需要知道对应的服务端订单 id。因此为了保险起见，将订单 id 加密后透传给 Google，由服务端去 Google 查询订单的时候，也可以直接获取到服务端的订单 id。

### 获取商品空

在测试的过程中，发现谷歌商品创建之后，在客户端发起购买的时候 querySkuDetailsAsync 获取到的商品为空。首先需要检查 App 是否已经发布内测版本，仅上传 Apk 是不够的，需要发布态。打包 apk 的时候需要按照 Play 的要求，设定 target sdk 和 64 位 so。然后信息中心中包含多个步骤都完成后，发布内测，使得应用呈现发布状态。但出现了一个奇怪的现象，一切设置都 ok 的情况下依然获取不到商品，但在间隔一天左右后，重试可以成功获取到商品。因此推测是创建商品到获取商品存在一定的延时。目前没有官方文档或者官方解释，只能做此推断。

### 无法购买您要买的商品

![https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/7920127256/c47a/534a/85dd/467b3d2d416f47694dbfbbfb890ebadd.png](https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/7920127256/c47a/534a/85dd/467b3d2d416f47694dbfbbfb890ebadd.png)

出现无法购买您要买的商品提示，第一，检查是否在内测用户列表和许可测试列表。第二，需要登录内测的账号，点击接受测试邀请链接（Google 后台的中可以找到这个链接，具体路径参照下文），接受邀请。第二点尤为关键，很容易遗漏。第三，签名是否对应。

许可测试的添加

![https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/7920237451/208e/f04a/e421/fb653b2d9a3e42a3e8bb67937f2f38d1.png](https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/7920237451/208e/f04a/e421/fb653b2d9a3e42a3e8bb67937f2f38d1.png)

内测用户的添加

![https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/7920260657/de71/2631/221c/8324c5d9a44b5a944af50d6a83462fa6.png](https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/7920260657/de71/2631/221c/8324c5d9a44b5a944af50d6a83462fa6.png)

### 消费状态异常

对于一次性商品，分为消耗型和非消耗型。（消耗型的定义在 [https://developer.android.com/google/play/billing?hl=zh-cn](https://developer.android.com/google/play/billing?hl=zh-cn)）。

消耗型的商品，在服务端校验订单成功后，需要调用 consumeAsync() 表明已经向用户授予权利，使得这个商品可以被再次购买。在官方文档中有提到，调用 consumeAsync 实际上也是隐式的调用了 acknowledgePurchase。

非消耗型的商品则是使用 acknowledgePurchase() 对商品进行确认。

根据 Google 的文档，在客户端购买完成后，需要调用服务端对 Google 的订单进行校验，根据 Google 的 API（[https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.products](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.products)），获取到的购买记录中包含 acknowledgementState consumptionState。对于这两个状态，官方文档没有很明确的解释。

acknowledgementState 状态表明商品是不是被确认， consumptionState 是商品是否被消费。在服务端判断一个订单是否正常需要被履约，使用了 acknowledgementState == 1 来判断订单是否被确认过，但是出现了一个奇怪的状态，acknowledgementState = 1，但是 consumptionState = 0，即订单已确认，订单未消费，这个状态暂时无法解释。

### 测试前置条件

总的来说，要完成一次支付的测试，需要准备以下的几个条件：

1. 测试机的 Google 服务已安装
2. Google PlayStore 检测 IP 非中国区（Google PlayStore 设置国家，非中国区，修改地区需要绑定支付方式），经多次试验，不修改国家也可以成功购买
3. 测试账号已添加到谷歌后台，包括内测用户添加和许可测试添加，两者都需要添加
4. 接受测试邀请，找到内测链接，点击接受内测邀请，这一步很重要，容易遗漏。Google 后台的内部测试 - 测试用户数量 - 测试人员参与测试的方式 - 复制链接。
5. 发布内测版本，应用发布状态
6. 安装包签名和包名与提交到 Google 后台的签名一致。网上看到有说版本号也要求一致或大于的，但是实操中发现不需要保持一致。