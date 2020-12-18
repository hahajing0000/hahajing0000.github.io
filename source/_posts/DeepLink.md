---
title: Android Deeplink
date: 2020-12-18
tags: [Deeplink]
categories: Android进阶
toc: true
---

## 什么是DeepLink？

​	deeplink是应用软件中的深度链接，一开始的时候主要是国外app运营中常用的技术，当然了，国内早已经开始使用了，一般主要是应用于app对app的广告或者社交分享之类的，简而言之，就是你在手机上面点击一个链接，可以跳转到另一个app内部的某一个页面，不是app正常打开时显示的首页内容。

[如上信息来源](http://m.xiuchuang.com/question/5486.html)

<!--more-->

## Android如何实现DeepLink

### 创建用于DeepLink拉起的Activity

配置Manifest文件中的对应Activity节点

```java
<activity android:name=".ui.DeepLinkActivity">
            <intent-filter>
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <action android:name="android.intent.action.VIEW" />
                <!-- http://myapp:8080 -->
                <data
                    android:host="myapp"
                    android:scheme="https"
                    android:path="/aa"/>
            </intent-filter>
        </activity>
```

注意Intent-filter部分

两个category分别为：DEFAULT BROWSABLE

一个action 为：VIEW

data标签如下设置：

> scheme——协议
>
> host——主机地址
>
> port——端口号
>
> path——相对路径
>
> ...

配置好后就可以获取到用于访问的地址，如上设置中访问地址即：协议://主机地址:端口号/相对地址

> https://myapp/aa

该地址可以在浏览器或者短信中分发给用户进行推广

### webview加载deeplink地址

我们用webview来加载这个地址从而拉起刚刚的activity并传入参数

建立一个测试activity其中放置webview

创建一个html页面

内容：

```html
<html>
    <head></head>
<body>
<a href="https://myapp/aa?username=xiaoming&pwd=123">DeepLink测试</a>

</body>
</html>
```

放置到assets目录下

webview进行加载该网页，因为在android5.1模拟器测试，无需考虑android7.0+的私有目录问题所以加载代码如下：

```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_browser);

        wvMain = (WebView) findViewById(R.id.wv_main);
        wvMain.loadUrl("file:///android_asset/test.html");
//        wvMain.setWebViewClient(new WebViewClient());
//        wvMain.getSettings().setJavaScriptEnabled(true);
    }
```

<font color="red">一定不要加其他设置webview的代码逻辑否则可能导致deeplink拉不起来对应activity，在没有进行其他设置前提下android才使用deeplink机制</font>

然后运行测试。不出意外应该可以拉起指定activity了。

### 获取传入参数

> https://myapp/aa?username=xiaoming&pwd=123

我们发现如上链接中结尾拼接了参数，如果在activity中获取呢，可以参见如下代码：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_deeplink);

    String host = getIntent().getData().getHost();
    String scheme = getIntent().getData().getScheme();
    String username = getIntent().getData().getQueryParameter("username");
    String pwd = getIntent().getData().getQueryParameter("pwd");
    String result = String.format("host %s | scheme %s | username %s | pwd %s", host, scheme, username, pwd);
    Log.d("123","result -> "+result);
    Toast.makeText(this,result , Toast.LENGTH_SHORT).show();
}
```

getIntent().getData().getQueryParameter即可以获取入口参数。



Android6.0以后引入了Applink，这里暂不做分析。