--- 
layout: post
title: 一次诡异的不能点击的事件
category: Android
tags: Android
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 很久以前的某一天，忽然收到反馈，搜索结果的列表中有些项目不可点击，并且这些项目是没有规律的。看到这个反馈的第一反应是这不可能，每个 Item 的点击事件都是统一处理的，不可能出现某些能点某些不能点的问题。抱着怀疑的态度，准备去 Review 代码，一步一步排查，尝试定位问题所在。
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

很久以前的某一天，忽然收到反馈，搜索结果的列表中有些项目不可点击，并且这些项目是没有规律的。看到这个反馈的第一反应是这不可能，每个 Item 的点击事件都是统一处理的，不可能出现某些能点某些不能点的问题。抱着怀疑的态度，准备去 Review 代码，一步一步排查，尝试定位问题所在。

根据现象有以下几个猜测：

1. 点击事件没有传递到对应的 View 上
2. View 被回收后 ClickListener 没有被设置上
3. ListView 的某些隐藏坑

### 定位问题

首先，简单 Review 一遍搜索结果列表的呈现和点击的代码，暂时没有看出什么问题来，初步怀疑可能是与 View 的复用和回收有关。

不能点击，极有可能是点击事件被其他 View 消费了，没有传递到指定的 View 上。根据这个思路，从 View 的事件分发流程着手。

