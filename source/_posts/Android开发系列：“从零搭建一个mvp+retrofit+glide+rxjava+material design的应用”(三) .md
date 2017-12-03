---
title: Android开发系列：“从零搭建一个mvp+retrofit+glide+rxjava+material design的应用”(三)

date: 2017-06-20 11:23:00

tags:

- Android
- Retrofit
- Rxjava


---




## 封装一个网络框架(Rxjava + Retrofit 2 +  OkHttp3)

**图片UML类图**



![img](https://github.com/SmallBlueWhale/SmallBlueWhale.github.io/blob/master/img/Retrofit%20+%20Rxjava%20+%20OkHttp.png?raw=true)



## 项目分包：

为了让项目更加清晰

api：网络层框架总包

basic：项目一些网络接口配置

builder：设置observable以及retrofit和okhttp的

Intercepter：okhtpp的拦截器用来设置网络的

observable：只有NetworkErrorCodeFunc1，用来处理返回的response

subscriber：用来设置subscriber

util：gson将map转化成需要的requestbody请求体

## 基本思想

先从整体最后调用的代码去谈。

```java
        /*
        * 调用model层来获取数据d
        * */
        NetworkRequest.with(mContext)
                .setShowingDialog()
                .setObservable(NetworkApi.register(hashMap))
                .subScriber(new NetworkSubscriber<ResponseInfo<User>>() {
                    //数据成功
                    @Override
                    public void onNext(ResponseInfo<User> result) {

                    }
                });
    }
```

1. Retrofit (用rxjava去实现的话其实，我们可以这样去构造，observable的start()函数可以用来提前处理由okhttp返回的observable<response>数据)
2. Observable (这样的话我们用一个转化ResponseInfo的observable也就是func1()将ResponseInfo转化成一个只有data数据的observable)
3. Subscriber (通过返回的observable之后，再由subscriber订阅时，在主线程去回调相应的方法，比如说)

observable.)



## Retrofit2 + Okhttp3

```java
package szu.whale.wenote.api;

import android.content.Context;

import java.lang.ref.WeakReference;

import rx.Observable;

/**
 * funtion :
 * author  :smallbluewhale.
 * date    :2017/6/5 0005.
 * version :1.0.
 */
public class NetworkRequest {

    public static final String TAG = "NetworkRequest";
    public static NetworkRequest instance = new NetworkRequest();           //单例饿汉模式 最简单的一种单例模式
    private static WeakReference<Context> wrContext;
    private Observable observable;
    private boolean isShowingDialog;

    public static NetworkRequest with(Context context) {
        wrContext = new WeakReference<Context>(context);
        return instance;
    }


    public NetworkRequest setShowingDialog() {
        this.isShowingDialog = false;
        return instance;
    }

    public NetworkRequest setObservable(Observable observable) {
        this.observable = observable;
        return instance;
    }


    /*
    * 在这里设置一个订阅者
    * 能够显示dialog以及根据不同网络状态显示做出不同反应
    * */
    public NetworkRequest subScriber(NetworkSubscriber networkSubscriber) {
        networkSubscriber.setContext(wrContext.get());
        networkSubscriber.setIsShowWaitDialog(isShowingDialog);
        observable.subscribe(networkSubscriber);
        return instance;
    }
}

```

这个NetworkRequest是一个单例类，主要是observable和subscriber两个部分。

observablebuilder类用来构造整个observable，这个observablebuilder控制发射所在线程，订阅所在线程，同时通过，func1()通过正确的response转化成observable.



## **NetworkErrorCodeFunc1**:

```java
package szu.whale.wenote.api.observable;

import rx.Observable;
import rx.functions.Func1;
import szu.whale.wenote.api.basic.NetworkException;
import szu.whale.wenote.api.basic.NetworkResponseType;
import szu.whale.wenote.api.basic.ResponseInfo;

/**
 * funtion :根据服务器返回的信息，统一处理错误码code以及正确码，并且正确时候的将data传给subscriber
 * author  :smallbluewhale.
 * date    :2017/6/5 0005.
 * version :1.0.
 */
public class NetworkErrorCodeFunc1<T> implements Func1<ResponseInfo<T>, Observable<T>> {
    @Override
    public Observable<T> call(ResponseInfo<T> responseInfo) {
        //0是正确标识码
        //自己根据服务器的错误码显示不同的错误信息
        if(!NetworkResponseType.STATUS_OK.equals(responseInfo.getStatus())){
            return Observable.error(new NetworkException(responseInfo.getStatus() , responseInfo.getMsg()));
        }
        return Observable.just(responseInfo.getData());
    }
}
```



## **NetworkSubscriber**:

接着通过subscriber来控制dialog , 同时对返回的错误码进行控制，

