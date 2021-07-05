# 21-View绘制流程源码解析

## 1. 第一部分Activity setContentView

```kotlin
class OnChildThreadActivity : Activity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_sample)

        val textView = findViewById<TextView>(R.id.textView)
        thread {
            textView.text = "onCreate!"
        }
    }
}
```

onCreate → setContentView

```java
public class Activity extends ContextThemeWrapper,xxxx {
    ...
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
}
```

想知道Activity生命周期回调方法触发时机，我们需要关注ActivityThread中的几个关键方法：handleLaunchActivity、handleStartActivity、handleResumeActivity

Activity被创建的地方

```java
//ActivityThread
public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
    ....
    final Activity a = performLaunchActivity(r, customIntent);
    
}
```

↓

```java
//ActivityThread
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
        ....
    } catch (Exception e) {
        ...
    }
}
```

↓

```java
// Instrumentation
public Activity newActivity(ClassLoader cl, String className,
                            Intent intent)
    throws InstantiationException, IllegalAccessException,
ClassNotFoundException {
    String pkg = intent != null && intent.getComponent() != null
        ? intent.getComponent().getPackageName() : null;
    return getFactory(pkg).instantiateActivity(cl, className, intent);
}
```

↓

```java
// AppComponentFactory
public @NonNull Activity instantiateActivity(@NonNull ClassLoader cl, @NonNull String className,
                                             @Nullable Intent intent)
    throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Activity) cl.loadClass(className).newInstance();
}
```

回到ActivityThread类handleLaunchActivity方法中，创建Activity后，会继续通过callActivityOnCreate方法调用activity的attach、performCreate方法

```java
//ActivityThread
public Activity handleLaunchActivity(ActivityClientRecord r,...) {
    ....
    final Activity a = performLaunchActivity(r, customIntent);
    ....
    activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, ...);
    ...
	mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
}
```

↓

最终会调用到activity的onCreate方法

```java
// Activity
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    ....
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
}
```

回过头来，我们继续查看activity.attach方法，可以看到在attach中，mWindow被赋值了

```java
// Activity
final void attach(Context context, ActivityThread aThread,
                  Instrumentation instr, IBinder token, int ident,
                  Application application, ....) {
    attachBaseContext(context);
    ...
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ...
}
```

接下来我们继续查看`setContentView(R.layout.activity_sample)`

```java
// Activity
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

这里，我们从上面知道了`getWindow()`实际上是`PhoneWindow`，所以直接查看PhoneWindow的`setContentView`方法中的`installDecor`

```java
// PhoneWindow
public void setContentView(int layoutResID) {
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        // 1、创建DecorView，并将默认的布局（R.layout.screen_simple）添加到DecorView当中
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        ....
    } else {
        // 2、将我们传递进来的layoutResID添加到mContentParent当中，即DecorView布局当中的FrameLayout（R.layout.screen_simple）
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
	....
}
```

↓

`generateDecor`方法总是返回一个`new DecorView()`对象，此时Activity中的成员变量mWindow（PhoneWindow）对象当中的mDecor已经关联上了

```java
// PhoneWindow
// 创建DecorView，并将默认的布局（R.layout.screen_simple）添加到DecorView当中
private void installDecor() {
    if (mDecor == null) {
            mDecor = generateDecor(-1);
    ....
    }
    if (mContentParent == null) {
		mContentParent = generateLayout(mDecor);
    ....
    }
}
```

 ## 插入流程图形ppt第2页

继续查看`generateLayout`方法，其中`layoutResource`局部变量通过`getLocalFeatures`方法返回的features值进行赋值；

```java
// PhoneWindow
protected ViewGroup generateLayout(DecorView decor) {
    ....
    // Inflate the window decor.
    int layoutResource;
    int features = getLocalFeatures();
	if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        ...
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        ...
	} else {
        // 1、可在本地Android SDK源码文件中查看该布局文件
        layoutResource = R.layout.screen_simple;
    }
    mDecor当中
    // 2、将R.layout.screen_simple布局添加到DecorView当中
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    // 3、找到screen_simple.xml布局当中的FrameLayout
    // int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    ....
    return contentParent;
}
```

↓

```java
// Window 
protected final int getLocalFeatures(){
    return mLocalFeatures;
}
```

mLocalFeatures是个什么东西呢？其实它就是window的特征，比如想让一个Activity没有标题栏，则可以通过在onCreate方法`setContentView(R.layout.activity_sample)`代码前面添加代码`requestWindowFeature(Window.FEATURE_NO_TITLE)`，这个方法其实修改的就是getLocalFeatures的返回值

```java
class OnChildThreadActivity : Activity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        requestWindowFeature(Window.FEATURE_NO_TITLE)
        setContentView(R.layout.activity_sample)
    }
}
```

↓

最终`layoutResource`的值被赋值为`R.layout.screen_simple`，我们可以去本地Android SDK源码文件中查看该文件

`D:\Program\Android\Sdk\platforms\android-29\data\res\layout\screen_simple.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

