# 第2章 IPC机制
## 2.1 Android IPC简介
1. 线程：CPU调度的最小单元，同时线程是一种有限的系统资源。
2. 进程：一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用。一个进程可以包含多个线程，因此进程和线程是包含与被包含的关系。
3. 最简单的情况下，一个进程中可以只有一个线程，即主线程，在Android里面主线程也叫UI线程。
4. 操作系统相应IPC机制：
    * Windows：剪贴板、管道和邮槽等
    * Linux：命名管道、共享内容、信号量等
    * Android：Binder（最有特色）、Socket
5. 多进程的情况分为两种：
    1. 第一种情况：一个应用因为某些原因自身需要采用多进程模式来实现，至于原因，比如有些模块由于特殊原因需要运行在单独的进程中，又或者为了加大一个应用可使用的内存
    2. 第二种情况：两个应用之间获取数据，必须采用跨进程方式
    3. 系统提供的ContentProvider，也是一种进程间通信
## 2.2 Android中的多进程模式
### 2.2.1 开启多进程模式
方式一：给四大组件（Activity、Service、Receiver、ContentProvider）在AndroidManifest中指定android:process属性
方式二：通过JNI在native层去fork一个新进程
查看进程信息：adb shell ps或者adb shell ps | grep com.ldj.chapter_2

```java
<activity
    android:name="com.ldj.chapter_2.SecondActivity"
    android:configChanges="screenLayout"
    android:label="@string/app_name"
    android:process=":remote"/>

<activity
    android:name="com.ldj.chapter_2.ThirdActivity"
    android:configChanges="screenLayout"
    android:label="@string/app_name"
    android:process="com.ldj.chapter_2.remote"/>
```

* SecondActivity和ThirdActivity的android:process属性分别为":remote"和"com.ldj.chapter_2.remote"，那么这两种方式有区别吗？
当然有，首先，":"的含义是指要在当前的进程名前面附加上当前的包名，这是一种简写的方法，对于SecondActivity来说，它完整的进程名为com.ldj.chapter_2:remote，这一点通过进程信息也能看出来，而对于ThirdActivity中的声明方式，它是一种完整的命名方式，不会附加包名信息；其次，进程名以":"开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，而进程名不以":"开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中（前提条件是需要这两个应用有相同的ShareUID并且签名相同才可以）。
### 2.2.2 多进程模式的运行机制
* 每一个应用分配了一个独立的虚拟机，或者说为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致在不同的虚拟机访问同一个类的对象会产生多份副本。
* 所有运行在不同进程中的四大组件，只要它们之间需要通过内存来共享数据，都会共享失败，这也是多进程所带来的主要影响。
* 多进程会造成如下几方面的问题：
    1. 静态成员和单例模式完全失效。
    2. 线程同步机制完全失效。
    3. SharedPreferences的可靠性下降
    4. Application会多次创建。
* 跨进程通信方式
    1. 通过Intent来传递数据
    2. 共享文件
    3. SharedPreferences
    4. 基于Binder的Messenger
    5. AIDL
    6. Socket
## 2.3 IPC基础概率介绍
主要包含三方面：Serializable接口、Parcelable接口以及Binder
### 2.3.1 Serializable接口
* 为对象提供标准的序列化核反序列化操作
* serialVersionUID的详细工作机制
    序列化的时候系统会把当前类的serialVersionUID写入序列化的文件中（也可能是其他中介），当反序列化的时候系统会去检测文件中的serialVersionUID，看它是否和当前类的serialVersionUID一致，如果一致就说明序列化的类的版本和当前类的版本是相同的，这个时候可以成功反序列化；否则就说明当前类核序列化的类相比发生了某些变换，比如成员变量的数量、类型可能发生了改变，这个时候是无法反序列的，会报如下错误：
    
```
java.io.InvalidClassException: Main; local class incompatible: stream classdesc serialVersionUID = 8711368828010083044,local class serial-VersionUID = 8711368828010083043
```

* 另外一种情况：如果类结构发生了非常规性改变，比如修改了类名、成员变量的类型，这个时候尽管serialVersionUID验证通过了，但是序列化过程还是会失败，因为类结构有了毁灭性的改变，根本无法从老版本值的数据中还原出一个新的类结构的对象。
* 两种不参与序列化过程的特殊情况：
    * 静态成员变量属于类不属于对象；
    * 用transient关键字标记的成员变量。
