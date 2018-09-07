#第1章 Android的生命周期和启动模式
## 1.1 Activity的生命周期全面分析
### 1.1.1 典型情况下的生命周期分析

1. 当用户打开新的Activity或者切换到桌面的时候，回调如下：onPause -> onStop。
这里有一种特殊情况，如果新Activity采用了透明主题，那么当前Activity不会回调onStop。
2. 两个问题
问题1：onStart 和 onResume、onPause和onStop从描述上来看差不多，对我们来说有什么实质的不同呢？
onStart和onStop是从Activity是否可见这个角度来回调的，而onResume和onPause是从Activity是否位于前台这个角度来回调的，除了这种区别，在实际使用中没有其他明显区别。
问题2：假设当前Activity为A，如果这时用户打开一个新ActivityB，那么B的onResume和A的onPause哪个先执行呢？
![Activity生命周期方法的回调顺序](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/010101.png)
图1-2 Activity生命周期方法的回调顺序

通过源码分析和日志打印可以发现，旧Acitivity的onPause先调用，然后新Activity才启动，这也证实了我们上面的分析过程。从另一个角度来说，Android官方文档对onPause的解释有这么一句：不能在onPause中做重量级的操作，因为必须onPause执行完成以后新Activity才能Resume，从这一点也能间接证明我们的结论。分析这个问题，我们知道onPause和onStop都不能执行耗时的操作，尤其是onPause，这也意味着，我们应当尽量在onStop中做操作，从而使得新的Activity尽快显示出来并切换到前台。
### 1.1.2 异常情况下的生命周期分析
#### 情况1：资源相关的系统配置发生改变导致Activity被杀死并重新创建
1. 当系统配置发生变化后，Activity会被销毁，其onPause、onStop、onDestroy均会被调用，同时Activity是在异常情况下终止的，系统会调用onSaveInstanceState来保存当前Activity的状态。这个方法的调用时机是在onStop之前，它和onPause没有既定的时序关系。
2. 当Activity被重新创建后，系统会调用onRestoreInstanceState，并且把Activity销毁时onSaveInstanceState方法所保存的Bundle对象作为参数同时传递给onRestoreInstanceState和onCreate方法。从时序上来说，onRestoreInstanceState的调用时机在onStart之后。
3. 系统自动为我们做了一定的恢复工作。当Activity在异常情况下需要重新创建时，系统会默认为我们保存当前Activity的视图结构，并且在Activity重启后为我们恢复这些数据，比如文本框中用户输入的数据、ListView滚动的位置等，这些View相关的状态系统都能默认为我们恢复。
4. 保存和恢复View层次结构，系统的工作流程：
    1. 首先Activity被意外终止时，Activity会调用onSaveInstanceState去保存数据。
    2. 然后Activity会委托window去保存数据，接着Window再委托它上面的顶级容器去保存数据。（顶级容器是一个ViewGroup，一般来说它很可能是DecorView。）
    3. 最后顶层容器再去一一通知它的子元素来保存数据
    4. 总结：这是一种典型的委托思想，上层委托下层、父容器委托子元素去处理一些事情，这种思想在Android中有很多应用，比如View的绘制过程、事件分发等。数据恢复过程也是类似的。
    * 举个栗子：查看源码，TextView#onSaveInstanceState，可以很容易看出，TextView保存了文本选中状态和文本内容。
    * 注意：系统只会在Activity即将被销毁并且有机会重新显示的情况下才会去调用它。简单理解，系统只会在Activity异常终止的时候才会调用onSaveInstanceState和onRestoreInstanceState来存储和恢复数据。
#### 情况2：资源内存不足导致低优先级的Activity被杀死
1. Activity按照优先级从高到低，分为以下三种：
    1. 前台Activity----正在和用户交互的Activity，优先级最高。
    2. 可见但非前台Activity----比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法和用户直接交互。
    3. 后台Activity----已经被暂停的Activity，比如执行了onStop，优先级最低。
