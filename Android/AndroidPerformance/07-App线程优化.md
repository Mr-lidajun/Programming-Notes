# 第7章 App线程优化

> 在开发中线程的使用必不可少，本章节带领大家学习线程调度的原理、常见的异步方式以及异步的优化，同时也会介绍大型项目中如何锁定线程创建位置、如何高效的收敛线程。
>
> **参考**：
>
> 深入探索Android启动速度优化
>
> https://juejin.im/post/5e6f18a951882549422ef333#heading-95
>
> 四、启动优化常规方案
>
> 3、异步初始化预备知识-线程优化

##  7-2 Android线程调度原理剖析

### 7.2.1 线程调度原理

- 1、任意时刻，**只有一个线程占用CPU，处于运行状态**。
- 2、多线程并发，轮流获取CPU使用权。
- 3、JVM负责线程调度，按照特定机制分配CPU使用权。

### 7.2.2 线程调度模型

**1、分时调度模型**

轮流获取、均分CPU。

##### 2、抢占式调度模型

优先级高的获取。

**如何干预线程调度？**

设置线程优先级。

### 7.2.3 Android线程调度

**1、nice值**

- Process中定义。
- 值越小，优先级越高。
- 默认是THREAD_PRIORITY_DEFAUT，0。

**2、cgroup**

它是一种更严格的群组调度策略，主要分为如下两种类型：

- 后台group（默认）。
- 前台group，保证前台线程可以获取到更多的CPU

### 7.2.4 注意点

* 线程过多会导致CPU频繁切换，降低线程运行效率。
* 正确认识任务重要性以决定使用哪种线程优先级。
* 优先级具有继承性。

## 7-3 Android异步方式汇总

### 1、Thread

* 最简单、常见的异步方式。
* 不易复用，频繁创建及销毁开销大。
* 复杂场景不易使用。

### 2、HandlerThread

* 自带消息循环的线程。
* 串行执行。
* 长时间运行，不断从队列中获取任务。

### 3、IntentService

* 继承自Service在内部创建HandlerThread。
* 异步，不占用主线程。
* 优先级较高，不易被系统Kill。

### 4、AsyncTask

* Android提供的工具类。
* 无需自己处理线程切换。
* 需注意版本不一致问题（API 14以上解决）

### 5、线程池

* Java提供的线程池。
* 易复用，减少频繁创建、销毁的时间。
* 功能强大，如定时、任务队列、并发数控制等。

### 6、RxJava

由强大的调度器Scheduler集合提供。

不同类型的Scheduler：

* IO
* Computation

### 异步方式总结

* 推荐度：从后往前排列。
* 正确场景选择正确的方式。

## 7-4 Android线程优化实战

### 线程使用准则

* 1、**严禁使用new Thread方式**。
* 2、提供**基础线程池**供各个业务线使用，避免各个业务线各自维护一套线程池，导致线程数过多。
* 3、**根据任务类型选择合适的异步方式**：优先级低，长时间执行，HandlerThread；定时执行耗时任务，线程池。
* 4、创建线程必须**命名**，以方便**定位线程归属**，在运行期 **Thread.currentThread().setName** 修改名字。
* 5、关键异步任务监控，注意**异步不等于不耗时**，建议使用**AOP**的方式来做**监控**。
* 6、**重视优先级设置（根据任务具体情况），Process.setThreadPriority() 可以设置多次**。

## 7-5 如何锁定线程创建者

### 锁定线程创建背景

* 项目变大之后收敛线程。
* 项目源码、三方库、aar中都有线程的创建。

### 锁定线程创建方案

特别适合Hook手段，**找Hook点：构造函数或者特定方法，如Thread的构造函数。**

**实战**

