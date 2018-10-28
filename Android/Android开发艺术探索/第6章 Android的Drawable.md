# 第6章 Android的Drawable

> Drawable表示的是一种可以在Canvas上进行绘制的抽象的概念，它的种类有很多，最常见的颜色和图片都可以是Drawable。

## 6.1 Drawable简介

在实际开发中，Drawable常被用来作为View的背景使用。Drawable是一个抽象类，它是所有Drawable对象的基类

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/6-1.png)

图6-1 Drawable的层次关系

Drawable的内部宽/高这个参数比较重要，通过getIntrinsicWidth和getIntrinsicHeight这两个方法可以获取到它们。但是并不是所有的Drawable都有内部宽/高，比如一张图片所形成的Drawable，它的内部宽/高就是图片的宽/高，但是一个颜色所形成的Drawable，它就没有内部宽/高的概念。另外需要注意的是，Drawable的内部宽/高不等同于它的大小，一般来说，Drawable是没有大小概念的，当用作View的背景时，Drawable会被拉伸至View的同等大小。

## 6.2 Drawable的分类

### 6.2.1 BitmapDrawable

### 6.2.2 ShapeDrawable

而对于shape来说，默认情况下它是没有固有宽/高这个概念的，这个时候getIntrinsicWidth和getIntrinsicHeight会返回-1，但是如果通过<size>标签来指定宽/高信息，那么这个时候shape就有了所谓的固有宽/高。因此，总结来说，<size>标签设置的宽/高就是ShapeDrawable的固有宽/高，但是作为View的背景时，shape还会被拉伸或者缩小为View的大小。

### 6.2.3 LayerDrawable

下面是一个layer-list具体使用的例子，它实现了微信中的文本输入框的效果，如图6-5所示。当然它只适用于白色背景上的文本输入框，另外这种效果也可以采用.9图来实现。

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/6-5.jpg)

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" >
    <item>
        <shape android:shape="rectangle" >
            <solid android:color="#0ac39e" />
        </shape>
    </item>
    <item android:bottom="6dp">
        <shape android:shape="rectangle" >
            <solid android:color="#ffffff" />
        </shape>
    </item>
    <item
    android:bottom="1dp"
    android:left="1dp"
    android:right="1dp">
        <shape android:shape="rectangle" >
            <solid android:color="#ffffff" />
        </shape>
    </item>
</layer-list>
```



### 6.2.4 StateListDrawable

### 6.2.5 LevelListDrawable

### 6.2.6 TransitionDrawable

### 6.2.7 InsetDrawable

InsetDrawable对应于<inset>标签，它可以将其他Drawable内嵌到自己当中，并可以在四周留出一定的间距。当一个View希望自己的背景比自己的实际区域小的时候，可以采用InsetDrawable来实现。

### 6.2.8 ScaleDrawable

ScaleDrawable有点费解，要理解它，我们首先要明白等级对ScaleDrawable的影响。等级0表示ScaleDrawable不可见，这是默认值，要想ScaleDrawable可见，需要等级不能为0，

### 6.2.9 ClipDrawable

## 6.3 自定义Drawable

Drawable的使用范围很单一，一个是作为ImageView中的图像来显示，另外一个就是作为View的背景，大多数情况下Drawable都是以View的背景这种形式出现的。Drawable的工作原理很简单，其核心就是draw方法。

通常我们没有必要去自定义Drawable，这是因为自定义的Drawable无法在XML中使用，这就降低了自定义Drawable的使用范围。

另外getIntrinsicWidth和getIntrinsicHeight这两个方法需要注意一下，当自定义的Drawable有固有大小时最好重写这两个方法，因为它会影响到View的wrap_content布局，比如自定义Drawable是绘制一张图片，那么这个Drawable的内部大小就可以选用图片的大小。