↓

`generateLayout`方法注释2处，将screen_simple.xml布局创建出来并添加到DecorView当中

```java
// DecorView
void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
    // 通过布局ID（R.layout.screen_simple）创建出root
    final View root = inflater.inflate(layoutResource, null);
    if () {
        ....
    } else {
        // 将root添加到DecorView当中
        addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }
    ....
}
```

## 插入流程图形ppt第3页

到现在为止，`setContentView(R.layout.activity_sample)`传递进来的布局ID依旧没有用到

我们继续查看PhoneWindow对象setContentView方法当中代码注释2`mLayoutInflater.inflate(layoutResID, mContentParent);`

到这里就明白了，通过`setContentView`传递进来的布局ID将被添加到DecorView布局(screen_simple.xml)的FrameLayout当中，**此时一个完整的布局树就已经出现了**

## 插入流程图形ppt第4页

## 插入流程图形ppt第5页

DecorView的parent是**ViewRootImpl**

## 2. 第二部分：ViewRootImpl checkThread

```java
// ViewRootImpl
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
            "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

我们来看一下成员变量mThread是在什么地方被赋值的，通过下面的代码，可以看到mThread是在ViewRootImpl构造方法中被赋值的

```java
// ViewRootImpl
public ViewRootImpl(Context context, Display display) {
    ...
    mThread = Thread.currentThread();
    ....
}
```

> 种种迹象表明，一旦知道了`ViewRootImpl`是在什么地方被创建的，还有什么时候和`DecorView`关联起来的，对于我们理解整个View的绘制是很有帮助的。

由于`ViewRootImpl`只有这么一个构造方法，我们可以通过打断点的方式查看构造在什么地方被调用的。

```java
// WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
    ....
    root = new ViewRootImpl(view.getContext(), display);//377
    ....
}
```

↓

继续查看`WindowManagerGlobal` `addView`方法在哪被调用的

```java
// WindowManagerImpl
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    ...
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

↓

最终，我们跟到了`handleResumeActivity`

```java
// ActivityThread
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,String reason) {
    .....
    // 1、会调用activity.onResume()
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ....
    if (r.window == null && !a.mFinished && willBeVisible) {
        // PhoneWindow
        r.window = r.activity.getWindow();
        // DecorView
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        // 2、WindowManagerImpl
        ViewManager wm = a.getWindowManager();
        // 3、getAttributes获取到一个宽高都为MATCH_PARENT的LayoutParams窗口属性
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        .....
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                // 4、WindowManagerImpl.addView
                wm.addView(decor, l);//4296
            } else {
                a.onWindowAttributesChanged(l);
            }
    	}
    ....
    }
    ....
}
```

↓

这里需要注意一下，查看`performResumeActivity`方法

```java
// ActivityThread
public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest, String reason) {
    ....
    r.activity.performResume(r.startsNotResumed, reason);//4195
    ....
}
```

↓

继续查看activity的`performResume`方法

```java
// Activity
final void performResume(boolean followedByPause, String reason) {
   ....
   mInstrumentation.callActivityOnResume(this);
   .....
}

// Instrumentation
public void callActivityOnResume(Activity activity) {
    activity.mResumed = true;
    activity.onResume();
    ....
}
```

这里留意一下`ActivityThread` `handleResumeActivity`方法代码注释3处，`getAttributes()`获取到一个宽高都为MATCH_PARENT的LayoutParams窗口属性

