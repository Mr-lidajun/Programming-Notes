# 第5章 理解RemoteViews

> RemoteViews表示的是一个View结构，它可以在其他进程中显示，提供了一组基础的操作用于跨进程更新它的界面。
>
> 使用场景：通知和桌面小部件

## 5.1 RemoteViews的应用

通知和桌面小部件的界面都运行在其他进程中，确切的说是系统的SystemServer进程。

### 5.1.1 RemoteViews在通知栏上的应用

### 5.1.2  RemoteViews在桌面小部件上的应用

AppWidgetProvider是Android中提供的用于实现桌面小部件的类，其本质是一个广播，即BroadcastReceiver，查看其类继承关系便知。

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/5-2.jpg)

图5-2 AppWidgetProvider的类继承关系

桌面小部件的开发步骤：

1. 定义小部件界面
2. 定义小部件配置信息
3. 定义小部件的实现类
4. 在AndroidManifest.xml中声明小部件

AppWidgetProvider常用方法的调用时机：

- onEnable：当该窗口小部件第一次添加到桌面时调用该方法，可添加多次但只在第一次调用。
- onUpdate：小部件被添加时或者每次小部件更新时都会调用一次该方法，小部件的更新时机由updatePeriodMillis来指定，每个周期小部件都会自动更新一次。
- onDeleted：每删除一次桌面小部件就调用一次。
- onDisabled：当最后一个该类型的桌面小部件被删除时调用该方法，注意是最后一个。
- onReceive：这是广播的内置方法，用于分发具体的事件给其他方法。

### 5.1.3 PendingIntent概述

顾名思义，PendingIntent表示一种处于pending状态的意图，而pending状态表示的是一种待定、等待、即将发生的意思，就是说接下来有一个Intent（即意图）将在某个待定的时刻发生。

PendingIntent支持三种待定意图：启动Activity、启动Service和发送广播，对应着它的三个接口方法，如表5-1所示。

| static PendingIntent | getActivity(Context context, int requestCode, Intent intent, int flags) <br />获得一个PendingIntent，该待定意图发生时，效果相当于Context.startActivity(intent) |
| -------------------- | ------------------------------------------------------------ |
| static PendingIntent | getService(Context context, int requestCode, Intent intent, int flags)<br />获得一个PendingIntent, 该待定意图发生时，效果相当于Context.startService(Intent intent) |
| static PendingIntent | getBroadcast(Context context, int requestCode, Intent intent, int flags)<br />获得一个PendingIntent，该待定意图发生时，效果相当于Context.sendBroadcast(intent) |

PendingIntent的匹配规则：如果两个PendingIntent它们内部的Intent相同并且requestCode也相同，那么这两个PendingIntent就是相同的。那么什么情况下Intent相同呢？Intent的匹配规则是：如果两个Intent的ComponentName和intent-filter都相同，那么这两个Intent就是相同的。需要注意的是Extras不参与Intent的匹配过程。

PendingIntent 四个flags标记位：

**FLAG_ONE_SHOT**

**FLAG_NO_CREATE**

**FLAG_CANCEL_CURRENT**

**FLAG_UPDATE_CURRENT**

当前描述的PendingIntent如果已经存在，那么它们都会被更新，即它们的Intent中的Extras会被替换成最新的。

如果notify方法的id是常量，那么不管PendingIntent是否匹配，后面的通知会直接替换前面的通知，这个很好理解。

如果notify方法的id每次都不同，那么当PendingIntent不匹配时，这里的匹配是指PendingIntent中的Intent相同并且requestCode相同，在这种情况下不管采用何种标记位，这些通知之间不会相互干扰。如果PendingIntent处于匹配状态时，这个时候要分情况讨论：如果采用了FLAG_ONE_SHOT标记位，那么后续通知中的PendingIntent会和第一条通知保持完全一致，包括其中的Extras，单击任何一条通知后，剩下的通知均无法再打开，当所有的通知都被清除后，会再次重复这个过程；如果采用FLAG_CANCEL_CURRENT标记位，那么只有最新的通知可以打开，之前弹出的所有通知均无法打开；如果采用FLAG_UPDATE_CURRENT标记位，那么之前弹出的通知中的PendingIntent会被更新，最终它们和最新的一条通知保持完全一致，包括其中的Extras，并且这些通知都是可以打开的。

