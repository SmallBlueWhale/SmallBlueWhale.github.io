---
title: Android学习总结

date: 2016-12-12 01:23:00

tags:

- Android


---
# 自定义View学习中的一些难点记录  

- what is the different about getLeft(),getX()？  

`Position`  

* [这个部分可以参照这个地方去理解]("http://gold.xitu.io/entry/571a591a2e958a006be9f473") 

**The geometry of a view is that of a rectangle. A view has a location, expressed as a pair of left and top coordinates, and two dimensions, expressed as a width and a height. The unit for location and dimensions is the pixel.**



**It is possible to retrieve the location of a view by invoking the methods getLeft() and getTop(). The former returns the left, or X, coordinate of the rectangle representing the view. The latter returns the top, or Y, coordinate of the rectangle representing the view. These methods both return the location of the view relative to its parent. For instance, when getLeft() returns 20, that means the view is located 20 pixels to the right of the left edge of its direct parent.**

**In addition, several convenience methods are offered to avoid unnecessary computations, namely getRight() and getBottom(). These methods return the coordinates of the right and bottom edges of the rectangle representing the view. For instance, calling getRight() is similar to the following computation: getLeft() + getWidth() (see Size for more information about the width.)**  

    right = left + width;

    bottom = top + height;

这里的意思是getLeft其实是相对父view的位置，而getX则是相对整个根布局的位置  

-    View的滑动							

     * [关于getscrollX以及ScrollX的理解和梳理]("http://gold.xitu.io/entry/576a264f80dda4005fb17639") 