```java
// Window
public final WindowManager.LayoutParams getAttributes() {
    return mWindowAttributes;
}

// Window
private final WindowManager.LayoutParams mWindowAttributes =
    new WindowManager.LayoutParams();

// WindowManager
public LayoutParams() {
    super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
    type = TYPE_APPLICATION;
    format = PixelFormat.OPAQUE;
}
```

接着我们看看代码注释2处`a.getWindowManager()`返回的是一个WindowManager接口，那它真正的实现类是谁呢？我们可以参考`Activity` `setContentView`方法代码`getWindow().setContentView(layoutResID)`

getWindow()实际返回的是一个`PhoneWindow`

回过头来，我们直接查看Activity的`mWindow`成员变量被赋值的地方

```java
// Activity
final void attach(Context context, ActivityThread aThread, xxxxx) {
    ....
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ....
	mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    ....
    mWindowManager = mWindow.getWindowManager();
    ....
}
```

↓

可以看到，`mWindow.getWindowManager`返回的是实际上是`WindowManagerImpl`对象

```java
// Window
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
                             boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated;
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

接着我们继续查看代码注释4处，`wm.addView(decor, l)`，此时我们已经知道了`wm`是个WindowManagerImpl对象，那我们直接去查看`WindowManagerImpl`对象的`addView`方法

```java
// WindowManagerImpl
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

`mGlobal`是一个WindowManagerGlobal的单例对象

```java
private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
```

↓

```java
// WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
    ....
    root = new ViewRootImpl(view.getContext(), display);
    ....
}
```

**小结：**

我们每个`Activity`都有一个`WindowManager`对象，所以一个程序里面可能会有很多个`WindowManager`对象，`WindowManager`都会把操作转发给`WindowManagerGlobal`，这样做的好处就是，如果我们以后想做一个统一的操作，在这个`WindowManagerGlobal`单例中会非常的简单

`WindowManagerGlobal`有什么作用呢？它会记录我们所有的View、ViewRootImpl

```java
public final class WindowManagerGlobal {
    private final ArrayList<View> mViews = new ArrayList<View>();
	private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    .....
}
```

回到我们最初`ViewRootImpl`的构造方法中查看`mThread`的赋值操作

```java
// ViewRootImpl
public ViewRootImpl(Context context, Display display) {
    ...
    mThread = Thread.currentThread();
    ....
}
```

由于`ViewRootImpl`构造方法是在`ActivityThread`的`handleLaunchActivity`方法中被调用的，所以`mThread`就是主线程了

## 3. 第三部分 将View绑定ViewRootImpl

查看`WindowManagerGlobal`中的`addView`方法

```java
// WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
    ....
    root = new ViewRootImpl(view.getContext(), display);
    ....
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);
    try {
       	// 1、将View绑定ViewRootImpl
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        ....
    }
}
```

↓

```java
// ViewRootImpl
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ....
    // 1、
    requestLayout();
    .....
    // 2、将ViewRootImpl本身传递给View，并赋值给了View的成员变量mParent
    view.assignParent(this);
    ....
}
```

↓

```java
// View
void assignParent(ViewParent parent) {
    if (mParent == null) {
        mParent = parent;
    }
    ....
}
```

## 4. 第四部分 ViewRootImpl->setView->requestLayout方法

```java
// ViewRootImpl
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

↓

```java
// ViewRootImpl
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 1、关键代码
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        .....
    }
}
```

查看`mTraversalRunnable`

 ```java
