## Bitmap 和 Drawabl

**Bitmap** **是什么**

Bitmap 是位图信息的存储，即⼀个矩形图像每个像素的颜色信息的存储器。

**Drawable 是什么**

Drawable是一个可以调用 Canvas 来进行绘制的上层工具。调用`Drawable.draw(Canvas) `可以把 Drawable 设置的绘制内容绘制到 Canvas中。

Drawable 内部存储的是绘制规则，这个规则可以是一个具体的 Bitmap，也可以是一个纯粹的颜色，甚至可以是一个抽象的、灵活的描述。Drawable 可以不含有具体的像素信息，只要它含有的信息足以在 `draw(Canvas)` 方法被调用时进行绘制就够了。

由于 Drawable 存储的只是绘制规则，因此在它的 `draw()` 方法被调用前，需要先调用 `Drawable.setBounds` 来为它设置绘制边界。

**Bitmap 和 Drawable 的互相转换**

事实上，由于 Bitmap 和 Drawable 是两个不同的概念，因此确切地说它们并不是互相「转换」，而是从其中一个获得另一个的对象：

* Bitmap -> Drawable：创建一个 BitmapDrawable。
* Drawable -> Bitmap：如果是 BitmapDrawable，使 `BitmapDrawable.getBitmap()` 直接获取，如果不是，创建一个Bitmap和Canvas，使用 Drawable 通过Canvas 把内容绘制到 Bitmap 中。

**自定义 Drawable**

* 怎么做？
  * 重写几个抽象方法
  * 重写 setAlpha() 的时候记得重写 getAlpha()
  * 重写 draw(Canvas) 方法，然后在里面做具体的绘制工作
  * 例如：MeshDrawable

* 有用吗？

  有用。它就是一个更加抽象和专注的、仅仅用于绘制的自定义 View 模块。

* 用来干嘛？

  * 需要共享在多个 View 之间的绘制代码，写在 Drawable里，然后在多个自定义 View 里面只要引用相同的 Drawable 就好，而不用互相粘贴代码。

  * 例如？

    股票软件的多个蜡烛图（candlestick graph）界面，可以把共享的蜡烛图界面放进去

**蜡烛图（candlestick graph）自定义控件：**

https://github.com/PhilJay/MPAndroidChart