### 2.3.2 Parcelable接口
* 实现该接口，一个类的对象就可以实现序列化并可以通过Intent和Binder传递。
#### 2.3.2.1 Parcelable和Serializable的区别
* Serializable是Java中的序列化接口，其使用起来简单但是开销很大，序列化和反序列化过程需要大量I/O操作。
* Parcelable是Android中的序列化方式，更适合用在Android平台上，缺点是使用起来稍微麻烦点，但是它的效率很高。Parcelable主要用在内存序列化上，通过Parcelable将对象序列化到存储设备中或者将对象序列化后通过网络传输也是可以的，但是过程稍显复杂，因此在这两种情况下建议大家使用Serializable。
### 2.3.3 Binder
直观来说，Binder是Android中的一个类，它继承了IBinder接口。
* 从IPC角度来说，Binder是一种跨进程通信方式，还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有；
* 从Android Framework角度来说，Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager，等等）和相应ManagerService的桥梁；
* 从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。
* Android开发中，Binder的主要用在Service中，包括AIDL和Messenger。
* 根据系统生成的Binder类来分析Binder的工作原理：
    1. 根据IBookManager.aidl系统为我们生成了IBookManager.java类，继承了IInterface接口。这个接口的核心实现就是它的内部类Stub和Stub的内部代理类Proxy
    2. 主要方法
        1. DESCRIPTOR: Binder的唯一标识，一般用当前Binder的类名表示
        2. asInterface(android.os.IBinder obj): 将服务端的Binder对象转换为客户端所需的AIDL接口类型的对象，这个转换过程是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身，否则返回系统封装后的Stub.proxy对象。
        3. asBinder：返回当前Binder对象
        4. onTransact: 此方法运行在服务端中的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。服务端通过code可以确定客户端所请求的目标方法是什么。如果此方法返回false，那么客户端的请求会失败，可以利用这个特性来做权限验证。
* 注意两点：
    1. 首先，当客户端发起远程请求时，由于当前线程会被挂起直至服务端进程返回数据，所以如果一个远程方法是耗时的，那么不能在UI线程中发起此远程请求。
    2. 其次，由于服务端的Binder方法运行在Binder的线程池中，所以Binder方法不算是否耗时都应该采用同步方式去实现，因为它已经运行在一个线程中了。
![Binder的工作机制](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/2-5.png)
图2-5 Binder的工作机制
* AIDL文件的本质：系统为我们提供了一种快速实现Binder的工具。
## 2.4 Android中的IPC方式
常见IPC方式：Bundle（通过在Intent中附加extras来传递信息）、共享文件的方式来共享数据、Binder方式跨进程通信、ContentProvider、Socket（网络通信）
### 2.4.1 使用Bundle
* 直接传递数据的典型场景：四大组件中的三大组件（Activity、Service、Receiver）都是支持在Intent中传递Bundle数据，Bundle实现了Parcelable接口，可以方便地在不同的进程间传输-----这是一种最简单的进程间通信方式。
* 一种特殊的使用场景：比如A进程正在进行计算，计算完成后它要启动B进程的一个组件并把计算结果传递给B进程。
    * 阻碍：这个计算结果不支持放入Bundle中，因此无法通过Intent来传输。
    * 解决方案：核心思想在于将原本需要在A进程的计算任务转移到B进程的后台Service中去执行。
### 2.4.2 使用文件共享
两个进程通过读/写同一个文件来交换数据，比如A进程把数据写入文件，B进程通过读取这个文件来获取数据。
* 特例：SharedPreferences，从本质上来说，它也属于文件的一种，但是由于系统对它的读/写有一定的缓存策略，即在内存中会有一份SharedPreferences文件的缓存，因此在多线程模式下，系统对它的读/写就变得不可靠，当面对高并发的读/写访问，有很大几率会丢失数据，因此，不建议在进程间通信中使用
### 2.4.3 使用Messenger
* Messenger可以翻译为信使，通过它可以在不同进程中传递Message对象，在Message中放入我们需要传递的对象，就可以轻松实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，它的底层实现是AIDL。
* Messenger对AIDL做了封装，由于它一次处理一个请求，因此服务端我们不用考虑线程同步的问题，这是因为服务端中不存在并发执行的情形。
* 缺点：
    1. Messenger是以串行的方式处理客户端发来的消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个处理，如果有大量的并发请求，那么Messenger就不太合适了。
    2. Messenger的主要作用是为了传递消息，很多我们需要跨进程调用服务端的方法，这种情形用Messenger就无法做到了。