-    关于自定义view中onMeasure方法的模板  
     ​      	   	private Bitmap mBitmap;
     ```
     	private void init() {
          mBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.meinv);
      	}

      	@Override
      	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          // 获取宽度测量规格中的mode和size
          int widthMode = MeasureSpec.getMode(widthMeasureSpec);
          int widthSize = MeasureSpec.getSize(widthMeasureSpec);
          int heightMode = MeasureSpec.getMode(heightMeasureSpec);
          int heightSize = MeasureSpec.getSize(heightMeasureSpec);
          // 声明一个临时变量来存储计算出的测量值
          int  heightResult = 0;
          int widthResult = 0;
        
           /*
     ```

     - 如果父容器心里有数
       */
        if (widthMode == MeasureSpec.EXACTLY) {

       ```
        // 那么子view就用父容器给定尺寸
        widthResult = widthSize;
       ```

        }
        /*

       - 如果父容器不确定自身大小
         */
         else{
          // 那么子view可要自己看看自己需要多大了
          widthResult = mBitmap.getWidth()+getPaddingLeft()+getPaddingRight();
          /*
         - 如果爹给儿子的是一个限制值

       ```
       */
       ```

          if (widthMode == MeasureSpec.AT_MOST) {

       ```
          // 那么儿子自己的需求就要跟爹的限制比比看谁小要谁
          widthResult = Math.min(widthSize, widthResult);
       ```

          }
       }

        if (heightMode == MeasureSpec.EXACTLY) {

     ```
        heightResult = widthSize;
     ```

        }else{

     ```
        //考虑padding
        heightResult = mBitmap.getHeight()+getPaddingBottom()+getPaddingTop();
        if (heightMode == MeasureSpec.AT_MOST) {
            heightResult = Math.min(widthSize, heightResult);
        }
     ```

        }
        // 设置测量尺寸
        setMeasuredDimension(widthResult, heightResult);

     }

-    关于自定义View滑动冲突的拦截以及模板总结、


#  外部拦截法

##   事件分发的过程图 
![](http://ww2.sinaimg.cn/large/005Xtdi2jw1f88i0q8uozj30nm0kqwhm.jpg)

外部拦截法即事件都经过父容器处理，如果父容器需要事件就处理事件，不需要则不拦截，下面来看一下伪代码：


        @Override
        public boolean onInterceptTouchEvent(MotionEvent event) {
            boolean intercept = false;
            int x = (int) event.getX();
            int y = (int) event.getY();
            int action = event.getActionMasked();
            switch (action) {
                case MotionEvent.ACTION_DOWN:
                    //如果希望子view能接收到事件，DOWN必然要返回false
                    intercept = false;
                    break;
    
                case MotionEvent.ACTION_MOVE:
                    //如果需要拦截事件，就返回true
                    if (needIntercept(event)) {
                        intercept = true;
                    } else {
                        intercept = false;
                    }
                    break;
    
                case MotionEvent.ACTION_UP:
                    //手指抬起，必须返回false，因为返回值对自己没有影响，而对子view可能有影响
                    intercept = false;
                    break;
            }
            //重新设置最后一次位置
            mLastEventX = x;
            mLastEventY = y;
            return intercept;
        }
    
        private boolean needIntercept(MotionEvent event) {
            return false;
        }

下面来分析一下这段伪代码的意思：

1. 首先ACTION_DOWN必须返回false，否则子view无法接收到事件，事件都会由自己处理
2. 对应ACTION_MOVE则对自己根据情况处理，需要就拦截，否则不拦截
3. 最后是ACTION_UP，必须返回false，原因有：
- ACTION_UP的返回值对自身并没有影响，自身始终能接收到事件
- 如果子一些列事件中，ViewGroup都始终没有拦截事件，却在ACTION_UP中返回true，这样导致子view无法接收到UP事件，那么就会影响子view的click事件，或者其他逻辑处理
4. 是否需要拦截事件都交给needIntercept方法处理，这个处理是根据业务来处理的，还可如果我们无法确定某些因素，还可以通过设置回调接口来处理，让其他对象通过接口来告知感兴趣的事。

 如下面代码：

         private boolean needIntercept(MotionEvent event) {
            if (mEventCallback != null) {
               return mEventCallback.isCanIntercept();
            }
            return false;
        }
    
        public EventCallback mEventCallback;
    
        public void setEventCallback(EventCallback eventCallback) {
            mEventCallback = eventCallback;
        }
        public interface EventCallback{
            boolean isCanIntercept();
        }

在外部拦截法中，子view最好不要使用requestDisallowInterceptTouchEvent来干预事件的处理






# 内部拦截法

内部拦截是指父容器不拦截任何事件，所有事件都传递给子view，如果子元素需要事件就直接消耗，否则交给父容器处理，这种拦截法需要配合requestDisallowInterceptTouchEvent方法来使用。我们需要重写子view的dispatchTouchEvent方法。

      private int mLastX, mLastY;
        @Override
        public boolean dispatchTouchEvent(MotionEvent event) {
            int action = event.getActionMasked();
            int x = (int) event.getX();
            int y = (int) event.getY();
            switch (action) {
                case MotionEvent.ACTION_DOWN:
                    //不让父View拦截事件
                    mLastX = x;
                    mLastY = y;
                    getParent().requestDisallowInterceptTouchEvent(true);
                    break;
    
                case MotionEvent.ACTION_MOVE:
                    //如果需要拦截事件，就返回true
                    if (!needIntercept(event)) {
                        getParent().requestDisallowInterceptTouchEvent(false);
                    }
                    break;
    
                case MotionEvent.ACTION_UP:
                    //手指抬起，必须返回false，因为返回值对自己没有影响，而对子view可能有影响
                    break;
            }
            mLastX = x;
            mLastY = y;
            return super.dispatchTouchEvent(event);
        }


代码说明：
- 首先，必须假定父view不拦截DOWN事件而拦截其他事件，否则子view无法获取任何事件。在子view调用requestDisallowInterceptTouchEvent(false)后，父view才能继续拦截事件
- 其次在ACTION_DOWN时，调用requestDisallowInterceptTouchEvent(true)来不允许父View拦截事件
- ACTION_MOVE中如果needIntercept返回false，则调用requestDisallowInterceptTouchEvent(false)让父view重新拦截事件，需要注意的是，一点调用此方法，就表示放弃了同系列的事件的所有事件。
- 最后调用requestDisallowInterceptTouchEvent后触发我们的onTouchEvent方法，处理时间


所以父元素的拦截逻辑如下：

      @Override
        public boolean onInterceptHoverEvent(MotionEvent event) {
            boolean intercept = false;
            int x = (int) event.getX();
            int y = (int) event.getY();
            int action = event.getActionMasked();
            if(action == MotionEvent.ACTION_DOWN){
                     return false;
            }else{
                     return true
            }
       }  




# 重定义dispatchTouchEvent方法

上面两种方法基本还是尊重系统的事件分发机制，但是还是有一些情况无法满足，这时候，我们需要根据业务需求来重新定义事件分发了。

比如一个下拉刷新模式

![](img/010_demopng.png)

首先我们定义：
下拉刷新容器为： A
列表布局为ListView：B
刷新头为：C

逻辑如下：
首先A或获取到事件，如果手机方向被认定为垂直滑动，A要判断C的位置和滑动方向：

1，C刚好隐藏，此时向下滑动，B这时无法向下滑动

A需要拦截事件，自己处理，让C显示出来，此时A需要拦截事件，自己处理，让C显示出来，如果手指又向上滑动，则A又要判断C是否隐藏，没有隐藏还是A拦截并处理事件，当C完全隐藏后，又要吧事件交给B处理，B来实现自己列表View该有的特性

就这个逻辑上述方案1和方案2就无法满足，**因为系统的事件分发有一个特点**：

- **当一个ViewGroup开始拦截并处理事件后，这个事件只能由它来处理，不可能再把事件交给它的子view处理，要么它消费事件，要么最后交给Activity的onTouchEvent处理**

 >在代码中就是，只要ViewGroup拦截了事件，他的dispatchTouchEvent方法中接收事件的子view就会被置为null，

此特点：
- 套用到方案1外部拦截法就是，在MOVE中，开始拦截事件，View收到一个Cancel事件后，之后都无法获取到同系列事件了。
- 套用到方案2就是在MOVE中调用requestDisallowInterceptTouchEvent(false)就表示完全放弃同系列事件的所有事件了



# Demo

说了这么方案，现在来一个实例，需求
定义一个ViewGroup，布局方向为横向布局，可以左右滑动切换子view，同时只显示一个子view，类似ViewPager，其次ViewGroup内部放置ListView，来制造滑动冲突，我们需要解决这种冲突。

我们的自定义HScrollLayout代码如下：

    package com.ztiany.view.views;

    import android.content.Context;
    import android.util.AttributeSet;
    import android.util.Log;
    import android.view.MotionEvent;
    import android.view.VelocityTracker;
    import android.view.View;
    import android.view.ViewGroup;
    import android.view.animation.AccelerateDecelerateInterpolator;
    import android.widget.Scroller;
    
    /**
     * @author Ztiany
     *         email 1169654504@qq.com & ztiany3@gmail.com
     *         date 2015-12-03 15:23
     *         description
     *         vsersion
     */
    public class HScrollLayout extends ViewGroup {


        public HScrollLayout(Context context) {
            this(context, null);
        }
    
        public HScrollLayout(Context context, AttributeSet attrs) {
            this(context, attrs, 0);
        }
    
        public HScrollLayout(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
            init();
        }
    
        public static final String TAG = HScrollLayout.class.getSimpleName();
    
        private int mLastEventX, mLastEventY;
        private VelocityTracker mVelocityTracker;
        private Scroller mScroller;
        private int mWidth;
        private int mCurrentPage;


        private void init() {
            //设置方向为横向布局
            mScroller = new Scroller(getContext(), new AccelerateDecelerateInterpolator());
            mVelocityTracker = VelocityTracker.obtain();
    
        }


        @Override
        public boolean onInterceptTouchEvent(MotionEvent event) {
    
            if (getChildCount() < 0) {
                return false;
            }


            boolean intercept = false;
            int x = (int) event.getX();
            int y = (int) event.getY();
            int action = event.getActionMasked();
            switch (action) {
                case MotionEvent.ACTION_DOWN:
    
                    if (!mScroller.isFinished()) {
                        mScroller.abortAnimation();
                        intercept = true;
                    } else {
                        //如果希望子view能接收到事件，DOWN必然要返回false
                        intercept = false;
                        mLastEventX = x;
                        mLastEventX = y;
                    }


                    break;

                case MotionEvent.ACTION_MOVE:
                    //计算移动差
                    int dx = x - mLastEventX;
                    int dy = y - mLastEventY;
                    if (Math.abs(dx) > Math.abs(dy)) {
                        intercept = true;
                    } else {
                        intercept = false;
                    }
                    break;
    
                case MotionEvent.ACTION_UP:
                    //手指抬起，必须返回false，因为返回值对自己没有影响，而对子view可能有影响
                    intercept = false;
                    break;
            }
            mLastEventX = x;
            mLastEventY = y;
            return intercept;
    
        }
    
        @Override
        protected void onLayout(boolean changed, int l, int t, int r, int b) {
            Log.d(TAG, "l:" + l);
            Log.d(TAG, "t:" + t);
            Log.d(TAG, "r:" + r);
            Log.d(TAG, "b:" + b);
            int left = l, top = t, right = r, bottom = b;
            if (changed) {
                int childCount = getChildCount();
                View child;
                for (int i = 0; i < childCount; i++) {
                    child = getChildAt(i);
                    if (child.getVisibility() == View.GONE) {
                        continue;
                    }
                    child.layout(left, top, left + child.getMeasuredWidth(), bottom);
                    left += child.getMeasuredWidth();
                }
            }
    
        }


        @Override
        public boolean onTouchEvent(MotionEvent event) {
    
            mVelocityTracker.addMovement(event);
            int x = (int) event.getX();
            int y = (int) event.getY();
            int action = event.getActionMasked();
            switch (action) {
                case MotionEvent.ACTION_DOWN:
                    mLastEventX = x;
                    mLastEventX = y;
                    break;
    
                case MotionEvent.ACTION_MOVE:
                    int dx = x - mLastEventX;
                    scrollBy(-dx, 0);
    
                    break;
    
                case MotionEvent.ACTION_UP:
                case MotionEvent.ACTION_CANCEL:
                    //将要滑动的距离
                    int distanceX;
                    mVelocityTracker.computeCurrentVelocity(1000);
                    float xVelocity = mVelocityTracker.getXVelocity();
    
                    Log.d(TAG, "xVelocity:" + xVelocity);
    
                    if (Math.abs(xVelocity) > 50) {
                        if (xVelocity > 0) {//向左
                            mCurrentPage--;
                        } else {
                            mCurrentPage++;
                        }


                    } else {
                        // 不考虑加速度
                        Log.d(TAG, "getScrollX():" + getScrollX());
                        if (getScrollX() < 0) {//说明超出左边界
                            mCurrentPage = 0;
                        } else {
                            int childCount = getChildCount();
                            int maxScroll = (childCount - 1) * mWidth;
                            Log.d(TAG, "maxScroll:" + maxScroll);
                            if (getScrollX() > maxScroll) {//超出了右边界
                                mCurrentPage = getChildCount() - 1;
                            } else {
    
                                //在边界范围内滑动
                                int currentScrollX = mCurrentPage * mWidth;//已近产生的偏移
                                int offset = getScrollX() % mWidth;
                                Log.d(TAG, "mWidth:" + mWidth);
                                Log.d(TAG, "offset:" + offset);
    
                                if (currentScrollX > Math.abs(getScrollX())) {//向左偏移
    
                                    if (offset < (mWidth - mWidth / 3)) {//小于其 2/3
                                        mCurrentPage--;
                                    } else {
    
                                    }
    
                                } else {//向右偏移
    
                                    if (offset > mWidth / 3) {//小于其 2/3
                                        mCurrentPage++;
                                    } else {
    
                                    }
    
                                }
    
                            }
                        }
                        //不考虑加速度
                    }
                    mCurrentPage = (mCurrentPage < 0) ? 0 : ((mCurrentPage > (getChildCount() - 1)) ? (getChildCount() - 1) : mCurrentPage);
                    distanceX = mCurrentPage * mWidth - getScrollX();
                    Log.d(TAG, "distanceX:" + distanceX);
                    smoothScroll(distanceX, 0);
                    mVelocityTracker.clear();
                    break;
            }
            mLastEventX = x;
            mLastEventY = y;
            //返回true，处理事件
            return true;
        }
    
        private void smoothScroll(int distanceX, int distanceY) {
            mScroller.startScroll(getScrollX(), 0, distanceX, 0, 500);
            invalidate();
        }
    
        @Override
        public void computeScroll() {
            if (mScroller.computeScrollOffset()) {
                scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
                invalidate();
            }
        }
    
        /**
         * 重写测量逻辑
         *
         * @param widthMeasureSpec
         * @param heightMeasureSpec 这里我们不考虑wrap_content的情况,也不考虑子view的margin情况
         */
        @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    
            int widthSize = MeasureSpec.getSize(widthMeasureSpec);
            int heightSize = MeasureSpec.getSize(heightMeasureSpec);
    
            int childCount = getChildCount();
            View child;
            for (int i = 0; i < childCount; i++) {
                child = getChildAt(i);
                if (child.getVisibility() == View.GONE) {
                    continue;
                }
                measureChild(child, MeasureSpec.makeMeasureSpec(widthSize, MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(heightSize, MeasureSpec.EXACTLY));
            }


            setMeasuredDimension(widthSize, heightSize);
        }
    
        @Override
        protected void onSizeChanged(int w, int h, int oldw, int oldh) {
            mWidth = w;
        }
    }




![](img/010_view滑动冲突解决方案.gif)

# 事件分发流程

通过前面的事件分发研究，我们可以总结出事件分发的流程：

在ViewGroup的dispatchTouchEvent中

1. 处理DOWN事件，在down事件中，如果拦截了事件则自己处理(onTouchEvent方法被调用)，子无法在获取事件了，如果不拦截DOWN事件，则会从外到内查找是否有子view能处理事件，如果有一个子view可以处理事件(down返回true)，则接下来的事件交给子view处理，ViewGroup的onInterceptTouchEvent方法还是会被调用，一旦其返回true，那么ViewGroup开始拦截事件，而子view以一个cancel事件结束

2. 接下来的move和up事件，如果在down中没有找到可以处理事件的子view，则自己处理接下来的事件。

3. 如果有子view可以处理事件，并且不拦截事件，则把事件都交给子view处理，一旦ViewGroup开始拦截，那么接收事件的子view将会被赋值为null，接下来事件遵循第二点。

4. 如果子view能接收到DWON事件，并且在接收到事件事件后，调用requestDisallowInterceptTouchEvent(true)方法，ViewGroup无法再拦截事件，也就说requestDisallowInterceptTouchEvent优先级高于onInterceptTouchEvent，但是requestDisallowInterceptTouchEvent不能干预父view对DOWN事件的处理。对于DOWN事件onInterceptTouchEvent说了算。


## Matrix的使用

### mapPoints

void mapPoints (float[] pts)

void mapPoints (float[] dst, float[] src)

void mapPoints (float[] dst, int dstIndex,float[] src, int srcIndex, int pointCount)

计算一组点基于当前Matrix变换后的位置，(由于是计算点，所以参数中的float数组长度一般都是偶数的,若为奇数，则最后一个数值不参与计算)。

它有三个重载方法:

1. void mapPoints (float[] pts) 方法仅有一个参数，pts数组作为参数传递原始数值，计算结果仍存放在pts中。

2. void mapPoints (float[] dst, float[] src) ，src作为参数传递原始数值，计算结果存放在dst中，src不变。
   如果原始数据需要保留则一般使用这种方法。

3. void mapPoints (float[] dst, int dstIndex,float[] src, int srcIndex, int pointCount) 可以指定只计算一部分数值。

   参数摘要
   dst目标数据
   dstIndex目标数据存储位置起始下标
   src源数据
   srcIndex源数据存储位置起始下标
   pointCount计算的点个数



## 关于事件分发的规律总结(参考Android开发艺术探索)


1. **同一个事件序列**是指从手指触摸屏幕那一刻起，到手指离开屏幕的那一刻结束,所以一些列事件由：
   `一个DOWN + 不数量的MOVE + 一个UP事件(可能CANCEL) `
   组成

2. 正常情况下，**一个事件只能被一个View拦截和消费**，也就是说同一个事件不可能被两个View共同来消费，但是如果一个View接收到事件并处理后有分发给其他View处理除外。

3. 如果一个ViewGroup能接收到事件，并且开始拦截事件，那么这一系列事件只能由它来处理。并且他的onInterceptTouchEvent方法不会再被调用。

 关于ViewGroup能否接收到事件又分为两种：
- 在DOWN就开始拦截事件
- 在DOWN没有拦截事件，但是子view处理了DOWN事件并且没有改变FLAG_DISALLOW_INTERCEPT这个标志位来不允许父View拦截事件，之后ViewGroup的onInterceptTouchEvent依然会被调用，如果返回true，ViewGroup还是可以拦截事件，之后可接收事件的子View收到一个CANCEL事件，然后在ViewGroup中被置为null

4. FLAG_DISALLOW_INTERCEPT 和 touchTarget在一系列事件的开始和结尾都会被重置,也就是说子View无法使用requestDisallowInterceptTouchEvent方法来要影响DOWN事件，如果ViewGroup在DOWN就开始拦截事件，子view不可能再得到事件

5. 如果一个View开始接收事件，如果它不消费DOWN事件(DOWN中返回false)，那么它不会接收到同系列事件中接下来的事件，在源码中的体现就是，ViewGroup在DOWN事件中没有找到可以处理事件的子view，接下来的同系列事件就会自己处理，即他的onTouchEvent方法被调用

6. 如果一个View开始接收事件，如果它消费了DOWN事件(DOWN中返回true),但是接下来的事件它返回false，这个View依然能继续接收这一系列的事件，直到UP(或CANCEL)事件结束，最终事件会回到Activity的onTouchEvent方法，由Activity处理

7. View不拦截事件，它接收到事件会里面调用onTouchEvent方法，ViewGroup默认不拦截事件

8. 如果一个View是可以被click或者longClick的，那么它的onTouchEvent方法默认都会消费事件，即使它是不可用的(disable)，disable只会导致click或者longClick不被调用：
- onClick发生前提，View可以点击，View能接收到Down和Up事件

9. focus对View的点击事件有影响，View的isFocusable和isFocusableInTouchMode为true并且当前没有获取到焦点，则会先请求焦点，此次点击不会响应click等事件

10. 事件是由外到内进行传递的，由内到外进行处理的，即事件总是先传给根View，再由根View传递给子View，而默认的处理顺序是子View到根View，ViewGroup可以全部拦截事件，子View可以调用requestDisallowInterceptTouchEvent干预父View的事件分发(DOWN事件无法被干预)

11. 当然对于某些特殊的需求，系统的dispatchTouchEvent方法可能不适用，那么需要重写ViewGroup的dispatchTouchEvent方法，那么事件分发的逻辑完全有我们定义。只要ViewGroup能接收到事件，它的dispatchTouchEvent每次都会被调用。

12. 如果ACTION_DOWN事件发生在某个View的范围之内，则后续的ACTION_MOVE，ACTION_UP和ACTION_CANCEL等事件都将被发往该View，即使事件已经出界了。

13. 第一根按下的手指触发ACTION_DOWN事件，之后按下的手指触发ACTION_POINTER_DOWN事件，中间起来的手指触发ACTION_POINTER_UP事件，最后起来的手指触发ACTION_UP事件（即使它不是触发ACTION_DOWN事件的那根手指）。

14. pointer id可以用于跟踪手指，从按下的那个时刻起pointer id生效，直至起来的那一刻失效，这之间维持不变（后续MotionEvent会详细解读）。

15. 如果父View在onInterceptTouchEvent中拦截了事件，则onInterceptTouchEvent中不会再收到Touch事件了，事件被直接交给它自己处理（按照普通View的处理方式）。

16. 如果一个事件首先由子view处理，但是如果子view在处理的过程中某个时刻返回了false，则此事件序列全部交给Activity处理。


## 关于事件分发中的滑动冲突

常见的滑动冲突场景：

- 1，外部滑动方向与内部滑动方向不一致
- 2，内部滑动方向与外部滑动方向一致
- 3，上述两种情况的嵌套


####滑动冲突场景：

场景1：

![](img/008_左右上下冲突.png)

类似ViewPager与多个ListFragemnt嵌套


场景2：


![](img/008_同向冲突.png)

类似ViewPager与ViewPager的嵌套

场景3：

![](img/008_复杂冲突.png)

类似SlidMenu加ViewPager加ListFragment






###解决滑动冲突的规则

- 对于场景1有以下方法来解决
- 判断滑动路径与水平方向夹角
- 判断水平方向与垂直方向的距离差(常用)
- 判断水平方向与垂直方向的速度差

- 对于场景2
- 这能通过业务需求来解决，比如某个情况只允许哪个View滑动



# 关于动画一些内容(下次用的时候可以直接从这里copy代码)
- 贝塞尔以及其它类型动画的一些总结  

    [各种常用动画效果的汇总]("http://www.jianshu.com/p/420da0f6e279")  

- AnimationSet  

  多个动画的组合  

      AnimationSet as = new AnimationSet(true);
- ScaleAnimation  

  变大变小动画

        ScaleAnimation sa = new ScaleAnimation(1f, 1.5f, 1f, 1.5f, ScaleAnimation.RELATIVE_TO_SELF,
                0.5f, ScaleAnimation.RELATIVE_TO_SELF, 0.5f);
        sa.setDuration(ANIMATIONEACHOFFSET * 3);
        sa.setRepeatCount(10);// 设置循环
- AlphaAnimation  

  透明度动画  

        AlphaAnimation aniAlp = new AlphaAnimation(1, 0.1f);
        aniAlp.setRepeatCount(10);// 设置循环
        as.setDuration(ANIMATIONEACHOFFSET * 3);
  最后用一个addAnimation讲动画加进这个控件中  

        as.addAnimation(sa);
        as.addAnimation(aniAlp);
  开始一个动画labelicon是一个view  

        labelIcon.startAnimation(as);
- 怎么让多个组合动画重复执行？  
  stackoverflow：
   [第二个回答，给animatorset加一个监听器]("http://stackoverflow.com/questions/17622333/repeat-animatorset")   

示例：  

	<?xml version="1.0" encoding="utf-8"?> 
	<!--First Stage  -->
	<set  xmlns:android="http://schemas.android.com/apk/res/android"   
	  android:ordering="together">  
	    <objectAnimator 
	        android:valueType="floatType"
	        android:duration="500"
	        android:propertyName="alpha"
	        android:valueFrom="1"
	        android:valueTo="0.8"
	        >
	    </objectAnimator>
	    
	    <objectAnimator 
	        android:valueType="floatType"
	        android:duration="500"
	        android:propertyName="scaleX"
	        android:valueFrom="1"
	        android:valueTo="0.8"
	        >
	    </objectAnimator>
	     
	    <objectAnimator 
	        android:valueType="floatType"
	        android:duration="500"
	        android:propertyName="scaleY"
	        android:valueFrom="1"
	        android:valueTo="0.8"
	        >
	    </objectAnimator>
	    
	    <objectAnimator
	        android:propertyName="x"
	        android:duration="500"
	        android:valueFrom="0"
	        android:valueTo="20"
	        >
	    </objectAnimator>
	    
	    <objectAnimator
	        android:propertyName="y"
	        android:duration="500"
	        android:valueFrom="0"
	        android:valueTo="-20"
	        >
	    </objectAnimator>


​	    
	    <!--Second Stage  -->
	    <objectAnimator 
	        android:valueType="floatType"
	        android:startOffset="500"
	        android:duration="500"
	        android:propertyName="alpha"
	        android:valueFrom="0.8"
	        android:valueTo="0.65"
	        >
	    </objectAnimator>
	    
	    <objectAnimator 
	        android:startOffset="500"
	        android:valueType="floatType"
	        android:duration="500"
	        android:propertyName="scaleX"
	        android:valueFrom="0.8"
	        android:valueTo="0.65"
	        >
	    </objectAnimator>
	     
	    <objectAnimator 
	        android:startOffset="500"
	        android:valueType="floatType"
	        android:duration="500"
	        android:propertyName="scaleY"
	        android:valueFrom="0.8"
	        android:valueTo="0.65"
	        >
	    </objectAnimator>
	    
	    <objectAnimator
	        android:startOffset="500"
	        android:propertyName="x"
	        android:duration="500"
	        android:valueFrom="20"
	        android:valueTo="-20"
	        >
	    </objectAnimator>
	
	    <!--Third Stage  -->
	    <objectAnimator 
	        android:startOffset="1000"
	        android:valueType="floatType"
	        android:duration="500"
	        android:propertyName="alpha"
	        android:valueFrom="0.65"
	        android:valueTo="1"
	        >
	    </objectAnimator>
	    
	    <objectAnimator 
	        android:startOffset="1000"
	        android:valueType="floatType"
	        android:duration="500"
	        android:propertyName="scaleX"
	        android:valueFrom="0.65"
	        android:valueTo="1"
	        >
	    </objectAnimator>
	     
	    <objectAnimator 
	        android:startOffset="1000"
	        android:valueType="floatType"
	        android:duration="500"
	        android:propertyName="scaleY"
	        android:valueFrom="0.65"
	        android:valueTo="1"
	        >
	    </objectAnimator>
	    
	    <objectAnimator
	        android:startOffset="1000"
	        android:propertyName="x"
	        android:duration="500"
	        android:valueFrom="-20"
	        android:valueTo="0"
	        >
	    </objectAnimator>
	    
	    <objectAnimator
	        android:startOffset="1000"
	        android:propertyName="y"
	        android:duration="500"
	        android:valueFrom="-20"
	        android:valueTo="0"
	        >
	    </objectAnimator>
	</set>

android代码部分：  
​			
	AnimatorSet animator =	(AnimatorSet) AnimatorInflater.loadAnimator(this,
				R.animator.clock_rotate);
		animator.setTarget(mFloatingImageView);
		animator.addListener(new AnimatorListener() {
			@Override
			public void onAnimationStart(Animator animation) {
				// TODO Auto-generated method stub
				mCanceled = false;
			}
			
			@Override
			public void onAnimationRepeat(Animator animation) {
				// TODO Auto-generated method stub
				
			}
			
			@Override
			public void onAnimationEnd(Animator animation) {
				// TODO Auto-generated method stub
			    if (!mCanceled) {
			        animation.start();
			      }
			}
			
			@Override
			public void onAnimationCancel(Animator animation) {
				// TODO Auto-generated method stub
				mCanceled = true;
			}
		});
		animator.start();

# Android Splash秒开以及Activity白屏，黑屏的原因和过程

* [ Android Splash页秒开]("http://blog.csdn.net/yanzhenjie1003/article/details/52201896") 

MultiDex的简要原理   

	我们以APK中有两个dex文件为例，第二个dex文件为classes2.dex。  
	兼容包在Applicaion实例化之后，会检查系统版本是否支持 multidex，classes2.dex是否需要安装。  
	如果需要安装则会从APK中解压出classes2.dex并将其拷贝到应用的沙盒目录下。  
	通过反射将classes2.dex注入到当前的classloader中。  

美团的解决方案(推荐后两篇文章)  

* [ Android拆分与加载Dex的多种方案对比(手机QQ，微信)]("http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207151651&idx=1&sn=9eab282711f4eb2b4daf2fbae5a5ca9a&3rd=MzA3MDU4NTYzMw==&scene=6#rd") 

手机QQ，微信的解决方案  

* [ 美团Android DEX自动拆包及动态加载简介 ]("http://tech.meituan.com/mt-android-auto-split-dex.html")  

综合解决方案  

* [ 其实你不知道MultiDex到底有多坑 ]("http://blog.zongwu233.com/the-touble-of-multidex")  

# 使用multidex时会出现的一些坑点，以及解决方式和部署步骤


汇集解决所有办法的关键文章与插件 

* [Android傻瓜式分包插件]("https://github.com/TangXiaoLv/Android-Easy-MultiDex") 

MultiDex的简要原理   

我们以APK中有两个dex文件为例，第二个dex文件为classes2.dex。  
	兼容包在Applicaion实例化之后，会检查系统版本是否支持 multidex，classes2.dex是否需要安装。  
	如果需要安装则会从APK中解压出classes2.dex并将其拷贝到应用的沙盒目录下。  
	通过反射将classes2.dex注入到当前的classloader中。  

Android Dex 分包并根据multidex源码解析(热修复的基础)  

* [ 少量multidex源码解析]("http://souly.cn/%E6%8A%80%E6%9C%AF%E5%8D%9A%E6%96%87/2016/02/25/android%E5%88%86%E5%8C%85%E5%8E%9F%E7%90%86/") 



Android Dex 分包指南  

* [ Android Dex 分包指南]("http://www.jianshu.com/p/b38124d332be") 

手机QQ，微信的解决方案  

* [ Android拆分与加载Dex的多种方案对比(手机QQ，微信)]("http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207151651&idx=1&sn=9eab282711f4eb2b4daf2fbae5a5ca9a&3rd=MzA3MDU4NTYzMw==&scene=6#rd") 

美团的解决方案(推荐后两篇文章)  

* [ 美团Android DEX自动拆包及动态加载简介 ]("http://tech.meituan.com/mt-android-auto-split-dex.html")  

综合解决方案  

* [ 其实你不知道MultiDex到底有多坑 ]("http://blog.zongwu233.com/the-touble-of-multidex")  

# 微信Tinker热修复原理
- 热修复基本原理
* [Android热修复原理(简单易懂)]("https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a") 


- dex包全量替换的原理  
  为什么是65536个方法，因为short只能表示65536个方法，具体计算，short两个字节代表16位，16位就是2的16次方。
- dex包全量替换的原理  
  因为这里他不能将新的dex直接加入补丁包中，这样会导致包大小太大，所以，这里他将差异包加入补丁包中(对比两个dex的差异)再将旧的dex和之前用bsdiff算法生成的差异对比后，形成一个新的dex，最后再解析出来，这样的话就很好的解决了这个问题。    
* [完整的方案以及tinker的原理]("https://www.qcloud.com/community/article/101") 
* [tinker源码分析]("http://www.w2bc.com/article/179241") 
* [dex文件格式及dexdiff原理]("http://sparkinlee.github.io/2016/08/dex%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E5%8F%8Adexdiff%E5%8E%9F%E7%90%86")  
* [微信热补丁 Tinker 的实践演进之路]("http://dev.qq.com/topic/57ad7a70eaed47bb2699e68e") 
* [和android app运行过程相关(how apps built and run)]("https://github.com/dogriffiths/HeadFirstAndroid/wiki/How-Android-Apps-are-Built-and-Run#put-classesdex-and-resources-into-a-package-file") 

# 一个Android Studio上的实用工具

这个工具是用来加入不同分辨率图片的工具  
* [ Android Drawable Importer的使用 ]("http://blog.inet198.cn/?ziwang_/article/details/51713623")  

这个工具是用来统计类里面方法数  
* [ APK method count的使用 ]("http://inloop.github.io/apk-method-count/")  

一个用来检测Android出现oom的库，测试，平时可用。  
* [  自动检测oom的库 ]("https://github.com/square/leakcanary")
# gradle一些有用的学习资料

* [ 如何编写一个gradle插件 ]("http://blog.csdn.net/sbsujjbcy/article/details/50782830")  

* [ gradle实现环境分离 ]("http://yifeng.studio/2016/09/06/apk-environment-separate")  

* [ 深入理解Android（一）：Gradle详解 ]("http://www.infoq.com/cn/articles/android-in-depth-gradle")  

# google上android的mvp结构基本实现方式以及如何规范的运用

[ 基于RxJava的一种MVP实现 ]("http://dev.qq.com/topic/57bfef673c1174283d60bac0")  

[ google规范mvp模式的运用 ]("https://github.com/googlesamples/android-architecture/")  

