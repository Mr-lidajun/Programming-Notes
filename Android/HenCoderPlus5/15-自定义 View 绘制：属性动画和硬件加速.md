## 硬件加速

**硬件加速是什么**

* 使用 CPU 绘制到 Bitmap，然后把 Bitmap 贴到屏幕，就是软件绘制；
* 使用 CPU 把绘制内容转换成 GPU 操作，交给 GPU，由 GPU 负责真正的绘制，
  就叫硬件绘制；
* 使用 GPU 绘制就叫做硬件加速

**怎么就加速了？**

* GPU 分摊了⼯作
* GPU 绘制简单图形（例如⽅形、圆形、直线）在硬件设计上具有先天优势，会更快
* 流程得到优化（重绘流程涉及的内容更少）

**硬件加速的缺陷**

  兼容性。由于使用 GPU 的绘制（暂时）无法完成某些绘制，因此对于⼀些特定的API，需要关闭硬件加速来转回到使用 CPU 进行绘制。

## 离屏缓冲

* 离屏缓冲是什么：单独的⼀个绘制 View（或 View 的⼀部分）的区域

* setLayerType() 和 saveLayer()
  * setLayerType() 是对整个 View，不能针对 onDraw() ⾥⾯的某⼀具体过程
    * 这个方法常用来关闭硬件加速，但它的定位和定义都不只是一个「硬件
      加速开关」。它的作用是为绘制设置一个离屏缓冲，让后⾯的绘制都单独写在这个离屏缓冲内。如果参数填写 `LAYER_TYPE_SOFTWARE`，会把离屏缓冲设置为⼀个 Bitmap ，即使用软件绘制来进行缓冲，这样就导致在设置离屏缓冲的同时，将硬件加速关闭了。但需要知道，这个方法被用来关闭硬件加速，只是因为 Android 并没有提供⼀个便捷的方法在 View 级别简单地开关硬件加速而已。
  * saveLayer() 是针对 Canvas 的，所以在 onDraw() 里可以使用 saveLayer()来圈出具体哪部分绘制要用离屏缓冲
    * 然而.....最新的文档表示这个方法太重了，能不用就别用，尽量用setLayerType() 代替

* setLayerType，layerType类型
  * LAYER_TYPE_HARDWARE：开启View的离屏缓冲，并且使用硬件绘制来实现离屏缓冲
  * LAYER_TYPE_SOFTWARE：开启View的离屏缓冲，使用软件绘制来实现离屏缓冲；相当于对于该View，间接关闭了硬件加速
  * LAYER_TYPE_NONE：关闭view的离屏缓冲

**注意：**Android并没有提供单独的去关闭硬件加速的开关，所以使用 `setLayerType(LAYER_TYPE_SOFTWARE, null) `这个方式来关闭硬件加速就成为了Android中唯一的对View这边关闭硬件加速的方式了。

**setLayerType的使用场景：**

* 使用`setLayerType(LAYER_TYPE_SOFTWARE, null) `来间接的，强行关闭View界别的硬件加速
* 通过在运行动画的时候，先调用`setLayerType(LAYER_TYPE_HARDWARE, null) `，然后再开启动画，动画完成之后再把它给关闭掉，通过这种方式可以提高动画性能，前提是自带属性的那些动画，自定义属性没有作用

```kotlin
view.animate().translationY(200.dp).withLayer()
```

