---
title: Android HandlerThread源码解析

date: 2017-10-26 11:23:00

tags:

- 源码
- Android
- Handler

---
# Android HandlerThread源码解析

## Handler  

**Handlerthread的源码其实就只有一点点，主要的函数只有重载的run函数以及**

* 懒汉式
* 饿汉式
* 双重校验锁
* 静态内部类

下面我总结前三种比较常用的设计模式的方式

### 饿汉式

```java
class SingleInstance{
  private final static SingleInstance singleInstance = new SingleInstance();
  	
  public SingleInstance getInstance(){
    	return singleInstance;
  	}
}
```

1. 当类的构造中有大量的逻辑操作或其它操作的时候，这个类加载的速度会变得更加慢。
2. 当类只是被实例化不做任何操作的时候，会占用并浪费资源。

### 懒汉式

```java
 class SingleInstance{
   private static SingleInstance singleInstance;
   public SingleInstance(){}
   public static SingleInstance getInstance(){
     if(null==singleInstance){
       singleInstance = new SingleInstance();
     }
     return singleInstance;
   }
}
```

1. 这种方式去加载的话，类的资源只有第一次加载才需要进行初始化。

如果需要同步锁校验的话代码如下：

```java
 class SingleInstance{
   private static SingleInstance singleInstance;
   public SingleInstance(){}
   public static synchronized SingleInstance getInstance(){
     if(null==singleInstance){
       singleInstance = new SingleInstance();
     }
     return singleInstance;
   }
}
```

但是这种方式相对来说会影响getInstance的效率，不能每一次都对getInstance进行检查。

那么就需要下面这一种

### 双重校验锁

```java
class SingleInstance{
   private static SingleInstance singleInstance;
   public SingleInstance(){}
   public static SingleInstance getInstance(){
     if(null==singleInstance){
       synchronized(SingleInstance.class){
         	if(null==singleInstance){
       			singleInstance = new SingleInstance();
             }
     	}
      }
     return singleInstance;
   }
}
```

这一种再singleinstance为空的时候才进行校验，解决了每次都要检查同步锁的问题，只有在singleinstance没有进行初始化的时候才进行校验.

### 总结

一般情况下我们使用**饿汉式**就可以了，如果构造函数有过多的操作，那么我觉得使用**懒汉式**或则**双重校验锁**会更加合适。



> http://hujiandong.com/2016/12/21/design_pattern_singleton/





## Builder模式







