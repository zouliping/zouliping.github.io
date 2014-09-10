---
layout: post
title: Android Search Dialog记录搜索历史
category: Android
tags: [Android]
comments: true
image:
  feature: texture-feature-05.jpg
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

在[上一篇博客](http://echo.vars.me/android/2014/08/14/search/)的基础之上，使用Search Dialog来记录搜索的历史。记录用户的搜索记录可以提高用户搜索的效率，例如在用户搜索过`test`后，用户输入了`t`，将用户搜索过的以`t`开头的词`test`显示出来。

### 创建搜索Actiivty

这在上篇博文中，已经创建了，与它类似即可。

### 创建Content Provider

创建的Content Provider要继承自SearchRecentSuggestionsProvider。

{% highlight Java linenos %}
public class SearchSuggestionProvider extends SearchRecentSuggestionsProvider {
    public final static String AUTHORY = "com.example.Test.SearchSuggestionProvider";
    public final static int MODE = DATABASE_MODE_QUERIES;

    public SearchSuggestionProvider(){
        setupSuggestions(AUTHORY, MODE);
    }
}
{% endhighlight %}

`AUTHORITY`是一个唯一的字符串，一般使用该类的完整的名字（包名 + 类名）。`DATABASE MODE`一般是`DATABASE_MODE_QUERIES`或`DATABASE_MODE_QUERIES | DATABASE_MODE_2LINES`。`DATABASE_MODE_2LINES`为suggestion表增加了一列，用于给每个建议提供一行文本。

### 声明Content Provider

在AndroidManifest.xml中，声明上一步创建的Content Provider。

{% highlight XML linenos %}
<provider android:authorities="com.example.Test.SearchSuggestionProvider" android:name=".SearchSuggestionProvider"></provider>
{% endhighlight %}

### 修改搜索配置

在res/xml/searchable.xml中，增加Content Provider的配置。其中的`android:searchSuggestAuthority`必须和Content Provider中的`AUTHORITY`保持一致。`android:searchSuggestSelection`是一个空格加一个问号。

{% highlight XML linenos %}
<searchable xmlns:android="http://schemas.android.com/apk/res/android"
            android:label="@string/app_name"
            android:hint="@string/hint"
            android:searchSuggestAuthority="com.example.Test.SearchSuggestionProvider"
            android:searchSuggestSelection=" ?"
        >
</searchable>
{% endhighlight %}

### 保存用户的搜索记录

在上篇博文中的搜索Activity，即MyActivity中，在用户每次执行搜索时，传递来的搜索关键字，保存在Content Provider。`saveRecentQuery()`的第二个参数，需和`DATABASE_MODE_2LINES`配合使用，否则传入`null`。

{% highlight Java linenos %}
Intent intent = getIntent();
if (Intent.ACTION_SEARCH.equals(intent.getAction())) {
	String query = intent.getStringExtra(SearchManager.QUERY);
	doSearch(query);

	SearchRecentSuggestions srs = new SearchRecentSuggestions(this, SearchSuggestionProvider.AUTHORY, SearchSuggestionProvider.MODE);
	srs.saveRecentQuery(query, null);
}
{% endhighlight %}

### 清除用户搜索记录

为了保护用户的隐私，一般需要提供一个清除搜索记录的功能。清除记录仅需调用`clearHistory()`方法即可。一般代码如下：


{% highlight Java linenos %}
SearchRecentSuggestions srs = new SearchRecentSuggestions(this, SearchSuggestionProvider.AUTHORY, SearchSuggestionProvider.MODE);
srs.clearHistory();
{% endhighlight %}