## 5.2 RemoteViews的内部机制

RemoteViews目前并不能支持所有的View类型，它所支持的所有类型如下：

**Layout**

FrameLayout、LinearLayout、RelativeLayout、GridLayout。

**View**

AnalogClock、Button、Chronometer、ImageButton、ImageView、ProgressBar、TextView、ViewFlipper、ListView、GridView、StackView、AdapterViewFlipper、ViewStub。

表5-2 RemoteViews的部分set方法

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/T5-2.jpg)

**RemoteViews的内部机制：**

由于RemoteViews主要用于通知栏和桌面小部件之中，这里就通过它们来分析RemoteViews的工作过程。我们知道，通知栏和桌面小部件分别由NotificationManager和AppWidgetManager管理，而NotificationManager和AppWidgetManager通过Binder分别和SystemServer进程中的NotificationManagerService以及AppWidgetService进行通信。由此可见，通知栏和桌面小部件中的布局文件实际上是在NotificationManagerService以及AppWidgetService中被加载的，而它们运行在系统的SystemServer中，这就和我们的进程构成了跨进程通信的场景。

首先RemoteViews会通过Binder传递到SystemServer进程，这是因为RemoteViews实现了Parcelable接口，因此它可以跨进程传输，系统会根据RemoteViews中的包名等信息去得到该应用的资源。然后会通过LayoutInflater去加载RemoteViews中的布局文件。在SystemServer进程中加载后的布局文件是一个普通的View，只不过相对于我们的进程它是一个RemoteViews而已。接着系统会对View执行一系列界面更新任务，这些任务就是之前我们通过set方法来提交的。set方法对View所做的更新并不是立刻执行的，在RemoteViews内部会记录所有的更新操作，具体的执行时机要等到RemoteViews被加载以后才能执行，这样RemoteViews就可以在SystemServer进程中显示了，这就是我们所看到的通知栏消息或者桌面小部件。当需要更新RemoteViews时，我们需要调用一系列set方法并通过NotificationManager和AppWidgetManager来提交更新任务，具体的更新操作也是在SystemServer进程中完成的。

从理论上来说，系统完全可以通过Binder去支持所有的View和View操作，但是这样做的话代价太大，因为View的方法太多了，另外就是大量的IPC操作会影响效率。为了解决这个问题，系统并没有通过Binder去直接支持View的跨进程访问，而是提供了一个Action的概念，Action代表一个View操作，Action同样实现了Parcelable接口。系统首先将View操作封装到Action对象并将这些对象跨进程传输到远程进程，接着在远程进程中执行Action对象中的具体操作。在我们的应用中每调用一次set方法，RemoteViews中就会添加一个对应的Action对象，当我们通过NotificationManager和AppWidgetManager来提交我们的更新时，这些Action对象就会传输到远程进程并在远程进程中依次执行，这个过程可以参看图5-3。远程进程通过RemoteViews的apply方法来进行View的更新操作，RemoteViews的apply方法内部则会去遍历所有的Action对象并调用它们的apply方法，具体的View更新操作是由Action对象的apply方法来完成的。上述做法的好处是显而易见的，首先不需要定义大量的Binder接口，其次通过在远程进程中批量执行RemoteViews的修改操作从而避免了大量的IPC操作，这就提高了程序的性能，由此可见，Android系统在这方面的设计的确很精妙。

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/5-3.jpg)

图5-3 RemoteViews的内部机制

**从源码的角度再来分析RemoteViews的工作流程：P233**

## 5.3 RemoteViews的意义 

本节这个例子是可以在实际中使用的，比如现在有两个应用，一个应用需要能够更新另一个应用中的某个界面，这个时候我们当然可以选择AIDL去实现，但是如果对界面的更新比较频繁，这个时候就会有效率问题，同时AIDL接口就有可能会变得很复杂。这个时候如果采用RemoteViews来实现就没有这个问题了，当然RemoteViews也有缺点，那就是它仅支持一些常见的View，对于自定义View它是不支持的。面对这种问题，到底是采用AIDL还是采用RemoteViews，这个要看具体情况，如果界面中的View都是一些简单的且被RemoteViews支持的View，那么可以考虑采用RemoteViews，否则就不适合用RemoteViews了。











