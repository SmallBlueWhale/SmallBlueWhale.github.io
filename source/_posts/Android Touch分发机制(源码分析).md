---
title: Android Touch分发机制源码分析

date: 2017-07-12 19:23:00

tags:

- Android
- Touch
- View
---
# Touch事件分发机制  

**先从Activity源码分析，分析写在函数内部**

```java
/**
 * Called to process touch screen events.  You can override this to
 * intercept all touch screen events before they are dispatched to the
 * window.  Be sure to call this implementation for touch screen events
 * that should be handled normally.
 *
 * @param ev The touch screen event.
 *
 * @return boolean Return true if this event was consumed.
 */
public boolean dispatchTouchEvent(MotionEvent ev) {
  	//如果先调用的是一个空方法
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
  	/**
  	* 再调用父类window的superDispatchTouchEvent()方法
    * 调用父类superDispatchTouchEvent()方法，如果父类拦截就返回ture
  	*/
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
  	//否则调用Activity的onTouchEvent()方法
    return onTouchEvent(ev);
}
 
```

然后我们看它调用Activity的onTouchEvent()

```java
/**
 * Called when a touch screen event was not handled by any of the views
 * under it.  This is most useful to process touch events that happen
 * outside of your window bounds, where there is no view to receive it.
 *
 * @param event The touch screen event being processed.
 *
 * @return Return true if you have consumed the event, false if you haven't.
 * The default implementation always returns false.
 */
public boolean onTouchEvent(MotionEvent event) {
  	//这里其实是就是考虑特殊情况，如果当前点击事件导致Activity关闭
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }
	//不然直接返回false
    return false;
}
```