```java
package szu.whale.wenote.api.subscriber;

import android.content.Context;
import android.text.TextUtils;

import com.google.gson.JsonParseException;

import org.json.JSONException;

import java.net.SocketTimeoutException;
import java.net.UnknownHostException;
import java.text.ParseException;

import retrofit2.HttpException;
import rx.Subscriber;
import szu.whale.wenote.api.basic.NetworkException;
import szu.whale.wenote.api.basic.NetworkResponseType;
import szu.whale.wenote.util.CustomProgressDialog;
import szu.whale.wenote.util.ToastUtil;


/**
 * funtion :一个控制展示dialog以及错误控制的subscriber
 * author  :smallbluewhale.
 * date    :2017/6/5 0005.
 * version :1.0.
 */
public abstract class NetworkSubscriber<T> extends Subscriber<T> {

    //对应HTTP的状态码,不是服務器状态码
    private static final int UNAUTHORIZED = 401;
    private static final int FORBIDDEN = 403;
    private static final int NOT_FOUND = 404;
    private static final int REQUEST_TIMEOUT = 408;
    private static final int INTERNAL_SERVER_ERROR = 500;
    private static final int BAD_GATEWAY = 502;
    private static final int SERVICE_UNAVAILABLE = 503;
    private static final int GATEWAY_TIMEOUT = 504;


    private Context context;
    private boolean isShowWaitDialog;
    private CustomProgressDialog waitDialog;

    public void setContext(Context context) {
        this.context = context;
    }

    public void setIsShowWaitDialog(boolean showWaitDialog) {
        isShowWaitDialog = showWaitDialog;
    }

    private void showWaitDialog() {
        waitDialog.show();
    }

    private void dissmissDialog() {
        waitDialog.dismiss();
    }

    @Override
    public void onStart() {
        super.onStart();
        if (isShowWaitDialog) {
            showWaitDialog();
        }
    }

    @Override
    public void onCompleted() {
        if (isShowWaitDialog) {
            dissmissDialog();
        }
    }

    @Override
    public void onError(Throwable e) {
        if (isShowWaitDialog) {
            dissmissDialog();
        }
        Throwable throwable = e;
        /*
        * 获取异常根源
        * */
        while (throwable.getCause() != null) {
            e = throwable;
            throwable = throwable.getCause();
        }
        //http请求错误
        if (e instanceof HttpException) {
            HttpException httpException = (HttpException) e;
            switch (httpException.code()) {
                case UNAUTHORIZED:
                    ToastUtil.showMsgShort(context, "未授权");
                    break;
                case FORBIDDEN:
                    ToastUtil.showMsgShort(context, "没有网络访问权限");
                    //  onPermissionError(ex);          //权限错误，需要实现
                    break;
                case NOT_FOUND:
                    ToastUtil.showMsgShort(context, "未找到网络");
                    break;
                case REQUEST_TIMEOUT:
                    ToastUtil.showMsgShort(context, "请求超时");
                    break;
                case GATEWAY_TIMEOUT:
                    ToastUtil.showMsgShort(context, "网关请求超时");
                    break;
                case INTERNAL_SERVER_ERROR:
                    ToastUtil.showMsgShort(context, "端口错误");
                    break;
                case BAD_GATEWAY:
                    ToastUtil.showMsgShort(context, "错误网关");
                    break;
                case SERVICE_UNAVAILABLE:
                    ToastUtil.showMsgShort(context, "找不到服务器");
                    break;
                default:
                    ToastUtil.showMsgShort(context, "网络错误");
                    break;
            }
        }
        //服务器返回的错误码
        else if (e instanceof NetworkException) {
            onNextError((NetworkException) e);
        } else if (e instanceof JsonParseException || e instanceof ParseException || e instanceof JSONException) {
            ToastUtil.showMsgShort(context, "json解析错误");
        } else if (e instanceof UnknownHostException) {
            ToastUtil.showMsgShort(context, "未知异常");
        } else if (e instanceof SocketTimeoutException) {
            ToastUtil.showMsgShort(context, "socket连接超时");
        } else {
            e.printStackTrace();
            ToastUtil.showMsgShort(context, "网络加载失败");
        }
    }

    //根据服务器返回的错误码来显示不同的toast
    private void onNextError(NetworkException e) {
        switch (e.getCode() + "") {
            case NetworkResponseType.STATUS_600211:
                ToastUtil.showMsgShort(context, "token已过期");
                break;
            case NetworkResponseType.STATUS_6003:
                ToastUtil.showMsgShort(context, "暂无数据");
                break;
            default:
                String errorMsg = e.getMessage();
                if (TextUtils.isEmpty(errorMsg)) {
                    ToastUtil.showMsgShort(context, errorMsg);
                } else {
                    ToastUtil.showMsgShort(context, errorMsg);
                }
                break;
        }
    }
}
```

这里NetworkException记录错误信息包括错误码以及错误信息，然后再subscriber中根据不同错误信息，做出不同的处理。



部分参考自：

[Android RxJava之网络处理专栏](http://blog.csdn.net/column/details/13297.html)

[RxJava+Retrofit+OkHttp封装](https://yuanjunli.github.io/2016/11/26/rxjava+retrofit+OkHttp%E5%B0%81%E8%A3%85/)







# 

