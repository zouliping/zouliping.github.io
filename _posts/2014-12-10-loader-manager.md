---
layout: post
title: Android LoaderManager的使用
category: Android
tags: Android ContentProvider LoaderManager Loader
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 在Android中实现数据的加载，总的来说有两种方式。一种是开发者利用AsyncTask（当然，也可使用Thread + Handler的方式，仅以AsyncTask作代表）在新的线程中开启数据库，读取数据，在数据读取完毕后刷新页面，展示数据。这种方式简单易懂，实现起来容易，但有点繁琐，需要开发者手动管理SQLite。在此不讨论在UI线程进行耗时的数据加载操作。另一种方式使用Content Provider，通常情况下数据也是存储在SQLite中，本文想要详细阐述的正是此种方式。
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

### 写在前面

&emsp;&emsp;在Android中实现数据的加载，总的来说有两种方式。一种是开发者利用AsyncTask（当然，也可使用Thread + Handler的方式，仅以AsyncTask作代表）在新的线程中开启数据库，读取数据，在数据读取完毕后刷新页面，展示数据。这种方式简单易懂，实现起来容易，但有点繁琐，需要开发者手动管理SQLite。在此不讨论在UI线程进行耗时的数据加载操作。另一种方式使用Content Provider，通常情况下数据也是存储在SQLite中，本文想要详细阐述的正是此种方式。上一篇文章[SQLite那些事儿](http://echo.vars.me/android/sqlite/)中提到对于Content Provider使用的不解，在这段时间里被bug折腾后去阅读了一些相关文章后似乎有点理解了。不需要手动管理SQLite的开闭，不需要知道数据何时发生变化，封装了数据的加载等诸多好处。

### Content Provider

&emsp;&emsp;在Android中，Content Provider用于多应用间共享数据，通常使用SQLite来存储数据。所以，在使用Content Provider也得设计数据库的schema和创建数据库等操作。在本文中不详细阐述常规的Content Provider使用（常规的使用方法可以参考[Working With Content Providers](http://code.tutsplus.com/tutorials/android-sdk_content-providers--mobile-5549)），本人采用了[Android ContentProvider Generator](https://github.com/BoD/android-contentprovider-generator)。这是一个用于生成Content Provider的工具，给定实体的定义，就能生成相应的类，如下(来源于此项目的readme)：

* a ContentProvider class
* a SQLiteOpenHelper class
* a SQLiteOpenHelperCallbacks class
* one BaseColumns class per entity
* one Cursor class per entity
* one ContentValues class per entity
* one Selection class per entity

### LoaderManager

&emsp;&emsp;LoaderManager是在Android 3.0(Honeycomb)才被引入的，因此，如果需要兼容低版本，那么就需要使用support v4包中的相关类，在导包的时候需要注意，不要import错了。这留下了一个隐患，关于这个隐患在本文的后半部分中会提到。

LoaderManager是用于管理Loader的。Loader是在单独的线程中加载数据，监测数据的变化，在数据发生变化的时候重新查询数据。每个Activity或Fragment只有一个LoaderManager的实例，在其生命周期中用于管理一个或多个Loader的初始化、开启、销毁等操作。

LoaderManager并不关心数据是怎么被加载的，它只需要告诉Loader什么时候去加载数据，什么时候停止加载等就可以了，然后提供一些接口将数据反馈给实现了该接口的Activity或Fragment，让其在合适的时机执行合适的操作。LoaderManager很重要的一个功能是对数据进行持续的监测，一旦数据发生了变化，就会接收到相关的Loader的异步的加载。

### 加载Loader

&emsp;&emsp;关于Loader的加载十分简单，在Activity的OnCreate()方法或Fragment的OnActivityCreated()方法中，使用getLoaderManager().initLoader()就可以获取到LoaderManager的实例（使用support v4的话，调用getSupportLoaderManager
().initLoader()）。调用initLoader()是为了确保Loader被正确初始化，如果Loader不存在就会创建一个新的Loader，如果Loader已经存在了，将会被重用。这是通过一个独有的id来判断其是否存在。

### 实现LoaderManager.LoaderCallbacks接口

&emsp;&emsp;实现LoaderCallbacks接口的Fragment，大约长成这样子：

{% highlight Java linenos %}
public class AcademicGroupFragment extends Fragment implements LoaderCallbacks\<Cursor\> {

@Override  
public void onActivityCreated(Bundle savedInstanceState) {  
    super.onActivityCreated(savedInstanceState);
getActivity().getSupportLoaderManager().initLoader(id, null, this);
}

@Override  
public Loader\<Cursor\> onCreateLoader(int loaderId, Bundle arg1) {  
    return new CursorLoader(getActivity(), CONTENT_URI, null, null, null, CREATE_TIME);  
}  
  
@Override  
public void onLoadFinished(Loader\<Cursor\> loader, Cursor cursor) {  
    mAdapter.changeCursor(cursor);  
}  
  
@Override  
public void onLoaderReset(Loader\<Cursor\> loader) {  
    mAdapter.changeCursor(null);  
}
}
{% endhighlight %}

&emsp;&emsp;下面简要说一下各个接口的含义。

* onCreateLoader

&emsp;&emsp;这是一个工厂方法，用于返回一个新的Loader。LoaderManager会在创建一个新的Loader的时候调用此方法。

* onLoadFinished

&emsp;&emsp;在Loader加载数据结束后，onLoadFinished被自动调用。在这个方法中，一般都是进行UI的刷新操作。

* onLoaderReset

&emsp;&emsp;在Loader的数据将要被重置的时候被调用。用此方法来移除旧的不可用的数据引用。

### 使用Content Provider的优势

&emsp;&emsp;在文章的开始部分提到了一些使用Content Provider的优势。

* 封装数据的加载

&emsp;&emsp;在Activity或Fragment中封装了数据的加载，Activity或Fragment实际上并不知晓数据如何加载的。Activity或Fragment通过代理的方式将此任务交给了Loader，不需要关心Loader如何开启线程，如何进行相关的操作，减少了数据加载中可能存在的bug。

* 不需要管理SQLite

&emsp;&emsp;正如上一点中提到的，封住了数据的加载，也就不需要手动管理数据库了。犹记得当时因为数据何时关闭的问题下出现的各种crash。所以在此强调了一次。

* 自动且持续监测数据变化

&emsp;&emsp;在数据发生变化的时候，自动进行加载，只要实现onLoadFinished方法刷新UI就可以了，其他事情都不需要管，交给Loader就可以了。从网络或其他渠道获取到数据后，只需要将数据保存到数据库，其余自动发生，页面就会成功刷新。
<br><br>
&emsp;&emsp;总体来说，配合着Android ContentProvider Generator使用，整个开发过程变得十分简洁优雅。

### 遇到的问题

* onLoadFinished被两次调用

&emsp;&emsp;这个问题在[Google Group](https://groups.google.com/forum/#!topic/android-developers/vc4-RUri9k8)和[StackOverFlow](http://stackoverflow.com/questions/11293441/android-loadercallbacks-onloadfinished-called-twice)中有多个帖子，大家普遍提到的解决方式是将initLoader从OnActivityCreated移动到OnResume中。但是我尝试这么做后发现没有效果，在Fragment被创建的时候，onLoadFinished仍被两次调用。暂未发现合适的解决方案。

但在另一种情景下，此解决方案生效。某APP有Tab A、B和C，有这样一个需求，需要从网页端打开Tab B。如果该应用是打开状态下，并且是Tab A或C可见时，从网页端调起APP的时候，Tab B的onLoadFinished将不被调用。在Tab B不可见的时候，onLoaderReset方法被调用了，当上述调起操作发生后，Loader却不再加载数据。将initLoader放到OnResume中后，Tab B就会重新加载，个中原因不详。

* 切换横屏，加载不出数据

&emsp;&emsp;Fragment A中有一个ViewPager，View Pager包含多个Fragment（有Fragment B），Fragment B打开其他Activity后，切换横屏，按后退键，回到Fragment B，显示空白内容。此时，Fragment B不重新加载数据，不调用onLoadFinished（即使在onResume中initLoader也不行），暂时找不到解决方案。
<br><br>
&emsp;&emsp;在上述的两个问题中，使用的都是support v4包的Loader相关类，所以推测这是support library的bug。

### 最后

&emsp;&emsp;实践出真知。
