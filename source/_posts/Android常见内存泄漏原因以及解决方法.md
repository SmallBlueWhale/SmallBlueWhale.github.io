---
title: Android常见内存泄漏原因以及解决方法

date: 2017-06-09 11:23:00

tags:

- Android
- Context
- Handler && InnerClass
- Memory Leak
- Static


---
# Handler  

**Handler出现内存泄漏的原因，handler内部持有了一个context对象，同时如果这个对象因为handler消息未发送完成导致context对象一直无法被销毁，出现内存泄漏，同时如果我们新开一个线程去发送消息的话，也会因此出现内存泄漏。如下：**

```java
new Thread(new Runnable(){
  @Override
  public void run(){
    
  }
}).start();
```

**Runnable是一个匿名内部类，会隐含的持有外部类对象，具体原因下面**

### Non-static Inner Class
**非静态匿名内部类导致的内存泄漏问题，主要是因为非静态匿名内部类是属于外部类的一部分，所以如果这个匿名内部类的周期比外部类的周期要长的话，它会隐式引用这个外部类，同时这个外部类就因为被持有，导致无法销毁，因此就泄漏了。**

****

### 解决方法

```java
package com.hyx.maizuo.base.utils.system;
import android.app.Activity;
import android.content.SharedPreferences;
import android.os.Bundle;
import android.support.multidex.MultiDex;

import com.tencent.bugly.beta.Beta;

import cn.com.mma.mobile.tracking.util.DeviceInfoUtil;

public class LoadDexActivity extends Activity {
	protected Context mContext;
	private MyHandler mHandler;
  	WeakReference<Activity> mActivity;
	private static class MyHandler extends Handler{
	  	MyHandler(Activity activity){
          	mActivity = New WeakReference<Activity>(activity);
	  	}
    @Override
    public void handleMessage(Message msg) {
        SampleActivity activity = mActivity.get();
        if (mActivity != null) {
          // 做自己的操作。
        }
      }
  
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
		mHandler = new Myhandler(this);
      	mHandler.sendEmptyMessage(0);
    }

}
}
```



## MVP模式的内存泄漏

首先贴一段错误的代码

```java
import android.content.Context;
import android.support.annotation.NonNull;

import com.hyx.maizuo.base.utils.network.basic.ResponseException;

import java.lang.ref.WeakReference;


/**
 * P层（Presenter）基类
 * 需要多个模块都采用MVP模式的时候，可以考虑在commonLibrary中添加BaseMVPActivity和BaseMVPFragment
 * Created by whale on 17/4/18.
 *
 * @TODO
 */

public abstract class BasePresenter<V extends IBaseView> {

    protected Context mContext;
    private V mView;

    protected BasePresenter(){
        this.mContext = getView().getContext();
    }

    /**
     * 需要在Activity和Fragment的onCreate()方法中
     */
    public void attachView(@NonNull V view) {
        mView = view;

    }

    /**
     * 需要在Activity的onDestroy()方法或者Fragment的OnDestroyView()中分离view和presenter
     */
    public void detachView() {
        mView = null;
    }


}
```

我们通常会在BaseActivity中去attachView()绑定activity对象，这里的View指的就是Activity，然而，因为view在这个结构里面指的是activity，presenter持有activity的强引用，如果网络请求没中断presenter生命周期比Acitity长，或则程序某种原因退出了，没有走ondestory方法，那么这个view就会持有activity的强引用，然后内存泄漏了。

那么如果要解决的话，应该怎么去解决？

#### 解决方式

