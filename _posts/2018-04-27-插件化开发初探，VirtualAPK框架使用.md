---
layout:     post
title:      插件化开发初探，VirtualAPK框架使用
subtitle:   插件化开发初探，通过滴滴的VirtualAPK框架实现插件化的功能
date:       2018-05-07
author:     smiling
header-img: img/2018-05-07/44752772_xl.jpg
catalog: true
tags:
     -VirtualAPK
     -插件化
---

## 目录

- [什么是插件化？](#1)
- [有哪些插件化框架？](#2)
- [接入VirtualAPK](#3)
- [基本的使用](#4)

<span id="1"></span>

## 什么是插件化？

> 插件是什么？插件就是对现有的程序添加额外的功能组件。这样的好处是可以对原有程序进行扩展，以Android Studio为例，我们可以安装FindBugs插件，这样IDE就具备了更强大的静态代码检查功能。插件化就是把APP的各个功能弄成插件，这样就可以随时更新、卸载，可以看到插件化使我们的程序更加灵活。


<span id="2"></span>

## 有哪些插件化框架？

要实现插件化其实便不难，现在各大厂商都推出了自己的插件实现框架。

| 特性 | DynamicLoadApk | DynamicAPK |  Small   | DroidPlugin | VirtualAPK |
|--- | :------: | :-------: | :------: | :-------: | :------: |
| 支持四大组件	| 只支持Activity	| 只支持Activity	| 只支持Activity	| 全支持	| 全支持 |
| 组件无需在宿主manifest中预注册	| √	| ×	| √	| √	| √ |
| 插件可以依赖宿主	| √	| √	| √	| ×	| √ |
| 支持PendingIntent	| ×	| ×	| ×	| √	| √ |
| Android特性支持	| 大部分	| 大部分	| 大部分	| 几乎全部	| 几乎全部 |
| 兼容性适配	一般	| 一般	| 中等	| 高	| 高 |
| 插件构建	| 无	| 部署aapt	| Gradle插件	| 无	| Gradle插件 |


### 这里我们简单介绍下的滴滴的VirtualAPK框架

#### VirtualAPK的特性

VirtualAPK是滴滴出行自研的一款优秀的插件化框架，主要有如下几个特性。

##### 功能完备

- 支持几乎所有的Android特性；
- 四大组件方面

##### 四大组件均不需要在宿主manifest中预注册，每个组件都有完整的生命周期。

1. Activity：支持显示和隐式调用，支持Activity的theme和LaunchMode，支持透明主题；
2. Service：支持显示和隐式调用，支持Service的start、stop、bind和unbind，并支持跨进程bind插件中的Service；
3. Receiver：支持静态注册和动态注册的Receiver；
4. ContentProvider：支持provider的所有操作，包括CRUD和call方法等，支持跨进程访问插件中的Provider。
5. 自定义View：支持自定义View，支持自定义属性和style，支持动画；
6. PendingIntent：支持PendingIntent以及和其相关的Alarm、Notification和AppWidget；
7. 支持插件Application以及插件manifest中的meta-data；
8. 支持插件中的so。

##### 优秀的兼容性

- 兼容市面上几乎所有的Android手机，这一点已经在滴滴出行客户端中得到验证；
- 资源方面适配小米、Vivo、Nubia等，对未知机型采用自适应适配方案；
- 极少的Binder Hook，目前仅仅hook了两个Binder：AMS和IContentProvider，hook过程做了充分的兼容性适配；
- 插件运行逻辑和宿主隔离，确保框架的任何问题都不会影响宿主的正常运行。

##### 入侵性极低

- 插件开发等同于原生开发，四大组件无需继承特定的基类；
- 精简的插件包，插件可以依赖宿主中的代码和资源，也可以不依赖；
- 插件的构建过程简单，通过Gradle插件来完成插件的构建，整个过程对开发者透明。

<span id="3"></span>

## 接入VirtualAPK

#### 主项目

在主项目的根目录中`build.gradle`的依赖项添加
{% highlight groovy %}
dependencies {
    classpath 'com.didi.virtualapk:gradle:0.9.7'
}
{% endhighlight %}
在主项目的app目录中`build.gradle`的依赖项添加插件与依赖
{% highlight groovy %}
apply plugin: 'com.didi.virtualapk.host'
compile 'com.didi.virtualapk:core:0.9.1'
{% endhighlight %}
在主项目的`application`中的`attachBaseContext()`方法中初始化VirtualAPK
{% highlight java %}
@Override
protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    PluginManager.getInstance(base).init();
}
{% endhighlight %}

#### 插件项目

在插件项目的根目录中`build.gradle`的依赖项添加
{% highlight groovy %}
dependencies {
    classpath 'com.didi.virtualapk:gradle:0.9.7'
}
{% endhighlight %}
在插件项目的app目录中`build.gradle`的依赖项添加插件与配置
{% highlight groovy %}
apply plugin: 'com.didi.virtualapk.plugin'
compile 'com.didi.virtualapk:core:0.9.1' //如果插件不需要访问宿主的内容，插件项目可以不依赖这个
//配置VirtualAPK。请记住在build.gradle的末尾放置以下几行代码
virtualApk {
    packageId = 0x6f             // 插件资源表中的packageId，需要确保不同插件有不同的packageId.
    targetHost='source/host/app' // 宿主工程application模块的路径，插件的构建需要依赖这个路径,绝对路径或者相对路径都可以。
    applyHostMapping = true      // 默认为true，如果插件有引用宿主的类，那么这个选项可以使得插件和宿主保持混淆一致
}
{% endhighlight %}
解释一下上面3个参数的作用

- `packageId`用于定义每个插件的资源id，多个插件间的资源Id前缀要不同，避免资源合并时产生冲突
- `targetHost`指明宿主工程的应用模块，插件编译时需要获取宿主的一些信息，比如mapping文件、依赖的SDK版本信息、R资源文件，一定不能填错，否则在编译插件时会提示找不到宿主工程。
- `applyHostMapping`表示插件是否开启apply mapping功能。当宿主开启混淆时，一般情况下插件就要开启applyHostMapping功能。因为宿主混淆后函数名可能有fun()变为a()，插件使用宿主混淆后的mapping映射来编译插件包，这样插件调用fun()时实际调用的是a()，才能找到正确的函数调用。

**插件项目的编写与正常的APP编写是一样的 ，`com.android.tools.build:gradle`版本最好是3.0.0，这是官方demo中的版本。我在测试时用的其他版本插件无法通过编译。**

#### 生成插件

运行Gradle插件`assemblePlugin`或者运行Grradle指令`gradle clean assemblePlugin` 生成插件，生成的插件位于`app-build-outputs-plugin`目录下
- 插件包均是Release包，不支持debug模式的插件包
- 如果存在多个productFlavors，那么将会构建出多个插件包

#### 混淆规则

{% highlight groovy %}
-keep class com.didi.virtualapk.internal.VAInstrumentation { *; }
-keep class com.didi.virtualapk.internal.PluginContentResolver { *; }
-dontwarn com.didi.virtualapk.**
-dontwarn android.content.pm.**
-keep class android.** { *; }
{% endhighlight %}

<span id="4"></span>

## 基本的使用

在适当的加载你所需的插件
{% highlight java %}
PluginManager pluginManager = PluginManager.getInstance(base);
        File apk = new File(Environment.getExternalStorageDirectory(), "Test.apk");
        if (apk.exists()) {
            try {
                pluginManager.loadPlugin(apk);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
{% endhighlight %}
跳转至插件的相关页面
{% highlight java %}
Intent intent = new Intent();
     intent.setClassName(this, "包名+页面路径");
     startActivity(intent);
{% endhighlight %}

**插件跳转到宿主页面不能使用`this`，需要完整的包名+页面路径**

#### 相关API
{% highlight java %}
public String getLocation()

public String getPackageName()

public PackageManager getPackageManager()

public AssetManager getAssets()

public Resources getResources()

public ClassLoader getClassLoader()
{% endhighlight %}