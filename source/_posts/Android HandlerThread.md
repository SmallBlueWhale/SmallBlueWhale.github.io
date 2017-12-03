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

**Handlerthread的源码其实就只有一点点，主要的函数只有重载的run函数以及getLoopper()函数**

### run()函数
```
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

#### getLoopper()函数


    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason is isAlive() returns false, this method will return null. If this thread 
     * has been started, this method will block until the looper has been initialized.  
     * @return The looper.
     */
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }   
        return mLooper;
    }