// ViewRootImpl
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
 ```

↓

`doTraversal`会在下一次消息队列循环中执行

## 5. 第五部分 ViewRootImpl -> performTraversals

```java
// ViewRootImpl
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        // 关键代码，ViewRootImpl的核心方法
        // ViewRootImpl拿着顶层的DecorView，调用它的生命周期方法
        // 就像ActivityThread拿着Activity的对象，调用它对应的生命周期方法
        performTraversals();
		....
    }
}
```

到这里我们知道了，`performTraversals`第一次执行是在下一个消息队列循环中执行的，它跟ActivityThread的`handleResumeActivity`或者说Activity的`onResume`不是在同一次消息处理里面的

```java
// ViewRootImpl
private void performTraversals() {
    ....
    // 1、这里的mWindowAttributes就是WindowManagerGlobal中addView方法中的root.setView(view, wparams, panelParentView);传递进来的
    // WindowManager.LayoutParams l = r.window.getAttributes();
    WindowManager.LayoutParams lp = mWindowAttributes;//1961
    ....
    if (mFirst) {//2000
        mFullRedrawNeeded = true;
        // 第一次肯定需要重新布局
        mLayoutRequested = true;
		....
        // 2、此时的lp值为：宽高都为MATCH_PARENT，所以会进行到else代码块中
        if (shouldUseDisplaySize(lp)) {
            ....
        } else {
            desiredWindowWidth = mWinFrame.width();
            desiredWindowHeight = mWinFrame.height();
        }
        ....
       	// 3、view.dispatchAttachedToWindow
        host.dispatchAttachedToWindow(mAttachInfo, 0);
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
        // 4、给DecorView加上状态栏和导航栏
        dispatchApplyInsets(host);//2030
        //.....
    }
    //....
    // 5、一个合适时机，遍历队列消息
    getRunQueue().executeActions(mAttachInfo.mHandler);//2065
    //....
    // 依赖方法成员变量mLayoutRequested
    // 15、每次执行之前都会判断mLayoutRequested的值，使用之后，将会被置为false,见代码注释13处
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {//2070
        //....
        // 6、第一次测量，对整个视图树的测量，测量的是window的大小
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                                                desiredWindowWidth, desiredWindowHeight);//2122
    }
    //.....
    if (layoutRequested) {
        //14、重点代码
        mLayoutRequested = false;//2188
    }
    //.....
    //依赖方法布局变量layoutRequested
    boolean windowShouldResize = layoutRequested && windowSizeMayChange&&xxxxxx;//2191
    //.....
    //windowShouldResize依赖方法布局变量layoutRequested
    if (mFirst || windowShouldResize || xxxxx) {// 2221
        //.....
        // 7、把尺寸调整告诉WMS
        relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);//2264
        ....
            //2280
            if (!mPendingMergedConfiguration.equals(mLastReportedMergedConfiguration)) {
                //....
                // 8、
                updatedConfiguration = true;//2285
            }
        //.....
        if (!mStopped || mReportNextDraw) {//2525
            //....
            if (xxxxx || updatedConfiguration) {//2528
                //....
                // 9、第二次测量，真正的开始测量View的大小；此时window窗口和视图树控件的尺寸都已经确定下来了
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);//2541
            }
        }
    }//2573
	//.....
    // 依赖方法局部变量layoutRequested
    didLayout = layoutRequested && (!mStopped || mReportNextDraw);//2586
    boolean triggerGlobalLayoutListener = didLayout
        || mAttachInfo.mRecomputeGlobalAttributes;
    if (didLayout) {
        // 10、布局
        performLayout(lp, mWidth, mHeight);//2590
        //....
    }
    if (triggerGlobalLayoutListener) {
        mAttachInfo.mRecomputeGlobalAttributes = false;
        // 11、视图树回调，此时View已经测量、布局完成，是拿到View大小的最佳位置
        mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();//2628
    }
    //....
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    if (!cancelDraw) {
        //....
        // 12、开始draw
        performDraw();//2755
    } else {
        if (isViewVisible) {
            // Try again
            // 13、重新执行performTraversals方法
            scheduleTraversals();//2759
        }
    }
}
```

**代码注释1处**，`mWindowAttributes`是什么呢？

这就要回到`ActivityThread`的`handleResumeActivity`方法说起了

```java
// ActivityThread
handleResumeActivity(xxxx) {
    ....
    WindowManager.LayoutParams l = r.window.getAttributes();//4277
    ....
    wm.addView(decor, l);//4296
}
```

↓

```java
// WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
    ....
    root = new ViewRootImpl(view.getContext(), display);
    ....
   	// 关键方法
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;//321
    ...
    root.setView(view, wparams, panelParentView);//387
    ....
}
```

↓

```java
// ViewRootImpl
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    mWindowAttributes.copyFrom(attrs);
    ....
    attrs = mWindowAttributes;
}
```

所以`mWindowAttributes`窗口属性就是：

```java
// Window
private final WindowManager.LayoutParams mWindowAttributes =
    new WindowManager.LayoutParams();
