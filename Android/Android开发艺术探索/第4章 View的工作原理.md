# 第4章 View的工作原理
> 需要掌握View的底层工作原理，比如View的测量流程、布局流程以及绘制流程。
> 除了View的三大流程以外，View常见的回调方法也是需要熟练掌握，比如构造方法、onAttach、onVisibilityChanged、onDetach等。我们还需要处理View的滑动，如果遇到滑动冲突就还需要解决相应的滑动冲突。总结来说，自定义View是有几种固定类型的，有的直接继承自View和ViewGroup，而有的则选择继承现有的系统控件，这些都可以，关键是要选择最适合当前需要的方式，选对自定义View的实现方式可以起到事半功倍的效果。

## 4.1 初识ViewRoot和DecorView
ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象（在WindowManagerGlobal中创建的），并将ViewRootImol对象和DecorView建立关联，这个过程可参看如下源码：

```java
root = new ViewRootImpl(view.getContext(),display);
root.setView(view,wparams,panelParentView);
```

View的绘制流程是从ViewRoot的performTraversals方法开始的，它经过measure、layout和draw三个过程才能最终将一个View绘制出来，其中measure用来测量View的宽和高，layout用来确定view在父容器中的放置位置，而draw则负责将View绘制在屏幕上。针对performTraversals的大致流程，可用流程图4-1来表示。
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/4-1.png)
图4-1 performTraversals的工作流程图

如图4-1所示，其中在performMeasure中会调用measure方法，在measure方法中又会调用onMeasure方法，**在DecorView的onMeasure方法中（DecorView extends FrameLayout，）则会对所有子元素进行measure过程**（调用ViewGroup的**measureChildWithMargins**方法），这个时候measure流程就从父容器传递到子元素中了，这样就完成了一次measure过程。接着子元素会重复父容器的measure过程，如此反复就完成了**整个View树的遍历**。

- measure过程决定了View的宽/高，Measure完成以后，可以通过getMeasuredWidth和getMeasureHeight方法来获取到View测量后的宽/高（存在特殊情况）。
- Layout过程决定了View的四个顶点的坐标和实际的View的宽/高，完成以后，可以通过getTop、getBottom、getLeft和getRight来拿到View的四个顶点的位置，并可以通过getWidth和getHeight方法来拿到View的最终宽/高。
- Draw过程则决定了View的显示，只有draw方法完成以后View的内容才能呈现在屏幕上。

如图4-2所示，DecorView作为顶级View，一般情况下它内部会包含一个竖直方向的LinearLayout，在这个LinearLayout里面有上下两个部分（具体情况和Android版本及主题有关），上面是标题栏，下面是内容栏。在Activity中我们通过setContentView所设置的布局文件其实就是被加到内容栏之中的，而内容栏的id是content。如何得到content呢？ViewGroup content = findViewById(R.id.content)。如何得到我们设置的View呢？content.getChildAt(0)。通过源码我们可以知道，DecorView其实是一个FrameLayout，View层的事件都先经过DecorView，然后才传递给我们的View。
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/4-2.jpeg)
图4-2 顶级View：DecorView的结构

## 4.2 理解MeasureSpec
MeasureSpec看起来像“测量规格”或者“测量说明书”，MeasureSpec在很大程度上决定了一个View的尺寸规格，之所以说是很大程度上是因为这个过程还受父容器的影响，因为父容器影响View的MeasureSpec的创建过程。在测量过程中，系统会将View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec来测量出View的宽/高。
### 4.2.1 MeasureSpec
MeasureSpec代表一个32位int值，高2位代表SpecMode，低30位代表SpecSize，SpecMode是指测量模式，而SpecSize是指在某种测量模式下的规格大小。
MeasureSpec通过将SpecMode和SpecSize打包成一个int值来避免过多的对象内存分配，为了方便操作，其提供了打包和解包方法。

SpecMode分为三类：
**UNSPECIFIED**
父容器不对View有任何限制，要多大给多大，这种情况一般用于系统内部，表示一种测量的状态。（主要用于系统内部多次Measure的情形，一般来说，我们不需要关注此模式）
**EXACTILY**
父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的值。它对应于LayoutParams中的match_parent和具体的数值这两种模式。
**AT_MOST**
父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么值要看不同View的具体实现。它对应于LayoutParams中的wrap_content。

### 4.2.2 MeasureSpec和LayoutParams的对应关系
在View测量时，系统会将LayoutParams在父容器的约束下转换成对应的MeasureSpec，然后再根据这个MeasureSpec来确定View测量后的宽/高。对于DecorView，其MeasureSpec由窗口的尺寸和其自身的LayoutParams来共同确定；对于普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定，MeasureSpec一旦确定后，onMeasure中就可以确定View的测量宽/高。

ViewRootImpl中的measureHierarchy方法：

```java
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth,lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight,lp.height);
performMeasure(childWidthMeasureSpec,childHeightMeasureSpec);
```

getRootMeasureSpec方法的实现：

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

DecorView的MeasureSpec产生过程遵守如下规则，根据它的LayoutParams中的宽/高的参数来划分。

- LayoutParams.MATCH_PARENT：精确模式，大小就是窗口的大小；
- LayoutParams.WRAP_CONTENT：最大模式，大小不定，但是不能超过窗口的大小；
- 固定大小（比如100dp）：精确模式，大小为LayoutParams中指定的大小。

对于普通View来说，这里是指我们布局中的View，View的measure过程由ViewGroup传递而来，先看一下ViewGroup的measureChildWithMargins方法：

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

从代码来看，很显然，子元素的MeasureSpec的创建与父容器的MeasureSpec和子元素本身的LayoutParams有关，此外还和View的margin及padding有关，具体情况可以看一下ViewGroup的getChildMeasureSpec方法，如下所示。

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = 0;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = 0;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

上述方法主要作用是根据父容器的MeasureSpec同时结合View本身的LayoutParams来确定子元素的MeasureSpec，参数中的padding是指父容器中已占用的空间大小，因此子元素可用的大小为父容器的尺寸减去padding，具体代码如下所示。

```java
int specSize = MeasureSpec.getSize(spec);
int size = Math.max(0,specSize -padding);
```

表4-1 普通View的MeasureSpec的创建规则
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/T4-1.jpeg)

再次做一下说明，对于普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定

## 4.3 View的工作流程

















