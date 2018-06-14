# 聊聊为什么在activity启动的时候获取不到View的宽高



一大早看到一篇鸿洋推了一篇博客《[onResume中Handler.post(Runnable)为什么获取不到宽高？](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650825566&idx=1&sn=95e855a44154b01fe23c37ea67c60a3d&chksm=80b7b7c0b7c03ed6c9b76421115ebdf553b9526b402d1eb2de059b42a0b1f1f329db50b24de6&mpshare=1&scene=1&srcid=06011fUFNDRtTi4rVS5Zg5Ka&key=a6892775801862d85b8f5955a290eed543683c4c4aca55c5579cd99937d30ce7d168fa120869d42426f066c7da6ed12c4ca5fbc8fd88d70214ce724f9c462e711b260860f8734c388c2e54f3e970ddc6&ascene=0&uin=MTc3NjI5OTcyMA%3D%3D&devicetype=iMac+MacBookPro11%2C4+OSX+OSX+10.12.6+build(16G29)&version=12020810&nettype=WIFI&lang=zh_CN&fontScale=100&pass_ticket=CK8diJkl62YNYMga0AepFGAqdxxtsEd0LoGbQtTQNJs47BqhXsal%2FrnhJ6M8dO5x)》，细细读了两遍，有些收获，但依然有些不明白的地方。于是自己梳理了一下，在翻看源码后我得出了一个这样的结论。

**在Activty启动的相关生命周期中提交到MainLooper的Message会在整个视图树注册时钟信号(垂直同步)之前处理，而且整个视图树会在注册后获得时钟信号的时候才去递归遍历进行测量和布局。**

PS：没看过Activity启动源码和setCotentView源码阅读本文可能引起不适

<!--more-->

## Activity的启动和视图树的创建

关于Activity的启动我们从`ActivityThread`中的`Handler`收到启动消息`LAUNCH_ACTIVITY`说起。这一块就带大家看源码了，建议大家自己打开源码按着以下顺序自行阅读了解整个流程，这样能加深印象。

简单概括一下我们需要关注的整个流程。在`handleLaunchActivity`方法中调用`performLaunchActivity`创建并启动`Activity`，接着调用`handleResumeActivity`方法，该方法最终会回调`Activity`的`onResume`方法，在回调了`onResume`之后，会调用`WindowManger`的`addView`方法把`decorView`添加进去以此构建整个视图树。

先来回顾一下，`decorView`是在哪里创建的？在`onCreate`方法中调用`setContentView`就会构建`decorView`。但这个时候这个顶层容器还没有被绘制到屏幕上。

贴一下自己画的时序图。

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/activity启动.png)

思考一下，不管在`onCreate`还是`onResume`甚至是`onStart`，发送给`MainLooper`的消息都会在执行完`LAUNCH_ACTIVITY`之后才得到处理。通过上面的时序图可以看到，在回调了`onResume`之后，`decorView`也已经通过`WindowManger`被添加进了`ViewRootImpl`。这个时候依然获取不到控件宽高，说明还没有执行整个视图的绘制流程。可是组件的启动已经处理完了，这里我理解为这个`Activity`已经启动了并且用户可见了，但这时候还没绘制视图的话什么时候绘制呢？

这就要说说**垂直同步信号**了。

Android系统每隔16ms会发出VSYNC信号绘制界面。如果我们在16ms内没有完成绘制，就会展示上一帧的画面，画面就出现了掉帧。我简单查了一下，这个信号是由native层的核心服务`SurfaceFlinger`发出的。

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/60fps.png)

当Java层收到垂直同步信号，会导致整个视图树的绘制，只有当整个视图树绘制完毕后，我们才能获取控件的宽高。

那这个时候问题又来了，垂直信号是如何通知视图树去绘制的？



## ViewRootImpl#requestLayout到底干了什么

通过上面的时序图，我们已经了解到最后会构建一个`ViewRootImpl`对象并把`decorView`添加进去。接下来我们看看垂直信号是如何和视图树的绘制关联到一起的，先看一下时序图。

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/requestlayout.png)

这里带着大家看看代码。

ViewRootImpl.class

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

MessageQueue.class