// WindowManager
public LayoutParams() {
    super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
    type = TYPE_APPLICATION;
    format = PixelFormat.OPAQUE;
}
```

**代码注释2处**，`shouldUseDisplaySize`方法返回fase

```java
// ViewRootImpl
performTraversals() {
    // lp值为：宽高都为MATCH_PARENT，所以会进行到else代码块中
    if (shouldUseDisplaySize(lp)) {
        ....
    } else {
        // 设置为手机屏幕的宽和高
        desiredWindowWidth = mWinFrame.width();
        desiredWindowHeight = mWinFrame.height();
    }
}
```

**代码注释3处**，将`ViewRootImpl`的成员变量`mAttachInfo`传递给了View，并且赋值给了`View`的成员变量`mAttachInfo`

```java
// View
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
    ....
}
```

从这里可以看出来，同一个界面中，一组控件它们的View当中的`mAttachInfo`，都是`ViewRootImpl`当中同一个成员变量`mAttachInfo`。当我们想要判断一个`View`是否依附到了`Window`上面时，就可以通过判断`View`成员变量`mAttachInfo`是否有值，可以查看View的源代码如下：

```java
//View
/**
  * Returns true if this view is currently attached to a window.
*/
public boolean isAttachedToWindow() {
    return mAttachInfo != null;
}
```

`ViewRootImpl`的成员变量`mAttachInfo`是在`ViewRootImpl`构造方法中被赋值的

```java
// ViewRootImpl
public ViewRootImpl(Context context, Display display) {
    ....
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this, context);
}
```



**代码注释4处**

### 插入流程图形ppt第7页

**代码注释5处**，是一个合适的时机处理队列消息，为什么呢？

那么我们需要看看`View`的`post`方法

```java
// View
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }

    // Postpone the runnable until we know on which thread it needs to run.
    // Assume that the runnable will be successfully placed after attach.
    getRunQueue().post(action);
    return true;
}
```

当在`Activity`的`onCreate`方法中调用`View.post`方法时，此时`View`的`mAttachInfo`局部变量还没有被赋值，通过上面的代码，我们知道`ViewRootImpl`的成员变量`mAttachInfo`是在`ViewRootImpl`构造方法中被赋值的，`ViewRootImpl`的构造方法是在`WindowManagerGlobal`中的`addView`方法调用的，`addView`方法最终又是在`ActivityThread`的`handleResumeActivity`方法中被调用到的。

所以当在`Activity`的`onCreate`方法中调用`View.post`方法时，`action`会被直接加入到队列当中，等待一个合适的时机被消费

**代码注释6处**

通过`measureHierarchy`方法对整个视图树的测量

```java
// ViewRootImpl
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    boolean goodMeasure = false;
    //此时，lp的宽高都是MATCH_PARENT
    if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
    	....
    }
    if (!goodMeasure) {
        // 都是window的宽和高
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        ....
    }
}
```

↓

```java
// ViewRootImpl
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        // 第一次测量 DecorView.measure
        // 调用DecorView的measure方法后，所有的子控件都会被DecorView一层一层向下遍历，子控件全部调用一次measure。这样就完成了对整个控件树的测量
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

}
```

**代码注释7处**

在Android中，一个window的大小，不是由我们应用层就能决定的，所有的window都是通过系统服务的WMS来完成的，通过`relayoutWindow`方法，把尺寸调整告诉WMS。同时，我们的window会由初始值变成固定的值。此时`updatedConfiguration`值也会发生改变，被赋值为`true`

**代码注释13、14、15处**

问：当变量isViewVisible为false时，执行`scheduleTraversals`方法时，还会重新执行测量和布局吗？

答：不会，因为在代码注释14处，成员变量`mLayoutRequested`已经被赋值为`false`，下次调用`performTraversals`方法时，会首先执行以下代码`boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);`，此时方法局部变量`layoutRequested `为`false`。测量和布局相关代码都不会执行

**代码注释12处**，解析`performDraw`方法

```java
// ViewRootImpl
private void performDraw() {
    //....
    boolean canUseAsync = draw(fullRedrawNeeded);
}
```

↓

```java
// ViewRootImpl
private boolean draw(boolean fullRedrawNeeded) {
    //....
    // 是否开启硬件加速，开启则硬件绘制，否则软件绘制
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {//3573
        //....
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);//3610
    } else {
        //....
        if (!drawSoftware(surface, mAttachInfo, xxxx)) {
            return false;
        }
}
```

## 第六部分 什么时候才会触发ViewRootImpl的requestLayout

```java
//ActivityThread
public void handleResumeActivity(xxx) {
    wm.addView(decor, l)//4296
}
```

↓

```java
//WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
                    Display display, Window parentWindow) {
    root = new ViewRootImpl(view.getContext(), display);//377
    root.setView(view, wparams, panelParentView);//387
}
```

↓

```java
// ViewRootImpl
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    requestLayout();//853
}
```

↓

```java
// ViewRootImpl
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

