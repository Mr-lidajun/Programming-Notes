# 第8章 理解Window和WindowManager
> **Window表示一个窗口的概念**，Window是一个抽象类，它的具体实现是PhoneWindow。**通过WindowManager创建一个Window**。WindowManager是外界访问Window的入口，Window的具体实现位于WindowManagerService中，**WindowManager和WindowManagerService的交互是一个IPC过程**。Android中所有的视图都是通过Window来呈现的，不管是Activity、Dialog还是Toast，它们的视图实际上都是附加在Window上的，因此**Window实际是View的直接管理者**。从第4章中所讲述的View的事件分发机制也可以知道，单击事件由Window传递给DecorView，然后再由DecorView传递给我们的View，就连Activity的设置视图的方法setContentView在底层也是通过Window来完成的。

## 8.1 Window和WindowManager
如何使用WindowManager添加一个Window？

```java
mFloatingButton = new Button(this);
mFloatingButton.setText("button");
mLayoutParams =new WindowManager.LayoutParams(
LayoutParams.WRAP_CONTENT,LayoutParams.WRAP_CONTENT,0,0,PixelFormat.TRANSPARENT);
mLayoutParams.flags =LayoutParams.FLAG_NOT_TOUCH_MODAL |LayoutParams.FLAG_NOT_FOCUSABLE |LayoutParams.FLAG_SHOW_WHEN_LOCKED;
mLayoutParams.gravity =Gravity.LEFT |Gravity.TOP;
mLayoutParams.x =100;
mLayoutParams.y =300;
mWindowManager.addView(mFloatingButton,mLayoutParams);
```

上述代码可以将一个Button添加到屏幕坐标为（100，300）的位置上。

- Flags参数表示Window的属性，这些选项可以控制Window的显示特性：

    **FLAG_NOT_FOCUSABLE**
    表示Window不需要获取焦点，也不需要接收各种输入事件，此标记会同时启用FLAG_NOT_TOUCH_MODAL，最终事件会直接传递给下层的具有焦点的Window。
    
    **FLAG_NOT_TOUCH_MODAL**
    在此模式下，系统会将当前Window区域以外的单击事件传递给底层的Window，当前Window区域以内的单击事件则自己处理。这个标记很重要，一般来说都需要开启此标记，否则其他Window将无法收到单击事件。
    
    **FLAG_SHOW_WHEN_LOCKED**
    开启此模式可以让Window显示在锁屏的界面上。

- Type参数表示Window的类型，Window有三种类型，分别是应用Window、子Window和系统Window。
    应用类Window对应着一个Activity。子Window不能单独存在，它需要附属在特定的父Window之中，比如常见的一些Dialog就是一个子Window。系统Window是需要声明权限在能创建的Window，比如Toast和系统状态栏这些都是系统Window。

- Window是分层的
    每个Window都有对应的z-ordered，层级大的会覆盖在层级小的Window的上面，这和HTML中的z-index的概念是完全一致的。这些层级范围对应着WindowManager.LayoutParams的type参数。

    - 应用Window：1~99
    - 子Window：1000~1999
    - 系统Window：2000~2999

如果想要Window位于所有Window的最顶层，那么采用较大的层级即可。显然系统Window的层级最大。一般我们可以选用TYPE_SYSTEM_OVERLAY或者TYPE_SYSTEM_ERROR，如果采用TYPE_SYSTEM_ERROR，只需要为type参数指定这个层级即可：mLayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR；同时声明权限：<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />。因为系统类型的Window是需要检查权限的

WindowManager所提供的功能很简单，常用的只有三个方法，即添加View、更新View和删除View，这三个方法定义在ViewManager中，而WindowManager继承了ViewManager。

