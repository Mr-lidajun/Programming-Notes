#第4章 Activity组件相关面试问题

> 这一章主要讲解Activity相关的机制，包括Activity的启动流程，显示原理等相关面试问题，通过本章的学习，我们不但能熟悉它，更能深入了解它。

#4-1 说说Activity的启动流程

## 面试官视角：这道题想考察什么？

* 启动Activity会经历哪些生命周期回调？
* 冷启动大致流程，涉及哪些组件，通信过程是怎么样的？
* Activity启动过程中，生命周期回调的原理？

## 冷启动大致流程，通信过程

![](img/040101.jpg)

<font color=red>创建Activity对象</font>

​			↓

<font color=red>准备好Application</font>

​			↓

<font color=red>创建ContextImpl</font>

​			↓

  <font color=red>attach上下文</font>

​			↓

   <font color=red>生命周期回调</font>



**流程及通信过程**：

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Framework/img/040102.jpg)

# 4-2 说说Activity的显示原理



## 相关问题

* Activity的显示原理（Window/DecorView/ViewRoot）
* Activity的UI刷新机制（Vsync/Choreographer）
* UI的绘制原理（Measure/Layout/Draw）
* Surface原理（Surface/SurfaceFlinger）

## 说说Activity的显示原理

* setContentView原理是什么？
* Activity在onResume之后才会显示的原因是什么？
* ViewRoot是干嘛的，是View Tree的rootView么？

## PhoneWindow

```java
final void attach(Context context,) {
    ....
    mWindow = new PhoneWindow(this);
    ....
}
```

* 创建Activity对象
* 创建Context对象
* 准备Application对象
* attach上下文
* Activity onCreate

## WMS（WindowManagerService）

时间：10:41

```java
mWidowSession.addToDisplay(mWindw, ...);
```

↓

mWidowSession

```java
IWindowManager windowManager = getWindowManagerService();
sWindowSession = windowManager.openSession(...);
```

↓

openSession

```java
IWindowSession openSession(IWindowSessionCallback callback, ...){
    Seesion session = new Session(this, callback, client, inputContext);
    return session;
}
```

```java
mWidowSession.addToDisplay(mWindw, ...);
```

↓

addToDisplay

```java
public int addToDisplay(IWindow window, int seq, ...) {
    return mService.addView(this, window, seq, attrs, ...);
}
```

↓

mWindw

```java
static class W extends IWindow.Stub {...}
```

### WMS主要作用

* 分配surface
* 掌管surface显示顺序及位置尺寸等
* 控制窗口动画
* 输入事件分发

## 总结（图文）

时间：12:46

##面试：说说Activity的显示原理

* PhoneWindow是什么，怎么创建的？
* setContentView原理，DecorView是什么？
* ViewRoot是什么？有什么作用？
* View的显示原理是什么？WMS发挥了什么作用？

# 4-3 应用的UI线程是怎么启动的

