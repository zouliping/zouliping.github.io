--- 
layout: post
title: 从一个小 bug 回顾 View 的测量
category: Android
tags: Android, View
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 从一个上线前的流式布局 FlowLayout 小 bug 入手回顾一下 View 的 measure。
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

### Bug 的前因后果

在发版前夕，发现若干一排二商品的高度显示异常，多出了一些空白的空间。仔细查看发现，少了关键属性的显示，关键属性是放在一个流式布局 FlowLayout 中，一行能展示下几个就显示几个。现在出现了这样一个奇怪的现象，一个属性的时候显示不了，但是两个及以上就能正常显示。现在基本确定了是一个属性显示不了的问题。

![image](https://haitao.nos.netease.com/80c8ec06-4248-47db-a2c0-bec3d8eda3e6_1080_2280.png)

#### 问题排查流程

1. 首先断点查看添加显示属性的代码，代码逻辑很简单，当属性列表不为空的时候，遍历列表，创建一个 TextView 设置文案，添加到 FlowLayout 中。这个过程是正常的，所有的 View 都正常的添加到 FlowLayout 中，但却没有正常显示出来。

2. 接着将所有设置这个 FlowLayout 的显示与隐藏的地方都加上断点，怀疑可能是这个 View 被设置成不可见了。跟着逻辑走一遍，怀疑设置 View 的 GONE 的逻辑被错误调用，但事实是没有异常的调用。那么问题就一定出在 FlowLayout 内部逻辑，FlowLayout 的最新的提交已是一年前了，大概率是由于使用方操作触发了隐藏 BUG。

3. 再往下查看属性自定义 View 及相关 XML 的提交记录，发现一个疑点，这个属性 View 的宽度从 `MATCH_PARENT` 改成了 `WRAP_CONTENT`，这个修改很可疑，可能导致了问题的出现。宽度的改变是因为视觉要求在某些情况下属性需要居中显示。

4. 先回滚修改重新编译打包，确认问题所在，结果果然是这行代码导致的。鉴于马上要发版，立刻将此代码回滚合到主分支，并和视觉沟通此版本暂无法实现居中功能。

#### Bug 的由来

接下来，我们分析一下这个 bug 的产生原因。

在 FlowLayout 的 onMeasure 方法中，发现走到了如下的代码逻辑中，导致它被隐藏不显示了。这段代码是 FlowLayout 为了实现第一个组件过长时隐藏的逻辑。

![image](https://haitao.nos.netease.com/63fe3bd8-c979-4aff-82d0-e527513ad861_1588_242.png)

设置 `WRAP_CONTENT` 和 `MATCH_PARENT` 两次测试对比结果如下表所示：

![image](https://haitao.nos.netease.com/af778c0f-b780-4037-934f-133b3db50b77_724_493.jpeg)

也就是说在设置 `WRAP_CONTENT` 的情况下，触发了 FlowLayout 的隐藏 bug，将 FlowLayout 隐藏了起来。

至此就可以解决这个问题了，解决方案有以下几个：

1. 说服视觉，不实现居中功能
2. 不需要第一个太长不显示的功能，规避此条件
3. 将这个判断语句中的 childWidth > sizeWidth

接下来老生常谈一下 View 的测量过程，来解释为什么在设置了 match_parent 和 wrap_content 的情况下，measure 的结果不同。

### View 测量过程

#### MeasureSpec 概念

View 的测量离不开 MeasureSpec。

> A MeasureSpec encapsulates the layout requirements passed from parent to child. Each MeasureSpec represents a requirement for either the width or the height. A MeasureSpec is comprised of a size and a mode.

MeasureSpec 封装了父 View 的 layout 需求，代表了 View 的规格尺寸。它有三种模式：

* UNSPECIFIED: 父 View 对子 View 的宽高没有限制。

* EXACTLY: 父 View 为子 View 决定了特定的大小，Match_Parent 和确定的大小是使用此模式。

* AT_MOST: 子 View 可以使用任意的大小，但不能超过父 View 设定的大小，Wrap_Content 适用。

一个 View 的 MeasureSpec 是由 Mode 和 Size 组成，用一个 32 位的 Int 表示，Int 的高两位表示 Mode 测量模式，剩余 30 位表示 Size，某测量模式下的大小。

#### MeasureSpec 如何生成

MeasureSpec 是由父 View 的 MeasureSpec 和自身的 LayoutParams 共同决定的。父 View 通过调用 getChildMeasureSpec 方法来生成子 View 的 MeasureSpec。

* 子 View 设置具体的宽高时，忽略父 View 的 MeasureSpec，直接使用子 View 设定的大小，mode 设置为 EXACTLY

* 子 View 设置为 `MATCH_PARENT` 时，
	* 当父 View 为 EXACTLY 或 AT_MOST，子 View 的宽高即为父 View 的宽高，mode 与父 View 保持一致。
	* 当父 View 为 UNSPECIFIED，这时候不限制子 View 的宽高，一般是用在 ScrollView 中。子 View mode 被设成 UNSPECIFIED ，size 根据 sUseZeroUnspecifiedMeasureSpec 设为 0 或其父 View 大小。

* 子 View 设置为 `WRAP_CONTENT` 时，
	* 当父 View 为 EXACTLY 或 AT_MOST，子 View 的宽高即为父 View 的宽高，mode 均为 AT_MOST。
	* 当父 View 为 UNSPECIFIED，子 View mode 被设成 UNSPECIFIED ，size 根据 sUseZeroUnspecifiedMeasureSpec 设为 0 或其父 View 大小。

#### UNSPECIFIED 是怎么回事

UNSPECIFIED 是指父 View 不会对子 View 的大小进行限制。如果设置 AT_MOST，子 View 最大也不能超出父 View 的范围。在 ScrollView 中，子 View 的大小很有可能会超出 ScrollView 本身，通过滚动可以展示超出的部分。因此在可滚动的容器中，子 View 设置的是 WRAP_CONTENT, 在滚动的时候会被强制设置成 UNSPECIFIED。

#### measure 的过程

说回 measure 的流程，首先被提到的一定是 ViewRootImpl 的 performTraversals，View 绘制的起点。

从绘制起来开始的方法调用栈：
① ViewRootImpl.performTraversals -> ② ViewRootImpl.performMeasure -> ③ View.measure -> ④ View.onMeasure

ViewRootImpl.performTraversals 会依次调用 performMeasure，performLayout，performDraw 来执行 View 的 measure，layout，draw 三个过程。调用 performMeasure 前，先调用了 getRootMeasureSpec，根据 DecorView 的 LayoutParams 来获取 DecorView 的 MeasureSpec。然后调用 measure 方法。

ViewGroup 中没有实现 measure 方法，实际调用的是 View 的 measure 方法。measure 方法主要是对查找该 View 应该显示多大，真实的 measure 操作是在 onMeasure 方法中。一般的 ViewGroupView （FrameLayout，LinearLayout 等）都会实现 onMeasure 方法来实现自己的布局方式。

以 LinearLayout 的 onMeasure 分析测量的过程。在 LinearLayout 的 onMeasure 中根据设置的排列方式，纵向和横向调用不同的方法，measureVertical 和 measureHorizontal。下面以 measureVertical 为例分析，该方法很长，不直接贴源码了，仅以文字简要说明。

1. 在第一个 for 循环中，对子 View 进行一次遍历测量，会忽略当 LinearLayout 为 `MATCH_PARENT`时，设置了高度 0， weight 大于 0 的子 View，这些 View 的高度在此时无法确定，需要通过其他子 View 的剩余高度来按比例分配。

2. 根据 1 中计算的子 View 得到一个 mTotalLength。

3. 对设置了 measureWithLargestChild true，并且 MeasureSpec 为 AT_MOST 或者 UNSPECIFIED，重新计算 mTotalLength。最后得到 LinearLayout 的高度。

4. 根据已得到的高度，对所有子 View 重新确定其大小。对于设置 weight 的 View 需要重新测量一次。

5. 最后得到 LinearLayout 的测量宽高，setMeasuredDimension 保存起来。

#### onMeasure 何时被调用

onMeasure 由父 View 在计算布局的时候来调用，会根据不同的 ViewGroup 调用多次。

#### onMeasure 到底被调用几次

onMeasure 到底被调用几次，是由其父布局来决定的。如果父 View 没法在一次测量中确定其子 View 的大小，就会进行两次 measure，例如 RelativeLayout 和 设置了 weight 的 LinearLayout。













