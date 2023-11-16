### 前言

主要分享一下绘制的触发，在应用层的主要核心流程，及其中重要的概念

涉及的源码基于API29

### 1、屏幕绘制

当最初的view和window开始关联的时候，启动了屏幕绘制的流程。

通过WMS将Window展示到页面之际，初始化ViewRootImpl，然后将DecorView，ViewRootImpl，Window三者关联起来

```java
//ActivityThread.class
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,String reason) {
    ...
    //调用WindowManager添加view
    ViewManager wm = a.getWindowManager();
    wm.addView(decor, l);
    ...
}
//WindowManagerGlobal.class 相当于WindowManger的实现类，是一个单例的
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
     ...
     ViewRootImpl root;
     root = new ViewRootImpl(view.getContext(), display);
     root.setView(view, wparams, panelParentView);
     ...
}
//ViewRootImpl.class 
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    //触发绘制流程
    requestLayout();
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
           getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
           mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
           mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
    ...
}
```

#### 自定义View

三个流程方法，测量（onMeasure），布局（onLayout），绘制(onDraw)

触发绘制流程的方法有两种，一种是requestLayout(), 一种是invalidate()

在view中调用，最终都会调到ViewRootImpl代码中

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        //检查线程是否不一致，和初始化ViewRootImpl的线程比较
        checkThread();
        //标志已经请求布局绘制，才会触发measure和layout流程
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        //防止多次调用
        mTraversalScheduled = true;
        //同步屏障
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //向Choreographer中的队列加入任务
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}
//mTraversalRunnable中执行的任务
void doTraversal() {
    if (mTraversalScheduled) {
        //移除标志
        mTraversalScheduled = false;
        //移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        //开始执行绘制
        performTraversals();
    }
}
private void performTraversals() {
    ...
    performMeasure();
    performLayout();
    performDraw();
    ...
}
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}

//子view调用invalidate递归调用到ViewRootImpl
@Override
 public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
     checkThread();
     if (DEBUG_DRAW) Log.v(mTag, "Invalidate child: " + dirty);

     if (dirty == null) {
         invalidate();
         return null;
     } else if (dirty.isEmpty() && !mIsAnimating) {
         return null;
     }
     ...
     //去触发scheduleTraversals
     invalidateRectOnScreen(dirty);
     return null;
}
```

回到前面，严格的来说

requestLayout不一定触发draw，但是会触发measure和layout流程

invalidate因为没有设置mLayoutRequested，所以不会触发测量和布局

所以当我们进行View更新时，若仅View的显示内容发生改变且新显示内容不影响View的大小、位置，则只需调用invalidate方法；

若View宽高、位置发生改变且显示内容不变，只需调用requestLayout方法；

若两者均发生改变，则需调用两者，按照View的绘制流程，推荐先调用requestLayout方法再调用invalidate方法。

### 2、Choreographer 编舞者

```java
//接上面的绘制任务
private void postCallbackDelayedInternal(int callbackType,
       Object action, Object token, long delayMillis) {
  ...
   synchronized (mLock) {
       final long now = SystemClock.uptimeMillis();
       final long dueTime = now + delayMillis;
    //往队列中插入绘制任务
       mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
//功能一致，只是一个是异步队列处理，一个是直接处理
       if (dueTime <= now) {
           scheduleFrameLocked(now);
      } else {
           Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
           msg.arg1 = callbackType;
           msg.setAsynchronous(true);
           mHandler.sendMessageAtTime(msg, dueTime);
      }
  }
}
private void scheduleFrameLocked(long now) {
 if (!mFrameScheduled) {
   mFrameScheduled = true;
   //是否使用脉冲模式，从Android4.1开始估计这个就一直是true了
   if (USE_VSYNC) {
    //去注册VSYNC信号
       if (isRunningOnLooperThreadLocked()) {
           scheduleVsyncLocked();
      } else {
           Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
           msg.setAsynchronous(true);
           mHandler.sendMessageAtFrontOfQueue(msg);
      }
} else {
   final long nextFrameTime = Math.max(
      mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
    //直接开始执行doFrame方法
    Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, nextFrameTime);
    }
}
}
//VSYNC信号回调
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
           implements Runnable {
@Override
 public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
  ...
     Message msg = Message.obtain(mHandler, this);
     msg.setAsynchronous(true);
     mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}
   @Override
 public void run() {
     mHavePendingVsync = false;
  //触发doFrame处理
     doFrame(mTimestampNanos, mFrame);
}
}
void doFrame(long frameTimeNanos, int frame) {
  ...
   try {
       Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
       AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

       mFrameInfo.markInputHandlingStart();
       doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

       mFrameInfo.markAnimationsStart();
       doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
       doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);

       mFrameInfo.markPerformTraversalsStart();
    //开始之前注册的绘制任务
       doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

       doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
  } finally {
       AnimationUtils.unlockAnimationClock();
       Trace.traceEnd(Trace.TRACE_TAG_VIEW);
  }