```java
public interface ViewManager{
    public void addView(View view,ViewGroup.LayoutParams params);
    public void updateViewLayout(View view,ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

WindowManager操作Window的过程更像是在操作Window中的View。我们时常见到那种可以拖动的Window效果，其实是很好实现的，只需要根据手指的位置来设定LayoutParams中的x和y的值即可改变Window的位置。首先给View设置onTouchListener：mFloatingButton.setOnTouchListener(this)。然后在onTouch方法中不断更新View的位置即可：

```java
public boolean onTouch(View v,MotionEvent event) {
    int rawX = (int) event.getRawX();
    int rawY = (int) event.getRawY();
    switch (event.getAction()) {
    case MotionEvent.ACTION_MOVE: {
        mLayoutParams.x = rawX;
        mLayoutParams.y = rawY;
        mWindowManager.updateViewLayout(mFloatingButton,mLayoutParams);
        break;
    }
    default:
        break;
    }
    return false;
}
```

## 8.2 Window的内部机制

Window是一个抽象的概念，每一个Window都对应着一个View和一个ViewRootImpl，Window和View通过ViewRootImpl来建立联系，因此Window并不是实际存在的，它是以View的形式存在。这点从WindowManager的定义也可以看出，它提供的三个接口方法addView、updateViewLayout以及removeView都是针对View的，这说明**View才是Window存在的实体**。实际使用中无法直接访问Window，对Window的访问必须通过WindowManager。

### 8.2.1 Window的添加过程

Window的添加过程需要通过WindowManager的addView来实现，WindowManager是一个接口，它的真正实现是WindowManagerImpl类。在WindowManagerImpl中Window的三大操作的实现如下：

```java
@Override
public void addView(View view,ViewGroup.LayoutParams params) {
    mGlobal.addView(view,params,mDisplay,mParentWindow);
}
@Override
public void updateViewLayout(View view,ViewGroup.LayoutParams params) {
    mGlobal.updateViewLayout(view,params);
}
@Override
public void removeView(View view) {
    mGlobal.removeView(view,false);
}
```

可以发现，WindowManagerImpl并没有直接实现Window的三大操作，而是全部交给了WindowManagerGlobal来处理，WindowManagerGlobal以工厂的形式向外提供自己的实例，在WindowManagerGlobal中有如下一段代码：private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance()。**WindowManagerImpl这种工作模式是典型的桥接模式，将所有的操作全部委托给WindowManagerGlobal来实现。**WindowManager-Global的addView方法主要分为如下几步。

1. **检查参数是否合法，如果是子Window那么还需要调整一些布局参数**

   ```java
   if (view == null) {
       throw new IllegalArgumentException("view must not be null");
   }
   if (display == null) {
       throw new IllegalArgumentException("display must not be null");
   }
   if (!(params instanceof WindowManager.LayoutParams)) {
       throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
   }
   final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)    params;
   if (parentWindow != null) {
       parentWindow.adjustLayoutParamsForSubWindow(wparams);
   }
   ```

2. **创建ViewRootImpl并将View添加到列表中**

   在WindowManagerGlobal内部有如下几个列表比较重要：

   ```java
   private final ArrayList<View> mViews = new ArrayList<View>();
   private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
   private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.LayoutParams>();
   private final ArraySet<View> mDyingViews = new ArraySet<View>();
   ```

   在上面的声明中，mViews存储的是所有Window所对应的View，mRoots存储的是所有Window所对应的ViewRootImpl，mParams存储的是所有Window所对应的布局参数，而mDyingViews则存储了那些正在被删除的View对象，或者说是那些已经调用removeView方法但是删除操作还未完成的Window对象。在addView中通过如下方式将Window的一系列对象添加到列表中：

   ```java
   root = new ViewRootImpl(view.getContext(),display);
   view.setLayoutParams(wparams);
   mViews.add(view);
   mRoots.add(root);
   mParams.add(wparams);
   ```

3. **通过ViewRootImpl来更新界面并完成Window的添加过程**

   这个步骤由ViewRootImpl的setView方法来完成，从第4章可以知道，View的绘制过程是由ViewRootImpl来完成的，这里当然也不例外，在setView内部会通过requestLayout来完成异步刷新请求。在下面的代码中，scheduleTraversals实际是View绘制的入口：

   ```java
   public void requestLayout() {
       if (!mHandlingLayoutInLayoutRequest) {
           checkThread();
           mLayoutRequested = true;
           scheduleTraversals();
       }
   }
   ```

   接着会通过WindowSession最终来完成Window的添加过程。在下面的代码中，mWindowSession的类型是IWindowSession，它是一个Binder对象，真正的实现类是Session，也就是Window的添加过程是一次IPC调用。

   ```java
   try {
       mOrigWindowType = mWindowAttributes.type;
       mAttachInfo.mRecomputeGlobalAttributes = true;
       collectViewAttributes();
       res = mWindowSession.addToDisplay(mWindow,mSeq,mWindowAttributes,
                       getHostVisibility(),mDisplay.getDisplayId(),
                       mAttachInfo.mContentInsets,mInputChannel);
   } catch (RemoteException e) {
       mAdded = false;
       mView = null;
       mAttachInfo.mRootView = null;
       mInputChannel = null;
       mFallbackEventHandler.setView(null);
       unscheduleTraversals();
       setAccessibilityFocus(null,null);
       throw new RuntimeException("Adding window failed",e);
   }
   ```

   在Session内部会通过WindowManagerService来实现Window的添加，代码如下所示。

   ```java
   public int addToDisplay(IWindow window,int seq,WindowManager.LayoutParams attrs, 
                           int viewVisibility,int displayId,Rect outContentInsets,
                           InputChannel outInputChannel) {
       return mService.addWindow(this,window,seq,attrs,viewVisibility,displayId,
               outContentInsets,outInputChannel);
   }
   ```

   如此一来，Window的添加请求就交给WindowManagerService去处理了，在Window-ManagerService内部会为每一个应用保留一个单独的Session。

### 8.2.2 Window的删除过程
Window的删除过程和添加过程一样，都是先通过WindowManagerImpl后，再进一步通过WindowManagerGlobal来实现的。

```java
public void removeView(View view, boolean immediate) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }

    synchronized (mLock) {
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }

        throw new IllegalStateException("Calling with view " + view
                + " but the ViewAncestor is attached to " + curView);
    }
}
```

removeView的逻辑很清晰，首先通过findViewLocked来查找待删除的View的索引，这个查找过程就是建立的数组遍历，然后再调用removeViewLocked来做进一步的删除，如下所示。

```java
private void removeViewLocked(int index, boolean immediate) {
    ViewRootImpl root = mRoots.get(index);
    View view = root.getView();

    if (view != null) {
        InputMethodManager imm = InputMethodManager.getInstance();
        if (imm != null) {
            imm.windowDismissed(mViews.get(index).getWindowToken());
        }
    }
    boolean deferred = root.die(immediate);
    if (view != null) {
        view.assignParent(null);
        if (deferred) {
            mDyingViews.add(view);
        }
    }
}
```

removeViewLocked是通过ViewRootImpl来完成删除操作的。在WindowManager中提供了两种删除接口removeView和removeViewImmediate，它们分别表示异步删除和同步删除


```java
boolean die(boolean immediate) {
    // Make sure we do execute immediately if we are in the middle of a traversal or the damage
    // done by dispatchDetachedFromWindow will cause havoc on return.
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }

    if (!mIsDrawing) {
        destroyHardwareRenderer();
    } else {
        Log.e(TAG, "Attempting to destroy the window while drawing!\n" +
                "  window=" + this + ", title=" + mWindowAttributes.getTitle());
    }
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}
```

如果是异步删除，那么就发送一个MSG_DIE的消息，如果是同步删除（立即删除）。在doDie内部会调用dispatchDetachedFromWindow方法，真正删除View的逻辑在dispatchDetachedFromWindow方法的内部实现。
dispatchDetachedFromWindow方法主要做四件事：

- （1）垃圾回收相关的工作，比如清除数据和消息、移除回调。
- （2）通过Session的remove方法删除Window：mWindowSession.remove(mWindow)，这同样是一个IPC过程，最终会调用WindowManagerService的removeWindow方法。
- （3）调用View的dispatchDetachedFromWindow方法，在内部会调用View的onDetached-FromWindow()以及onDetachedFromWindowInternal()。对于onDetachedFromWindow()大家一定不陌生，当View从Window中移除时，这个方法就会被调用，可以在这个方法内部做一些资源回收的工作，比如终止动画、停止线程等。
- （4）调用WindowManagerGlobal的doRemoveView方法刷新数据，包括mRoots、mParams以及mDyingViews，需要将当前Window所关联的这三类对象从列表中删除。

### 8.2.3 Window的更新过程

```java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

    view.setLayoutParams(wparams);

    synchronized (mLock) {
        int index = findViewLocked(view, true);
        ViewRootImpl root = mRoots.get(index);
        mParams.remove(index);
        mParams.add(index, wparams);
        root.setLayoutParams(wparams, false);
    }
}
```

WindowManagerGlobal的updateViewLayout方法 -> ViewRootImpl的setLayoutParams方法 -> 在ViewRootImpl中会通过scheduleTraversals方法来对View重新布局，包括测量、布局、重绘这三个过程。 -> WindowSession更新Window视图 -> WindwoManagerService的relayoutWindow，IPC过程

## Window的创建过程
> View是Android中的视图的呈现方式，但是View不能单独存在，它必须附着在Window这个抽象的概念上面，因此有视图的地方就有Window。有视图的地方就有Window，因此Activity、Dialog、Toast等视图都对应着一个Window。

### 8.3.1 Activity的Window创建过程
Activity的启动过程很复杂，最终会由ActivityThread中的performLaunchActivity()来完成整个启动过程，在这个方法内部会通过类加载器创建Activity的实例对象，并调用其attach方法为其关联运行过程中所依赖的一系列上下文环境变量，代码如下所示。

ActivityThread的performLaunchActivity -> 通过类加载器创建Activity的实例对象 -> 调用其attach方法关联运行过程中所依赖的上下文变量 -> Activity所属Window对象 -> PolicyManager的makeNewWindow方法
由此可以发现，Window的具体实现的确是PhoneWindow

```java
public Window makeNewWindow(Context context) {
    return new PhoneWindow(context);
}
```

分析Activity的视图是怎么附属在Window上的？
Activity setContentView -> PhoneWindow的setContentView

PhoneWindow的setContentView方法大致遵循如下几个步骤：

1. 如果没有DecorView，那么就创建它
2. 将View添加到DecorView的mContentParent中
3. 回调Activity的onContentChanged方法通知Activity视图已经发生改变

**这里需要正确理解Window的概念，Window更多表示的是一种抽象的功能集合，虽然说早在Activity的attach方法中Window就已经被创建了，但是这个时候由于DecorView并没有被WindowManager识别，所以这个时候的Window无法提供具体功能，因为它还无法接收外界的输入信息。**

ActivityThread的handleResumeActivity方法 -> Activity的onResume -> Activity的makeVisible -> DecorView的添加和显示

### 8.3.2 Dialog的Window创建过程
1. 创建Window
2. 初始化DecorView并将Dialog的视图添加到DecorView中
3. 将DecorView添加到Window中并显示

普通的Dialog有一个特殊之处，那就是必须采用Activity的Context，如果采用Application的Context，那么就会报错。错误原因是因为没有应用token，而应用token一般只有Activity拥有，另外，系统Window比较特殊，它可以不需要token，可以选用TYPE_SYSTEM_OVERLAY来指定对话框的Window类型为系统Window。
    
```java
dialog.getWindow().setType(LayoutParams.TYPE_SYSTEM_ERROR)
```

然后在AndroidManifest文件中声明权限

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
```

