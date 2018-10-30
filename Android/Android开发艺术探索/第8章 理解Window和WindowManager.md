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



















