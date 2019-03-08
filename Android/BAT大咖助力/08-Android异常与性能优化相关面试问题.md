#第8章 Android异常与性能优化相关面试问题
## 8.1 anr异常面试问题讲解
1. 什么是anr
    Application Not Responding
2. 造成anr的主要原因
    1. 应用程序的响应性是由Activity Manager和WindowManager系统服务监视的
    2. 主线程被IO操作（从4.0之后网络IO不允许在主线程中）阻塞
    3. 主线中存在耗时的计算
3. 造成anr的主要原因-Android中哪些操作是在主线程呢？
    * Activity的所有生命周期回调都是执行在主线程的
    * Service默认是执行在主线程的
    * BroadcastReceiver的onReceive回调是执行在主线程的
    * 没有使用子线程的looper的Handler的handleMessage，post(Runnable)是执行在主线程的
    * AsyncTask的回调中除了doInBackground，其他都是执行在主线程
4. 如何解决anr
    * 使用AsyncTask处理耗时操作
    * 使用Thread或者HandlerThread提高优先级
    * 使用handler来处理工作线程的耗时任务
    * Activity的onCreate和onResume回调中尽量避免耗时的代码

## 8.2 oom异常面试问题讲解
1. 什么是oom？
    当前占用的内存加上我们申请的内存资源超过了Dalvik虚拟机的最大内存限制就会抛出Out of memory异常。
2. 一些容易混淆的概念
    内存溢出/内存抖动/内存泄露
3. 如何解决oom
    * 3.1 有关bitmap
        * 图片显示
        * 及时释放内存
        * 图片压缩
        * inBitmap属性
        * 捕获异常
    * 3.2 其他办法
        * listview: convertview/lru
        * 避免在onDraw方法里面执行对象的创建
        * 谨慎使用多进程

## 8.3 bitmap面试问题讲解
1. recycle
2. LRU
3. 计算inSampleSize
4. 缩略图（options.inJustDecodeBounds = true）
5. 三级缓存

## 8.4 ui卡顿面试问题讲解
* 一.UI卡顿原理
    * 60fps --> 16ms
        60fps：每秒60帧（Android规定流畅的帧率在60fps，每秒帧数，换算过来也就是一秒60帧，为了保证不丢失帧数，我们一定要在16ms处理完所有的CPU计算和GPU渲染操作）
        16ms：1000（ms）/60 ≈ 16
    * overdraw
* 二. UI卡顿原因分析
    1. 人为在UI线程中做轻微耗时操作，导致UI线程卡顿；
    2. 布局Layout过于复杂，无法在16ms内完成渲染
    3. 同一时间动画执行的次数过多，导致CPU或GPU负载过重
    4.View过度绘制，导致某些像素在同一帧时间内被绘制多次，从而使 CPU或GPU负载过重
    4. View频繁的触发measure、layout, 导致measure、layout累计耗时过多及整个View频繁的重新渲染 
    5. 内存频繁触发GC过多，导致暂时阻塞渲染操作 
    6. 冗余资源及逻辑等导致加载和执行缓慢
    7. ANR
* 三. UI卡顿总结
    1. 布局优化
    2. 列表及Adapter优化
    3. 背景和图片等内存分配优化
    4. 避免ANR

## 8.5 内存泄漏面试问题讲解
* 一.Java内存泄露基础知识
    * 1.Java内存的分配策略
        * 1)静态存储区
        * 2)栈区
        * 3)堆区
    * 2.Java是如何管理内存的
        ![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/BAT大咖助力/img/0800501.png)
    * 3.Java中的内存泄露
        内存泄露是指无用对象（不再使用的对象）持续占有内存或无用对象 的内存得不到及时释放，从而造成的内存空间的浪费称为内存泄露