### 8.3.3 Toast的Window创建过程
由于Toast具有定时取消这一功能，所以系统采用了Handler。
在Toast的内部有两类IPC过程，第一类是Toast访问NotificationManagerService，第二类是Notification-ManagerService回调Toast里的TN接口。

Toast属于系统Window，它内部的视图由两种方式指定，一种是系统默认的样式，另一种是通过setView方法来指定一个自定义View，不管如何，它们都对应Toast的一个View类型的内部成员mNextView。Toast提供了show和cancel分别用于显示和隐藏Toast，它们的内部是一个IPC过程，show方法和cancel方法的实现如下：

```java
public void show() {
    if (mNextView == null) {
        throw new RuntimeException("setView must have been called");
    }

    INotificationManager service = getService();
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    tn.mNextView = mNextView;

    try {
        service.enqueueToast(pkg, tn, mDuration);
    } catch (RemoteException e) {
        // Empty
    }
}

public void cancel() {
    mTN.hide();

    try {
        getService().cancelToast(mContext.getPackageName(), mTN);
    } catch (RemoteException e) {
        // Empty
    }
}
```
NMS: NotificationManagerService
显示和隐藏Toast -> NMS（运行在系统进程中）-> 跨进程回调TN（Binder类，运行在Binder线程池中）-> 通过Handler将其切换到当前线程中（这里的当前线程是指发送Toast请求所在的线程）