## 第七部分 子线程更新UI会崩溃吗？

> 只要子线程更新UI没有触发到`ViewRootImpl`的`checkThread`方法，就不会报错

思考：为什么下面的代码在子线程中更新UI不会报错？

```java
class OnClickActivity : Activity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_sample)

        val textView = findViewById<TextView>(R.id.textView)
        textView.setOnClickListener {
            thread {
                textView.text = "onCreate!"
            }
        }
    }
}
```

activity_sample布局文件

```java
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout 
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/textView"
        android:layout_width="100dp"
        android:layout_height="100dp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        android:text="Hello"
        />

</androidx.constraintlayout.widget.ConstraintLayout>
```

查看TextView的`setText`方法

```java
//TextView
private void setText(CharSequence text, BufferType type,
                         boolean notifyBefore, int oldlen) {
    //....
    if (mLayout != null) {
        checkForRelayout();
    }
}
```

↓

```java
//TextView
private void checkForRelayout() {
    // 如果布局宽不为WRAP_CONTENT，则进入
    if ((mLayoutParams.width != LayoutParams.WRAP_CONTENT
                || (mMaxWidthMode == mMinWidthMode && mMaxWidth == mMinWidth))
                && (mHint == null || mHintLayout != null)
                && (mRight - mLeft - getCompoundPaddingLeft() - getCompoundPaddingRight() > 0)) {
        if (mEllipsize != TextUtils.TruncateAt.MARQUEE) {
            //.....
            //如果宽高没发生改变，则进入
            if (mLayout.getHeight() == oldht
                && (mHintLayout == null || mHintLayout.getHeight() == oldht)) {
                autoSizeText();
                // 重点代码
                invalidate();//会执行到这里
                return;
            }
        }
        // Dynamic height, but height has stayed the same,
        // so use our new text layout.
        requestLayout();//这里不会被执行
        invalidate();
    }
}
```

↓

```java
//View
public void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }
```

↓

```java
//View
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
                        boolean fullInvalidate) {
    final AttachInfo ai = mAttachInfo;
    final ViewParent p = mParent;
    if (p != null && ai != null && l < r && t < b) {
        final Rect damage = ai.mTmpInvalRect;
        damage.set(l, t, r, b);
        //重点代码
        p.invalidateChild(this, damage);
    }
}

```

↓

```java
//ViewGroup
public final void invalidateChild(View child, final Rect dirty) {
    final AttachInfo attachInfo = mAttachInfo;
    //如果支持硬件加速，则触发硬件加速的捷径
    if (attachInfo != null && attachInfo.mHardwareAccelerated) {
        // HW accelerated fast path
        onDescendantInvalidated(child, child);
        return;
    }
}
```

↓

```java
//ViewGroup
public void onDescendantInvalidated(@NonNull View child, @NonNull View target) {
    //.....
    if (mParent != null) {
        //ViewRootImpl.onDescendantInvalidated
        mParent.onDescendantInvalidated(this, target);
    }
}
```

↓

```java
//ViewRootImpl
public void onDescendantInvalidated(@NonNull View child, @NonNull View descendant) {
    //....
    invalidate();
}
```

↓

```java
//ViewRootImpl
void invalidate() {
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        scheduleTraversals();
    }
}

//ViewRootImpl
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();//检查线程
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

到这里，我们可以发现ViewRootImpl的`invalidate`方法与`requestLayout`方法的区别：

`invalidate`：不检查线程

`requestLayout`：会检查线程

如果我们关闭硬件加速，则会调用`requestLayout`方法，当然也就会去检查线程了，随之就会报错