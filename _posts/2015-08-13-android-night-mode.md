---
layout: post
title: Android Night Mode 夜间模式实现
category: Android
tags: Android NightMode UiModeManager CarMode Theme
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 最近有个需求是夜间模式，在实现之前肯定得看看官方文档有没有相关的 tips，结果真的有。Providing Resources 这一节提到了夜间模式可以用 UiModeManager 实现，这是个令人高兴的事情。那么这就是方法一了。提到夜间模式，想起最近很火的知乎夜间模式，在知乎上搜索一番，看见了几个问题，知乎安卓客户端夜间模式切换动画是如何实现的？，这里有个匿名用户提到其是知乎夜间模式的实现者，实现的方法是截当前页面显示，改变主题，完成后渐隐截图。
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

### 准备阶段

&emsp;&emsp;最近有个需求是夜间模式，在实现之前肯定得看看官方文档有没有相关的 tips，结果真的有。[Providing Resources](http://developer.android.com/guide/topics/resources/providing-resources.html) 这一节提到了夜间模式可以用 [UiModeManager](http://developer.android.com/reference/android/app/UiModeManager.html) 实现，这是个令人高兴的事情，那么这就是方法一了。

提到夜间模式，想起最近很火的知乎夜间模式，在知乎上搜索一番，看见了几个问题，[知乎安卓客户端夜间模式切换动画是如何实现的？](http://www.zhihu.com/question/25902652)，这里有个匿名用户提到其是知乎夜间模式的实现者，实现的方法是：

> 知乎 Android 客户端的夜间模式是用 Android 本身的 Theme 实现的。动画的实现就是在 setTheme(...) 之前截图，然后在 setTheme(...) 之后把之前截好的图 Alpha 渐隐掉就好了。

&emsp;&emsp;还有一个问题（忘了收藏了）有个靠谱的回答，作者放在专栏了，[知乎Android客户端不重启Activity设置夜间模式实现分析](http://zhuanlan.zhihu.com/gracker/20077589)。这里分析的很详细，使用了 Android Studio 的 Method Trace 进行分析。那么设置主题就是方法二。

这两个方案各有利弊。

#### 夜间模式实现方法一 UiModeManager.setNightMode

#### UiModeManager.setNightMode 探索

&emsp;&emsp;仔细看看这个方案的可行性。UiModeManger 的描述是这样的，这个类可以调用系统的服务来控制 UI，它提供了 Car Mode （车载模式）和 Night Mode （夜间模式）的设置。

> This class provides access to the system uimode services. These services allow applications to control UI modes of the device. It provides functionality to disable the car mode and it gives access to the night mode settings.

&emsp;&emsp;看到这里就有一连串的疑问了，车载模式是什么，为什么和夜间模式放在一起？官方并没有说的更多，那先暂且放下这些疑问，看看具体怎么实现。

UiModeManger 提供了一个方法 setNigthMode()，但是这个方法的描述里提到了设置夜间模式的前提是开启了车载模式或者 Desk 模式。因为 UiModeManager 仅提供了方法设置车载模式，没有 Desk 模式，所以以下仅提到车载模式的设置。

> Sets the night mode. Changes to the night mode are only effective when the car or desk mode is enabled on a device.

&emsp;&emsp;看到这里就有更多的疑惑了，必须打开车载模式才能设置夜间模式，好坑的样子。既然代码写起来并不麻烦，于是乎决定尝试一下，看看到底有什么奇怪的地方。

#### UiModeManager.setNightMode 实现

&emsp;&emsp;建立若干资源文件夹，drawable-night-hdpi，drawable-night-xhdpi，drawable-night-xxhdpi，和 values-night。在 drawable 文件夹中放入与不带 night 的文件夹对应的图标，保存图标名一致。在 values-night 中建立新的 color.xml，存放夜间模式所需的颜色设置。使用这种方式的好处就是通过建立带 -night 的资源文件夹，就可以通过改变设置，使其读取指定目录下的资源，减少很多原有代码的更改。

例如，（颜色值仅供参考）

colors.xml in values

{% highlight XML linenos %}
<resources>
    <color name="night_mode_color">#DD7321</color>
    <color name="night_mode_dark_color">#DD4814</color>
    <color name="background_color">#FFFFFF</color>
</resources>
{% endhighlight %}

colors.xml in values-night

{% highlight XML linenos %}
<resources>
    <color name="night_mode_color">#7D4112</color>
    <color name="night_mode_dark_color">#7D4112</color>
    <color name="background_color">#1F1F1F</color>
</resources>
{% endhighlight %}

styles.xml in values

{% highlight XML linenos %}
<style name="Theme.Test" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="colorPrimary">@color/night_mode_color </item>
    <item name="colorPrimaryDark">@color/night_mode_dark_color </item>
    <item name="colorAccent">@color/night_mode_color </item>
    <item name="android:windowBackground">@color/background_color</item>
</style>
{% endhighlight %}

完成这些配置后，在需要切换的地方，加上如下代码

{% highlight JAVA linenos %}
UiModeManager uiManager = (UiModeManager) getSystemService(Context.UI_MODE_SERVICE);
if (isNightMode) {
    uiManager.enableCarMode(0);
    uiManager.setNightMode(UiModeManager.MODE_NIGHT_YES);
} else {
    uiManager.disableCarMode(0);
    uiManager.setNightMode(UiModeManager.MODE_NIGHT_NO);
}
{% endhighlight %}

#### UiModeManager.setNightMode 结果

&emsp;&emsp;运行一下，看看效果。这个效果真的好赞，没有闪屏，就是切换了颜色和 icon 等。

![image]({{ site.url }}/assets/images/setnightmode.gif)

但是有个烦人的通知常驻通知栏。

![image]({{ site.url }}/assets/images/skitch.png)

这就是车载模式导致的。因为此方案的实现简单，效果优雅，实在不愿意直接抛弃它，首先想到的是不显示这个通知。因此我在 StackOverFlow 上提了一个[问题](http://stackoverflow.com/questions/31934503/to-implement-android-night-mode-using-uimodemanager-and-enable-car-mode-but-sh)，大意就是能不能想办法不显示通知，如果不行的话这是否说明了使用 `UiModeManager.setNightMode` 这个方式是不可行的，还有没有其他更好的办法之类的。但是，并没有人能解答我的疑惑。

经过一番思考，我意识到不显示通知这个想法本身就是错误的，既然允许了车载模式，那不显示通知的意义何在，仅是为了让用户体会不到车载模式。而且，在 Android 5.0 以上，开启车载模式，手机会发生一系列的变化。在 Nexus 7 Android 5.1.1 下，打开任何应用后，点击 Home 键，会出现一个蓝色的界面，上面写着

> Launch Android Auto

> Look for the Android Auto button on your car's display to start

![image]({{ site.url }}/assets/images/carmode.png)

这说明为了实现夜间模式开启车载模式就是不合理的。但是这就更加令我疑惑了，为什么 Google 要将车载模式和夜间模式绑定在一起。也许这就是为了实现车载模式而存在的。

在这样的困境下，只能阅读 Android 源码，试图根据它的实现原理来实现一套不带车载模式的夜间模式。追踪到 /services/core/java/com/android/server/UiModeManagerService.java 这个文件的 `setNightMode()` 方法

{% highlight JAVA linenos %}
@Override
public void setNightMode(int mode) {
	switch (mode) {
		case UiModeManager.MODE_NIGHT_NO:
		case UiModeManager.MODE_NIGHT_YES:
		case UiModeManager.MODE_NIGHT_AUTO:
			break;
		default:
			throw new IllegalArgumentException("Unknown mode: " + mode);
	}
	final long ident = Binder.clearCallingIdentity();
	try {
		synchronized (mLock) {
		if (isDoingNightModeLocked() && mNightMode != mode) {
		Settings.Secure.putInt(getContext().getContentResolver(),Settings.Secure.UI_NIGHT_MODE, mode);
			mNightMode = mode;
			updateLocked(0, 0);
		}
	}
	} finally {
		Binder.restoreCallingIdentity(ident);
	}
}
{% endhighlight %}

&emsp;&emsp;关键在 `updateLocked()` 中（代码有点长），发了一堆的广播通知 View 改变设置，并且调用 `adjustStatusBarCarModeLocked()` 这个方法来显示车载模式通知。

{% highlight JAVA linenos %}
    void updateLocked(int enableFlags, int disableFlags) {
        String action = null;
        String oldAction = null;
        if (mLastBroadcastState == Intent.EXTRA_DOCK_STATE_CAR) {
            adjustStatusBarCarModeLocked();
            oldAction = UiModeManager.ACTION_EXIT_CAR_MODE;
        } else if (isDeskDockState(mLastBroadcastState)) {
            oldAction = UiModeManager.ACTION_EXIT_DESK_MODE;
        }
        if (mCarModeEnabled) {
            if (mLastBroadcastState != Intent.EXTRA_DOCK_STATE_CAR) {
                adjustStatusBarCarModeLocked();
                if (oldAction != null) {
                    getContext().sendBroadcastAsUser(new Intent(oldAction), UserHandle.ALL);
                }
                mLastBroadcastState = Intent.EXTRA_DOCK_STATE_CAR;
                action = UiModeManager.ACTION_ENTER_CAR_MODE;
            }
        } 
        // ... 省略部分源码
        if (action != null) {
            // Send the ordered broadcast; the result receiver will receive after all
            // broadcasts have been sent. If any broadcast receiver changes the result
            // code from the initial value of RESULT_OK, then the result receiver will
            // not launch the corresponding dock application. This gives apps a chance
            // to override the behavior and stay in their app even when the device is
            // placed into a dock.
            Intent intent = new Intent(action);
            intent.putExtra("enableFlags", enableFlags);
            intent.putExtra("disableFlags", disableFlags);
            getContext().sendOrderedBroadcastAsUser(intent, UserHandle.CURRENT, null,
                    mResultReceiver, null, Activity.RESULT_OK, null, null);
            // Attempting to make this transition a little more clean, we are going
            // to hold off on doing a configuration change until we have finished
            // the broadcast and started the home activity.
            mHoldingConfiguration = true;
            updateConfigurationLocked();
        } 
        // ... 省略部分源码
        // keep screen on when charging and in car mode
        boolean keepScreenOn = mCharging &&
                ((mCarModeEnabled && mCarModeKeepsScreenOn &&
                  (mCarModeEnableFlags & UiModeManager.ENABLE_CAR_MODE_ALLOW_SLEEP) == 0) ||
                 (mCurUiMode == Configuration.UI_MODE_TYPE_DESK && mDeskModeKeepsScreenOn));
        if (keepScreenOn != mWakeLock.isHeld()) {
            if (keepScreenOn) {
                mWakeLock.acquire();
            } else {
                mWakeLock.release();
            }
        }
    }
{% endhighlight %}

&emsp;&emsp;因为没有找到接收广播的类，且时间紧迫，这个根据源码来实现的方案暂时搁置了。

### 夜间模式实现方法二 Change Theme

#### Change Theme 探索

&emsp;&emsp;这个方式较为普遍，通过定义两套 Theme，在切换模式的时候，保存 Theme Id，重建 Activity。在 Activity 中，在 `setContentView()` 之前，读取保存的 Theme id，再 `setTeme(Theme Id)` 一下即可。

#### Change Theme 实现

colors.xml in values，定义两套颜色。

{% highlight XML linenos %}
<resources>
    <color name="night_mode_color">#40DD7321</color>
    <color name="night_mode_dark_color">#40DD4814</color>
    <color name="background_color">#1F1F1F</color>
    
    <color name="night_mode_color_night">#40DD7321</color>
    <color name="night_mode_dark_color_night">#40DD4814</color>
    <color name="background_color_night">#1F1F1F</color>
</resources>
{% endhighlight %}

styles.xml in values，定义两套主题。

{% highlight XML linenos %}
<style name="Theme.Test.Light" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="colorPrimary">@color/night_mode_color </item>
    <item name="colorPrimaryDark">@color/night_mode_dark_color </item>
    <item name="colorAccent">@color/night_mode_color </item>
    <item name="android:windowBackground">@color/background_color</item>
</style>

<style name="Theme.Test.Dark" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="colorPrimary">@color/night_mode_color_night </item>
    <item name="colorPrimaryDark">@color/night_mode_dark_color_night </item>
    <item name="colorAccent">@color/normal_color</item>
    <item name="android:windowBackground">@color/background_color_night </item>
</style>
{% endhighlight %}

在点击切换的时候，保存 Theme Id，重启页面。

{% highlight JAVA linenos %}
if (isNightMode) {
    mThemeId = R.style.Theme_Idxyer_NoActionBar_Dark;
}else {
    mThemeId = R.style.Theme_Idxyer_NoActionBar;
}
saveThemeId(); // means save theme in SharedPreferences
MainActivity.this.recreate();
{% endhighlight %}

在 Activity 的 onCreate() 方法中，设置主题。

{% highlight JAVA linenos %}
@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	int themeId = getThemeId(); // means get theme id from SharedPreferences
	setTheme(themeId);
	setContentView(R.layout.activity_main);
}
{% endhighlight %}

#### Change Theme 结果

&emsp;&emsp;结果显而易见，出现了闪屏，因为调用 `Activity.recreate()` 整个 Activity 重建了。

![image]({{ site.url }}/assets/images/changetheme.gif)

而且这个方案改起来特别繁琐，要改的东西特别多，几乎每个 Layout 都得改动，得建立两套的主题，怎么看都不像是一个好的解决方案。我上面提到的实现看起来并不麻烦，主要是仅修改 ActionBar 的颜色和背景色，其他部分没有修改，所以显得修改也不是很多的样子，实际操作起来就不是这么回事儿了。

类似这个方案的实现可以参考 [MultipleTheme](https://github.com/dersoncheng/MultipleTheme) 这个开源库，号称不需要重启页面即可实现夜间模式的切换，但需要将切换页面的控件都替换为它的控件。它实现 Change Theme 这种方案的方法比较标准，可以参考。如果切换夜间模式是在设置页面里，可以考虑该方法。这样做的麻烦之处在于，需要将每个控件都继承一遍，想想都很头疼。Android 缺乏类似 iOS 的 category 机制，其可以横向扩展一个类，不需要通过继承一个类方式来为其添加新的方法。

据说知乎就是用 Change Theme 这个方案，只是截图盖住了闪屏的过程。

### 是否还有其他方案？

&emsp;&emsp;理想的状态是，开发者仅需很少的改动现有代码，通过建立一些资源文件的配置来满足夜间模式的需要。切换夜间模式的时候，以某种机制通知系统来读取新的资源，改变自身的状态。

所幸，在 Gist 中找到了一个 [NightModeHelper](https://gist.github.com/slightfoot/c508cdc8828a478572e0)，这大概算是一个折中的方式，改动很少的代码，但是闪屏。虽然看起来的效果和第二种方案相似，但是代码修改量将会减少很多很多，而且可扩展性好，将来如果有一天，Google 提供了 UiModeManger 的设置夜间模式和车载模式的分离，切换成本低。

代码写起来非常简单，仅需要在 Activity.onCreate() 的 `super.onCreate(savedInstanceState);` 之后加上

{% highlight JAVA linenos %}
mNightModeHelper = new NightModeHelper(this, R.style.AppTheme_Light);
{% endhighlight %}

&emsp;&emsp;并且和第一种方式类似的添加 -night 资源文件夹，即可实现切换。

实现效果如下：

![image]({{ site.url }}/assets/images/nightmodehelper.gif)

因为使用了 `Activity.recreate()`，所以仍然会出现闪屏的问题。

### 小结

&emsp;&emsp;本文中提到了三种实现夜间模式的方案：

* UiModeManager.setNightMode()
* Change Theme
* Use NightModeHelper

&emsp;&emsp;这三个方案中，第一个是绝对不可取的，开启车载模式的代价太大。第二个和第三个看起来实现效果一样，但是从代码层面来看，第三个要优于第二个。闪屏的问题还需后续再研究。

本文的 Demo 可以在 https://github.com/zouliping/AndroidNightMode 找到。这个 Demo 的三种方式会相互影响，所以请一次仅设置一种方式，在切回正常模式后再尝试下一种方案。