Toast显示过程：
NMS中的enqueueToast方法 -> ToastRecord被添加到mToastQueue中 -> NMS通过showNextToastLocked方法显示当前Toast -> 由ToastRecord的callback来完成（这个callback是Toast中的TN对象的远程Binder，通过callback来访问TN中的方法是需要跨进程来完成的，最终被调用的TN中的方法会运行在发起Toast请求的应用的Binder线程池中。） -> NMS通过scheduleTimeoutLocked发送一个延时消息 -> 延时相应时间后，NMS通过cancelToastLocked隐藏Toast并将其从mToastQueue中移除 -> 这个时候如果mToastQueue中还有其他Toast，那么NMS就继续显示其他Toast

Toast隐藏过程：
同样通过ToastRecord的callback来完成，也是一个IPC过程。

通过上面的分析，大家知道Toast的显示和影响过程实际上是通过Toast中的TN这个类来实现的，它有两个方法show和hide，分别对应Toast的显示和隐藏。由于这两个方法是被NMS以跨进程的方式调用的，因此它们运行在Binder线程池中。为了将执行环境切换到Toast请求所在的线程，在它们的内部使用了Handler，如下所示。

```java
/**
 * schedule handleShow into the right thread
 */
@Override
public void show() {
    if (localLOGV) Log.v(TAG, "SHOW: " + this);
    mHandler.post(mShow);
}

/**
 * schedule handleHide into the right thread
 */
@Override
public void hide() {
    if (localLOGV) Log.v(TAG, "HIDE: " + this);
    mHandler.post(mHide);
}
```

