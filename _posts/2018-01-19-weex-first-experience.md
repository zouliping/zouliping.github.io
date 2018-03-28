--- 
layout: post
title: Weex 初体验
category: Weex
tags: Android Weex 初体验
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 最近在 Vue、HTML、CSS 几乎小白的情况下，接到了一个 Weex 页面需求，于是乎开始了 Weex 的学习，遇到了一些坑，在这里做一些总结。Weex 是阿里开源的一个「使用 Web 开发体验来开发高性能原生应用的框架」，号称「Write Once, Run Everywhere」。
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

### 背景

最近在 Vue、HTML、CSS 几乎小白的情况下，接到了一个 Weex 页面需求，于是乎开始了 Weex 的学习，遇到了一些坑，在这里做一些总结。

Weex 是阿里开源的一个「使用 Web 开发体验来开发高性能原生应用的框架」，号称「Write Once, Run Everywhere」。

### Weex vs React Native

提到 Weex，就不得不和 React Native 进行一番对比。鉴于没有真正的写过 RN，只是在当年尝试过 Demo，只能从较为浅层的方面来做一些比较。

* 框架选择方面，Weex 是使用 Vue.js，React Native 是使用 React。Vue.js 的学习成本较低。
* 社区活跃度方面，React Native 社区活跃度高，在 github 的 star 数也远多于 Weex。
* 调试方面，都可以使用 Chrome 来调试 JS 代码。也都支持 Hot Reload。
* 性能方面，List 的实现，React Native 是基于 ScrollView 的，Weex 是基于 RecyclerView 的，显而易见，RN  在这方面的性能不如 Weex。
* 打包方面，React Native 只能将基础 JS 和业务 JS 打成一个包，bundle 较大，而 Weex 可以单独打业务 JS，bundle 较小。
* React Native 支持两端（Adnroid、iOS），Weex 是三端（Android、iOS、Web）。
* React Native 是「Learn Once, Write Everywhere」，Weex 是「Write Once，Run Everywhere」。

### 踩坑记录

在第一次写 Weex 的情况下，遇到了还多坑，其中有部分是因为知识水平所限不了解 Weex 支持哪些属性不支持哪些属性，在这里作个记录，如有不对，望指正。

* `border` 不支持组合写法，只能将 `border-width`，`border-radius`，`border-color` 这些属性都分开来写。
* `flex-grow` 属性不支持，后来发现，Weex 中的 `flex` 属性似乎是对应 `flex-grow`，而 `flex-shrink` 不支持。这个在写圣杯布局的时候发现，没有 `flex-grow` 的情况下，只能多嵌套一层 `flex` 布局，最后将中间的元素设置了 `flex: 1;` 实现。 Weex 中只支持 `flex` 布局，结果很多属性不支持。
* background-image 不支持设置图片，但是可以写渐变背景，`background-image: linear-gradient(to bottom, #EC63E9, #633BDF);`。
* `border` 不能直接作用在 `img` 标签上，只能再多包裹一层 `div`。现在写的 Weex 页面 View 的嵌套层次较深，这个问题后续需要考虑，是否是因为不熟悉语法，嵌套过多的 `div` 导致。
* `v-bind:class` 不生效，有一个根据标签的类型来显示不同标签的需求，如果 Weex 能给支持 `v-bind:class` 写起来比较优雅，现在是在 HTML 使用了 `:class="[getTagStyle(item.tagType)]"`，在 JS 中通过类型返回对应的 CSS class 名的方法实现。
* `flex row` 默认是没有占满行高的，这一点比较坑，在多个元素中有若干可能不显示的时候，没有居中才发现的，后来只能加上 `height: 100%;`。
* `for item in list` 不生效，只能用 `for index in list`。
* 在 HTML 中写错变量，没有看到报错，很坑，Debug 找不到原因，通过读代码来解决，不清楚是否是忽略了报错。
* 可以直接写 `if (list.length)`，在 JS 里， 数字 0 是 false，其他数字是 true。
* 网络请求的回调可以使用箭头函数，就不需要 `let that = this` 了。
* 在同一个 `List` 中，`cell` 不能根据 `type` 混排，只能写成一个 `cell`。
* 直接更新数组元素中不存在的属性，不会更新视图，需要使用主动调用 `this.$set()` 刷新视图。这一点是由一个 “玄学问题” 引发的，在页面上加了一个 `test` 标签，在网络请求 List 的部分属性成功的时候，有一个更新标签的操作，删除这个标签后发现页面没有更新，保留标签的情况下 List 是有变化的。实际上是由于 `test` 标签引发了页面更新，进而引发部分属性顺带被更新了。
* 在 Weex 中实现一个标题 + 一个标签，当标签过长的时候，优先完全显示标签，标题截断，想不到完美的解决方案。目前的需求，标签是固定的长度，所以给标签设置了一个最大宽度来实现的。如果标签不固定，则不知怎么实现。
* 在 Android 中的 .9 图显示有问题，拉伸的不太正确。目前的解决方案是将 .9 图放在 native 中，但需要注意的是在 native 中需要手动引用一下，否则在打 release 包的时候会因为没有引用而被排除。
* load more 的实现方式是使用 loading 圈的 `appear` 和 `disappear` 来触发，这样实现有一个问题，如果请求的数量较少，假设只有一条数据的情况，不足以占满屏幕，load more 仅会被触发一次，因为没有 `disappear`。
* 在与 iOS 使用同一份代码中，也遇到了一些问题，并不如官方所说的「写一次，处处运行」那么轻松。Android 与 iOS 引用 native 图片的路径写法不一致。点击左上角返回按钮，Android 中可以写 `navigator.close();`，在 iOS 不生效，只能改成 `navigator.pop();`。诸如此类的问题。
* 在 Android 和 iOS 上 text 上下的 padding 是明显不同的，在实现一个标签由文案 + padding 撑开画上一个圆弧背景的时候，就需要对 Android 和 iOS 写上不同的 padding 了。而且 Android 是支持 padding 组合起来写的，类似这样，`padding: 2wx 7wx;`，而 iOS 是不支持的，`padding-left: 7wx; padding-right: 7wx;`。
* iOS 上如果想要不超出布局边界，则需要设置 `overflow: hidden;`。

### 后续

后续还需要补上 JS 相关的缺漏和更多的实践。

### 参考

* [React Native 官网](https://facebook.github.io/react-native/)
* [Weex 教程](https://weex.apache.org/guide/)
* [How to Become a React Native Developer in 2018](https://hackernoon.com/how-to-become-a-react-native-developer-in-2018-d9bc85e1d91f)