![image](https://haitao.nos.netease.com/c2154959-2236-4be9-95b5-2050cd3bcbe8_850_746.jpg)
（事件分发流程图，来源于网络（http://www.gcssloop.com/customview/dispatch-touchevent-theory））

恰巧在同事的手机上复现了这个问题，按照如果 Bug 可以被复现一定可以被修复的原则。顺着事件分发流程，逐步断点调试。在 View 的 onTouchEvent 方法中找到了一些线索。

```
public boolean onTouchEvent(MotionEvent event) {
    // 省略部分代码      
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                    // 省略部分代码      
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }

                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }

                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // If the post failed, unpress right now
                        mUnsetPressedState.run();
                    }

                    removeTapCallback();
                }
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
                // 省略部分代码
                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    setPressed(true, x, y);
                    checkForLongClick(0, x, y);
                }
                break;

        // 省略部分代码    
        }

        return true;
    }

    return false;
}

```

在 onTouchEvent 的 ACTION_UP 中找到了线索，PerformClick 的 post 比较关键。

```
/**
 * <p>Causes the Runnable to be added to the message queue.
 * The runnable will be run on the user interface thread.</p>
 *
 * @param action The Runnable that will be executed.
 *
 * @return Returns true if the Runnable was successfully placed in to the
 *         message queue.  Returns false on failure, usually because the
 *         looper processing the message queue is exiting.
 *
 * @see #postDelayed
 * @see #removeCallbacks
 */
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }

    // Postpone the runnable until we know on which thread it needs to run.
    // Assume that the runnable will be successfully placed after attach.
    getRunQueue().post(action);
    return true;
}
```

AttachInfo 是一个 View Attach 到他的父 Window 的时候保存的一些信息合集。在 post 方法中会拿到这个 View 的 attachInfo，当 View 的 AttachInfo 不为空的时候，使用 AttachInfo 的 Handler 发送了一条消息到 UI 线程，将事件添加到了队列当中。如果事件被成功抛出，这次点击事件可以视为完成，如果没有成功，在 onTouchEvent 中会立刻执行 performClick。在这种情况下，视为正常的点击流程完成。

当 View 的 AttachInfo 为空的时候，就会拿到 RunQueue 将此条消息发送出去，直到该 View 被成功 Attach 上被执行，在这里会默认这次事件已经完成，不会再走后续的流程了。我们这次的问题极有可能出现在此，不能点击的 View 的 AttachInfo 为空。

为什么会没有 AttachInfo 呢？有两种可能，一是视图树还未创建，二是这个 View 没有被添加到视图树。这得结合 AttachInfo 的赋值来看。

AttachInfo 是什么时候被赋值的呢？在 View 的 dispatchAttachedToWindow 方法中，拿到 AttachInfo，并且执行刚才 post 到 RunQueue 中的消息。也就是说如果点击的时候 AttachInfo 为空，dispatchAttachedToWindow 一直没有被执行到，那么这次的点击事件将永远不会响应。这就是我们的问题所在。

```
/**
 * @param info the {@link android.view.View.AttachInfo} to associated with
 *        this view
 */
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
    if (mOverlay != null) {
        mOverlay.getOverlayView().dispatchAttachedToWindow(info, visibility);
    }
    mWindowAttachCount++;
    // We will need to evaluate the drawable state at least once.
    mPrivateFlags |= PFLAG_DRAWABLE_STATE_DIRTY;
    if (mFloatingTreeObserver != null) {
        info.mTreeObserver.merge(mFloatingTreeObserver);
        mFloatingTreeObserver = null;
    }

    registerPendingFrameMetricsObservers();

    if ((mPrivateFlags&PFLAG_SCROLL_CONTAINER) != 0) {
        mAttachInfo.mScrollContainers.add(this);
        mPrivateFlags |= PFLAG_SCROLL_CONTAINER_ADDED;
    }
    // Transfer all pending runnables.
    if (mRunQueue != null) {
        mRunQueue.executeActions(info.mHandler);
        mRunQueue = null;
    }
    performCollectViewAttributes(mAttachInfo, visibility);
    onAttachedToWindow();
    // 省略部分代码
}
```

### 问题本质的原因

在搜索结果页面滚动后，部分 View 会被回收，这些 View 将变成「悬空」的 View，在当前的视图树中不存在了。虽然你能看见它正常的显示，但是它并不隶属于当前的视图树，这听起来似乎有些玄学，但却真实的发生了。

回收后的 View 却没有被 Attach 上，这就得从 ListView 的回收机制说起。ListView 的回收机制中关键的类是 RecycleBin，RecycleBin 用于保存无用的可以被回收的 View，避免下次 layout 的时候一直创建新的 View。

RecycleBin 中有两个关键的成员，mActiveViews 即为当前在屏幕上可见的 View，mScrapViews 为滑出屏幕被回收的 View。

接着分析 View 的复用逻辑，通过方法的调用链 makeAndAddView() -> obtainView() -> getScrapView()，找到 getScrapView()。getScrapView() 方法中通过 item type 来找到可以复用的 scrap view。

```
/**
 * @return A view from the ScrapViews collection. These are unordered.
 */
View getScrapView(int position) {
    final int whichScrap = mAdapter.getItemViewType(position);
    if (whichScrap < 0) {
        return null;
    }
    if (mViewTypeCount == 1) {
        return retrieveFromScrap(mCurrentScrap, position);
    } else if (whichScrap < mScrapViews.length) {
        return retrieveFromScrap(mScrapViews[whichScrap], position);
    }
    return null;
}
```

在 Adapter 中通过 getItemViewType() 和 getViewTypeCount() 就可以拿到当前 View 的类型和 ListView 有几种类型的 View。另外一个关键的位置在 AbListView.setViewTypeCount()，这里说明了 ListView 的回收是通过 itemType 来区分不同的缓存，即每一个 type 都会启用一个单独的缓存。

```
public void setViewTypeCount(int viewTypeCount) {
    if (viewTypeCount < 1) {
        throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
    }
    //noinspection unchecked
    ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
    for (int i = 0; i < viewTypeCount; i++) {
        scrapViews[i] = new ArrayList<View>();
    }
    mViewTypeCount = viewTypeCount;
    mCurrentScrap = scrapViews[0];
    mScrapViews = scrapViews;
}
```

真相逐步被揭开，前人在这里挖了一个坑，搜索列表中的某几种类型（商品、活动等）的 View 的类型是通过具体数据模型的 type 字段来决定的。也就是说对于 Adapter 来说，仅知道商品类型，而不知道活动或者其他的类型，在复用的时候，会将商品、活动等类型的 View 放在一起进行回收利用，十分容易出现错落并导致 View 没有被 Attach 上。

在 ListView 的 setupChild() 方法中，如果满足 if 中的条件，将不会调用 addViewInLayout() 也就是说不会调用 dispatchAttachedToWindow()，就会导致我们接收不到点击事件。这就和上一个 Part 中的内容串联起来。

```
private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
    boolean selected, boolean isAttachedToWindow) {
    // 省略部分代码
    if ((isAttachedToWindow && !p.forceAdd) || (p.recycledHeaderFooter
            && p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
        attachViewToParent(child, flowDown ? -1 : 0, p);

        // If the view was previously attached for a different position,
        // then manually jump the drawables.
        if (isAttachedToWindow
                && (((AbsListView.LayoutParams) child.getLayoutParams()).scrappedFromPosition)
                        != position) {
            child.jumpDrawablesToCurrentState();
        }
    } else {
        p.forceAdd = false;
        if (p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
            p.recycledHeaderFooter = true;
        }
        addViewInLayout(child, flowDown ? -1 : 0, p, true);
        // add view in layout will reset the RTL properties. We have to re-resolve them
        child.resolveRtlPropertiesIfNeeded();
    }
}
```

### 解决方法

基于上述分析，有以下几种解决方案：

1. 改变一个 Type 对应多个 View 的状况，实现一对一
2. 在 Adapter.getView() 的反射获取 mAttachInfo 进行判断，如果为 null 的时候不进行复用

方案一可以比较彻底的解决此问题，但是历史代码复杂，代码修改量大，出错的可能性也变得很大。根据最小代码修改原则，采用了方案二，在 getView 的时候获取 mAttachInfo。

```
/**
 * 获取 view 的 mAttachInfo
 */
private Object getAttachInfo(View view) {
    try {
        Field field = View.class.getDeclaredField("mAttachInfo");
        field.setAccessible(true);
        return field.get(view);
    } catch (Throwable e) {
        return null;
    }
}
```

因为在 View 中 mAttachInfo 是 private 的，所以通过反射的方式拿到它。在 getView 的时候判断一下，如果没有 mAttachInfo 的时候重新创建一个 View。

```
if (view == null || !(view instanceof ItemView) || getAttachInfo(view) == null) {
	view = new ItemView(getContext);
}
```

至此就解决了这个诡异的不能点击的问题。

### 其他发现

在查看 View 的源码的时候，发现在 Android 6.0 与 Android 7.0 的一点差异。在 6.0 中，attachInfo 为空的时候，调用的是将事件发送到 ViewRootIml.getRunQueue() 中，事件被抛到了 ViewRootImpl 中，当 ViewRootImpl 的 performTraversals() 执行到的时候，就会使得我们的点击事件得到响应。而在 Android 7.0 中使用的是当前 View 的 handler，当前 View 如果一直没有机会被 attach，该事件将一直无法得到响应。也就是说在 Android 7.0 以下的手机上，点击事件是正常的。

![image](https://haitao.nos.netease.com/d4e5373d-f41b-4a4b-9488-07996be9da9b_2880_744.png)

### 总结

当时选择了比较 trick 的方式来解决问题，这毕竟不是长久之计。因此，在此后的一段时间内，对这部分代码进行了重构，使用 RecyclerView 来重写这部分逻辑。彻底改变了通过空数据设置类型，拿到对应的类型获取真实数据，再映射到不同的 View 来展现的现象。新方案中将不同类型的数据映射到不同的 ViewHolder 上，使得代码的可维护性大大的提高了，新增类型变得十分简单，可以更好的支撑业务的发展。

### 参考链接

* [安卓自定义View进阶-事件分发机制原理](http://www.gcssloop.com/customview/dispatch-touchevent-theory)

* [Android ListView工作原理完全解析，带你从源码的角度彻底理解](https://blog.csdn.net/guolin_blog/article/details/44996879)