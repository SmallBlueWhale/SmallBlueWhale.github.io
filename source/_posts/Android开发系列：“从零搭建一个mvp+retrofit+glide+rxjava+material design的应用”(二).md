---
title: Android开发系列：“从零搭建一个mvp+retrofit+glide+rxjava+material design的应用”(二)

date: 2017-04-01 11:23:00

tags:

- MVP
- Android
- Retrofit
- Rxjava
- Material Design
- Glide
- Gradle


---
# Rxjava生命周期管理

对于这个大量运用mvp,rxjava的项目，我们很容易出现内存泄漏的问题，因此需要一个统一管理rxjava生命周期的类。命名为rxmanager。

这里我们可以使用compositeSubscription这个类的unsubscribe方法来管理生命周期。

## RxManager

```java
package com.hyx.whale.whalenote.base;

import java.util.HashMap;
import java.util.Map;

import rx.Observable;
import rx.Subscription;
import rx.android.schedulers.AndroidSchedulers;
import rx.functions.Action;
import rx.functions.Action1;
import rx.subscriptions.CompositeSubscription;

/**
 * funtion :用于管理单个presenter的rxbus事件和rxjava相关代码的生命周期处理。
 * author  :smallbluewhale.
 * date    :2017/4/5 0005.
 * version :1.0.
 */
public class RxManager {
    public RxBus mRxbus = RxBus.getInstance();
    /*
    * 管理Observable和Subscribe订阅
    * 取消以及注册订阅
    * */
    private CompositeSubscription compositeSubscription = new CompositeSubscription();

    /*
    * 单纯的obserables和subscribers管理
    * */
    public void add(Subscription subscription){
        /*订阅管理*/
        compositeSubscription.add(subscription);
    }


    /**
     * 单个presenter生命周期结束，取消订阅和所有rxbus观察
     */
    public void clear(){
        compositeSubscription.unsubscribe();
    }

}


```

## RxBus

**另外我觉得一个项目免不了事件传递和事件发送，因此我们可以利用rxjava来编写一个类似eventbus的事件处理类，代码很简洁，也很简单，相应的注释都写在类的头部**

```java
package com.hyx.whale.whalenote.base;

import android.support.annotation.NonNull;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;

import rx.Observable;
import rx.android.schedulers.AndroidSchedulers;
import rx.functions.Action;
import rx.functions.Action1;
import rx.subjects.PublishSubject;
import rx.subjects.Subject;

import static android.R.attr.tag;
import static android.R.string.no;

/**
 * funtion :用rxbus和rxandroid实现的一个事件监听的类
 * author  :smallbluewhale.
 * date    :2017/3/21 0021.
 * version :1.0.
 */
public class RxBus {
    private static RxBus instance;


    RxBus(){

    }
    public static RxBus getInstance(){
        if(null==instance){
            synchronized (RxBus.class){
                instance = new RxBus();
            }
        }
        return  instance;
    }


    //每一种tag类型有一个同种类型事件的lsit
    private ConcurrentHashMap<Object , List<Subject>> subjectMapper = new ConcurrentHashMap<>();

    /**
     * 订阅事件源
     *
     * @param mObservable
     * @param mAction1
     * @return
     */
    public RxBus onEvent(Observable<?> mObservable , Action1<Object> mAction1){
        mObservable.observeOn(AndroidSchedulers.mainThread()).subscribe(mAction1, new Action1<Throwable>() {
            @Override
            public void call(Throwable throwable) {
                throwable.printStackTrace();
            }
        });
        return getInstance();
    }


    /**
     * 注册事件源
     * 1.将事件按照tag标志放置在一个map中，然后不同tag事件对应不同一个subjectlist
     * 2.再创建一个tag对应的subject放在
     * @param tag
     * @return
     */
    public <T> Observable<T> register(@NonNull Object tag){
        List<Subject> subjectList = subjectMapper.get(tag);
        if(null == subjectList){
            subjectList = new ArrayList<>();
            subjectMapper.put(tag,subjectList);
        }
        Subject<T,T> subject = PublishSubject.create();
        subjectList.add(subject);
        return subject;
    }

    /*
    * 简单发送事件
    * 默认设置content的类名为tag标志
    * */
    public void post(@NonNull Object content){
        post(content.getClass().getName(),content);
    }


    /**
     * 触发事件
     * 遍历全部相同tag事件的list，然后将相同的时间全部post出去，利用subject的一些有用的特性，例如可以构造一个无发送事件的observable
     * @param content
     */
    public void post(@NonNull Object tag,@NonNull Object content){
        List<Subject> subjectList = subjectMapper.get(tag);
        if(!isEmpty(subjectList)){
            for (Subject subject :
                    subjectList) {
                subject.onNext(content);
            }

        }
    }

    /**
     * 解绑事件
     * @param content
     */
    public void unRegister(@NonNull Object content){
        List<Subject> subjectList = subjectMapper.get(content);
        if(null != subjectList){
            subjectMapper.remove(content);
        }
    }

    /**
     * 取消监听
     *
     * @param tag
     * @param observable
     * @return
     */
    public RxBus unRegister(@NonNull Object tag , @NonNull Observable<?>observable){
        if(null == observable){
            getInstance();
        }
        List<Subject> subjectList = subjectMapper.get(tag);
        if(null != subjectList){
            subjectList.remove((Subject<?,?>)observable);
            if(isEmpty(subjectList)){
                subjectMapper.remove(tag);
            }
        }
        return getInstance();
    }

    @SuppressWarnings("rawtypes")
    public static boolean isEmpty(Collection<Subject> collection) {
        return null == collection || collection.isEmpty();
    }



}
```

当然，我们可以用rxmanager再统一处理rxbus。建立一个map存储

# 

