---
title: Android开发系列：“从零搭建一个mvp+retrofit+glide+rxjava+material design的应用”(一)

date: 2017-03-08 11:23:00

tags:

- MVP
- Android
- Retrofit
- Rxjava
- Material Design
- Glide
- Gradle


---




## gradle配置

首先项目的开始，我们需要先就项目需要用到的第三方库对gradle进行相应的配置。

**Material Design(module中)**

```groovy
compile 'com.android.support:support-v4:23.+'
compile 'com.android.support:support-annotations:23.+'
compile 'com.android.support:design:23.+'
compile 'com.android.support:cardview-v7:23.+'
compile 'com.android.support:recyclerview-v7:23.+'
compile 'com.android.support:appcompat-v7:23.+'
```

**Retrofit+Rxjava+Glide(module中):**

```groovy
    compile 'com.squareup.retrofit2:retrofit:2.0.0-beta3'
    compile 'com.squareup.okhttp3:logging-interceptor:3.1.2'
    compile 'io.reactivex:rxjava:1.0.1'
    compile 'io.reactivex:rxandroid:1.0.1'
    compile 'com.squareup.retrofit2:converter-gson:2.0.0-beta4'
    compile 'com.squareup.retrofit2:adapter-rxjava:2.0.0-beta4'
    compile 'com.github.bumptech.glide:glide:3.7.0'
```

**Multidex(module中):**

```groovy
    compile 'com.android.support:multidex:1.0.0'//分包
```

**ButterKnife(module中):**

```groovy
compile 'com.jakewharton:butterknife:7.0.1'//view注解
```



**Freeline*(project中的gradle):**

```groovy
classpath 'com.antfortune.freeline:gradle:0.8.6
```

**接着在项目的app module中:**

```groovy
apply plugin: 'com.antfortune.freeline'
```

这个是一个完整的mvp项目该有的样子，可以看到业务业务相关的代码全部被移到了presenter中，activity实现减负。

![img](http://oa5504rxk.bkt.clouddn.com/week5_MVP/2.jpg)



## 项目分包：

为了让项目更加清晰

api：负责网络服务，例如定义网络接口，以及固定常量。

app：项目单独的application

bean：负责实体类的定义

db：负责view和接口层

ui：整个mvp中最关键的一环，每一个功能模块在这内部都应分出一个属于自己的包。这内部中有几个功能模块，比如main，main模块是总组成，又可以分成activity，fragment，contract，model，presenter。其中activity和fragment属于view，contract定义了这个view和model以及presenter全部的功能接口，属于一个能让结构更加清晰的类，每次分析项目，只需要从contract看起就可以清楚的理解这个模块主要承载了什么样子的功能特点。同时，我们构建项目也只需要先从contract入手，这样就不会在mvp这个架构中被一堆的接口绕晕。model层定义的是数据类型，用来拼装一系列数据，将数据拼装好再传到presenter中通过Presenter层接口进行调用。

util：负责项目一些公有的项目大部分功能。

widget：项目中的自定义控件





部分参考自google mvp architecture 以及bugly文章

http://dev.qq.com/topic/57bfef673c1174283d60bac0

https://github.com/googlesamples/android-architecture/







# 