上述代码中，mShow和mHide是两个Runnable，它们内部分别调用了handleShow和handleHide方法。由此可见，handleShow和handleHide才是真正完成显示和隐藏Toast的地方。TN的handleShow中会将Toast的视图添加到Window中，如下所示。

```java
mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
mWM.addView(mView,mParams)
```

而NT的handleHide中会将Toast的视图从Window中移除，如下所示。

```java
public void handleHide() {
    if (localLOGV) Log.v(TAG, "HANDLE HIDE: " + this + " mView=" + mView);
    if (mView != null) {
        if (mView.getParent() != null) {
            if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
            mWM.removeView(mView);
        }

        mView = null;
    }
}
```

#### 一个问题：一个应用中到底有多少个Window呢？
从Activity启动的attach就能看出来是无限的，因为Window的唯一实现类是PhoneWindow，比如说我们现在启动一个Activity，在ActivityThread中开始，调用启动Activity，到最后的实例化完成Activity之后会调用Activity的attach方法，该方法中就对PhoneWindow做了实例化。

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, IVoiceInteractor voiceInteractor) {
    attachBaseContext(context);

    mFragments.attachActivity(this, mContainer, null);

    mWindow = PolicyManager.makeNewWindow(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
}
```

在Activity的attach方法里，系统会创建Activity所属的Window对象并为其设置回调接口，Window对象的创建是通过PolicyManager的makeNewWindow方法实现的。

从上面的分析可以看出，Activity的Window是通过PolicyManager的一个工厂方法来创建的，但是从PolicyManager的类名可以看出，它不是一个普通的类，它是一个策略类。PolicyManager中实现的几个工厂方法全部在策略接口IPolicy中声明了，IPolicy的定义如下：


```java
public interface IPolicy {
     public Window makeNewWindow(Context context);
     public LayoutInflater makeNewLayoutInflater(Context context);
     public WindowManagerPolicy makeNewWindowManager();
     public FallbackEventHandler makeNewFallbackEventHandler(Context
     context);
}
```

在实际的调用中，PolicyManager的真正实现是Policy类，Policy类中的makeNewWindow方法的实现如下，由此可以发现，Window的具体实现的确是PhoneWindow。


```java
public Window makeNewWindow(Context context) {
    return new PhoneWindow(context);
}
```



