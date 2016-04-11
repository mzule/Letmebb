---
title: 通过 URL 打开 Activity
date: 2016-04-11 21:28:21
tags: ActivityRouter
---

为每个 Activity 绑定一个 url 可以方便的让第三方 app 直接打开这些 Activity。也可以方便在 app 内部进行页面跳转，解耦。

![](http://7sbl54.com1.z0.glb.clouddn.com/blog_QQ20160411-22.png)

<!-- more -->

## 背景

举一个常见的案例，假设我们有个产品 A，产品 A 包含 h5 网页端和客户端，当用户在手机打开我们的 h5 网页端的时候，我们会期望如果用户手机安装了我们的客户端，则直接打开 app，否则停留在网页端浏览。

这是一个很常见的需求，但是实现需要 h5 和 Android 的配合，本文会先说下原理，然后单独描述 Android 端需要做的事情，最后会给一个链接说明 h5 的工作。

## 原理

Android 端先给 Activity 绑定一个 url ，比如说是 `myapp://main`. 

用户访问 `http://myapp.com` 网页时，h5 尝试访问 `myapp://main`，如果用户安装了客户端，则会打开相应的 Activity，否则会继续留在 h5 浏览网页。

那么，如何给 Activity 绑定一个 url 是在 Android 端的关键。

## Android 实现

创建一个空的 ViewActivity.

``` java
public class ViewActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    }
}
```

在 `AndroidManifest.xml` 里面注册 ViewActivity，包含一个 action 为  `android.intent.action.VIEW` 的 `intent-filter`

``` xml
<activity
    android:name=".ViewActivity"
    android:theme="@android:style/Theme.NoDisplay">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" />
    </intent-filter>
</activity>
```

这样，`ViewActivity` 就具备了接收 `myapp` 协议的 `android.intent.action.VIEW` 事件的能力。比如说，如果某个 app 执行了下面的这段逻辑，我们的 ViewActivity 就可能被打开。

```java
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setData(Uri.parse("myapp://dosomething"));
startActivity(intent);
```
刚才之所以说 "可能被打开"，是为了严谨。因为完全可能用户手机上，还有另外一个 app 也声明了一样协议的 intent-filter. 这时候系统就会给出一个弹出框，让用户选择一个期望的应用来打开该地址。

![](http://7sbl54.com1.z0.glb.clouddn.com/blog_chooser.png)

既然入口找到了，接下来就简单了，无非就是实现一下 ViewActivity 处理跳转逻辑，比如这样。

``` java
public class ViewActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Uri uri = getIntent().getData();
        String url = uri.toString();
        if ("myapp://main".equals(url)) {
            startActivity(new Intent(ViewActivity.this, MainActivity.class));
        } else if ("myapp://user".equals(url)) {
            startActivity(new Intent(ViewActivity.this, UserActivity.class));
        }
        finish();
    }
}
```
但是上面这种，这是一个理想的情况，因为现实情况会复杂的多。比如说会遇到传参问题，打开一个 UserActivity 可能都是需要指定一个 userId 的。不过再怎么复杂，无非就是对一个 url 的解析。

下面会介绍一个我写的开源项目，省掉了解析 url 的麻烦，下面会基于这个库来说明。

## 库 

这个库叫 [ActivityRouter](https://github.com/mzule/ActivityRouter)，通过给 Activity 添加注解来绑定 url. 首先我们要把库添加到项目里面来, 需要修改两个 build.gradle 文件。

项目根目录的 build.gradle

``` groovy
buildscript {
  dependencies {
    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.7'
  }
}
```

app 项目的 build.gradle

``` groovy
apply plugin: 'android-apt'

dependencies {
    compile 'com.github.mzule.activityrouter:activityrouter:1.1.1'
    apt 'com.github.mzule.activityrouter:compiler:1.1.1'
}
```

这样，等项目 sync 完 ActivityRouter 就已经被集成在项目里面了。

此外还需要改一个配置，之前在 `AndroidManifest.xml` 上注册的 ViewActivity 现在要换成 ActivityRouter 里面自带的 Activity，这样 ActivityRouter 才有机会处理相关的 Intent 事件。修改完的 `AndroidManifest.xml` 如下：

``` xml
<activity
    android:name="com.github.mzule.activityrouter.router.RouterActivity"
    android:theme="@android:style/Theme.NoDisplay">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" />
    </intent-filter>
</activity>
```

接下来就是去修改需要绑定 url 的 Activity，添加注解即可，一个典型的实现如下：

``` java
@Router("main")
public class MainActivity extends Activity {
}
```

一个需要参数的 Activity 可以这样声明：

``` java
@Router("user/:userId") // :userId 代表参数名为 userId
public class UserActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        String userId = getIntent().getStringExtra("userId"); // 获取参数 userId
    }
}
```
也可以给参数指定类型，比如说常见的 id 为 long 型.

``` java
@Router(value = "user/:userId", longExtra = "userId")
```

这样我们就可以通过 `myapp://main` 来访问 MainActivity, `myapp://user/89757` 来访问 UserActivity 并且 userId = 89757 了。

## H5 实现

如上文所说网页自动跳转到 app 需要 h5 配合，由于 h5 已经超越了 Android 的范畴，这边就直接贴个链接。

[http://t.cn/RqMTBDZ](http://t.cn/RqMTBDZ)

[http://t.cn/RzOQWGU](http://t.cn/RzOQWGU)

## App 内应用场景

除了第三方 app 跳转场景外，还可以在 app 内部 Activity 跳转时采用 Router 来实现，比如在 Android 端和后端约定好页面对应的 url，后端在发送 push 的时候，就可以发送特定的 url，客户端只需要处理打开 url 即可，可以有效减少 push 通知的适配工作。相关 API 如下：

``` java
Routers.open(Context, String)
Routers.open(Context, Uri)
```

## 结语

哈哈，感谢你看到这里。