...
}
```

### 总结下流程：

1. 一个Window对应一个Surface，对应一个ViewRootImpl，以及DecorView的根View
2. 请求绘制最终会将消息传递给ViewRootImpl处理，此处会将消息设置为异步消息，绘制期间产生的都是异步消息
3. 会将绘制任务放入队列中，注册Vsync信号通知
4. 信号到达后，触发绘制任务出队列，然后就是熟知的measure，layout，draw流程了

### 3、RenderThread 渲染线程

定义：RenderThread是Android系统中的一个专门用于处理UI渲染工作的线程。在Android 5.0（Lollipop）及以上版本中，Google引入了这个新的线程，以帮助提高应用的性能和流畅度。

在之前的版本中，所有的UI更新都是在主线程（也称为UI线程）中进行的。这意味着，如果主线程正在处理其他任务（如数据计算或网络请求），UI渲染就可能会被阻塞，导致界面卡顿或延迟。

引入RenderThread后，UI渲染工作可以在一个单独的线程中进行，而不会影响主线程的其他任务。这样，即使主线程忙于处理其他任务，UI也能保持流畅。

作用：RenderThread主要负责GPU绘制工作，包括OpenGL命令的执行、硬件加速的绘制等。它可以并行于主线程和其他工作线程运行，提高了UI渲染的效率和性能。

接上面所述，performDraw流程触发最终的应用层绘制，会走到核心的分发

```java
//ViewRootImpl.class 由performDraw流程触发
private boolean draw(boolean fullRedrawNeeded) {
  	...
	//是否启用硬件加速
	if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
		mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
	} else {
		drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets);
	}
  	...
}
```

当启动硬件加速后，Android 使用 “display list” 组件进行绘制而非直接使用 CPU 绘制每一帧。Display List 是一系列绘制操作的记录，抽象为 RenderNode 类

```java
//ThreadedRenderer.class
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
  	...
    //更新DisplayList
    updateRootDisplayList(view, callbacks);
    //触发渲染线程
    int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
  	...
}

private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
	...
  	updateViewTreeDisplayList(view);
	...
}

private void updateViewTreeDisplayList(View view) {
  	view.mPrivateFlags |= View.PFLAG_DRAWN;
  	view.mRecreateDisplayList=(view.mPrivateFlags&View.PFLAG_INVALIDATED
    		== View.PFLAG_INVALIDATED;
  	view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
	//触发子view迭代,从此处就会调起自定义View的onDraw方法了
  	view.updateDisplayListIfDirty();
  	view.mRecreateDisplayList = false;
}
//View.class
public RenderNode updateDisplayListIfDirty() {
			...
			//RecordingCanvas是专供于生成RenderNode的画布
            final RecordingCanvas canvas = renderNode.beginRecording(width, height);
            try {
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    buildDrawingCache(true);
                    Bitmap cache = getDrawingCache(true);
                    if (cache != null) {
                        canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                    }
                } else {
                    ...
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        dispatchDraw(canvas);
                        drawAutofilledHighlight(canvas);
                        if (mOverlay != null && !mOverlay.isEmpty()) {
                            mOverlay.getOverlayView().draw(canvas);
                        }
                        if (isShowingLayoutBounds()) {
                            debugDrawFocus(canvas);
                        }
                    } else {
						//调用draw，触发view绘制的迭代流程
                        draw(canvas);
                    }
                }
            } finally {
                renderNode.endRecording();
                setDisplayListProperties(renderNode);
            }
        } else {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        }
        return renderNode;
    }