```java
package com.hyx.maizuo.base.utils.mvp;

import android.content.Context;
import android.support.annotation.NonNull;

import com.hyx.maizuo.base.utils.network.basic.ResponseException;

import java.lang.ref.WeakReference;


/**
 * P层（Presenter）基类
 * 需要多个模块都采用MVP模式的时候，可以考虑在commonLibrary中添加BaseMVPActivity和BaseMVPFragment
 * Created by frank on 17/4/18.
 *
 * @TODO
 */

public abstract class BasePresenter<V extends IBaseView> {

    protected Context mContext;
    private WeakReference<V> mView;

    protected BasePresenter(){
        this.mContext = getView().getContext();
    }

    /**
     * 需要在Activity和Fragment的onCreate()方法中
     */
    public void attachView(@NonNull V view) {
        mView = new WeakReference<V>(view);
    }

    /**
     * 需要在Activity的onDestroy()方法或者Fragment的OnDestroyView()中分离view和presenter
     */
    public void detachView() {
        mView = null;
    }


    /**
     * 判断当前的VIEW引用是否为空
     */
    public final boolean isViewAttached() {
        return mView != null && mView.get() != null;
    }

}
```

因为用的是weakReference ，这样的话android的回收机制会因为没有其它强引用去引用它的原因，将这个初始化的view回收，这样的话就不会导致内存泄漏。

但是用WeakReference的话，可能会有这样一种情况，系统误回收，这样我们在使用之前，最好判断它是否被回收。

```java
if(isViewAttached()){
  	//自己的操作
}
```

这里如果想知道更多，贴一段mosby的原文吧。

> In general I agree, it's painful to write all those `isViewAttached()` checked.
>
> First let me explain why `MvpBasePresenter` uses a `WeakReference`. Usually there is no need to wrap the `MvpView` in a `WeakReference` because:
>
> 1. Any async call should be canceled in `detachView()`, so that no callback should be invoked afterwards. For instance with `RxJava` and `unsubscribe()` this works like a charm.
> 2. However, there might be scenarios where you don't want to cancel running async background tasks, like when using retaining Fragments. But also in that case the Fragment gets detached and reattached temporarily during orientation changes. However, Mosby assumes that every interaction with the view takes place on the main UI Thread. So every interaction with the view from a callback / presenter **must** run on the main UI Thread. Hence it's **not possible** that a callback accesses the view during a screen orientation change, since screen orientation changes run on the main ui thread as well. Therefore, either screen orientation changes run before callback or vice versa, but **never** in parallel.
>
> So why does Mosby's default implementation uses a `WeakReference`?
> The only reason why Mosby's default implementation uses a `WeakReference` is that I wanted to avoid memory leaks if other developers misuse Mosby by writing a custom delegate that i.e. forget to call`presenter.detachView()`.
>
> Thus at a first glance you might come to the conclusion that `View` can never be `null`. Unfortunately, it can be in case that `View` is detached (i.e. user has closed the activity by pressing back button) but an async running background thread has not been canceled in `detachView()`. i.e. as far as I know Retrofit 1.x has no API to cancel running requests. Therefore, we have to check if `view != null` anyway.
>
> Alright, so back to your suggestion to use the null object pattern. Your implementation looks good (there is a little issue with WeakReference, I have added a comment in your commit to point out the problem) and I think in that case it is reasonable to use reflections.
>
> But I have in general a problem with the null object pattern in general. I don't like it because usually it only tries to hide other problems (might be an implementation detail like libraries that don't support canceling async threads or a lack of java programming language syntax / expressiveness).
>
> On one hand you are right and those `view != null` checks are pretty annoying. I have to think about it again, but I think we could try to find a compromise by adding those kind of base presenter, but I see it more as an addition and not as a replacement of `MvpBasePresenter`. However, having a second null object based `MvpNullObjectBasePresenter` implementation may result in having also a bunch of other presenter implemetation like `RxNullObjectPresenter`, `RetrofitNullObjectPresenter` and so on.
>
> On the other hand that kind of problem could be reduced / solved by Kotlin, where you could replace
>
> ```java
>  if (isViewAttached()){
>     getView.showContent();
> }
> ```
>
> with
>
> ```java
> getView()?showContent()
> ```
>
> What do you think?



[ 内存泄漏之Handler & InnerClass](http://www.androiddesignpatterns.com/2013/01/inner-class-handler-memory-leak.html)