2. 如果一个进程中没有四大组件在执行，那么这个进程将很快被系统杀死；解决方法，将后台工作放入Service中从而保证进程有一定的优先级。
3. 如果当某项内容发生改变后，我们不想系统重新创建Activity，可以给Activity指定configChanges属性。比如不想让Activity在屏幕旋转的时候重新创建，就可以给configChanges属性添加orientation这个值
    ```
    android:configChanges="orientation"    
    ```
#### 疑问
当按Home键时，打印日志如下，执行了onSaveInstanceState方法（MainActivity跳转到ActivityB也会调用该方法），这个回调方法书上说是系统只在Activity异常终止的时候才会调用onSaveInstanceState和onRestoreInstanceState来存储和恢复数据，其他情况不会触发这个过程。这里有点奇怪 .....

```
02-08 14:00:30.567 31012-31012/com.ldj.androidexploreart D/MainActivity: onPause
02-08 14:00:30.876 31012-31012/com.ldj.androidexploreart D/MainActivity: onSaveInstanceState
02-08 14:00:30.877 31012-31012/com.ldj.androidexploreart D/MainActivity: onStop
```

## 1.2 Activity的启动模式
### 1.2.1 Activity的LaunchMode
1. singleTask启动模式中，多次提到某个Activity所需的任务栈，什么是Activity所需要的任务栈呢？这要从一个参数说起：TaskAffinity，可以翻译为任务相关性。
2. 当TaskAffinity和singleTask启动模式配对使用的时候，它是具有该模式的Activity的目的任务栈的名字，待启动的Activity会运行在名字和TaskAffinity相同的任务栈中。
3. 当一个应用A启动了应用B的某个Activity后，如果这个Activity的allowTaskReparenting属性为true的话，那么当应用B被启动后，此Activity会直接从应用A的任务栈转移到应用B的任务栈中。

```
执行命令：adb shell dumpsys activity
关键字：ACTIVITY MANAGER ACTIVITIES
```

### 1.2.2 Activity的Flags
* FLAG_ACTIVITY_NEW_TASK：为Activity指定“singleTask”启动模式，其效果和在XML中指定该启动模式相同。
* FLAG_ACTIVITY_SINGLE_TOP：为Activity指定“singleTop”启动模式，其效果和在XML中指定该启动模式相同。
* FLAG_ACTIVITY_CLEAR_TOP：在同一个任务栈中所有位于它上面的Activity都要出栈。一般需要和FLAG_ACTIVITY_TASK配合使用，在这种情况下，被启动Activity的实例如果已经存在，那么系统就会调用它的onNewIntent。如果被启动的Activity采用standard模式启动，那么它连同它之上的Activity都要出栈，系统会创建新的Activity实例并放入栈顶。
* FLAG_ACTIVITY_EXCLUDE_FROM_RECE：具有这个标记的Activity不会出现在历史Activity的列表中，等同于在XML中指定Activity的属性android:excludeFromRecents="true"。
## 1.3 IntentFilter的匹配规则
* IntentFilter中的过滤信息有action、category、data

```
<activity
    android:name="com.ldj.chapter_1.ThirdActivity"
    android:configChanges="screenLayout"
    android:label="@string/app_name"
    android:launchMode="singleTask"
    android:taskAffinity="com.ldj.task1" >
    <intent-filter>
        <action android:name="com.ldj.charpter_1.c"/>
        <action android:name="com.ldj.charpter_1.d"/>
        <category android:name="com.ldj.category.c"/>
        <category android:name="com.ldj.category.d"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="text/plain"/>
    </intent-filter>
</activity>
```

