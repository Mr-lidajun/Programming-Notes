#第7章 Android Framework用到了哪些跨进程通信方式

> 本章主要讲UI体系相关面试问题，包括UI刷新机制，涉及到vsync和choreographer原理。另外还会讲到surface的相关原理，涉及到应用和WMS、surfaceFlinger通信。 

# 6-1 说说屏幕刷新的机制

## 看几个问题？

* 丢帧一般是什么原因引起的？
  * 主线程有耗时操作，耽误了view的绘制
* Android刷新频率60帧/秒，每隔16ms调onDraw绘制一次？
  * 这个刷新频率实际上指的Vsync信号发生的频率，但是不一定每次Vsync信号都会去绘制，需要应用端先主动发起重绘，这样才会向SurfaceFlinger请求接收Vsync信号。再下次Vsync信号来的时候才会真正的去绘制
* onDraw完之后屏幕会马上刷新么？
  * onDraw之后，屏幕不会立即刷新，它是要等到下次Vsync信号来的时候才刷新
* 如果界面没有重绘，还会每隔16ms刷新屏幕么？
  * 如果界面没有重绘，应用就不会收到Vsync信号，但是屏幕还是会以60帧/秒的速度刷新，只不过这个画面数据一直用的是旧的，所以看起来画面没什么变化而已
* 如果在屏幕快要刷新的时候才去onDraw绘制会丢帧么？
  * 代码发起的view重绘，并不会马上执行，它都是等到下次Vsync信号来的时候才开始，所以你什么时候发起重绘操作其实并没有太大关系

## 说说Android的UI刷新机制

* Vsync的原理是怎样的？
* Choreographer的原理是怎样的？
* UI刷新的大致流程，应用和SurfaceFlinger的通信过程？

# 6-3 surface跨进程传递原理

* 怎么理解surface，它是一块buffer吗？
* 如果是，surface跨进程传递怎么带上这个buffer？
* 如果不是，那surface跟buffer又是什么关系？
* surface到底是怎么跨进程传递的？

## 总结

* surface本质是GraphicBufferProducer，而不是buffer
* surface跨进程传递，本质就是GraphicBufferProducer的传递
  * GraphicBufferProducer实际上是一个binder对象，跨进程传递很快

## 回答这几个问题

* 怎么理解surface，它是一块buffer吗？
  * surface不是一个buffer，它只是一个壳子，里面包含了能生产buffer的对象，就是GraphicBufferProducer，简称GBP
* 如果是，surface跨进程传递怎么带上这个buffer？
* 如果不是，surface又是怎么跨进程传递的？
  * surface跨进程传递主要是这个GraphicBufferProducer对象
* Activity的surface在系统中创建后，是怎么跨进程传回应用的？
  * 这个系统中创建的是surfaceControl对象，不是surface对象，这个surfaceControl里面有我们感兴趣的东西，就是GBP，有了这个GBP，其他的都不是事儿，我们可以创建一个surface，然后跨进程返回到应用

```java
private void performTraversals() {
	final View host = mView;
    
    if (mFirst) {
        relayoutResult = relayoutWindow(params, ...);
        ....
    }
    ....
}
```

```java
int relayoutWindow(WindowManager.LayoutParams params, ...) {
    // final Surface mSurface = new Surface();
    return mWindowSession.relayout(mWindow, ..., mSurface);
}
```

```java
public int relayout(IWindow window, ..., Surface outSurface) {
    int res = mService.relayoutWindow(this, window, ..., outSurface);
    return res;
}
```

```java
public int relayoutWindow(Session session, ..., Surface outSurface) {
	....
    SurfaceControl surfaceControl = winAnimator.createSurfaceLocked();
    outSurface.copyFrom(surfaceControl);
}
```

## 6-4 surface的绘制原理



## 总结

* surface绘制的buffer是怎么来的？
  * 通过GBP（GraphicBufferProducer）向BufferQueue申请的
* buffer绘制完了又是怎么提交的？
  * 通过GBP（GraphicBufferProducer）向BufferQueue提交的

<font color=red>**所以对于surface来说，GBP是它的灵魂一点也不为过**</font>

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Framework/img/060401.png)

