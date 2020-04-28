# 第3章 App启动优化
> App启动速度是用户的第一印象，本章会介绍精准度量启动速度的方式，启动优化的相关工具、常规优化手段等，同时我会介绍异步初始化以及延迟初始化的最优解，以最优雅、可维护性高的的方式获得闪电般的启动速度。...

## 3-3 启动时间测量方式

### 3.3.1 adb命令

* adb shell am start -W packagename/首屏Activity
  * ThisTime: 最后一个Activity启动耗时
  * TotalTime: 所有Activity启动耗时
  * WaitTime: AMS启动Activity的总耗时
* 缺点：
  * 线下使用方便，不能带到线上
  * 非严谨、精确时间

### 3.3.2 手动打点

* 启动时埋点，启动结束埋点，二者差值
* 误区：onWindowFocusChanged只是首帧时间
* 正解：真实数据展示，Feed（列表数据）第一条展示
  * 精准，可带到线上，推荐使用
  * 避开误区，采用Feed第一条展示

### 3.3.3 总结

* 启动时间测量的两种方式
* 严谨真实的启动时间（误区）



## 3-4 启动优化工具选择-1

两种工具：traceView、systrace

**注意**

* 两种方式互相补充
* 正确认识工具及不同场景选择合适的工具

### 3.4.1 traceView

* 图形的形式展示执行时间、调用栈等
* 信息全面，包含所有线程
* 使用方式
  * Debug.startMethodTracing("文件名");
  * Debug.stopMethodTracing();
  * 生成文件在sd卡：Android/data/packagename/files

#### 3.4.1.1指标解读

Call Chart

* 横向：函数调用时间段消耗
* 竖向：函数调用关系

Flame Chart: 倒置的调用图表

Top Down：函数的调用列表

* Wall Clock Time：线程真正执行的时间
* Thread Time: CPU执行时间

Bottom Up：一个函数的调用列表（与Top Down相反）

**重点关注**：Call Chart、Top Down

#### 3.4.1.2 traceView缺点

* 运行时开销严重，整体都会变慢
* 可能会带偏优化方向
* traceView与cpu profiler

### 3.4.2 systrace

* 结合Android内核的数据，生成Html报告
* API 18以上使用，推荐TraceCompat（向下兼容类）

#### 3.4.2.1 使用方式

* python systrace.py -t 10 [other-options] [categories]
* https://developer.android.com/studio/command-line/systrace#command_options

#### 3.4.2.2 优点

* 轻量级，开销小
* 直观反映cpu利用率

#### 3.4.2.3 cputime与walltime区别

* walltime是代码执行时间
* cputime是代码消耗cpu的时间（重点指标）
* 举例：锁冲突



##  3-6 优雅获取方法耗时讲解

### 3.6.1 常规方式

* 背景：需要知道启动阶段所有方法耗时
* 实现：手动埋点

**具体实现**

* long time = System.currentTimeMillis();
* long cost = System.currentTimeMillis() - time;
* 或者SystemClock.currentThreadMillis();

**缺点**

* 侵入性强
* 工作量大

### 3.6.2 AOP介绍

* Aspect Oriented Programming，面向切面编程
  * 针对同一类问题的统一处理
  * 无侵入添加代码
* 优点
  * 无侵入性
  * 修改方便

### 3.6.3 AspectJ使用

* classpath 'com.hujiang.aspectjx:gradle-android-plugin-aspectjx:2.0.0'
* implementation 'org.aspectj:aspectjrt:1.8.+'
* apply plugin: 'android-aspectjx'

#### 3.6.3.1 Join Points

* 程序运行时的执行点，可以作为切面的地方
  * 函数调用、执行
  * 获取、设置变量
  * 类初始化

#### 3.6.3.2 PointCut

* 带条件的JoinPoints

#### 3.6.3.3 Advice

* 一种Hook，要插入代码的位置
  * Before: PointCut之前执行
  * After: PointCut之后执行
  * Around: PointCut之前、之后分别执行

#### 3.6.3.4 语法简介