* 只有一个Intent同时匹配action类别、category类别、data类别才算完全匹配，只有完全匹配才能成功启动目标Activity。
* 一个Activity中可以有多个intent-filter，一个Intent只要能匹配任何一组intent-filter即可出锅启动对应的Activity
### 1.3.1 action的匹配规则
Intent中的action必须能够和过滤规则中的action匹配，这里说的匹配是指action的字符串值完全一样。一个过滤规则中可以有多个action，那么只要Intent中的action能够和过滤规则中的任何一个action相同即可匹配成功。
### 1.3.2 category的匹配规则
* category的匹配规则和action不同，它要求Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。换句话说，Intent中如果出现了category，不管有几个category，对于每个category来说，它必须是过滤规则中已经定义了的category。Intent中可以没有category，如果没有category，这个Intent仍然可以匹配成功
* 为什么不设置category也可以匹配呢？原因是系统在调用startActivity或者startActivityForResult的时候会默认为Intent加上“android.intent.category.DEFAULT”这个category。同时，为了我们的activity能够接收隐式调用，就必须在intent-filter中指定“android.intent.category.DEFAULT”这个category。
### 1.3.3 data的匹配规则
* data的匹配规则和action类似，如果过滤规则中定义了data，那么Intent中必须也要定义可匹配的data
* data由两部分组成，mimeType和URI
    1. mimeType：指媒体类型，比如image/jpeg、audio/mpeg4-generic和video/*等
    2. URI：URI结构，<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]
        * content://com.example.project:200/folder/subfolder/etc
        * http://www.baidu.com:80/search/info
        * 每个数据的含义：
            * Scheme：URI的模式，比如http、file、content等
            * Host：URI的主机名，比如www.baidu.com
            * Port：URI中的端口号，比如80
            * Path、pathPattern、pathPrefix：路径信息，其中path表示完整路径信息；pathPattern也表示完整路径信息，但是可以包含通配符“*”，“*”表示0个或多个任意字符，需要注意的是，由于正则表达式的规范，如果想表示真实的字符串，那么“*”要写成“\\*”，“\”要写成“\\\\”；pathPrefix表示路径的前缀信息。
            * URI的默认值为content和file。也就是说，虽然没有指定URI，但是Intent中的URI部分的schema必须为content或者file才能匹配。
### 1.3.4 判断是否有Activity匹配我们的隐式Intent
* 原因：当我们他隐式方式启动一个Activity时，如果没有Activity能够匹配我们的隐式Intent，就会报错

```
08-28 14:34:19.032 D/AndroidRuntime(10021): Shutting down VM
08-28 14:34:19.033 E/AndroidRuntime(10021): FATAL EXCEPTION: main
08-28 14:34:19.033 E/AndroidRuntime(10021): Process: com.ldj.chapter_1, PID: 10021
08-28 14:34:19.033 E/AndroidRuntime(10021): android.content.ActivityNotFoundException: No Activity found to handle Intent { act=com.ldj.charpter_1.c cat=[com.ldj.category.c] dat=http://abc/... typ=text/plain (has extras) }
08-28 14:34:19.033 E/AndroidRuntime(10021): at android.app.Instrumentation.checkStartActivityResult(Instrumentation.java:1816)
```

* 两种判断方法：
    1. PackageManager的resolveActivity方法，另外，PackageManager还提供了queryIntentActivities方法返回所有成功匹配的Activity信息
        * public abstract List<ResolveInfo> queryIntentActivities(Intent intent,int flags);
        * public abstract ResolveInfo resolveActivity(Intent intent,int flags);
        * 第二个参数需要注意，我们要使用MATCH_DEFAULT_ONLY这个标记位，含义是仅仅匹配那些在intent-filter中声明了<category android:name="android.intent.category.DEFAULT"/>这个category的Activity。使用这个标记位的意义在于，只要上述两个方法不返回null，那么startActivity一定可以成功，如果不用这个标记，就可以把intent-filter中category不含DEFAULT的那些Activity给匹配出来，从而导致startActivity可能失败。因为不含有DEFAULT这个category的Activity是无法接收隐式Intent的。
    2. Intent的resolveActivity方法