```
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

`ViewRootImpl.class`第4行调用了`postSyncBarrier`方法，这个方法在`messageQueue`中**插入了一条sync barrier消息**。这个`Message`的特殊之处在于没有对应的处理消息的`target`，熟悉Handler机制的应该知道，你发送消息的时候，`message`中的`target`就是你发送消息的`handler`。因为该消息不依赖handler去发送消息，因此也理所应当不需要设置`target`。

第5行调用了`Choreographer`对象`postCallback`方法，最后调用到了`postCallbackDelayedInternal`。

**这个Choreographer就是一个时钟信号的接收者和处理者。**

在Choreographer中有三种消息

- MSG_DO_FRAME：开始渲染下一帧的操作
- MSG_DO_SCHEDULE_VSYNC：请求Vsync信号
- MSG_DO_SCHEDULE_CALLBACK：请求执行callback

Choreographer.class

```java
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            //这里把回调加入了队列
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
           
            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);//设置为异步消息
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
```

首先会**把回调存入一个队列**中，接着判断这个消息是否已经超时。根据我看源码的流程，这样走下来应该会dueTime是等于now的，也就是调用了`scheduleFrameLocked`方法。但我们还是先看看另外一半干了什么，调用`setAsynchronous`设置该消息为异步消息，并将之前的runnable作为回调。

这个`action`就是上面传过来的`mTraversalRunnable`，而`TraversalRunnable`是`ViewRootImpl`的内部类。来看看它的实现。在其内部调用了`doTraversal`方法，**这里就是整个视图树测量、布局的起点**。

```
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
```

那`scheduleFrameLocked`干了什么呢？

在`scheduleFrameLocked`中调用了`scheduleVsyncLocked`方法

接着调用`mDisplayEventReceiver.scheduleVsync()`，这是一个native方法，入参是个句柄。我的理解是注册了一个垂直同步信号的监听器。当垂直同步信号发送过来的时候，会通过`FrameDisplayEventReceiver#onVsync`接收，接着发送消息到主线程，请求执行`doFrame`。



```java
    void doFrame(long frameTimeNanos, int frame) {
		//..省略大量代码
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

```

在`doFrame`最后调用了`doCallbacks`，这个方法是把队列中的回调依次执行。还记得我们上面存入队列的`TraversalRunnable`么？至此当垂直信号到来的时候，就会最终回调到`doTraversal`遍历测量整个视图。



## 一些思考和补充

问：这个回调已经持久化的保存在了`Choreographer`中，而每16ms的垂直同步信号为什么不会导致布局每16ms就测量一次呢？？

答：对于这个问题我的理解有两点，不一定对

1. 在`Choreographer`执行完一次回调后，会回收所有的回调。关于这一点暂时没找到其他博客作为依据，只是看到在执行完回调后调用了`recycleCallbackLocked`方法。
2. 调用view的requestLayout方法会修改`mPrivateFlags`标志位。而在 `performLayout`中会请求重新布局的view。如果这个标志位在一次绘制后设置为不请求绘制，则下次垂直信号来的时候并不会重回这些布局。



问：上文说到接收到垂直同步信号后会发送消息到主线程请求执行doFrame，这个流程是怎么样的？

答：这里就要说回之前的**sync barrier消息**

在MessageQueue.class的`next`方法中有这么一段代码。如果mst的target为null，说明该message为sync barrier消息。

```java
	Message msg = mMessages;
		if (msg != null && msg.target == null) {
			// Stalled by a barrier.  Find the next asynchronous message in the queue.
			do {
				prevMsg = msg;
				msg = msg.next;
				} while (msg != null && !msg.isAsynchronous());
			}
```

当碰到这样一条消息的时候，就会轮询去找下一条异步消息。回到刚才接收垂直同步信号的地方，当`FrameDisplayEventReceiver`接收到垂直信号，会回调`onVsync`。上文我们提高过，`Choreographer`会通过`FrameDisplayEventReceiver#onVsync`接收，接着发送消息到主线程，请求执行`doFrame`。而这个消息会被设置为异步消息。也就是说当我们的**消息队列中有sync barrier消息才会执行这个异步消息**

```java
        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
			//...省略部分代码
            mTimestampNanos = timestampNanos;
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);//设置成异步消息在这里出现很多次了
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

```



参考资料：

[Android Choreographer 源码分析](https://www.jianshu.com/p/996bca12eb1d)