```java
@Before("excution(* android.app.Activity.on**(..))")
public void onActivityCalled(JoinPoint joinPoint) throws Throwable {
    .....
}
```

* Before: Advice，具体插入位置
* execution: 处理Join Point的类型，call、execution
* (* android.app.Activity.on**(..)): 匹配规则
* onActivityCalled: 要插入的代码

## 3-7 优雅获取方法耗时实操

```java
@Aspect
public class PerformanceApp {
    @Around("call(* com.optimize.performance.PerformanceApp.**(..))")
    public void getTime(ProceedingJoinPoint joinPoint) {
        Signature signature = joinPoint.getSignature();
        String name = signature.toShortString();
        long time = System.currentTimeMillis();
        try {
            joinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        LogUtils.i(name + " cost " + (System.currentTimeMillis() - time));
    }
}
```

### 总结

* 优雅获取方法耗时的方式
* AOP的理解及使用

## 3-8 异步优化详解

**优化小技巧**

* Theme切换：感觉上的快

```xml
<layer-list>
	<item android:drawable="@android:color/white" />
    <item>
    	<bitmap
           android:src="@drawable/product_logo_144p"
           android:gravity="center"/>
    </item>
</layer-list>
```



```xml
<activity...
android:theme="@style/AppTheme.Launcher" />

@Override
protected void onCreate(Bundle savedInstanceState) {
	//super.onCreate之前切换回来
	setTheme(R.style.Theme_MyApp);
	super.onCreate(savedInstanceState);
}
```

**异步优化**

* 核心思想：子线程分担主线程任务，并行减少时间
  * 首先根据CPU核数来确定线程池中的线程个数
  * 然后创建指定线程个数的线程池，将主线程任务放入其中

```java
public final CPU_COUNT = Runtime.getRuntime().availableProcessors();
public final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1), 4);
public final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;

@Override
public void onCreate() {
    super.onCreate();
    mApplication = this;
    LaunchTimer.startRecord();
    ExecutorService service = Executors.newFixedThreadPool(CORE_POOL_SIZE);
    service.submit(new Runnable() {
       @Override
        public void run() {
            // Bugly
            initBugly();
        }
    });
    
    service.submit(new Runnable() {
       @Override
        public void run() {
            // 高德地图
            initAMap();
        }
    });
    
    LaunchTimer.endRecord();
}
```

**异步优化注意**

* 不符合异步要求
* 需要再某阶段完成
* 区分CPU密集型和IO密集型任务

**不符合异步要求**

如果初始化代码中包含必须在主线程执行的内容，此时需要修改初始化代码块，如果是第三方SDK，无法修改其内容，则使用第二种方案

方案一：修改初始化代码块

```java
private void initStetho() {
    Handler handler = new Handler(Looper.getMainLooper());
    Stetho.initializeWithDefaults(this);
}
```

**需要在某阶段完成**

方案二：使用CountDownLatch

```java
private CountDownLatch mCountDownLatch = new CountDownLatch(1);

@Override
public void onCreate() {
    super.onCreate();
    mApplication = this;
    LaunchTimer.startRecord();
    ExecutorService service = Executors.newFixedThreadPool(CORE_POOL_SIZE);
    service.submit(new Runnable() {
       @Override
        public void run() {
            // Bugly
            initBugly();
            mCountDownLatch.countDown();
        }
    });
    
    service.submit(new Runnable() {
       @Override
        public void run() {
            // 高德地图
            initAMap();
        }
    });
    try {
        mCountDownLatch.await();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
  
    LaunchTimer.endRecord();
}
```



## 3-9 异步初始化最优解-启动器-1

* 常规异步痛点
  * 代码不优雅
  * 场景不好处理（依赖关系）
  * 维护成本高
* 启动器介绍
* 启动器实战

### 启动器介绍

* 核心思想：充分利用CPU多核，自动梳理任务顺序
  * 关键字：充分、自动

### 启动器流程

* 代码Task优化，启动逻辑抽象为Task
* 根据所有任务依赖关系排序生成一个有向无环图
* 多线程按照排序后的优先级依次执行

### 启动器流程图

​	略