这里我们直接使用维数的 [epic](https://github.com/tiann/epic) 对Thread进行Hook。在attachBaseContext中调用DexposedBridge.hookAllConstructors方法即可，如下所示：

```java
DexposedBridge.hookAllConstructors(Thread.class, new XC_MethodHook() { 
    @Override protected void afterHookedMethod（MethodHookParam param）throws Throwable {                         
        super.afterHookedMethod(param); 
        Thread thread = (Thread) param.thisObject; 
        LogUtils.i(thread.getName() + "stack " + Log.getStackTraceString(new Throwable());
    }
);
```

从log找到线程创建信息，根据堆栈信息跟相关业务方沟通解决方案。

## 7-6 线程收敛优雅实践初步

### 线程收敛常规方案

* 根据线程创建堆栈考量合理性，使用同一线程库。
* 各业务线下掉自己的线程库。

### 问题：基础库怎么使用线程？

直接依赖线程库，但问题在于**线程库更新可能会导致基础库更新**。

### 基础库优雅使用线程

* 基础库内部暴露API：setExecutor
* 初始化的时候注入统一的线程库

### 统一线程库时区分任务类型

* **IO密集型任务**：IO密集型任务不消耗CPU，核心池可以很大。常见的IO密集型任务如文件读取、写入，网络请求等等。
* **CPU密集型任务**：核心池大小和CPU核心数相关。常见的CPU密集型任务，如比较复杂的计算操作，此时需要使用大量的CPU计算单元。

### 实现用于执行多类型任务的基础线程池组件

目前基础线程池组件位于启动器sdk之中，使用非常简单，示例代码如下所示：

```java
// 如果当前执行的任务是CPU密集型任务，则从基础线程池组件
// DispatcherExecutor中获取到用于执行 CPU 密集型任务的线程池
DispatcherExecutor.getCPUExecutor().execute(YourRunable());

// 如果当前执行的任务是IO密集型任务，则从基础线程池组件
// DispatcherExecutor中获取到用于执行 IO 密集型任务的线程池
DispatcherExecutor.getIOExecutor().execute(YourRunable());
```

具体的实现源码也比较简单，并且我对每一处代码都进行了详细的解释，就不一一具体分析了。代码如下所示：

```java
public class DispatcherExecutor {

    /**
     * CPU 密集型任务的线程池
     */
    private static ThreadPoolExecutor sCPUThreadPoolExecutor;

    /**
     * IO 密集型任务的线程池
     */
    private static ExecutorService sIOThreadPoolExecutor;

    /**
     * 当前设备可以使用的 CPU 核数
     */
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();

    /**
     * 线程池核心线程数，其数量在2 ~ 5这个区域内
     */
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 5));

    /**
     * 线程池线程数的最大值：这里指定为了核心线程数的大小
     */
    private static final int MAXIMUM_POOL_SIZE = CORE_POOL_SIZE;

    /**
    * 线程池中空闲线程等待工作的超时时间，当线程池中
    * 线程数量大于corePoolSize（核心线程数量）或
    * 设置了allowCoreThreadTimeOut（是否允许空闲核心线程超时）时，
    * 线程会根据keepAliveTime的值进行活性检查，一旦超时便销毁线程。
    * 否则，线程会永远等待新的工作。
    */
    private static final int KEEP_ALIVE_SECONDS = 5;

    /**
    * 创建一个基于链表节点的阻塞队列
    */
    private static final BlockingQueue<Runnable> S_POOL_WORK_QUEUE = new LinkedBlockingQueue<>();

    /**
     * 用于创建线程的线程工厂
     */
    private static final DefaultThreadFactory S_THREAD_FACTORY = new DefaultThreadFactory();

    /**
     * 线程池执行耗时任务时发生异常所需要做的拒绝执行处理
     * 注意：一般不会执行到这里
     */
    private static final RejectedExecutionHandler S_HANDLER = new RejectedExecutionHandler() {
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
            Executors.newCachedThreadPool().execute(r);
        }
    };

    /**
     * 获取CPU线程池
     *
     * @return CPU线程池
     */
    public static ThreadPoolExecutor getCPUExecutor() {
        return sCPUThreadPoolExecutor;
    }

    /**
     * 获取IO线程池
     *
     * @return IO线程池
     */
    public static ExecutorService getIOExecutor() {
        return sIOThreadPoolExecutor;
    }

    /**
     * 实现一个默认的线程工厂
     */
    private static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger POOL_NUMBER = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                    Thread.currentThread().getThreadGroup();
            namePrefix = "TaskDispatcherPool-" +
                    POOL_NUMBER.getAndIncrement() +
                    "-Thread-";
        }

        @Override
        public Thread newThread(Runnable r) {
            // 每一个新创建的线程都会分配到线程组group当中
            Thread t = new Thread(group, r,
                    namePrefix + threadNumber.getAndIncrement(),
                    0);
            if (t.isDaemon()) {
                // 非守护线程
                t.setDaemon(false);
            }
            // 设置线程优先级
            if (t.getPriority() != Thread.NORM_PRIORITY) {
                t.setPriority(Thread.NORM_PRIORITY);
            }
            return t;
        }
    }

    static {
        sCPUThreadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                S_POOL_WORK_QUEUE, S_THREAD_FACTORY, S_HANDLER);
        // 设置是否允许空闲核心线程超时时，线程会根据keepAliveTime的值进行活性检查，一旦超时便销毁线程。否则，线程会永远等待新的工作。
        sCPUThreadPoolExecutor.allowCoreThreadTimeOut(true);
        // IO密集型任务线程池直接采用CachedThreadPool来实现，
        // 它最多可以分配Integer.MAX_VALUE个非核心线程用来执行任务
        sIOThreadPoolExecutor = Executors.newCachedThreadPool(S_THREAD_FACTORY);
    }

}
```

## 7-7 线程优化模拟面试（线程优化核心问题）

### 1、线程使用为什么会遇到问题？

项目发展阶段忽视基础设施建设，没有采用统一的线程池，导致线程数量过多。

**表现形式**

异步任务执行太耗时，导致主线程卡顿。

**问题原因**

* **1、Java线程调度是抢占式的，线程优先级比较重要，需要区分。**
* **2、没有区分IO和CPU密集型任务，导致主线程抢不到CPU。**

### 2、怎么在项目中对线程进行优化？

**核心：线程收敛**

* **通过Hook方式找到对应线程的堆栈信息，和业务方讨论是否应该单独起一个线程，尽可能使用统一线程池。**
* **每个基础库都暴露一个设置线程池的方法，以避免线程库更新导致基础库需要更新的问题。**
* **统一线程池应注意IO、CPU密集型任务区分。**
* 其它细节：**重要异步任务统计耗时、注重异步任务优先级和线程名的设置。**