* 二.Android内存泄露
    * 1.单例

    ```java
    public class AppManager {
    
        //有内存泄漏的问题：
    //    private static AppManager instance;
    //    private Context context;
    //    private AppManager(Context context) {
    //        this.context = context;
    //    }
    //    public static AppManager getInstance(Context context) {
    //        if (instance == null) {
    //            instance = new AppManager(context);
    //        }
    //        return instance;
    //    }
    //    修复内存泄漏的写法：
        private static AppManager instance;
        private Context context;
        private AppManager(Context context) {
            this.context = context.getApplicationContext();// 使用Application 的context
        }
        public static AppManager getInstance(Context context) {
            if (instance == null) {
                instance = new AppManager(context);
            }
            return instance;
        }
    }
    ```

    * 2.匿名内部类
    
    ```java
    public class MainActivity extends AppCompatActivity {
        //容易造成内存泄漏的写法：
    //    private Handler mHandler = new Handler() {
    //        @Override
    //        public void handleMessage(Message msg) {
    //            //...
    //        }
    //    };
    //
    //    @Override
    //    protected void onCreate(Bundle savedInstanceState) {
    //        super.onCreate(savedInstanceState);
    //        setContentView(R.layout.activity_main);
    //        loadData();
    //    }
    //
    //    private void loadData() {
    //        //...request
    //        Message message = Message.obtain();
    //        mHandler.sendMessage(message);
    //    }
    //
    //    static class TestResource { // 非静态内部类持有外部类的应用
    //        private static final String TAG = "";
    //        //...
    //    }
    
    //    修复内存泄漏的方法：
        private MyHandler mHandler = new MyHandler(this);
        private TextView mTextView ;
        private static class MyHandler extends Handler {
            private WeakReference<Context> reference;
            public MyHandler(Context context) {
                reference = new WeakReference<>(context);
            }
            @Override
            public void handleMessage(Message msg) {
                MainActivity activity = (MainActivity) reference.get();
                if(activity != null){
    //                activity.mTextView.setText("");
                }
            }
        }
    
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            loadData();
        }
    
        private void loadData() {
            //...request
            Message message = Message.obtain();
            mHandler.sendMessage(message);
        }
    
        @Override
        protected void onDestroy() {
            super.onDestroy();
            mHandler.removeCallbacksAndMessages(null);
        }
    }
    ```
    
    * 3.handler
    * 4.避免使用static变量
    * 资源未关闭造成的内存泄露
    * AsyncTask造成的内存泄露

## 8.6 内存管理面试问题讲解
* 一.内存管理机制概述
    * 分配机制
    * 回收机制
* 二.Android内存管理机制
    * 分配机制
    * 回收机制
* 三.内存管理机制的特点
    1. 更少的占用内存
    2. 在合适的时候，合理的释放系统资源。
    3. 在系统内存紧张的情况下，能释放掉大部分不重要的资源，来为 Android系统提供可用的内存。
    4. 能够很合理的在特殊生命周期中，保存或者还原重要数据，以至千 系统能够正确的重新恢复该应用
* 四.内存优化方法
    1. 当Service完成任务后，尽量停止它。
    2. 在UI不可见的时候，释放掉一些只有UI使用的资源
    3. 在系统内存紧张的时候，尽可能多的释放掉些非重要资源
    4. 避免滥用Bitmap导致的内存浪费
    5. 使用针对内存优化过的数据容器
    6. 避免使用依赖注入的框架
    7. 使用ZIP对齐的APK
    8. 使用多进程
* 五.内存溢出vs内存泄露
    1. 内存溢出
    2. 内存泄露

## 8.7 冷启动优化面试问题讲解
* 一.什么是冷启动
    1. 冷启动的定义
        冷启动就是在启动应用前，系统中没有该应用的任何进程信息
    2. 冷启动/热启动的区别
        热启动: 用户使用返回键退出应用，然后马上又重新启动应用。
    3. 冷启动时间的计算
        这个时间值从应用启动(创建进程)开始计算，到完成视图的第一次绘制(即Activit内容对用户可见)为止。
* 二.冷启动流程
    Zygote进程中fork创建出一个新的进程创建和初始化Application类、创建MainActivity类inflate布局、当onCreate/onStart/onResume方 法都走完contentView的measure/layout/draw显示在界面上
    * 冷启动流程-总结
    Application的构造器方法->attachBaseContext()->onCreate()->Activity的构造方法一>onCreate()->配置主题中背景等属性->onStart()->onResume()->测量布局绘制显示在界面上。
* 三.如何对冷启动的时间进行优化
    * 1.减少onCreate()方法的工作量
    * 2.不要让Application参与业务的操作
    * 3.不要在APPlication进行耗时操作
    * 4.不要以静态变量的方式在Application中保存数据
    * 5.布局/ mainThread

## 8.8 其他优化面试问题讲解
* 一.android不用静态变量存储数据
    * 1.静态变量等数据由于进程已经被杀死而被初始化
    * 2.使用其他数据传输方式:文件/ sp / contentProvider。。
* 二.有关sp的安全问题
    * 1.不能跨进程同步
    * 2.存储SharePreference的文件过大问题
* 三.内存对象序列化
    * 序列化:将对象的状态信息转换为可以存储或传输的形式的过程
        * 1.Serializable
        * 2.Parcelable
    * 内存对象序列化-总结：
        * 1.Serializable是java的序列化方式，Parcelable是 Android特有的序列化方式
        * 2.在使用内存的时候，Parcelable比Serializable性能高
        * 3.Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC
4.Parcelable不能使用在要将数据存储在磁盘上的情况
* 四.避免在ui线程中做繁重的操作
    老生常谈啦