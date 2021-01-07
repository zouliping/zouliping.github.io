### MotionLayout 是什么

MotionLayout 是 ConstraintLayout 的子类，可以用于布局的状态转换添加动画效果。MotionLayout 完全是声明式的，使用 XML 描述转换。需要注意的是 MotionLayout 的所有直接子 View 都需赋予一个 id，否则会报 All children of ConstraintLayout must have ids to use ConstraintSet 错误。

MotionLayout 需要链接到一个 MotionScene 文件，使用 app:layoutDescription 将 MotionLayout 和 MotionScene 链接在一起。MotionScene 用于描述两个场景的过渡动画，存放于 res/xml 目录。布局和运动的描述分开描述，MotionLayout 引用单独的 MotionScene。MotionScene 分为三个部分：StateSet、ConstraintSet 和 Transition。StateSet 用于描述状态，是可选的。ConstraintSet 用于定义一个场景的约束集，用来描述限制的位置。Transition 用于描述两个状态或者 ConstraintSet 间的变换。更多请参考[官方文档](https://developer.android.com/training/constraint-layout/motionlayout?hl=zh-cn)。下文会展开具体的用法和写法。

![](https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/5499960560/5780/337f/83c2/8fd388f0d0b366909a2c42200ab8a01a.png)

### 使用 MotionLayout 写一个折叠动画

#### 展开折叠动效展示

在项目中展开折叠的效果如下：

![](https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/5500581100/223e/4cc7/67e3/e6e1b1de649b38616e7ba979a63304ea.gif)


#### 现有方案

通常情况下，在 RecyclerView 中展开折叠动画有几种实现方式，

展开 View 的 VISIBLE 和 GONE 切换，动画生硬，效果不佳。可以对展开 View 进行 translate 动画或者改变高度的动画
增加和移除 item，涉及 RecyclerView 元素的变动

在非 RecyclerView 中，和 RecyclerView 类似，也是采用第一种方案。通常代码是这样的：

```
private fun expandAnimation() {
        val valueAnimator = ValueAnimator()
        valueAnimator.apply {
            setIntValues(0, animatedHeight)
            interpolator = AccelerateInterpolator()
            addUpdateListener {
                animationView?.layoutParams?.height = it.animatedValue as Int?
                animationView?.requestLayout()
            }
            doOnEnd {
                setFolded(!folded)
            }
        }
        valueAnimator.start()
    }
```

现在我们可以采用 MotionLayout 来实现顺滑的展开折叠动画。

#### MotionLayout 的实现

##### 基础的布局文件

```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.motion.widget.MotionLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/motion_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp"
    app:layoutDescription="@xml/view_item_scene">

    <ImageView
        android:id="@+id/avatar_view"
        android:layout_width="60dp"
        android:layout_height="60dp"
        android:src="@drawable/cat_hug" />

    <TextView
        android:id="@+id/title_view"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/title"
        app:layout_constraintStart_toEndOf="@id/avatar_view"
        app:layout_constraintTop_toTopOf="@id/avatar_view" />

    <TextView
        android:id="@+id/desc_view"
        android:layout_width="0dp"
        android:layout_height="100dp"
        android:paddingTop="5dp"
        android:text="@string/desc"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/avatar_view" />

    <TextView
        android:id="@+id/fold_view"
        android:layout_width="wrap_content"
        android:layout_height="40dp"
        android:paddingStart="20dp"
        android:paddingEnd="40dp"
        android:text="@string/collapse"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/desc_view" />

</androidx.constraintlayout.motion.widget.MotionLayout>
```
布局文件很简单，只有一些基础的元素：头像、昵称、简介、展开收起按钮。这里需要注意的一点是在布局文件中生命的 ConstraintSet 的优先级是低于 MotionScene 中设定的。

##### MotionScene 文件

```
<?xml version="1.0" encoding="utf-8"?>
<MotionScene xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <ConstraintSet android:id="@+id/start">
        <Constraint
            android:id="@+id/desc_view"
            android:layout_width="0dp"
            android:layout_height="100dp"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/avatar_view" />
    </ConstraintSet>

    <ConstraintSet android:id="@+id/end">
        <Constraint
            android:id="@id/desc_view"
            android:layout_width="0dp"
            android:layout_height="0dp"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/avatar_view" />
    </ConstraintSet>

    <Transition
        app:constraintSetEnd="@id/end"
        app:constraintSetStart="@+id/start"
        app:duration="500"
        app:motionInterpolator="easeInOut">
        <OnClick
            app:clickAction="toggle"
            app:targetId="@id/fold_view" />
    </Transition>
</MotionScene>
```

在 MotionScene 中定义了两个 ConstraintSet，分别对应了动画的开始和结束状态，结束的时候将用户简介 View 收起。触发条件是 Transition 中的 OnClick 设定的 targetId，展示收起 View 点击的时候触发操作。clickAction 设置的是 toggle，在开始场景和结束场景间切换。

简单介绍一下 Transition 的属性：

* constraintSetStart：设定开始状态的布局 id
* constraintSetEnd：设定结束状态的布局 id
* duration：动画时长
* motionInterpolator：过渡动画的差值器。
	* easeInOut 缓入缓出
	* easeIn 缓入
	* easeOut 缓出
	* linear 线性
	* bounce 弹簧
	
OnClick 中的属性：

* targetId：触发过渡动画的 view 的 id
* clickAction：点击执行的动作
	* toggle 开始和结束状态循环切换
	* transitionToEnd 过渡到结束状态
	* transitionToStart 过渡到开始状态
	* jumpToEnd 无过渡动画到结束状态
	* jumpToStart 无过渡动画到开始状态

除了 OnClick 还有 OnSwipe，设置拖拽的动作。也可以不在此设置，在代码里显式的触发。Transition 可以设置多个 OnClick，都能生效，但是 OnSwipe 只有最后设置的一个生效。

##### 监听过渡的变化

```
motionLayout.setTransitionListener(object : MotionLayout.TransitionListener {
            override fun onTransitionTrigger(p0: MotionLayout?, p1: Int, p2: Boolean, p3: Float) {
            }

            override fun onTransitionStarted(p0: MotionLayout?, p1: Int, p2: Int) {
            }

            override fun onTransitionChange(p0: MotionLayout?, p1: Int, p2: Int, p3: Float) {
            }

            override fun onTransitionCompleted(p0: MotionLayout?, p1: Int) {
                folded = !folded
                if (folded) {
                    foldView.setText(R.string.fold)
                } else {
                    foldView.setText(R.string.collapse)
                }
            }

        })
```
可以通过设置 MotionLayout.TransitionListener 得到过渡状态的回调，例如可以在过渡结束后，改变按钮的状态。

##### 成果

通过上述的代码，可以实现和文中开头提到的效果一致的展开收起动画。

![](https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/5500625383/4a8d/a844/b03e/288ba0071016eccb3f142be44dac6044.gif)

##### RecyclerView 实现只展开一项

在 RecyclerView 中的展开和折叠通常需要仅展开一项，其他项自动关闭。在 RecyclerView 实现的时候，不在 XML 中处理点击，将点击的处理放到 adapter 中，通过 MotionLayout 的 progress 可以得知当前 View 的状态。

* 当 MotionLayout 处于开始状态，即 progress 等于 0.0，执行过渡到结束态的方法 transitionToEnd()
* 当 MotionLayout 处于结束状态，即 progress 等于 1.0，执行过去到开始态的方法 transitionToStart()
* 结合 RecyclerView 的 payload，局部刷新，折叠其他的 item
* 在 onBindView 的时候需要初始化 MotionLayout 的状态，否则复用的时候可能会发生错乱

#### 总结

使用 MotionLayout 可以很轻松实现展开折叠动画，比常规方法简单，且体验较佳。在两个场景间的切换，通常可以使用 MotionLayout。文中展示了较为简单的一种场景，通过该场景熟悉 MotionLayout 后，可以实现更多更好的动画。使用 MotionLayout 可以做出很多酷炫的动画，可参考 [https://developer.android.com/training/constraint-layout/motionlayout/examples?hl=zh-cn](https://developer.android.com/training/constraint-layout/motionlayout/examples?hl=zh-cn) 和
[https://mp.weixin.qq.com/s/dF32SX61qTzV-hTC2Rx8EA](https://mp.weixin.qq.com/s/dF32SX61qTzV-hTC2Rx8EA)。

### 参考

* [MotionLayout 官方文档](https://developer.android.com/training/constraint-layout/motionlayout?hl=zh-cn)
* [Introduction to MotionLayout](https://medium.com/google-developers/introduction-to-motionlayout-part-i-29208674b10d)
* [MotionLayout 基础教程](https://juejin.cn/post/6844903816249212935)
* [MotionLayout 示例](https://developer.android.com/training/constraint-layout/motionlayout/examples?hl=zh-cn) 
* [用 MotionLayout 实现这些不可思议的效果](https://mp.weixin.qq.com/s/dF32SX61qTzV-hTC2Rx8EA)