```

最后会触发syncAndDrawFrame方法，通知渲染线程开始工作，其核心实现位于 libhwui 库里面

#### 主线程和渲染线程的分工

主线程负责处理进程 Message、处理 Input 事件、处理 Animation 逻辑、处理 Measure、Layout、Draw ，更新 DIsplayList ，但是不涉及 SurfaceFlinger 打交道；

渲染线程负责渲染渲染相关的工作，一部分工作也是 CPU 来完成的，一部分操作是调用 OpenGL 函数来完成的

当启动硬件加速后，在 Measure、Layout、Draw 的 Draw 这个环节，Android 使用 DisplayList 进行绘制而非直接使用 CPU 绘制每一帧。DisplayList 是一系列绘制操作的记录，抽象为 RenderNode 类

### 4、SurfeceFlinger

官方定义：SurfaceFlinger 接受缓冲区，对它们进行合成，然后发送到屏幕

- 大多数应用在屏幕一共显示3层：状态栏，导航栏（底部或侧面），应用界面。有的应用拥有更多或更少的层
- 状态栏和导航栏由系统进行渲染，应用界面由应用渲染。
- App部分的绘制流程负责生产出合成需要的Surface

工作流程：

当 VSYNC 信号到达时，SurfaceFlinger 会遍历它的层列表，以寻找新的缓冲区。如果找到新的缓冲区，它会获取该缓冲区；否则，它会继续使用以前获取的缓冲区。

SurfaceFlinger 必须始终显示内容，因此它会保留一个缓冲区。如果在某个层上没有提交缓冲区，则该层会被忽略。SurfaceFlinger 在收集可见层的所有缓冲区之后，便会询问 Hardware Composer 应如何进行合成

### 总结后续流程

是否开启了硬件加速

是：

1. Android 使用 “display list” 组件进行绘制，对画布(Canvas)进行一系列操作后，会记录下绘制操作，这个记录就是RenderNode
2. 之后会将数据交给RenderThread线程，内部会调用GPU绘制命令，做数据栅格化
3. 然后SurfeceFlinger会将缓冲区数据合并，最后将合成的数据交给硬件的缓冲区
4. 屏幕以固定频率从前缓冲区拿出数据渲染，渲染完毕后发送VSYNC，此时前后缓冲区数据交换，屏幕绘制下一帧

否：

1.  直接在draw流程中调用drawSoftware绘制view，对画布(Canvas)进行一系列操作，直接在内存中绘制（实际是内存中的Bitmap上做绘制）
2. 后续流程同上面3，4

Tips：

- Android从3.0引入硬件加速功能，从4.0默认开启了硬件加速
- 硬件加速能够提升性能，提高系统流畅性，但是会增加资源的消耗，如果没有复杂的图形渲染，可以尝试关闭硬件加速以达到节省内存和电量的目的。
- 并非所有的2D绘图操作都可以被硬件加速。有些操作在开启硬件加速后可能无法正常工作，或者效果与预期不符

### Vsync和三缓冲

两个概念

- 屏幕刷新率(Hz): 屏幕在一秒内刷新的次数，Android手机一般都是60Hz，也就是一秒刷新60次，当然也有高刷的，但是60Hz足矣
- 帧速率(FPS): cpu在一秒内合成的帧数，比如60FPS，就是60 frame per sconds，意思就是一秒合成60帧。 如上所述，当屏幕刷新率大于帧速率的时候，会发生卡顿；屏幕刷新率小于帧速率的时候，会发生撕裂。那么怎么解决这个问题呢，我们一个一个来解决，先来看撕裂

我们知道，撕裂是因为: cpu太快 从而导致 屏幕还没渲染完毕 就把正在渲染的数据 给覆盖掉了，那么我们可以限制cpu的速度吗？当然可以，但是不划算，因为这样就等于把cpu的长处给扼杀了，所以我们只要让cpu的数据不覆盖掉屏幕正在渲染的数据即可，也就是说，给cpu新来的数据提供一个存放点，而不是往帧缓冲里面写，这个存放点叫做**后缓冲(BackBuffer)**，相应的，**帧缓冲(FrameBuffer)也叫做前缓冲**，这样，cpu新来的数据就会放在后缓冲，而屏幕则继续从前缓冲取数据来渲染，等到后缓冲数据写入完了，前后缓冲的数据就会交换，屏幕此时读取的数据就是后缓冲的数据，也就是下一帧的数据，循环往复，我们就看到了画面。但是！还是不行，举个列子，如果cpu非常快，前缓冲数据还没刷新完毕，后缓冲已经写满，此时，就会交换数据，又发生了撕裂！那么怎么办呢？

![没有VSYNC](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f176b8fcc795432bb49c91d5308bb238~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

从图中可以看到: 没有vsync的情况下，cpu在任意地方开始，随心所欲!

我们追究原因: 核心点在与数据交换的时机由谁来控制，**数据交换的发生点应该是在屏幕渲染完一帧后，而不是cpu写入一帧数据后**，所以，控制数据是否交换应该由屏幕来决定，但是！计算机五大组成部分各司其职，屏幕只是输出设备和输入设备(因为能触屏)，他不是控制器，如何控制数据的交换呢？当然可以，答案就是:VSYNC。

VSYNC(vertical sync): 也就是垂直同步，当屏幕渲染完一帧数据后，即将开始渲染下一帧之前，发出的一个同步信号。

cpu只要监听VSYNC信号，接收到信号后再开始交换后缓冲和前缓冲的数据，就等价于屏幕控制了数据交换，也就解决了撕裂问题，这很明显是设计模式中的监听器模式。

现在我们来捋一下流程:

- 1 屏幕正在从前缓冲读取第一帧数据并渲染，此时cpu计算完第二帧数据，放在后缓冲，等待VSYNC信号。
- 2 屏幕将第一帧数据渲染完毕，发出VSYNC信号，cpu收到VSYNC信号，将后缓冲的第二帧数据复制到前缓冲。
- 3 同时屏幕继续绘制第二帧数据，cpu开始计算下一帧数据，循环往复。

![没有VSYNC](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01c031299de3461b8c1c26c9cfc8a750~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

从图中可以看到，有了VSYNC，cpu总是在指定的地方开始。

有人会问: 说白了，真正解决问题的是VSYNC，而不是双缓冲，那不要双缓冲只要VSYNC不是也可以吗？

好，我们假设只有VSYNC，现在假设屏幕正在渲染数据，而cpu在等VSYNC信号，屏幕将数据渲染完毕后，发送VSYNC信号，cpu收到信号后，就去计算数据，计算完后才会写入帧缓冲，那么，在cpu计算数据这段时间内，屏幕干什么呢？嗯，它接着刷新帧缓冲的数据，反正cpu还没有将新数据计算完毕刷入帧缓冲，所以还是上一帧的数据，这样就会卡顿，说白了，有双缓冲的情况下，cpu使用后缓冲计算数据，屏幕使用前缓冲渲染数据，两者可以同时工作，你计算一个我渲染一个，典型的"生产者消费者模式"，只不过使用VSYNC信号来进行数据的交换；而没有双缓冲的情况下，两者需要排队使用帧缓冲，不能同时工作，就变成了我等着你计算，你计算完了等着我渲染，VSYNC此时的作用就是进行排队，这样会大大增加卡顿率，所以: VSYNC真正**解决了撕裂问题**，而双缓**冲优化了卡顿问题**。

那么，怎么解决卡顿问题呢？答曰: 无法根本解决，只能优化!

### 优化卡顿问题(多缓冲)

我们知道，卡顿是因为**帧速率<屏幕刷新率**，这是不严谨的，准确的说应该是因为:**帧速率<60fps**，因为现在屏幕刷新率基本都是60hz的，所以帧速率只要取下限60fps即可，换句话说，1秒内需要计算60个帧，也就是16.7ms就能计算完一帧。如果计算不完，那么在一个vsync信号过来后，cpu还在计算，缓冲区的数据并没有改变，就还是老数据，屏幕就又把老数据刷新一遍，就出现了卡顿，所以，cpu要尽可能在16.7ms内把所有数据计算完准备好，以等待vsync信号过来后直接交换数据。



![双缓冲卡顿问题](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/151f71e0738d4ed88d26585f6afb2f79~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

我们可以将Display那一行看作是前缓冲，将GPU和CPU两行叠加起来看作是后缓冲(因为它俩排队使用)，将VSYNC线隔离开的竖行看作一个帧。

我们看到，在第一帧里面，GPU墨迹了半天没搞完，以至于在第二帧里面，Display(屏幕)显示的还是第一帧的A数据，此时就产生了Jank(卡顿)，并且在一个vsync信号过来后，cpu什么都没做，因为gpu占着后缓冲(那个绿色的长B块)，所以cpu只能再等下一个vsync，在下一个vsync里面，cpu终于拿到了后缓冲的使用权，但是cpu计算时间比较长，导致了gpu时间不够用，数据又没算完，再次发生了卡顿，可以说，这次卡顿直接受到了第一次卡顿的影响，试想: 如果在第一次卡顿的时候，cpu也能计算数据，那么，第二次卡顿可能就不存在了，因为cpu已经在第一次卡顿的时候把蓝色的A给计算完了，第二次完全可以让gpu独自计算(绿色的A)，就不存在因为排队导致的时间不够用了，但是！cpu和gpu共用后缓冲，这就导致它们只能轮流使用后缓冲，怎么解决呢？再加一个后缓冲区，让cpu、gpu各用一块。我们来看引入三缓冲后的效果:

![引入三缓冲](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61a74ac470404940bf22ca02fe69390c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

我们看到，在第一次jank内，cpu使用了第三块缓冲区，自己计算了C帧的数据，假如此时没有三缓冲，那么cpu就只能再继续等下一个vsync信号，也就是在图中蓝色A块的地方，才能开始计算C帧数据，就又引发下一次卡顿。我们看到，通过引入三缓冲，虽然不能避免卡顿问题，但是却可以大幅优化卡顿问题，尤其是避免连续卡顿，但是，三缓冲也有缺点，就是耗资源，所以系统并非一直开启三缓冲，要想真正解决问题，还需要在cpu层对数据尽量优化，从而减小cpu和gpu的计算量，比如:View尽量扁平化，少嵌套，少在UI线程做耗时操作等。

Tips：

- Android 4.1引入了黄油计划(VSYNC)，上层开始接收VSYNC(Choreographer)，并且加入了三缓冲.
- VSYNC不仅控制了后缓冲和前缓冲的数据交换，还控制了cpu何时开始进行绘制计算。

参考链接:

1、https://juejin.cn/post/7017377999385280526