图020403
![Messenger的工作原理](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/2-6.png)
图2-6 Messenger的工作原理

### 2.4.4 使用AIDL
5.客户端的实现
疑问1：页码P87~88
客户端调用远程服务的方法，被调用的方法运行在服务端的Binder线程池中，同时客户端会被挂起，这个时候如果服务端方法执行比较耗时，就会导致客户端线程长时间阻塞在这里，而如果这个客户端线程是UI线程的话，就会导致客户端ANR，这当然不是我们想要看到的。因此，如果我们明确知道某个远程方法是耗时的。那么就要避免在客户端的UI线程中去访问远程方法。由于客户端的onServiceConnected和onServiceDisconnected方法都运行在UI线程中，所以也不可以在它们里面直接调用服务端的耗时方法，这点要尤其注意。另外，由于服务端的方法本身就运行在服务端的Binder线程池中，所以服务端方法本身就可以执行大量耗时操作，这个时候切记不要在服务端方法中开线程进行异步任务，除非你明确知道自己在干什么，否则不建议这么做。
然后作者接着说：
同理，当远程服务端需要调用客户端的listener中的方法时，被调用的方法和运行在Binder线程池中，只不过是客户端的线程池。所以，我们同样不可以在服务端中调用客户端的耗时方法。比如针对BookManagerService的onNewBookArrived方法，如下所示。它在内部调用了客户端的IOnNewBookArrivedListener中的onNewBookArrived方法，如果客户端的这个onNewBookArrived方法比较耗时的话，那么请确保BookManagerService中的onNewBookArrived运行在非UI线程中，否则将导致服务端无法响应。
另外，由于客户端的IOnNewBookArrivedListener中的onNewBookArrived方法运行在客户端的Binder线程池中，所以不能再它里面去访问UI相关的内容，如果要访问UI，请使用Handler切换到UI线程....
这里有疑问，前面说服务端的方法本身就运行在服务端的Binder线程池中，所以服务端本身就可以执行大量耗时操作。后面说同理，....我们同样不可以在服务端中调用客户端的耗时方法？这里需要重读一下。
AIDL 权限验证（P90）
方法一：在onBind中进行验证，验证不通过就直接返回null，这样验证失败的客户端直接无法绑定服务，至于验证方式可以有很多种，比如使用permission验证。使用这种验证方式，我们要先在AndroidManifest中声明所需的权限，比如：

```xml
<permission android:name="com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE"
android:protectionLevel="normal" />
```

定义权限以后，就可以在BookManagerService的onBind方法中做权限验证了，如下所示。

```Java,default
@Override 
public IBinder onBind(Intent intent) {
    int check =
            checkCallingOrSelfPermission("com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE");
    Log.d(TAG, "onbind check=" + check);
    if (check == PackageManager.PERMISSION_DENIED) {
        return null;
    }
    return mBinder;
}
```

一个应用来绑定我们的服务时，会验证这个应用的权限，如果它没有使用这个权限，onBind方法就会直接返回null，最终结果是这个应用无法绑定到我们的服务，这样就到达了权限验证的效果，这种方法同样适用于Messenger中，读者可以自行扩展。
如果我们自己内部的应用想绑定到我们的服务中，只需要在它的AndroidManifest文件中采用如下方式使用permission即可。

```xml
<uses-permission android:name="com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE" />
```

方法二：在服务端的onTransact方法中进行权限验证，如果验证失败就直接返回false，这样服务端就不会终止执行AIDL中的方法从而达到保护服务端的效果。验证方式也有很多种，可以采用permission验证，具体实现方式和第一种方法一样，还可以采用Uid和Pid来做验证，通过getCallingUid和getCallingPid可以拿到客户端所属应用的Uid和Pid，通过这两个参数我们可以做一些验证工作，比如验证包名。略...

```Java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    int check =
            checkCallingOrSelfPermission("com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE");
    Log.d(TAG, "check=" + check);
    if (check == PackageManager.PERMISSION_DENIED) {
        return false;
    }
    String packageName = null;
    String[] packages = getPackageManager().getPackagesForUid(getCallingUid());
    if (packages != null && packages.length > 0) {
        packageName = packages[0];
    }
    Log.d(TAG, "onTransact: " + packageName);
    if (!packageName.startsWith("com.ryg")) {
        return false;
    }
    return super.onTransact(code, data, reply, flags);
}
```
其他方法：比如为Service指定android:permission属性等。










