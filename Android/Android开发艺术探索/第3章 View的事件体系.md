# 第3章 View的事件体系
> 很多情况下我们的应用都需要支持滑动操作，当处于不同层级的View都可以响应用户的滑动操作时，就会带来一个问题，那就是滑动冲突。如何解决滑动冲突呢？它需要我们对View的事件分发机制有一定的了解。

## 3.1 View基础知识
> 主要内容：View的位置参数、MotionEvent和TouchSlop对象、VelocityTracker、GestureDetector和Scroller对象

### 3.1.1 什么是view
View是Android中所有控件的基类，View是一种界面层的控件的一种抽象，它代表了一个控件。ViweGroup，翻译为控件组，言外之意是ViewGroup内部包含了许多个控件，即一组View。ViewGroup也继承了View，这就意味着View本身就可以是单个控件也可以是由多个控件组成的一组控件，通过这种关系就形成了View树的结构，这和Web前端中的DOM树的概念是相似的。
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/3-1.png)
图3-1 TestButton的层次结构
### 3.1.2 View的位置参数
> _View的位置主要由它的四个顶点来决定，分别对应于View的四个属性：top、left、right、bottom_，其中top是左上角纵坐标，left是左上角横坐标，right是右下角横坐标，bottom是右下角纵坐标。需要注意的是， _这些坐标都是相对于View的父容器来说的，因此它是一种相对坐标。在Android中，x轴和y轴的正方向分别为右和下，_这点不难理解， _不仅仅是Android，大部分显示系统都是按照这个标准来定义坐标系的。_

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/3-2.png)
图3-2 View的位置坐标和父容器的关系

* 根据图3-2，我们很容易得出View的宽高和坐标的关系：

```java
width = right - left
height = bottom - top
```

* 从Android3.0开始，View增加了额外的几个参数：x、y、translationX和translationY，其中**x和y是View左上角的坐标，而translationX和translationY是View左上角相当于父容器的偏移量。这几个参数也是相对于父容器的坐标**，并且translationX和translationY的默认值是0，和View的四个基本的位置参数一样，View也为它们提供了get/set方法，**这几个参数的换算关系如下所示**。

    ```java
    x = left + translationX
    y = left + translationY
    ```

* 需要注意的是，**View在平移的过程中，top和left表示的是原始左上角的位置信息，其值并不会发生改变**，此时发生改变的是x、y、translationX和translationY这个四个参数。

### 3.1.3 MotionEvent和TouchSlop
1. MotionEvent    
* 1-1 一次手指触摸屏幕的行为会触发一系列点击事件， 考虑如下几种情况：
    * 点击屏幕后离开松开，事件序列为DOWN -> UP；
    * 点击屏幕滑动一会再松开，事件序列为DOWN -> MOVE  ->...> MOVE -> UP
* 通过MotionEvent对象我们可以得到点击事件发生的x和y坐标。为止，系统提供了两组方法：getX/getY和getRawX/getRawY。它们的区别其实很简单，**getX/getY返回的是相对于当前View左上角的 x 和 y 坐标，而getRawX/getRawY返回的是相对于手机屏幕左上角的 x 和 y 坐标。**
2. TouchSlop    
TouchSlop是系统所能识别出的被认为是滑动的最小距离。这是一个常量，和设备有关，在不同设备上这个值可能是不同的，通过如下方式即可获取这个常量：

```java    
ViewConfiguration.get(getContext()).getScaledTouchSlop()。
```

* 这个常量有什么意义呢？**当我们在处理滑动时，可以利用这个常量来做一些过滤**，比如当两次滑动事件的滑动距离小于这个值，我们就可以认为未达到滑动距离的临界值，因此就可以认为它们不是滑动，这样可以有更好的用户体验。

* 可以在源码中找到这个常量的定义，在frameworks/base/core/res/res/values/config.xml文件中。

```xml
<!-- Base"touch slop" value used by ViewConfiguration as a movement threshold where scrolling should begin. -->
<dimen name="config_viewConfigurationTouchSlop">8dp</dimen>
```

### 3.1.4 VelocityTracker、GestureDetector和Scroller
1. VelocityTracker
    速度追踪，用于追踪手指在滑动过程中的速速，包括水平和竖直方法的速度。
    * 首先，要在View的onTouchEvent方法中添加要追踪的事件
    
    ```java
    VelocityTracker velocityTracker = VelocityTracker.obtarn();
    velocityTracker.addMovement(event);
    ```
    
    * 接着，获得当前速度
    
    ```java
    velocityTracker.computeCurrentVelocity(1000);
    int xVelocity = (int) velocityTracker.getXVelocity();
    int yVelocity = (int) velocityTracker.getYVelocity();
    ```
    
    * 这里需要注意的是：
        * 1-1 必须先计算速度再获取速度，即必须先调用computeCurrentVelocity方法才可以调用getX/YVelocity方法
        * 1-2 这个速度是可以为负的，它指的是一段时间内手指所滑过的像素数，当手指逆着Android坐标滑动，结果即为负数了。
        * 1-3 computeCurrentVelocity方法的参数是一个时间单元，单位为ms，如果参数为100，手指在100ms内划过了10像素，那水平速度即为10。参数为1000，手指在1000ms内划过了100个像素，那水平速度即为100。其实这两个速度是相等的（假设滑动过程都是均速），但结果却不同，因为这个速度是相对于这个时间单元参数的，这里需要理解一下。
    * 最后，当不需要它的时候，需要回收内存
    
        ```java
        velocityTracker.clear();
        velocityTracker.recycle();
        ```
    
2. GestureDetector
    手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。
    * 1-1 首先，需要创建一个GestureDetector对象并实现OnGestureListener接口，根据需要我们还可以实现OnDoubleTapListener从而能够监听双击行为：
    
    ```java
    GestureDetector  mGestureDetector = new GestureDetector(this);
    //解决长按屏幕后无法拖动的现象
    mGestureDetector.setIsLongpressEnabled(false);
    ```
    
    * 1-2 接着，接管目标View的OnTouchEvent方法
    
    ```java
    boolean consume = mGestureDetector.onTouchEvent(event);
    return consume;
    ```
    
    表3-1 OnGestureListener和OnDoubleTapListener中的方法
    ![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/T3-1.jpg)
    * 建议：如果只是监听滑动相关的，建议自己在onTouchEvent中实现，如果要监听双击这种行为的话，那么就使用GestureDetector
    
3. Scroller 
    **弹性滑动对象，用于实现View的弹性滑动。**我们知道，当使用View的scrollTo/scrollBy方法来进行滑动时，其过程是瞬间完成的，这个没有过渡效果的滑动用户体验不好。这个时候就可以使用Scroller来实现有过渡效果的滑动，其过程不是瞬间完成的，而是在一定的时间间隔内完成的。Scroller本身无法让View弹性滑动，它需要和View的computeScroll方法配合使用才能共同完成这个功能。那么如何使用Scroller呢？它的典型代码是固定的，代码略了~，至于它为什么能实现弹性滑动，这个在3.2节中会进行详细介绍。

```java
Scroller scroller = new Scroller(mContext);
// 缓慢滚动到指定位置
private void smoothScrollTo(int destX, int destY) {
  int scrollX = getScrollX();
  int delta = destX - scrollX;
  // 1000ms内滑向destX，效果就是慢慢滑动
  mScroller.startScroll(scrollX, 0, delta, 0, 1000);
  invalidate();
}

@Override public void computeScroll() {
  if (mScroller.computeScrollOffset()) {
    scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
    postInvalidate();
  }
}
```

## 3.2 View的滑动
掌握滑动的方法是实现绚丽的自定义控件的基础。通过三种方式可以实现View的滑动：
* 第一种是通过View本身提供的scrollTo/scrollBy方法来实现滑动；
* 第二种是通过动画给View施加平移效果来实现滑动；
* 第三种是通过改变View的LayoutParams使得View重新布局从而实现滑动。
### 3.2.1 使用scrollTo/scrollBy
    为了实现View的滑动，View提供了专门的方法来实现这个功能，那就是scrollTo和scrollBy。
参考博文：Android Scroller完全解析，关于Scroller你所需知道的一切
http://blog.csdn.net/guolin_blog/article/details/48719871

```java
/**
 * Set the scrolled position of your view. This will cause a call to
 * {@link #onScrollChanged(int, int, int, int)} and the view will be
 * invalidated.
 *
 * @param x the x position to scroll to
 * @param y the y position to scroll to
 */
public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) {
            postInvalidateOnAnimation();
        }
    }
}
```

```java
/**
 * Move the scrolled position of your view. This will cause a call to
 * {@link #onScrollChanged(int, int, int, int)} and the view will be
 * invalidated.
 *
 * @param x the amount of pixels to scroll by horizontally
 * @param y the amount of pixels to scroll by vertically
 */
public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}
```

从源代码可以看出，scrollBy实际上也是调用了scrollTo方法，它实现了基于当前位置的相对滑动，而scrollTo则实现了基于所传递参数的绝对滑动，这个不难理解。但是我们要明白滑动过程中View内部的两个属性mScrollX和mScrollY的改变规则，这两个属性可以通过getScrollX和getScrollY方法分别得到。这里先简要概括一下：**在滑动过程中，mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离，而mScrollY的值总是等于View上边缘和View内容上边缘在竖直方向的距离。**View边缘是指View的位置，由四个顶点组成，而View内容边缘是指View中的内容的边缘，**scrollTo和scrollBy只能改变View内容的位置而不能改变View在布局中的位置**。mScrollX和mScrollY的单位为像素，并且当View左边缘在View内容左边缘的右边时，mScrollX为正值，反之为负值；当View上边缘在View内容上边缘的下边时，mScrollY为正值，反之为负值。换句话说，**如果从左向右滑动，那么mScrollX为负值，反之为正值；如果从上往下滑动，那么mScrollY为负值，反之为正值。**
    注意：使用scrollTo和scrollBy来实现View的滑动，只能将View的内容进行移到，并不能将View本身进行移到，也就是说，不管怎么滑动，也不可能将当前View滑动到附近View所在的区域，这个需要仔细体会一下。
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/3-3.jpg)
图3-3 mScrollX和mScrollY的变换规律示意
### 3.2.2 使用动画
通过动画我们能够让一个View进行平移，而平移就是一种滑动。使用动画来移到View，主要是操作View的translationX和translationY属性，既可以采用传统的View动画，也可以采用属性动画。**这里需要注意传统的View动画的一个弊端**：传统的View动画并不能真正改变View的位置，这位带来一个很严重的问题。比如我们通过View动画将一个Button向右移到100px，并且这个View设置的有单击事件，然后你会惊奇地发现，单击新位置无法触发onClick事件，而单击原始位置仍然可以触发onClick事件，尽管Button已经不在原始位置了。这个问题带来的影响是致命的，但是它却又是可以理解的，因为**不管Button怎么做变换，但是它的位置信息（四个顶点和宽/高）并不会随着动画而改变，因此在系统眼里，这个Button并没有发生任何改变，它的真身仍然在原始位置。在这种情况下，单击新位置当然不会触发onClick事件了，因为Button的真身并没有发生改变，在新位置上只是View的影像而已**。
* 那么这种问题难道就无法解决了吗？
    解决方案：我们可以在新位置预先创建一个和目标Button一模一样的Button，它们不但外观一样连OnClick事件也一样。当目标Button完成平移动画后，就把目标Button隐藏，同时把预先创建的Button显示出来。
### 3.2.3 改变布局参数
第三种实现View滑动的方法，那就是改变布局参数，即改变LayoutParams。比如我们想把一个Button向右平移100px，我们只需要将这个Button的LayoutParams里的marginLeft参数的值增加100px即可，是不是很简单呢？还有一种情形，为了达到移到Button的目的，我们可以在Button的左边放置一个空的View，这个空View的默认宽度为0，当我们需要向右移到Button时，只需要重置空View的宽度即可，当空View的宽度增大时（假设Button的父容器是水平方向的LinearLayout），Button就自动被挤向右边，即实现了向右平移的效果。如何重新设置一个View的LayoutParams呢？很简单，如下所示。

```java
MarginLayoutParams params = (MarginLayoutParams) mButton1.getLayoutParams();
params.width +=100;
params.leftMargin +=100;
mButton1.requestLayout();
// 或者mButton1.setLayoutParams(params);
```

### 3.2.4 各种滑动方式的对比
上面分别介绍了三种不同的滑动方式，他们都能实现View的滑动，那么它们之间的差别是什么呢？
总结如下：
scrollTo/scrollBy: 操作简单，适合对View内容的滑动；
动画：操作简单，主要适用于没有交互的View和实现复杂的动画效果；
改变布局参数：操作稍微复杂，适用于有交互的View。    
## 3.3 弹性滑动
知道了View的滑动，我们还要知道如何实现View的弹性滑动，比较生硬地滑动过去，这种滑动方式的用户体验实在太差了，因此我们要实现溅近式滑动。那么如何实现弹性滑动呢？其实实现方法有很多，但是它们都有一个共同的思想：将一次大的滑动分成若干次小的滑动并在一个时间段内完成，弹性滑动的具体实现方式有很多，比如通过Scroller、Handle#postDelayed以及Thread#sleep等，下面一一进行介绍。
### 3.3.1 使用Scroller
我们来分析一下它的源码，从而探究为什么它能实现View的弹性滑动。

```java
Scroller scroller = new Scroller(mContext);
// 缓慢滚动到指定位置
private void smoothScrollTo(int destX, int destY) {
    int scrollX = getScrollX();
    int scrollY = destX - scrollX;
    // 1000ms内滑向destX，效果就是慢慢滑动
    mScoller.startScroll(scrollX, 0, deltaX, 0, 1000);
    invalidate();
}

@Override 
public void computeScroll() {
    if (mScroller.computeScrollOffset()) {
        scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
    }
}
```

上面是Scroller的典型的使用方法，这里先描述它的工作原理：当我们构造一个Scroller对象并且调用它的startScroll方法时，Scroller内部其实什么都没做，它只是保存了我们传递的几个参数，这几个参数从startScroll的原型上就可以看出来，如下所示。

```java
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
    mMode = SCROLL_MODE;
    mFinished = false;
    mDuration = duration;
    mStartTime = AnimationUtils.currentAnimationTimeMillis();
    mStartX = startX;
    mStartY = startY;
    mFinalX = startX + dx;
    mFinalY = startY + dy;
    mDeltaX = dx;
    mDeltaY = dy;
    mDurationReciprocal = 1.0f / (float) mDuration;
}
```

#### Scroller的基本用法其实还是比较简单的，主要可以分为以下几个步骤： 
1. 创建Scroller的实例 
2. 调用startScroll()方法来初始化滚动数据并刷新界面 
3. 重写computeScroll()方法，并在其内部完成平滑滚动的逻辑
参考博文：
Android Scroller完全解析，关于Scroller你所需知道的一切
http://blog.csdn.net/guolin_blog/article/details/48719871

这个方法的参数很清楚，startX和startY表示的是滑动的起点，dx和dy表示的是要滑动的距离，而duration表示的是滑动时间，即整个滑动过程完成所需要的时间，注意这里的滑动是指View内容的滑动而非View本身位置的改变。可以看到，仅仅调用startScroll方法是无法让View滑动的，因为它内部并没有做滑动相关的事，那么Scroller到底是如何让View弹性滑动的呢？答案就是startScroll方法下面的invalidate方法，虽然有点不可思议，但是的确是这样的。invalidate方法会导致View重绘，在View的draw方法中又会去调用computeScroll方法，computeScroll方法在View中是一个空实现，因此需要我们自己去实现，上面的代码已经实现了computeScroll又会去向Scroller获取当前的scrollX和scrollY；然后通过scrollTo方法实现滑动；接着又调用postInvalidate方法来进行第二次重绘，这一次重绘的过程和第一次重绘一样，还是会导致computeScroll方法被调用；然后继续向Scroller获取当前的scrollX和scrollY，并通过scrollTo方法滑动到新的位置，如此反复，直到整个滑动过程结束。
我们再看一下Scroller的computeScrollOffset方法的实现，如下所示。

```java
/**
 * Call this when you want to know the new location. If it returns true,
 * the animation is not yet finished.
 */
public boolean computeScrollOffset() {
    if (mFinished) {
        return false;
    }
    int timePassed = (int) (AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    if (timePassed < mDuration) {
        switch (mMode) {
            case SCROLL_MODE:
                final float x =
                        mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
            case FLING_MODE:
        ..............

                break;
        }
    } else {
        mCurrX = mFinalX;
        mCurrY = mFinalY;
        mFinished = true;
    } return true;
}
```

是不是突然就明白了？这个方法会根据时间的流逝来计算出当前的scrollX和scrollY的值。计算方法也很简单，大意就是根据时间流逝的百分比来算出scrollX和scrollY的值。计算方法也很简单，大意就是根据时间流逝的百分比来算出scrollX和scrollY改变的百分比并计算出当前的值，这个过程类似于动画中的插值器的概念，这里我们先不去深究这个具体过程。这个方法的返回值也很重要，它返回true表示滑动还未结束，false则表示滑动已经结束，因此当这个方法返回true时，我们要继续进行View的滑动。
通过上面的分析，我们应该明白Scroller的工作原理了，这里做一下概括：Scroller本身并不能实现View的滑动，它需要配合View的computeScroll方法才能完成弹性滑动的效果，它不断地让View重绘，而每一次重绘距滑动起始时间会有一个时间间隔，通过这个时间间隔Scroller就可以得到View当前的滑动距离，知道了滑动位置就可以通过scrollTo方法来完成View的滑动。就这样，View的每一次重绘都会导致View进行小幅度的滑动，而多次的小幅度滑动就组成了弹性滑动，这就是Scroller的工作机制。由此可见，Scroller的设计思想是多么值得称赞，整个过程中它对View没有丝毫的引用，甚至在它内部连计时器都没有。
### 3.3.2 通过动画
动画本身就是一种渐近的过程，因此通过它来实现的滑动天然就具有弹性效果，比如以下代码可以让一个View的内容在100ms内向左移动100像素。

```java
ObjectAnimator.ofFloat(targetView, "translationX", 0, 100).setDuration(100).start();
```

不过这里想说的并不是这个问题，我们可以利用动画的特性来实现一些动画不能实现的效果。还拿ScrollTo来说，我们也想模仿Scroller来实现View的弹性滑动，那么利用动画的特性，我们可以采用如下方式来实现：

```java
final int startX = 0;
final int startY = 100;
ValueAnimator animator = ValueAnimator.ofInt(0, 1).setDuration(1000);
aimator.addUpdateListener(new AnimatorUpdateListener()) {
    @Override public void onAnimationUpdate (ValueAnimator animator){
    float fraction = animator.getAnimatedFraction();
    mButton1.scrollTo(startX + (int) (deltaX * fraction), 0);
    }
};
animator.start();
```

在上述代码中，我们的动画本质上没有作用于任何对象上，它只是在1000ms内完成了整个动画过程。利用这个特性，我们就可以在动画的每一帧到来时获取动画完成的比例，然后再根据这个比例计算出当前View所要滑动的距离。注意，这里的滑动针对的是View的内容而非View本身。可以发现，这个方法的思想其实和Scroller比较类似，都是通过改变一个百分比配合scrollTo方法来完成View的滑动。需要说明一点，采用这种方法除了能够完成弹性滑动以外，还可以实现其他动画效果，我们完全可以在onAnimationUpdate方法中加上我们想要的其他操作。
### 3.3.3 使用延时策略
它的核心思想是通过发送一系列延时消息从而达到一种渐近式的效果，具体来说可以使用Handle或View的postDelayed方法，也可以使用线程的sleep方法。对于postDelayed方法来说，我们可以通过它来延时发送一个消息，然后在消息中进行View的滑动。如果连接不断发送这种延时消息，那么就可以实现弹性滑动的效果。对于sleep方法来说，通过在while循环中不断地滑动View和sleep，就可以实现弹性滑动的效果。
下面采用Handle开做个示例，其他方法思想都是类似的。下面的代码在大约1000ms内将View的内容向左移到了100像素。

```java
private static final int MESSAGE_SCROLL_TO = 1;
private static final int FRAME_COUNT = 30;
private static final int DELAYED_TIME = 33;
private int mCount = 0;

@SuppressLint("HandlerLeak") 
private Handler mHandler = new Handler() {
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MESSAGE_SCROLL_TO: {
                mCount++;
                if (mCount <= FRAME_COUNT) {
                    float fraction = mCount / (float) FRAME_COUNT;
                    int scrollX = (int) (fraction * 100);
                    mButton1.scrollTo(scrollX, 0);
                    mHandler.sendEmptyMessageDelayed(MESSAGE_SCROLL_TO, DELAYED_TIME);
                }
                break;
            }

            default:
                break;
        }
    };
};
```

## 3.4 View的事件分发机制
### 3.4.1 点击事件的传递规则
> 所谓点击事件的事件分发，其实就是对MotionEvent事件的分发过程，即当一个MotionEvent产生了以后，系统需要把这个事件传递给一个具体的View，而这个传递的过程就是分发过程。
> 点击事件的分发过程由三个很重要的方法来共同完成：dispatchTouchEvent、onInterceptTouchEvent和onTouchEvent

**public boolean dispatchTouchEvent(MotionEvent ev)**
用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。

**public boolean onInterceptTouchEvent(MotionEvent event)**
在上述方法内部调用，**用来判断是否拦截某个事件**，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会再次调用，返回结果表示是否拦截当前事件。

**public boolean onTouchEvent(MotionEvent event)**
在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。

* 上述三个方法到底有什么区别呢？它们是什么关系呢？其实它们的关系可以用如下伪代码表示：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean consume = false;
    if (onInterceptTouchEvent(ev)) {
        consume = onTouchEvent(ev);
    } else {
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
```
上述伪代码已经将三者的关系表现得淋漓尽致。通过伪代码，我们可以大致了解点击事件的传递规则：对于一个根ViewGroup来说，点击事件产生后，首先会传递给它，这时它的dispatchTouchEvent就会被调用，如果这个ViewGroup的onInterceptTouchEvent方法返回true就表示它要拦截当前事件，接着事件就会交给这个ViewGroup处理，即它的onTouchEvent方法就会被调用；如果这个ViewGroup的onInterceptTouchEvent方法返回false就表示它不拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素的dispatchTouchEvent方法就会被调用，如此反复直到事件被最终处理。

* 当一个点击事件产生后，它的传递过程遵循如下顺序：Activity -> Window -> View，即事件总是先传递给Activity，Activity再传递给Window，最后Window再传递给顶级View。顶级View接收到事件后，就会按照事件分发机制去分发事件。考虑一种情况，如果一个View的onTouchEvent返回false，那么它的父容器的onTouchEvent将会被调用，依此类推。
* 关于事件传递的机制，这里给出一些结论，根据这些结论可以更好地理解整个传递机制，如下所示。
* （1）同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。

* （2）正常情况下，一个事件序列只能被一个View拦截且消耗。这一条的原因可以参考（3），因为一旦一个元素拦截了某此事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别由两个View同时处理，但是通过特殊手段可以做到，比如一个View将本该自己处理的事件通过onTouchEvent强行传递给其他View处理。
* （3）某个View一旦决定拦截，那么这一个事件序列都只能由它来处理（如果事件序列能够传递给它的话），并且它的onInterceptTouchEvent不会再被调用。这条也很好理解，就是说当一个View决定拦截一个事件后，那么系统会把同一个事件序列内的其他方法都直接交给它来处理，因此就不用再调用这个View的onInterceptTouchEvent去询问它是否要拦截了。
* （4）某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件（onTouchEvent返回了false），那么同一事件序列中的其他事件都不会再交给它来处理，并且事件将重新交由它的父元素去处理，即父元素的onTouchEvent会被调用。意思就是事件一旦交给一个View处理，那么它就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理了，这就好比上级交给程序员一件事，如果这件事没有处理好，短期内上级就不敢再把事情交给这个程序员做了，二者是类似的道理。
* （5）如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。
* （6）ViewGroup默认不拦截任何事件。Android源码中ViewGroup的onInterceptTouch-Event方法默认返回false。
* （7）View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。
* （8）View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击的（clickable 和longClickable同时为false）。View的longClickable属性默认都为false，clickable属性要分情况，比如Button的clickable属性默认为true，而TextView的clickable属性默认为false。
* （9）View的enable属性不影响onTouchEvent的默认返回值。哪怕一个View是disable状态的，只要它的clickable或者longClickable有一个为true，那么它的onTouchEvent就返回true。
* （10）onClick会发生的前提是当前View是可点击的，并且它收到了down和up的事件。
* （11）事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外。

### 3.4.2 事件分发的源码解析
1. Activity对点击事件的分发过程
2. 顶级View对点击事件的分发过程
    * 2-1 当ViewGroup不拦截事件并将事件交由子元素处理时，mFirstTouchTarget!=null。反过来，一旦事件由当前ViewGroup拦截时，mFirstTouchTarget!=null就不成立。
    * 2-2 总结起来有两点：第一点，onInterceptTouchEvent不是每次事件都会被调用的，如果我们想提前处理所有的点击事件，要选择dispatchTouchEvent方法，只有这个方法能确保每次都会调用，当然前提是事件能够传递到当前的ViewGroup；另外一点，FLAG_DISALLOW_INTERCEPT标记位的作用给我们提供了一个思路，当面对滑动冲突时，我们可以是不是考虑用这种方法去解决问题？
    * 2-3 mFirstTouchTarget是否被赋值，将直接影响到ViewGroup对事件的拦截策略，如果mFirstTouchTarget为null，那么ViewGroup就默认拦截接下来同一序列中所有的点击事件
3. View对点击事件的处理过程
    如果OnTouchListener中的onTouch方法返回true，那么onTouchEvent就不会被调用，可见OnTouchListener的优先级高于onTouchEvent，这样做的好处是方便在外界处理点击事件。
    
## 3.5 View的滑动冲突

> 滑动冲突是如何产生的呢？
>
> 其实在界面中只要内外两层同时可以滑动，这个时候就会产生滑动冲突。

### 3.5.1 常见的滑动冲突场景

常见的滑动冲突场景可以简单分为如下三种：

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/3-4.jpg)

图3-4 滑动冲突的场景

- 场景1----外部滑动方向和内部滑动方向不一致；
- 场景2----外部滑动方向和内部滑动方向一致；
- 场景3----上面两种情况的嵌套。

场景1，主要是将ViewPager和Fragment配合使用所组成的页面滑动效果，每个页面内部往往又是一个ListView。

场景2，要么只有一层滑动，要么就是内外两层都滑动得很卡顿。在实际的开发中，这种场景主要是指内外两层同时能上下滑动或内外两层同时能左右滑动。

场景3，场景3是场景1和场景2两种情况的嵌套，比如在许多应用中会有这么一个效果：内层有一次场景1中的滑动效果，然后外层又有一个场景2中的滑动效果。具体说就是，外部有一个SlidedMenu效果，然后内部有一个ViewPager，ViewPager的每一个页面中又是一个ListView。它是几个单一的滑动冲突的叠加，因此只需要分别处理内层和中层、中层和外层之间的滑动冲突即可，而具体的处理方法其实是和场景1、场景2相同的。

### 3.5.2 滑动冲突的处理规则

- 场景1处理规则：当用户左右滑动时，需要让外部的View拦截点击事件，当用户上下滑动时，需要让内部View拦截点击事件。具体来说是：**根据滑动时水平滑动还是竖直滑动来判断到底由谁来拦截事件**，如图3-5所示，根据滑动过程中两个点之间的坐标就可以得出到底是水平滑动还是竖直滑动。如何根据坐标来得到滑动的方向呢？方法有很多种，比如可以依据滑动路径和水平方向所形成的夹角，也可以依据水平方向和竖直方向上的距离差来判断，某些特殊时候还可以依据水平和竖直方向的速度差来判断。这里我们可以通过水平和竖直方向的距离差来判断，比如竖直方向滑动的距离大就判断为竖直滑动，否则判断为水平滑动。

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/3-5.jpg)

图3-5 滑动过程示意

- 场景2处理规则：它比较特殊，它无法根据滑动的角度、距离差以及速度差来做判断，但是这个时候一般都能**在业务上找到突破点**，比如业务上有规定：当处于某种状态时需要外部View响应用户的滑动，而处于另外一种状态时则需要内部View来响应View的滑动，
- 场景3处理规则：滑动规则就更复杂了，和场景2一样，它无法直接根据滑动的角度、距离差以及速度差来做判断，**同样还是只能从业务上找到突破点**，具体方法和场景2一样，都是从业务的需求上得出相应的处理规则。

### 3.5.3 滑动冲突的解决方式

1. 外部拦截法

  指点击事件都先经过父容器的拦截处理，如果父容器需要此事件就拦截，如果不需要此事件就不拦截。外部拦截法需要重写父容器的onInterceptTouuchEvent方法，在内部做相应的拦截即可，这种方法的伪代码如下所示。

  ```java
  @Override
  public boolean onInterceptTouchEvent(MotionEvent event) {
      boolean intercepted = false;
      int x = (int) event.getX();
      int y = (int) event.getY();
      switch (event.getAction()) {
          case MotionEvent.ACTION_DOWN:
              intercepted = false;
              break;
          case MotionEvent.ACTION_MOVE:
              if (父容器需要当前点击事件) {
                  intercepted = true;
              } else {
                  intercepted = false;
              }
              break;
          case MotionEvent.ACTION_UP:
              intercepted = false;
              break;
          default:
              break;
      }
      mLastXIntercept = x;
      mLastYIntercept = y;
      return intercepted;
  }
  ```

  上述代码是外部拦截法的典型逻辑，针对不同的滑动冲突，只需要修改”父容器需要当前点击事件“这个条件即可，其他均不需做修改并且也不能修改。

  - 这里对上述代码再描述一下：

    - 在onInterceptTouochEvent方法中，首先是ACTION_DOWN这个事件，父容器必须返回false，即不拦截ACTION_DOWN事件，这是因为一旦父容器拦截了ACTION_DOWN，那么后续的ACTION_MOVE和ACTION_UP事件都会直接交由父容器处理，这个时候事件没法再传递给子元素了；

    - 其次是ACTION_MOVE事件，这个事件可以根据需要来决定是否拦截，如果父容器需要拦截就返回true，否则返回false；

    - 最后是ACTION_UP事件，这里必须要返回false，因为ACTION_UP事件本身没有太多意义。

      ​

  - 考虑一种情况，假设事件交由子元素处理，如果父容器在ACTION_UP时返回了true，就会导致子元素无法接收到ACTION_UP事件，这个时候子元素中的onclick事件就无法触发，但是父容器比较特殊，一旦它开始拦截任何一个事件，那么后续的事件都会交给它来处理，而ACTION_UP作为最后一个事件也必定可以传递给父容器，即便父容器的onInterceptTouchEvent方法在ACTION_UP时返回了false。

2. 内部拦截法

   指父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就交由父容器进行处理，这种方法和Android中的事件分发机制不一致，需要配合**requestDisallowInterceptTouchEvent**方法才能正常工作，它的伪代码如下，我们需要重写子元素的**dispatchTouchEvent**方法：

```java
@Override
public boolean dispatchTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            parent.requestDisallowInterceptTouchEvent(true);
            break;
        case MotionEvent.ACTION_MOVE:
            int deltaX = x - mLastX;
            int deltaY = y - mLastY;
            if (父容器需要此类点击事件) {
                parent.requestDisallowInterceptTouchEvent(false);
            }
            break;
        case MotionEvent.ACTION_UP:
            break;
        default:
            break;
    }
    mLastX = x;
    mLastY = y;
    return super.dispatchTouchEvent(event);
}
```

​	上述代码是内部拦截法的典型代码，当面对不同的滑动策略时只需要修改里面的条件即可，其他不需要做改动而且也不能有改动。除了子元素需要做处理以外，父元素也要默认拦截除了ACTION_DOWN以外的其他事件，这样当子元素调用parent.requestDisallowInterceptTouchEvent(false)方法时，父元素才能继续拦截所需的事件。

​	为什么父容器不能拦截ACTION_DOWN事件呢？

​	那是因为ACTION_DOWN事件并不受FLAG_DISALLOW_INTERCEPT这个标记位的控制（此时还未走子元素的dispatchTouchEvent方法，并未调用parent.requestDisallowInterceptTouchEvent(true)方法，所以disallowIntercept为默认值false），所以一旦父容器拦截ACTION_DOWN事件，那么所有的事件都无法传递到子元素中去，这样内部拦截就无法起作用了。父元素所做的修改如下所示。

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent event) {
    int action = event.getAction();
    if (action == MotionEvent.ACTION_DOWN) {
        return false;
    } else {
        return true;
    }
}
```

​	通过一个实例来分别介绍这两种方法。实现一个类似于ViewPager中嵌套ListView的效果，为了制造滑动冲突，我们写一个类似于ViewPager的控件即可，名字就叫HorizontalScrollViewEx

​	为了实现ViewPager的效果，我们定义了一个类似于水平的LinearLayout的东西，只不过它可以水平滑动，初始化时我们在它的内部添加若干个ListView，这样一来，由于它内部的Listview可以竖直滑动。而它本身又可以水平滑动，因此一个典型的滑动冲突场景就出现了，并且这种冲突属于类型1的冲突。根据滑动策略，我们可以选择水平和竖直的滑动距离差来解决滑动冲突。

​	P161

​	**具体代码请参考github中 Programming-Notes-Code项目代码**

- 案例一：外部拦截法

  Activity初始化代码

  ```java
  public class DemoActivity_1 extends Activity {
      private static final String TAG = "DemoActivity_1";
      private HorizontalScrollViewEx mListContainer;

      @Override protected void onCreate(@Nullable Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.demo_1);
          Log.d(TAG, "onCreate");
          initView();
      }

      private void initView() {
          LayoutInflater inflater = getLayoutInflater();
          mListContainer = (HorizontalScrollViewEx) findViewById(R.id.container);
          final int screenWidth = MyUtils.getScreenMetrics(this).widthPixels;
          final int screenHeight = MyUtils.getScreenMetrics(this).heightPixels;
          for (int i = 0; i < 3; i++) {
              ViewGroup layout =
                      (ViewGroup) inflater.inflate(R.layout.content_layout, mListContainer, false);
              layout.getLayoutParams().width = screenWidth;
              TextView textView = (TextView) layout.findViewById(R.id.title);
              textView.setText("page " + (i + 1));
              layout.setBackgroundColor(Color.rgb(255 / (i + 1), 255 / (i + 1), 0));
              createList(layout);
              mListContainer.addView(layout);
          }
      }

      private void createList(ViewGroup layout) {
          ListView listView = (ListView) layout.findViewById(R.id.list);
          ArrayList<String> datas = new ArrayList<>();
          for (int i = 0; i < 50; i++) {
              datas.add("name " + i);
          }

          ArrayAdapter<String> adapter =
                  new ArrayAdapter<>(this, R.layout.content_list_item, R.id.name, datas);
          listView.setAdapter(adapter);
          listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
              @Override
              public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                  Toast.makeText(DemoActivity_1.this, "click item", Toast.LENGTH_SHORT).show();
              }
          });
      }

  }
  ```




HorizontalScrollViewEx的具体实现


  ```java
  public class HorizontalScrollViewEx extends ViewGroup {
      private static final String TAG = "HorizontalScrollViewEx";

      private int mChildrenSize;
      private int mChildWidth;
      private int mChildIndex;

      // 分别记录上次滑动的坐标
      private int mLastX = 0;
      private int mLastY = 0;
      // 分别记录上次滑动的坐标(onInterceptTouchEvent)
      private int mLastXIntercept = 0;
      private int mLastYIntercept = 0;

      private Scroller mScroller;
      private VelocityTracker mVelocityTracker;

      public HorizontalScrollViewEx(Context context) {
          super(context);
          init();
      }

      public HorizontalScrollViewEx(Context context, AttributeSet attrs) {
          super(context, attrs);
          init();
      }

      public HorizontalScrollViewEx(Context context, AttributeSet attrs, int defStyleAttr) {
          super(context, attrs, defStyleAttr);
          init();
      }

      private void init() {
          mScroller = new Scroller(getContext());
          mVelocityTracker = VelocityTracker.obtain();
      }

      @Override
      public boolean onInterceptTouchEvent(MotionEvent ev) {
          boolean intercepted = false;
          int x = (int) ev.getX();
          int y = (int) ev.getY();

          switch (ev.getAction()) {
              case MotionEvent.ACTION_DOWN:
                  intercepted = false;
                  if (!mScroller.isFinished()) {
                      mScroller.abortAnimation();
                      intercepted = true;
                  }
              break;
              case MotionEvent.ACTION_MOVE:
                  int deltaX = x - mLastXIntercept;
                  int deltaY = y - mLastYIntercept;
                  if (Math.abs(deltaX) > Math.abs(deltaY)) {
                      intercepted = true;
                  } else {
                      intercepted = false;
                  }
              break;
              case MotionEvent.ACTION_UP:
                  intercepted = false;
                  break;
              default:
                  break;
          }

          Log.d(TAG, "intercepted=" + intercepted);
          mLastX = x;
          mLastY = y;
          mLastXIntercept = x;
          mLastYIntercept = y;

          return intercepted;
      }

      @Override public boolean onTouchEvent(MotionEvent event) {
          mVelocityTracker.addMovement(event);
          int x = (int) event.getX();
          int y = (int) event.getY();

          switch (event.getAction()) {
              case MotionEvent.ACTION_DOWN:
                  if (!mScroller.isFinished()) {
                      mScroller.abortAnimation();
                  }
                  break;
              case MotionEvent.ACTION_MOVE:
                  int deltaX = x - mLastX;
                  int deltaY = y - mLastY;
                  scrollBy(-deltaX, 0);
                  break;
              case MotionEvent.ACTION_UP:
                  int scrollX = getScrollX();
                  int scrollToChildIndex = scrollX / mChildWidth;
                  mVelocityTracker.computeCurrentVelocity(1000);
                  float xVelocity = mVelocityTracker.getXVelocity();
                  if (Math.abs(xVelocity) >= 50) {
                      mChildIndex = xVelocity > 0 ? mChildIndex - 1 : mChildIndex + 1;
                  } else {
                      mChildIndex = (scrollX + mChildWidth / 2) / mChildWidth;
                  }
                  mChildIndex = Math.max(0, Math.min(mChildIndex, mChildrenSize - 1));
                  int dx = mChildIndex * mChildWidth - scrollX;
                  smoothScrollBy(dx, 0);
                  mVelocityTracker.clear();
                  break;
              default:

              break;
          }

          mLastX = x;
          mLastY = y;
          return true;
      }

      private void smoothScrollBy(int dx, int dy) {
          mScroller.startScroll(getScrollX(), 0, dx, 0, 500);
          invalidate();
      }

      @Override
      public void computeScroll() {
          if (mScroller.computeScrollOffset()) {
              scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
              postInvalidate();
          }
      }

      @Override
      protected void onDetachedFromWindow() {
          mVelocityTracker.recycle();
          super.onDetachedFromWindow();
      }

      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          super.onMeasure(widthMeasureSpec, heightMeasureSpec);
          int measureWidth = 0;
          int measureHeight = 0;
          final int childCount = getChildCount();
          measureChildren(widthMeasureSpec, heightMeasureSpec);

          int widthSpaceSize = MeasureSpec.getSize(widthMeasureSpec);
          int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
          int heightSpaceSize = MeasureSpec.getSize(heightMeasureSpec);
          int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
          if (childCount == 0) {
              setMeasuredDimension(0, 0);
          } else if (heightSpecMode == MeasureSpec.AT_MOST) {
              final View childView = getChildAt(0);
              measureHeight= childView.getMeasuredHeight();
              setMeasuredDimension(widthSpaceSize, childView.getMeasuredHeight());
          } else if (widthSpecMode == MeasureSpec.AT_MOST) {
              final View childView = getChildAt(0);
              measureWidth = childView.getMeasuredWidth() * childCount;
              setMeasuredDimension(measureWidth, heightSpaceSize);
          } else {
              final View childView = getChildAt(0);
              measureWidth = childView.getMeasuredWidth() * childCount;
              measureHeight = childView.getMeasuredHeight();
              setMeasuredDimension(measureWidth, measureHeight);
          }

      }

      @Override
      protected void onLayout(boolean changed, int l, int t, int r, int b) {
          int childLeft = 0;
          final int childCount = getChildCount();
          mChildrenSize = childCount;

          for (int i = 0; i < childCount; i++) {
              final View childView = getChildAt(i);
              if (childView.getVisibility() != View.GONE) {
                  final int childWidth = childView.getMeasuredWidth();
                  mChildWidth = childWidth;
                  childView.layout(childLeft, 0, childLeft + childWidth, childView.getMeasuredHeight());
                  childLeft += childWidth;
              }
          }
      }
  }
  ```

  

- 案例一：内部拦截法

  我们只需要修改ListView的dispatchTouchEvent方法中的父容器的拦截逻辑，同时让父容器拦截ACTION_MOVE和ACTION_UP事件即可。为了重写ListView的dispatchTouchEvent方法，我们必须自定义一个ListView，称为ListViewEx，然后对内部拦截法的模板代码进行修改。

  ListViewEx的实现如下所示：

  ```java
  public class ListViewEx extends ListView {
      private static final String TAG = "ListViewEx";

      private HorizontalScrollViewEx2 mHorizontalScrollViewEx2;

      // 分别记录上次滑动的坐标
      private int mLastX = 0;
      private int mLastY = 0;

      public ListViewEx(Context context) {
          super(context);
      }

      public ListViewEx(Context context, AttributeSet attrs) {
          super(context, attrs);
      }

      public ListViewEx(Context context, AttributeSet attrs, int defStyleAttr) {
          super(context, attrs, defStyleAttr);
      }

      public void setHorizontalScrollViewEx2(HorizontalScrollViewEx2 horizontalScrollViewEx2) {
          this.mHorizontalScrollViewEx2 = horizontalScrollViewEx2;
      }

      @Override
      public boolean dispatchTouchEvent(MotionEvent ev) {
          int x = (int) ev.getX();
          int y = (int) ev.getY();

          switch (ev.getAction()) {
              case MotionEvent.ACTION_DOWN:
                  mHorizontalScrollViewEx2.requestDisallowInterceptTouchEvent(true);
                  break;
              case MotionEvent.ACTION_MOVE:
                  int deltaX = x - mLastX;
                  int deltaY = y - mLastY;
                  Log.d(TAG, "dx:" + deltaX + " dy:" + deltaY);
                  if (Math.abs(deltaX) > Math.abs(deltaY)) {
                      mHorizontalScrollViewEx2.requestDisallowInterceptTouchEvent(false);
                  }
                  break;
              case MotionEvent.ACTION_UP:
                  break;
              default:
                  break;
          }

          mLastX = x;
          mLastY = y;
          return super.dispatchTouchEvent(ev);
      }
  }
  ```

  ​

  除了上面对ListView所做的修改，我们还需要修改HorizontalScrollViewEx的onInterceptTouchEvent方法，修改如下：

  ```java
  @Override
  public boolean onInterceptTouchEvent(MotionEvent ev) {
      int x = (int) ev.getX();
      int y = (int) ev.getY();

     if (ev.getAction() == MotionEvent.ACTION_DOWN) {
         mLastX = x;
         mLastY = y;
         if (!mScroller.isFinished()) {
             mScroller.abortAnimation();
             return true;
         }
         return false;
     } else {
         return true;
     }
  }
  ```

  ​

  ​	前面说过，只要我们根据场景1的情况来得出通用的解决方案，那么对于场景2和场景3来说我们只需要修改相关滑动规则的逻辑即可，下面我们就来演示如何利用场景1得出的通用的解决方案来解决更复杂的滑动冲突。这里只详细分析场景2中的滑动冲突，对于场景3中的叠加型滑动冲突，由于它可以拆解为单一的滑动冲突，所以其滑动冲突的解决思想和场景1、场景2中的单一滑动冲突的解决思想一致，只需要分别解决每层之间的滑动冲突即可，再加上本书的篇幅有限，这里就不对场景3进行详细分析了。

  ​	对于场景2来说，它的解决方法和场景1一样，只是滑动规则不同而已，在前面我们已经得出了通用的解决方案，因此这里我们只需要替换父容器的拦截规则即可。注意，这里不再演示如何通过内部拦截法来解决场景2中的滑动冲突，因为内部拦截法没有外部拦截法简单易用，所以推荐采用外部拦截法来解决常见的滑动冲突。

  ​	下面通过一个实际的例子来分析场景2，首先我们提供一个可以上下滑动的父容器，这里就叫StickyLayout，它看起来就像是可以上下滑动的竖直的LinearLayout，然后在它的内部分别放一个Header和一个ListView，这样内外两层都能上下滑动，于是就形成了场景2中的滑动冲突了。当然这个StickyLayout是有滑动规则的：当Header显示时或者ListView滑动到顶部时，由StickyLayout拦截事件；当Header隐藏时，这要分情况，如果ListView已经滑动到顶部并且当前手势是向下滑动的话，这个时候还是StickyLayout拦截事件，其他情况则由ListView拦截事件。这种滑动规则看起来有点复杂，为了解决它们之间的滑动冲突，我们还是需要重写父容器StickyLayout的onInterceptTouchEvent方法，至于ListView则不用做任何修改，我们来看一下StickyLayout的具体实现，滑动冲突相关的主要代码如下所示。


```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    int intercepted = 0;
    int x = (int) ev.getX();
    int y = (int) ev.getY();

    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            mLastXIntercept = x;
            mLastYIntercept = y;
            mLastX = x;
            mLastY = y;
            intercepted = 0;
            break;
        case MotionEvent.ACTION_MOVE:
            int deltaX = x - mLastXIntercept;
            int deltaY = y - mLastYIntercept;
            if (mStatus == STATUS_EXPANDED && deltaY <= -mTouchSlop) {
                intercepted = 1;
            } else if (mGiveUpTouchEventListener != null) {
                if (mGiveUpTouchEventListener.giveUpTouchEvent(ev) && deltaY >= mTouchSlop) {
                    intercepted = 1;
                }
            }
            break;
        case MotionEvent.ACTION_UP:
            intercepted = 0;
            mLastXIntercept = mLastYIntercept = 0;
            break;
        default:
            break;
    }

    Log.d(TAG, "intercepted=" + intercepted);
    return intercepted != 0;
}

@Override
public boolean onTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    Log.d(TAG, "x=" + x + "  y=" + y + "  mlastY=" + mLastY);
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            break;
        case MotionEvent.ACTION_MOVE:
            int deltaX = x - mLastX;
            int deltaY = y - mLastY;
            Log.d(TAG, "mHeaderHeight=" + mHeaderHeight + "  deltaY=" + deltaY + "  mlastY=" + mLastY);
            mHeaderHeight += deltaY;
            setHeaderHeight(mHeaderHeight);
            break;
        case MotionEvent.ACTION_UP:
            // 这里做了下判断，当松开手的时候，会自动向两边滑动，具体向哪边滑，要看当前所处的位置
            int destHeight = 0;
            if (mHeaderHeight <= mOriginalHeaderHeight * 0.5) {
                destHeight = 0;
                mStatus = STATUS_COLLAPSED;
            } else {
                destHeight = mOriginalHeaderHeight;
                mStatus = STATUS_EXPANDED;
            }
            // 慢慢滑向终点
            this.smoothSetHeaderHeight(mHeaderHeight, destHeight, 500);
            break;
        default:
            break;
    }
    mLastX = x;
    mLastY = y;
    return true;
}
```



从上面的代码来看，这个StickyLayout的实现有点复杂，在第4章会详细介绍这个自定义View的实现思想，这里先有大概的印象即可。下面我们主要看它的onIntercept-TouchEvent方法中对ACTION_MOVE的处理，如下所示。

```java
case MotionEvent.ACTION_MOVE: {
    int deltaX = x -mLastXIntercept;
    int deltaY = y -mLastYIntercept;
    if (mDisallowInterceptTouchEventOnHeader && y <= getHeaderHeight()) {
        intercepted = 0;
    } else if (Math.abs(deltaY) <= Math.abs(deltaX)) {
        intercepted = 0;
    } else if (mStatus == STATUS_EXPANDED && deltaY <= -mTouchSlop) {
        intercepted = 1;
    } else if (mGiveUpTouchEventListener != null) {
        if (mGiveUpTouchEventListener.giveUpTouchEvent(event) && deltaY => mTouchSlop) {
            intercepted = 1;
        }
    }
    break;
}
```



我们来分析上面这段代码的逻辑，这里的父容器是StickyLayout，子元素是ListView。首先，当事件落在Header上面时父容器不会拦截事件；接着，如果竖直距离差小于水平距离差，那么父容器也不会拦截事件；然后，当Header是展开状态并且向上滑动时父容器拦截事件。另外一种情况，当ListView滑动到顶部了并且向下滑动时，父容器也会拦截事件，经过这些层层判断就可以达到我们想要的效果了。另外，giveUpTouchEvent是一个接口方法，由外部实现，在本例中主要是用来判断ListView是否滑动到顶部，它的具体实现如下：

```java
public boolean giveUpTouchEvent(MotionEvent event) {
    if (expandableListView.getFirstVisiblePosition() == 0) {
        View view = expandableListView.getChildAt(0);
        if (view != null && view.getTop() => 0) {
            return true;
        }
    }
    return false;
}
```



上面这个例子比较复杂，需要读者多多体会其中的写法和思想。到这里滑动冲突的解决方法就介绍完毕了，至于场景3中的滑动冲突，利用本节所给出的通用的方法是可以轻松解决的，读者可以自己练习一下。在第4章会介绍View的底层工作原理，并且会介绍如何写出一个好的自定第3章　View的事件体系

本章将介绍Android中十分重要的一个概念：View，虽然说View不属于四大组件，但是它的作用堪比四大组件，甚至比Receiver和Provider的重要性都要大。在Android开发中，Activity承担这可视化的功能，同时Android系统提供了很多基础控件，常见的有Button、TextView、CheckBox等。很多时候仅仅使用系统提供的控件是不能满足需求的，因此我们就需要能够根据需求进行新控件的定义，而控件的自定义就需要对Android的View体系有深入的理解，只有这样才能写出完美的自定义控件。同时Android手机属于移动设备，移动设备的一个特点就是用户可以直接通过屏幕来进行一系列操作，一个典型的场景就是屏幕的滑动，用户可以通过滑动来切换到不同的界面。很多情况下我们的应用都需要支持滑动操作，当处于不同层级的View都可以响应用户的滑动操作时，就会带来一个问题，那就是滑动冲突。如何解决滑动冲突呢？这对于初学者来说的确是个头疼的问题，其实解决滑动冲突本不难，它需要读者对View的事件分发机制有一定的了解，在这个基础上，我们就可以利于这个特性从而得出滑动冲突的解决方法。上述这些内容就是本章所要介绍的内容，同时，View的内部工作原理和自定义View相关的知识会在第4章进行介绍。

## 3.1　View基础知识

本节主要介绍View的一些基础知识，从而为更好地介绍后续的内容做铺垫。主要介绍的内容有：View的位置参数、MotionEvent和TouchSlop对象、VelocityTracker、GestureDetector和Scroller对象，通过对这些基础知识的介绍，可以方便读者理解更复杂的内容。类似的基础概念还有不少，但是本节所介绍的都是一些比较常用的，其他不常用的基础概念读者可以自行了解。

### 3.1.1　什么是View

在介绍View的基础知识之前，我们首先要知道到底什么是View。View是Android中所有控件的基类，不管是简单的Button和TextView还是复杂的RelativeLayout和ListView，它们的共同基类都是View。所以说，View是一种界面层的控件的一种抽象，它代表了一个控件。除了View，还有ViewGroup，从名字来看，它可以被翻译为控件组，言外之意是ViewGroup内部包含了许多个控件，即一组View。在Android的设计中，ViewGroup也继承了View，这就意味着View本身就可以是单个控件也可以是由多个控件组成的一组控件，通过这种关系就形成了View树的结构，这和Web前端中的DOM树的概念是相似的。根据这个概念，我们知道，Button显然是个View，而LinearLayout不但是一个View而且还是一个ViewGroup，而ViewGroup内部是可以有子View的，这个子View同样还可以是ViewGroup，依此类推。

明白View的这种层级关系有助于理解View的工作机制。如图3-1所示，可以看到自定义的TestButton是一个View，它继承了TextView，而TextView则直接继承了View，因此不管怎么说，TestButton都是一个View，同理我们也可以构造出一个继承自ViewGroup的控件。

图3-1　TestButton的层次结构

### 3.1.2　View的位置参数

View的位置主要由它的四个顶点来决定，分别对应于View的四个属性：top、left、right、bottom，其中top是左上角纵坐标，left是左上角横坐标，right是右下角横坐标，bottom是右下角纵坐标。需要注意的是，这些坐标都是相对于View的父容器来说的，因此它是一种相对坐标，View的坐标和父容器的关系如图3-2所示。在Android中，x轴和y轴的正方向分别为右和下，这点不难理解，不仅仅是Android，大部分显示系统都是按照这个标准来定义坐标系的。

图3-2　View的位置坐标和父容器的关系

根据图3-2，我们很容易得出View的宽高和坐标的关系：

```
    width = right - left
    height = bottom - top
```

那么如何得到View的这四个参数呢？也很简单，在View的源码中它们对应于mLeft、mRight、mTop和mBottom这四个成员变量，获取方式如下所示。

- Left=getLeft()；
- Right=getRight()；
- Top=getTop；
- Bottom=getBottom()。

从Android3.0开始，View增加了额外的几个参数：x、y、translationX和translationY，其中x和y是View左上角的坐标，而translationX和translationY是View左上角相对于父容器的偏移量。这几个参数也是相对于父容器的坐标，并且translationX和translationY的默认值是0，和View的四个基本的位置参数一样，View也为它们提供了get/set方法，这几个参数的换算关系如下所示。

x=left+translationX

y=top+translationY

需要注意的是，View在平移的过程中，top和left表示的是原始左上角的位置信息，其值并不会发生改变，此时发生改变的是x、y、translationX和translationY这四个参数。

### 3.1.3　MotionEvent和TouchSlop

\1. MotionEvent

在手指接触屏幕后所产生的一系列事件中，典型的事件类型有如下几种：

- ACTION_DOWN——手指刚接触屏幕；
- ACTION_MOVE——手指在屏幕上移动；
- ACTION_UP——手机从屏幕上松开的一瞬间。

正常情况下，一次手指触摸屏幕的行为会触发一系列点击事件，考虑如下几种情况：

- 点击屏幕后离开松开，事件序列为DOWN -> UP；
- 点击屏幕滑动一会再松开，事件序列为DOWN -> MOVE -> … > MOVE -> UP。

上述三种情况是典型的事件序列，同时通过MotionEvent对象我们可以得到点击事件发生的x和y坐标。为此，系统提供了两组方法：getX/getY和getRawX/getRawY。它们的区别其实很简单，getX/getY返回的是相对于当前View左上角的x和y坐标，而getRawX/getRawY返回的是相对于手机屏幕左上角的x和y坐标。

\2. TouchSlop

TouchSlop是系统所能识别出的被认为是滑动的最小距离，换句话说，当手指在屏幕上滑动时，如果两次滑动之间的距离小于这个常量，那么系统就不认为你是在进行滑动操作。原因很简单：滑动的距离太短，系统不认为它是滑动。这是一个常量，和设备有关，在不同设备上这个值可能是不同的，通过如下方式即可获取这个常量：ViewConfiguration. get(getContext()).getScaledTouchSlop()。这个常量有什么意义呢？当我们在处理滑动时，可以利用这个常量来做一些过滤，比如当两次滑动事件的滑动距离小于这个值，我们就可以认为未达到滑动距离的临界值，因此就可以认为它们不是滑动，这样做可以有更好的用户体验。其实如果细心的话，可以在源码中找到这个常量的定义，在frameworks/base/core/res/res/values/config.xml文件中，如下所示。这个“config_viewConfigurationTouchSlop”对应的就是这个常量的定义。

```
    <!--Base "touch slop" value used by ViewConfiguration as a movement threshold
    where scrolling should begin. -->
    <dimen name="config_viewConfigurationTouchSlop">8dp</dimen>
```

### 3.1.4　VelocityTracker、GestureDetector和Scroller

\1. VelocityTracker

速度追踪，用于追踪手指在滑动过程中的速度，包括水平和竖直方向的速度。它的使用过程很简单，首先，在View的onTouchEvent方法中追踪当前单击事件的速度：

```
    VelocityTracker velocityTracker = VelocityTracker.obtain();
    velocityTracker.addMovement(event);
```

接着，当我们先知道当前的滑动速度时，这个时候可以采用如下方式来获得当前的速度：

```
    velocityTracker.computeCurrentVelocity(1000);
    int xVelocity = (int) velocityTracker.getXVelocity();
    int yVelocity = (int) velocityTracker.getYVelocity();
```

在这一步中有两点需要注意，第一点，获取速度之前必须先计算速度，即getXVelocity和getYVelocity这两个方法的前面必须要调用computeCurrentVelocity方法；第二点，这里的速度是指一段时间内手指所滑过的像素数，比如将时间间隔设为1000ms时，在1s内，手指在水平方向从左向右滑过100像素，那么水平速度就是100。注意速度可以为负数，当手指从右往左滑动时，水平方向速度即为负值，这个需要理解一下。速度的计算可以用如下公式来表示：

速度=（终点位置-起点位置）/时间段

根据上面的公式再加上Android系统的坐标系，可以知道，手指逆着坐标系的正方向滑动，所产生的速度就为负值。另外，computeCurrentVelocity这个方法的参数表示的是一个时间单元或者说时间间隔，它的单位是毫秒（ms），计算速度时得到的速度就是在这个时间间隔内手指在水平或竖直方向上所滑动的像素数。针对上面的例子，如果我们通过velocityTracker.computeCurrentVelocity(100)来获取速度，那么得到的速度就是手指在100ms内所滑过的像素数，因此水平速度就成了10像素/每100ms（这里假设滑动过程是匀速的），即水平速度为10，这点需要好好理解一下。

最后，当不需要使用它的时候，需要调用clear方法来重置并回收内存：

```
    velocityTracker.clear();
    velocityTracker.recycle();
```

上面就是如何使用VelocityTracker对象的全过程，看起来并不复杂。

\2. GestureDetector

手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。要使用GestureDetector也不复杂，参考如下过程。

首先，需要创建一个GestureDetector对象并实现OnGestureListener接口，根据需要我们还可以实现OnDoubleTapListener从而能够监听双击行为：

```
    GestureDetector  mGestureDetector = new GestureDetector(this);
    //解决长按屏幕后无法拖动的现象
    mGestureDetector.setIsLongpressEnabled(false);
```

接着，接管目标View的onTouchEvent方法，在待监听View的onTouchEvent方法中添加如下实现：

```
    boolean consume = mGestureDetector.onTouchEvent(event);
    return consume;
```

做完了上面两步，我们就可以有选择地实现OnGestureListener和OnDoubleTapListener中的方法了，这两个接口中的方法介绍如表3-1所示。

表3-1　OnGestureListener和OnDoubleTapListener中的方法介绍

表3-1里面的方法很多，但是并不是所有的方法都会被时常用到，在日常开发中，比较常用的有：onSingleTapUp（单击）、onFling（快速滑动）、onScroll（拖动）、onLongPress（长按）和onDoubleTap（双击）。另外这里要说明的是，实际开发中，可以不使用GestureDetector，完全可以自己在View的onTouchEvent方法中实现所需的监听，这个就看个人的喜好了。这里有一个建议供读者参考：如果只是监听滑动相关的，建议自己在onTouchEvent中实现，如果要监听双击这种行为的话，那么就使用GestureDetector。

\3. Scroller

弹性滑动对象，用于实现View的弹性滑动。我们知道，当使用View的scrollTo/scrollBy方法来进行滑动时，其过程是瞬间完成的，这个没有过渡效果的滑动用户体验不好。这个时候就可以使用Scroller来实现有过渡效果的滑动，其过程不是瞬间完成的，而是在一定的时间间隔内完成的。Scroller本身无法让View弹性滑动，它需要和View的computeScroll方法配合使用才能共同完成这个功能。那么如何使用Scroller呢？它的典型代码是固定的，如下所示。至于它为什么能实现弹性滑动，这个在3.2节中会进行详细介绍。

```
    Scroller scroller = new Scroller(mContext);
    // 缓慢滚动到指定位置
    private void smoothScrollTo(int destX,int destY) {
        int scrollX = getScrollX();
        int delta = destX -scrollX;
        // 1000ms内滑向destX，效果就是慢慢滑动
        mScroller.startScroll(scrollX,0,delta,0,1000);
        invalidate();
    }
    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
                scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
                postInvalidate();
        }
    }
```

## 3.2　View的滑动

3.1节介绍了View的一些基础知识和概念，本节开始介绍很重要的一个内容：View的滑动。在Android设备上，滑动几乎是应用的标配，不管是下拉刷新还是SlidingMenu，它们的基础都是滑动。从另外一方面来说，Android手机由于屏幕比较小，为了给用户呈现更多的内容，就需要使用滑动来隐藏和显示一些内容。基于上述两点，可以知道，滑动在Android开发中具有很重要的作用，不管一些滑动效果多么绚丽，归根结底，它们都是由不同的滑动外加一些特效所组成的。因此，掌握滑动的方法是实现绚丽的自定义控件的基础。通过三种方式可以实现View的滑动：第一种是通过View本身提供的scrollTo/scrollBy方法来实现滑动；第二种是通过动画给View施加平移效果来实现滑动；第三种是通过改变View的LayoutParams使得View重新布局从而实现滑动。从目前来看，常见的滑动方式就这么三种，下面一一进行分析。

### 3.2.1　使用scrollTo/scrollBy

为了实现View的滑动，View提供了专门的方法来实现这个功能，那就是scrollTo和scrollBy，我们先来看看这两个方法的实现，如下所示。

```
    /**
      * Set the scrolled position of your view. This will cause a call to
      * {@link #onScrollChanged(int,int,int,int)} and the view will be
      * invalidated.
      * @param x the x position to scroll to
      * @param y the y position to scroll to
      */
    public void scrollTo(int x,int y) {
        if (mScrollX != x || mScrollY != y) {
                int oldX = mScrollX;
                int oldY = mScrollY;
                mScrollX = x;
                mScrollY = y;
                invalidateParentCaches();
                onScrollChanged(mScrollX,mScrollY,oldX,oldY);
                if (!awakenScrollBars()) {
                        postInvalidateOnAnimation();
                }
        }
    }
     /**
      * Move the scrolled position of your view. This will cause a call to
      * {@link #onScrollChanged(int,int,int,int)} and the view will be
      * invalidated.
      * @param x the amount of pixels to scroll by horizontally
      * @param y the amount of pixels to scroll by vertically
      */
    public void scrollBy(int x,int y) {
        scrollTo(mScrollX + x,mScrollY + y);
    }
```

从上面的源码可以看出，scrollBy实际上也是调用了scrollTo方法，它实现了基于当前位置的相对滑动，而scrollTo则实现了基于所传递参数的绝对滑动，这个不难理解。利用scrollTo和scrollBy来实现View的滑动，这不是一件困难的事，但是我们要明白滑动过程中View内部的两个属性mScrollX和mScrollY的改变规则，这两个属性可以通过getScrollX和getScrollY方法分别得到。这里先简要概况一下：在滑动过程中，mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离，而mScrollY的值总是等于View上边缘和View内容上边缘在竖直方向的距离。View边缘是指View的位置，由四个顶点组成，而View内容边缘是指View中的内容的边缘，scrollTo和scrollBy只能改变View内容的位置而不能改变View在布局中的位置。mScrollX和mScrollY的单位为像素，并且当View左边缘在View内容左边缘的右边时，mScrollX为正值，反之为负值；当View上边缘在View内容上边缘的下边时，mScrollY为正值，反之为负值。换句话说，如果从左向右滑动，那么mScrollX为负值，反之为正值；如果从上往下滑动，那么mScrollY为负值，反之为正值。

为了更好地理解这个问题，下面举个例子，如图3-3所示。在图中假设水平和竖直方向的滑动距离都为100像素，针对图中各种滑动情况，都给出了对应的mScrollX和mScrollY的值。根据上面的分析，可以知道，使用scrollTo和scrollBy来实现View的滑动，只能将View的内容进行移动，并不能将View本身进行移动，也就是说，不管怎么滑动，也不可能将当前View滑动到附近View所在的区域，这个需要仔细体会一下。

图3-3　mScrollX和mScrollY的变换规律示意

### 3.2.2　使用动画

上一节介绍了采用scrollTo/scrollBy来实现View的滑动，本节介绍另外一种滑动方式，即使用动画，通过动画我们能够让一个View进行平移，而平移就是一种滑动。使用动画来移动View，主要是操作View的translationX和translationY属性，既可以采用传统的View动画，也可以采用属性动画，如果采用属性动画的话，为了能够兼容3.0以下的版本，需要采用开源动画库nineoldandroids（http://nineoldandroids.com/）。

采用View动画的代码，如下所示。此动画可以在100ms内将一个View从原始位置向右下角移动100个像素。

```
    <?xml version="1.0" encoding="utf-8"?>
    <set xmlns:android="http://schemas.android.com/apk/res/android"
         android:fillAfter="true"
         android:zAdjustment="normal" >
         <translate
             android:duration="100"
             android:fromXDelta="0"
             android:fromYDelta="0"
             android:interpolator="@android:anim/linear_interpolator"
             android:toXDelta="100"
             android:toYDelta="100" />
    </set>
```

如果采用属性动画的话，就更简单了，以下代码可以将一个View在100ms内从原始位置向右平移100像素。

```
    ObjectAnimator.ofFloat(targetView,"translationX",0,100).setDuration
     (100).start();
```

上面简单介绍了通过动画来移动View的方法，关于动画会在第5章中进行详细说明。使用动画来做View的滑动需要注意一点，View动画是对View的影像做操作，它并不能真正改变View的位置参数，包括宽/高，并且如果希望动画后的状态得以保留还必须将fillAfter属性设置为true，否则动画完成后其动画结果会消失。比如我们要把View向右移动100像素，如果fillAfter为false，那么在动画完成的一刹那，View会瞬间恢复到动画前的状态；如果fillAfter为true，在动画完成后，View会停留在距原始位置100像素的右边。使用属性动画并不会存在上述问题，但是在Android 3.0以下无法使用属性动画，这个时候我们可以使用动画兼容库nineoldandroids来实现属性动画，尽管如此，在Android 3.0以下的手机上通过nineoldandroids来实现的属性动画本质上仍然是View动画。

上面提到View动画并不能真正改变View的位置，这会带来一个很严重的问题。试想一下，比如我们通过View动画将一个Button向右移动100px，并且这个View设置的有单击事件，然后你会惊奇地发现，单击新位置无法触发onClick事件，而单击原始位置仍然可以触发onClick事件，尽管Button已经不在原始位置了。这个问题带来的影响是致命的，但是它却又是可以理解的，因为不管Button怎么做变换，但是它的位置信息（四个顶点和宽/高）并不会随着动画而改变，因此在系统眼里，这个Button并没有发生任何改变，它的真身仍然在原始位置。在这种情况下，单击新位置当然不会触发onClick事件了，因为Button的真身并没有发生改变，在新位置上只是View的影像而已。基于这一点，我们不能简单地给一个View做平移动画并且还希望它在新位置继续触发一些单击事件。

从Android 3.0开始，使用属性动画可以解决上面的问题，但是大多数应用都需要兼容到Android 2.2，在Android 2.2上无法使用属性动画，因此这里还是会有问题。那么这种问题难道就无法解决了吗？也不是的，虽然不能直接解决这个问题，但是还可以间接解决这个问题，这里给出一个简单的解决方法。针对上面View动画的问题，我们可以在新位置预先创建一个和目标Button一模一样的Button，它们不但外观一样连onClick事件也一样。当目标Button完成平移动画后，就把目标Button隐藏，同时把预先创建的Button显示出来，通过这种间接的方式我们解决了上面的问题。这仅仅是个参考，面对这种问题时读者可以灵活应对。

### 3.2.3　改变布局参数

本节将介绍第三种实现View滑动的方法，那就是改变布局参数，即改变LayoutParams。这个比较好理解了，比如我们想把一个Button向右平移100px，我们只需要将这个Button的LayoutParams里的marginLeft参数的值增加100px即可，是不是很简单呢？还有一种情形，为了达到移动Button的目的，我们可以在Button的左边放置一个空的View，这个空View的默认宽度为0，当我们需要向右移动Button时，只需要重新设置空View的宽度即可，当空View的宽度增大时（假设Button的父容器是水平方向的LinearLayout），Button就自动被挤向右边，即实现了向右平移的效果。如何重新设置一个View的LayoutParams呢？很简单，如下所示。

```
    MarginLayoutParams params = (MarginLayoutParams)mButton1.getLayoutParams();
    params.width += 100;
    params.leftMargin += 100;
    mButton1.requestLayout();
    //或者mButton1.setLayoutParams(params);
```

通过改变LayoutParams的方式去实现View的滑动同样是一种很灵活的方法，需要根据不同情况去做不同的处理。

### 3.2.4　各种滑动方式的对比

上面分别介绍了三种不同的滑动方式，它们都能实现View的滑动，那么它们之间的差别是什么呢？

先看scrollTo/scrollBy这种方式，它是View提供的原生方法，其作用是专门用于View的滑动，它可以比较方便地实现滑动效果并且不影响内部元素的单击事件。但是它的缺点也是很显然的：它只能滑动View的内容，并不能滑动View本身。

再看动画，通过动画来实现View的滑动，这要分情况。如果是Android 3.0以上并采用属性动画，那么采用这种方式没有明显的缺点；如果是使用View动画或者在Android 3.0以下使用属性动画，均不能改变View本身的属性。在实际使用中，如果动画元素不需要响应用户的交互，那么使用动画来做滑动是比较合适的，否则就不太适合。但是动画有一个很明显的优点，那就是一些复杂的效果必须要通过动画才能实现。

最后再看一下改变布局这种方式，它除了使用起来麻烦点以外，也没有明显的缺点，它的主要适用对象是一些具有交互性的View，因为这些View需要和用户交互，直接通过动画去实现会有问题，这在3.2.2节中已经有所介绍，所以这个时候我们可以使用直接改变布局参数的方式去实现。

针对上面的分析做一下总结，如下所示。

- scrollTo/scrollBy：操作简单，适合对View内容的滑动；
- 动画：操作简单，主要适用于没有交互的View和实现复杂的动画效果；
- 改变布局参数：操作稍微复杂，适用于有交互的View。

下面我们实现一个跟手滑动的效果，这是一个自定义View，拖动它可以让它在整个屏幕上随意滑动。这个View实现起来很简单，我们只要重写它的onTouchEvent方法并处理ACTION_MOVE事件，根据两次滑动之间的距离就可以实现它的滑动了。为了实现全屏滑动，我们采用动画的方式来实现。原因很简单，这个效果无法采用scrollTo来实现。另外，它还可以采用改变布局的方式来实现，这里仅仅是为了演示，所以就选择了动画的方式，核心代码如下所示。

```
    public boolean onTouchEvent(MotionEvent event) {
        int x = (int) event.getRawX();
        int y = (int) event.getRawY();
        switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
                break;
        }
        case MotionEvent.ACTION_MOVE: {
                int deltaX = x -mLastX;
                int deltaY = y -mLastY;
                Log.d(TAG,"move,deltaX:" + deltaX + " deltaY:" + deltaY);
                int translationX = (int)ViewHelper.getTranslationX(this) + deltaX;
                int translationY = (int)ViewHelper.getTranslationY(this) + deltaY;
                ViewHelper.setTranslationX(this,translationX);
                ViewHelper.setTranslationY(this,translationY);
                break;
        }
        case MotionEvent.ACTION_UP: {
                break;
        }
        default:
                break;
        }
        mLastX = x;
        mLastY = y;
        return true;
    }
```

通过上述代码可以看出，这一全屏滑动的效果实现起来相当简单。首先我们通过getRawX和getRawY方法来获取手指当前的坐标，注意不能使用getX和getY方法，因为这个是要全屏滑动的，所以需要获取当前点击事件在屏幕中的坐标而不是相对于View本身的坐标；其次，我们要得到两次滑动之间的位移，有了这个位移就可以移动当前的View，移动方法采用的是动画兼容库nineoldandroids中的ViewHelper类所提供的setTranslationX和setTranslationY方法。实际上，ViewHelper类提供了一系列get/set方法，因为View的setTranslationX和setTranslationY只能在Android 3.0及其以上版本才能使用，但是ViewHelper所提供的方法是没有版本要求的，与此类似的还有setX、setScaleX、setAlpha等方法，这一系列方法实际上是为属性动画服务的，更详细的内容会在第5章进行进一步的介绍。这个自定义View可以在2.x及其以上版本工作，但是由于动画的性质，如果给它加上onClick事件，那么在3.0以下版本它将无法在新位置响应用户的点击，这个问题在前面已经提到过。

## 3.3　弹性滑动

知道了View的滑动，我们还要知道如何实现View的弹性滑动，比较生硬地滑动过去，这种方式的用户体验实在太差了，因此我们要实现渐近式滑动。那么如何实现弹性滑动呢？其实实现方法有很多，但是它们都有一个共同思想：将一次大的滑动分成若干次小的滑动并在一个时间段内完成，弹性滑动的具体实现方式有很多，比如通过Scroller、Handler#postDelayed以及Thread#sleep等，下面一一进行介绍。

### 3.3.1　使用Scroller

Scroller的使用方法在3.1.4节中已经进行了介绍，下面我们来分析一下它的源码，从而探究为什么它能实现View的弹性滑动。

```
    Scroller scroller = new Scroller(mContext);
    // 缓慢滚动到指定位置
    private void smoothScrollTo(int destX,int destY) {
        int scrollX = getScrollX();
        int deltaX = destX -scrollX;
        // 1000ms内滑向destX，效果就是慢慢滑动
        mScroller.startScroll(scrollX,0,deltaX,0,1000);
        invalidate();
    }
    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
                scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
                postInvalidate();
        }
    }
```

上面是Scroller的典型的使用方法，这里先描述它的工作原理：当我们构造一个Scroller对象并且调用它的startScroll方法时，Scroller内部其实什么也没做，它只是保存了我们传递的几个参数，这几个参数从startScroll的原型上就可以看出来，如下所示。

```
    public void startScroll(int startX,int startY,int dx,int dy,int duration){
        mMode = SCROLL_MODE;
        mFinished = false;
        mDuration = duration;
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
        mStartX = startX;
        mStartY = startY;
        mFinalX = startX + dx;
        mFinalY = startY + dy;
        mDeltaX = dx;
        mDeltaY = dy;
        mDurationReciprocal = 1.0f / (float) mDuration;
    }
```

这个方法的参数含义很清楚，startX和startY表示的是滑动的起点，dx和dy表示的是要滑动的距离，而duration表示的是滑动时间，即整个滑动过程完成所需要的时间，注意这里的滑动是指View内容的滑动而非View本身位置的改变。可以看到，仅仅调用startScroll方法是无法让View滑动的，因为它内部并没有做滑动相关的事，那么Scroller到底是如何让View弹性滑动的呢？答案就是startScroll方法下面的invalidate方法，虽然有点不可思议，但是的确是这样的。invalidate方法会导致View重绘，在View的draw方法中又会去调用computeScroll方法，computeScroll方法在View中是一个空实现，因此需要我们自己去实现，上面的代码已经实现了computeScroll方法。正是因为这个computeScroll方法，View才能实现弹性滑动。这看起来还是很抽象，其实这样的：当View重绘后会在draw方法中调用computeScroll，而computeScroll又会去向Scroller获取当前的scrollX和scrollY；然后通过scrollTo方法实现滑动；接着又调用postInvalidate方法来进行第二次重绘，这一次重绘的过程和第一次重绘一样，还是会导致computeScroll方法被调用；然后继续向Scroller获取当前的scrollX和scrollY，并通过scrollTo方法滑动到新的位置，如此反复，直到整个滑动过程结束。

我们再看一下Scroller的computeScrollOffset方法的实现，如下所示。

```
    /**
      * Call this when you want to know the new location. If it returns true,
      * the animation is not yet finished.
      */
    public boolean computeScrollOffset() {
        ...
        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() -
        mStartTime);
        if (timePassed < mDuration) {
                switch (mMode) {
                case SCROLL_MODE:
                        final float x = mInterpolator.getInterpolation(timePassed *
                        mDurationReciprocal);
                        mCurrX = mStartX + Math.round(x * mDeltaX);
                        mCurrY = mStartY + Math.round(x * mDeltaY);
                        break;
                ...
                }
        }
        return true;
    }
```

是不是突然就明白了？这个方法会根据时间的流逝来计算出当前的scrollX和scrollY的值。计算方法也很简单，大意就是根据时间流逝的百分比来算出scrollX和scrollY改变的百分比并计算出当前的值，这个过程类似于动画中的插值器的概念，这里我们先不去深究这个具体过程。这个方法的返回值也很重要，它返回true表示滑动还未结束，false则表示滑动已经结束，因此当这个方法返回true时，我们要继续进行View的滑动。

通过上面的分析，我们应该明白Scroller的工作原理了，这里做一下概括：Scroller本身并不能实现View的滑动，它需要配合View的computeScroll方法才能完成弹性滑动的效果，它不断地让View重绘，而每一次重绘距滑动起始时间会有一个时间间隔，通过这个时间间隔Scroller就可以得出View当前的滑动位置，知道了滑动位置就可以通过scrollTo方法来完成View的滑动。就这样，View的每一次重绘都会导致View进行小幅度的滑动，而多次的小幅度滑动就组成了弹性滑动，这就是Scroller的工作机制。由此可见，Scroller的设计思想是多么值得称赞，整个过程中它对View没有丝毫的引用，甚至在它内部连计时器都没有。

### 3.3.2　通过动画

动画本身就是一种渐近的过程，因此通过它来实现的滑动天然就具有弹性效果，比如以下代码可以让一个View的内容在100ms内向左移动100像素。

```
    ObjectAnimator.ofFloat(targetView,"translationX",0,100).setDuration
     (100).start();
```

不过这里想说的并不是这个问题，我们可以利用动画的特性来实现一些动画不能实现的效果。还拿scrollTo来说，我们也想模仿Scroller来实现View的弹性滑动，那么利用动画的特性，我们可以采用如下方式来实现：

```
    final int startX = 0;
    final int deltaX = 100;
    ValueAnimator animator = ValueAnimator.ofInt(0,1).setDuration(1000);
    animator.addUpdateListener(new AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animator) {
                float fraction = animator.getAnimatedFraction();
                mButton1.scrollTo(startX + (int) (deltaX * fraction),0);
        }
    });
    animator.start();
```

在上述代码中，我们的动画本质上没有作用于任何对象上，它只是在1000ms内完成了整个动画过程。利用这个特性，我们就可以在动画的每一帧到来时获取动画完成的比例，然后再根据这个比例计算出当前View所要滑动的距离。注意，这里的滑动针对的是View的内容而非View本身。可以发现，这个方法的思想其实和Scroller比较类似，都是通过改变一个百分比配合scrollTo方法来完成View的滑动。需要说明一点，采用这种方法除了能够完成弹性滑动以外，还可以实现其他动画效果，我们完全可以在onAnimationUpdate方法中加上我们想要的其他操作。

### 3.3.3　使用延时策略

本节介绍另外一种实现弹性滑动的方法，那就是延时策略。它的核心思想是通过发送一系列延时消息从而达到一种渐近式的效果，具体来说可以使用Handler或View的postDelayed方法，也可以使用线程的sleep方法。对于postDelayed方法来说，我们可以通过它来延时发送一个消息，然后在消息中来进行View的滑动，如果接连不断地发送这种延时消息，那么就可以实现弹性滑动的效果。对于sleep方法来说，通过在while循环中不断地滑动View和sleep，就可以实现弹性滑动的效果。

下面采用Handler来做个示例，其他方法请读者自行去尝试，思想都是类似的。下面的代码在大约1000ms内将View的内容向左移动了100像素，代码比较简单，就不再详细介绍了。之所以说大约1000ms，是因为采用这种方式无法精确地定时，原因是系统的消息调度也是需要时间的，并且所需时间不定。

```
    private static final int MESSAGE_SCROLL_TO = 1;
    private static final int FRAME_COUNT = 30;
    private static final int DELAYED_TIME = 33;
    private int mCount = 0;
    @SuppressLint("HandlerLeak")
    private Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
                switch (msg.what) {
                case MESSAGE_SCROLL_TO: {
                        mCount++;
                        if (mCount <= FRAME_COUNT) {
                                float fraction = mCount / (float) FRAME_COUNT;
                                int scrollX = (int) (fraction * 100);
                                mButton1.scrollTo(scrollX,0);
                                mHandler.sendEmptyMessageDelayed(MESSAGE_SCROLL_TO,
                                DELAYED_TIME);
                        }
                        break;
                }
                default:
                        break;
                }
        };
    };
```

上面几种弹性滑动的实现方法，在介绍中侧重更多的是实现思想，在实际使用中可以对其灵活地进行扩展从而实现更多复杂的效果。

## 3.4　View的事件分发机制

上面几节介绍了View的基础知识以及View的滑动，本节将介绍View的一个核心知识点：事件分发机制。事件分发机制不仅仅是核心知识点更是难点，不少初学者甚至中级开发者面对这个问题时都会觉得困惑。另外，View的另一大难题滑动冲突，它的解决方法的理论基础就是事件分发机制，因此掌握好View的事件分发机制是十分重要的。本节将深入介绍View的事件分发机制，在3.4.1节会对事件分发机制进行概括性地介绍，而在3.4.2节将结合系统源码去进一步分析事件分发机制。

### 3.4.1　点击事件的传递规则

在介绍点击事件的传递规则之前，首先我们要明白这里要分析的对象就是MotionEvent，即点击事件，关于MotionEvent在3.1节中已经进行了介绍。所谓点击事件的事件分发，其实就是对MotionEvent事件的分发过程，即当一个MotionEvent产生了以后，系统需要把这个事件传递给一个具体的View，而这个传递的过程就是分发过程。点击事件的分发过程由三个很重要的方法来共同完成：dispatchTouchEvent、onInterceptTouchEvent和onTouchEvent，下面我们先介绍一下这几个方法。

public boolean dispatchTouchEvent(MotionEvent ev)

用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。

public boolean onInterceptTouchEvent(MotionEvent event)

在上述方法内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

public boolean onTouchEvent(MotionEvent event)

在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。

上述三个方法到底有什么区别呢？它们是什么关系呢？其实它们的关系可以用如下伪代码表示：

```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        boolean consume = false;
        if (onInterceptTouchEvent(ev)) {
                consume = onTouchEvent(ev);
        } else {
                consume = child.dispatchTouchEvent(ev);
        }
        return consume;
    }
```

上述伪代码已经将三者的关系表现得淋漓尽致。通过上面的伪代码，我们也可以大致了解点击事件的传递规则：对于一个根ViewGroup来说，点击事件产生后，首先会传递给它，这时它的dispatchTouchEvent就会被调用，如果这个ViewGroup的onInterceptTouchEvent方法返回true就表示它要拦截当前事件，接着事件就会交给这个ViewGroup处理，即它的onTouchEvent方法就会被调用；如果这个ViewGroup的onInterceptTouchEvent方法返回false就表示它不拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素的dispatchTouchEvent方法就会被调用，如此反复直到事件被最终处理。

当一个View需要处理事件时，如果它设置了OnTouchListener，那么OnTouchListener中的onTouch方法会被回调。这时事件如何处理还要看onTouch的返回值，如果返回false，则当前View的onTouchEvent方法会被调用；如果返回true，那么onTouchEvent方法将不会被调用。由此可见，给View设置的OnTouchListener，其优先级比onTouchEvent要高。在onTouchEvent方法中，如果当前设置的有OnClickListener，那么它的onClick方法会被调用。可以看出，平时我们常用的OnClickListener，其优先级最低，即处于事件传递的尾端。

当一个点击事件产生后，它的传递过程遵循如下顺序：Activity -> Window -> View，即事件总是先传递给Activity，Activity再传递给Window，最后Window再传递给顶级View。顶级View接收到事件后，就会按照事件分发机制去分发事件。考虑一种情况，如果一个View的onTouchEvent返回false，那么它的父容器的onTouchEvent将会被调用，依此类推。如果所有的元素都不处理这个事件，那么这个事件将会最终传递给Activity处理，即Activity的onTouchEvent方法会被调用。这个过程其实也很好理解，我们可以换一种思路，假如点击事件是一个难题，这个难题最终被上级领导分给了一个程序员去处理（这是事件分发过程），结果这个程序员搞不定（onTouchEvent返回了false），现在该怎么办呢？难题必须要解决，那只能交给水平更高的上级解决（上级的onTouchEvent被调用），如果上级再搞不定，那只能交给上级的上级去解决，就这样将难题一层层地向上抛，这是公司内部一种很常见的处理问题的过程。从这个角度来看，View的事件传递过程还是很贴近现实的，毕竟程序员也生活在现实中。

关于事件传递的机制，这里给出一些结论，根据这些结论可以更好地理解整个传递机制，如下所示。

（1）同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。

（2）正常情况下，一个事件序列只能被一个View拦截且消耗。这一条的原因可以参考（3），因为一旦一个元素拦截了某此事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别由两个View同时处理，但是通过特殊手段可以做到，比如一个View将本该自己处理的事件通过onTouchEvent强行传递给其他View处理。

（3）某个View一旦决定拦截，那么这一个事件序列都只能由它来处理（如果事件序列能够传递给它的话），并且它的onInterceptTouchEvent不会再被调用。这条也很好理解，就是说当一个View决定拦截一个事件后，那么系统会把同一个事件序列内的其他方法都直接交给它来处理，因此就不用再调用这个View的onInterceptTouchEvent去询问它是否要拦截了。

（4）某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件（onTouchEvent返回了false），那么同一事件序列中的其他事件都不会再交给它来处理，并且事件将重新交由它的父元素去处理，即父元素的onTouchEvent会被调用。意思就是事件一旦交给一个View处理，那么它就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理了，这就好比上级交给程序员一件事，如果这件事没有处理好，短期内上级就不敢再把事情交给这个程序员做了，二者是类似的道理。

（5）如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。

（6）ViewGroup默认不拦截任何事件。Android源码中ViewGroup的onInterceptTouch-Event方法默认返回false。

（7）View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。

（8）View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击的（clickable 和longClickable同时为false）。View的longClickable属性默认都为false，clickable属性要分情况，比如Button的clickable属性默认为true，而TextView的clickable属性默认为false。

（9）View的enable属性不影响onTouchEvent的默认返回值。哪怕一个View是disable状态的，只要它的clickable或者longClickable有一个为true，那么它的onTouchEvent就返回true。

（10）onClick会发生的前提是当前View是可点击的，并且它收到了down和up的事件。

（11）事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外。

### 3.4.2　事件分发的源码解析

上一节分析了View的事件分发机制，本节将会从源码的角度去进一步分析、证实上面的结论。

\1. Activity对点击事件的分发过程

点击事件用MotionEvent来表示，当一个点击操作发生时，事件最先传递给当前Activity，由Activity的dispatchTouchEvent来进行事件派发，具体的工作是由Activity内部的Window来完成的。Window会将事件传递给decor view，decor view一般就是当前界面的底层容器（即setContentView所设置的View的父容器），通过Activity.getWindow.getDecorView()可以获得。我们先从Activity的dispatchTouchEvent开始分析。

源码：Activity#dispatchTouchEvent

```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

现在分析上面的代码。首先事件开始交给Activity所附属的Window进行分发，如果返回true，整个事件循环就结束了，返回false意味着事件没人处理，所有View的onTouchEvent都返回了false，那么Activity的onTouchEvent就会被调用。

接下来看Window是如何将事件传递给ViewGroup的。通过源码我们知道，Window是个抽象类，而Window的superDispatchTouchEvent方法也是个抽象方法，因此我们必须找到Window的实现类才行。

源码：Window#superDispatchTouchEvent

```
    public abstract boolean superDispatchTouchEvent(MotionEvent event);
```

那么到底Window的实现类是什么呢？其实是PhoneWindow，这一点从Window的源码中也可以看出来，在Window的说明中，有这么一段话：

Abstract base class for a top-level window look and behavior policy. An instance of this class should be used as the top-level view added to the window manager. It provides standard UI policies such as a background, title area, default key processing, etc.

The only existing implementation of this abstract class is android. policy. PhoneWindow,which you should instantiate when needing a Window. Eventually that class will be refactored and a factory method added for creating Window instances without knowing about a particular implementation.

上面这段话的大概意思是：Window类可以控制顶级View的外观和行为策略，它的唯一实现位于android.policy.PhoneWindow中，当你要实例化这个Window类的时候，你并不知道它的细节，因为这个类会被重构，只有一个工厂方法可以使用。尽管这看起来有点模糊，不过我们可以看一下android.policy.PhoneWindow这个类，尽管实例化的时候此类会被重构，仅是重构而已，功能是类似的。

由于Window的唯一实现是PhoneWindow，因此接下来看一下PhoneWindow是如何处理点击事件的，如下所示。

源码：PhoneWindow#superDispatchTouchEvent

```
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```

到这里逻辑就很清晰了，PhoneWindow将事件直接传递给了DecorView，这个DecorView是什么呢？请看下面：

```
    private final class DecorView extends FrameLayout implements RootViewSur-
    faceTaker
    // This is the top-level view of the window,containing the window decor.
    private DecorView mDecor;
    @Override
    public final View getDecorView() {
        if (mDecor == null) {
                installDecor();
        }
        return mDecor;
    }
```

我们知道，通过((ViewGroup)getWindow().getDecorView().findViewById(android.R.id.content)).getChildAt(0)这种方式就可以获取Activity所设置的View，这个mDecor显然就是getWindow().getDecorView()返回的View，而我们通过setContentView设置的View是它的一个子View。目前事件传递到了DecorView这里，由于DecorView继承自FrameLayout且是父View，所以最终事件会传递给View。换句话来说，事件肯定会传递到View，不然应用如何响应点击事件呢？不过这不是我们的重点，重点是事件到了View以后应该如何传递，这对我们更有用。从这里开始，事件已经传递到顶级View了，即在Activity中通过setContentView所设置的View，另外顶级View也叫根View，顶级View一般来说都是ViewGroup。

\3. 顶级View对点击事件的分发过程

关于点击事件如何在View中进行分发，上一节已经做了详细的介绍，这里再大致回顾一下。点击事件达到顶级View（一般是一个ViewGroup）以后，会调用ViewGroup的dispatchTouchEvent方法，然后的逻辑是这样的：如果顶级ViewGroup拦截事件即onInterceptTouchEvent返回true，则事件由ViewGroup处理，这时如果ViewGroup的mOnTouchListener被设置，则onTouch会被调用，否则onTouchEvent会被调用。也就是说，如果都提供的话，onTouch会屏蔽掉onTouchEvent。在onTouchEvent中，如果设置了mOnClickListener，则onClick会被调用。如果顶级ViewGroup不拦截事件，则事件会传递给它所在的点击事件链上的子View，这时子View的dispatchTouchEvent会被调用。到此为止，事件已经从顶级View传递给了下一层View，接下来的传递过程和顶级View是一致的，如此循环，完成整个事件的分发。

首先看ViewGroup对点击事件的分发过程，其主要实现在ViewGroup的dispatchTouch-Event方法中，这个方法比较长，这里分段说明。先看下面一段，很显然，它描述的是当前View是否拦截点击事情这个逻辑。

```
    // Check for interception.
    final boolean intercepted;
    if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_                    INTERCEPT) != 0;
        if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
        } else {
                intercepted = false;
        }
    } else {
        // There are no touch targets and this action is not an initial down
        // so this view group continues to intercept touches.
        intercepted = true;
    }
```

从上面代码我们可以看出，ViewGroup在如下两种情况下会判断是否要拦截当前事件：事件类型为ACTION_DOWN或者mFirstTouchTarget != null。ACTION_DOWN事件好理解，那么mFirstTouchTarget != null是什么意思呢？这个从后面的代码逻辑可以看出来，当事件由ViewGroup的子元素成功处理时，mFirstTouchTarget会被赋值并指向子元素，换种方式来说，当ViewGroup不拦截事件并将事件交由子元素处理时mFirstTouchTarget != null。反过来，一旦事件由当前ViewGroup拦截时，mFirstTouchTarget != null就不成立。那么当ACTION_MOVE和ACTION_UP事件到来时，由于(actionMasked == MotionEvent. ACTION_DOWN || mFirstTouchTarget != null)这个条件为false，将导致ViewGroup的onInterceptTouchEvent不会再被调用，并且同一序列中的其他事件都会默认交给它处理。

当然，这里有一种特殊情况，那就是FLAG_DISALLOW_INTERCEPT标记位，这个标记位是通过requestDisallowInterceptTouchEvent方法来设置的，一般用于子View中。FLAG_DISALLOW_INTERCEPT一旦设置后，ViewGroup将无法拦截除了ACTION_DOWN以外的其他点击事件。为什么说是除了ACTION_DOWN以外的其他事件呢？这是因为ViewGroup在分发事件时，如果是ACTION_DOWN就会重置FLAG_DISALLOW_INTERCEPT这个标记位，将导致子View中设置的这个标记位无效。因此，当面对ACTION_DOWN事件时，ViewGroup总是会调用自己的onInterceptTouchEvent方法来询问自己是否要拦截事件，这一点从源码中也可以看出来。在下面的代码中，ViewGroup会在ACTION_DOWN事件到来时做重置状态的操作，而在resetTouchState方法中会对FLAG_DISALLOW_INTERCEPT进行重置，因此子View调用request-DisallowInterceptTouchEvent方法并不能影响ViewGroup对ACTION_DOWN事件的处理。

```
    // Handle an initial down.
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // Throw away all previous state when starting a new touch gesture.
        // The framework may have dropped the up or cancel event for the previous gesture.
        // due to an app switch,ANR,or some other state change.
        cancelAndClearTouchTargets(ev);
        resetTouchState();
    }
```

从上面的源码分析，我们可以得出结论：当ViewGroup决定拦截事件后，那么后续的点击事件将会默认交给它处理并且不再调用它的onInterceptTouchEvent方法，这证实了3.4.1节末尾处的第3条结论。FLAG_DISALLOW_INTERCEPT这个标志的作用是让ViewGroup不再拦截事件，当然前提是ViewGroup不拦截ACTION_DOWN事件，这证实了3.4.1节末尾处的第11条结论。那么这段分析对我们有什么价值呢？总结起来有两点：第一点，onInterceptTouchEvent不是每次事件都会被调用的，如果我们想提前处理所有的点击事件，要选择dispatchTouchEvent方法，只有这个方法能确保每次都会调用，当然前提是事件能够传递到当前的ViewGroup；另外一点，FLAG_DISALLOW_INTERCEPT标记位的作用给我们提供了一个思路，当面对滑动冲突时，我们可以是不是考虑用这种方法去解决问题？关于滑动冲突，将在3.5节进行详细分析。

接着再看当ViewGroup不拦截事件的时候，事件会向下分发交由它的子View进行处理，这段源码如下所示。

```
    final View[] children = mChildren;
    for (int i = childrenCount -1; i => 0; i--) {
        final int childIndex = customOrder
                 ? getChildDrawingOrder(childrenCount,i) : i;
        final View child = (preorderedList == null)
                 ? children[childIndex] : preorderedList.get(childIndex);
        if (!canViewReceivePointerEvents(child)
                 || !isTransformedTouchPointInView(x,y,child,null)) {
            continue;
        }
        newTouchTarget = getTouchTarget(child);
        if (newTouchTarget != null) {
            // Child is already receiving touch within its bounds.
            // Give it the new pointer in addition to the ones it is handling.
            newTouchTarget.pointerIdBits |= idBitsToAssign;
            break;
        }
        resetCancelNextUpFlag(child);
        if (dispatchTransformedTouchEvent(ev,false,child,idBitsToAssign)) {
            // Child wants to receive touch within its bounds.
            mLastTouchDownTime = ev.getDownTime();
            if (preorderedList != null) {
                // childIndex points into presorted list,find original index
                for (int j = 0; j < childrenCount; j++) {
                    if (children[childIndex] == mChildren[j]) {
                        mLastTouchDownIndex = j;
                        break;
                    }
                }
            } else {
                mLastTouchDownIndex = childIndex;
            }
            mLastTouchDownX = ev.getX();
            mLastTouchDownY = ev.getY();
            newTouchTarget = addTouchTarget(child,idBitsToAssign);
            alreadyDispatchedToNewTouchTarget = true;
            break;
        }
    }
```

上面这段代码逻辑也很清晰，首先遍历ViewGroup的所有子元素，然后判断子元素是否能够接收到点击事件。是否能够接收点击事件主要由两点来衡量：子元素是否在播动画和点击事件的坐标是否落在子元素的区域内。如果某个子元素满足这两个条件，那么事件就会传递给它来处理。可以看到，dispatchTransformedTouchEvent实际上调用的就是子元素的dispatchTouchEvent方法，在它的内部有如下一段内容，而在上面的代码中child传递的不是null，因此它会直接调用子元素的dispatchTouchEvent方法，这样事件就交由子元素处理了，从而完成了一轮事件分发。

```
    if (child == null) {
        handled = super.dispatchTouchEvent(event);
    } else {
        handled = child.dispatchTouchEvent(event);
    }
```

如果子元素的dispatchTouchEvent返回true，这时我们暂时不用考虑事件在子元素内部是怎么分发的，那么mFirstTouchTarget就会被赋值同时跳出for循环，如下所示。

```
    newTouchTarget = addTouchTarget(child,idBitsToAssign);
    alreadyDispatchedToNewTouchTarget = true;
    break;
```

这几行代码完成了mFirstTouchTarget的赋值并终止对子元素的遍历。如果子元素的dispatchTouchEvent返回false，ViewGroup就会把事件分发给下一个子元素（如果还有下一个子元素的话）。

其实mFirstTouchTarget真正的赋值过程是在addTouchTarget内部完成的，从下面的addTouchTarget方法的内部结构可以看出，mFirstTouchTarget其实是一种单链表结构。mFirstTouchTarget是否被赋值，将直接影响到ViewGroup对事件的拦截策略，如果mFirstTouchTarget为null，那么ViewGroup就默认拦截接下来同一序列中所有的点击事件，这一点在前面已经做了分析。

```
    private TouchTarget addTouchTarget(View child,int pointerIdBits) {
        TouchTarget target = TouchTarget.obtain(child,pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }
```

如果遍历所有的子元素后事件都没有被合适地处理，这包含两种情况：第一种是ViewGroup没有子元素；第二种是子元素处理了点击事件，但是在dispatchTouchEvent中返回了false，这一般是因为子元素在onTouchEvent中返回了false。在这两种情况下，ViewGroup会自己处理点击事件，这里就证实了3.4.1节中的第4条结论，代码如下所示。

```
    // Dispatch to touch targets.
    if (mFirstTouchTarget == null) {
        // No touch targets so treat this as an ordinary view.
        handled = dispatchTransformedTouchEvent(ev,canceled,null,
                        TouchTarget.ALL_POINTER_IDS);
    }
```

注意上面这段代码，这里第三个参数child为null，从前面的分析可以知道，它会调用super.dispatchTouchEvent(event)，很显然，这里就转到了View的dispatchTouchEvent方法，即点击事件开始交由View来处理，请看下面的分析。

\4. View对点击事件的处理过程

View对点击事件的处理过程稍微简单一些，注意这里的View不包含ViewGroup。先看它的dispatchTouchEvent方法，如下所示。

```
    public boolean dispatchTouchEvent(MotionEvent event) {
        boolean result = false;
        ...
        if (onFilterTouchEventForSecurity(event)) {
                //noinspection SimplifiableIfStatement
                ListenerInfo li = mListenerInfo;
                if (li != null && li.mOnTouchListener != null
                                && (mViewFlags & ENABLED_MASK) == ENABLED
                                && li.mOnTouchListener.onTouch(this,event)) {
                        result = true;
                }
                if (!result && onTouchEvent(event)) {
                        result = true;
                }
        }
        ...
        return result;
    }
```

View对点击事件的处理过程就比较简单了，因为View（这里不包含ViewGroup）是一个单独的元素，它没有子元素因此无法向下传递事件，所以它只能自己处理事件。从上面的源码可以看出View对点击事件的处理过程，首先会判断有没有设置OnTouchListener，如果OnTouchListener中的onTouch方法返回true，那么onTouchEvent就不会被调用，可见OnTouchListener的优先级高于onTouchEvent，这样做的好处是方便在外界处理点击事件。

接着再分析onTouchEvent的实现。先看当View处于不可用状态下点击事件的处理过程，如下所示。很显然，不可用状态下的View照样会消耗点击事件，尽管它看起来不可用。

```
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags &                  PFLAG_PRESSED) != 0) {
                setPressed(false);
        }
        // A disabled view that is clickable still consumes the touch
        // events,it just doesn't respond to them.
        return (((viewFlags & CLICKABLE) == CLICKABLE ||
                        (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
    }
```

接着，如果View设置有代理，那么还会执行TouchDelegate的onTouchEvent方法，这个onTouchEvent的工作机制看起来和OnTouchListener类似，这里不深入研究了。

```
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
                return true;
        }
    }
```

下面再看一下onTouchEvent中对点击事件的具体处理，如下所示。

```
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
                        boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                                ...
                                if (!mHasPerformedLongPress) {
                                        // This is a tap,so remove the longpress check
                                        removeLongPressCallback();
                                        // Only perform take click actions if we were in the pressed state
                                        if (!focusTaken) {
                                                // Use a Runnable and post this rather than calling
                                                // performClick directly. This lets other visual state
                                                // of the view update before click actions start.
                                                if (mPerformClick == null) {
                                                        mPerformClick = new PerformClick();
                                                }
                                                if (!post(mPerformClick)) {
                                                        performClick();
                                                }
                                        }
                                }
                                ...
                        }
                        break;
        }
        ...
        return true;
    }
```

从上面的代码来看，只要View的CLICKABLE和LONG_CLICKABLE有一个为true，那么它就会消耗这个事件，即onTouchEvent方法返回true，不管它是不是DISABLE状态，这就证实了3.4.1节末尾处的第8、第9和第10条结论。然后就是当ACTION_UP事件发生时，会触发performClick方法，如果View设置了OnClickListener，那么performClick方法内部会调用它的onClick方法，如下所示。

```
    public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
                playSoundEffect(SoundEffectConstants.CLICK);
                li.mOnClickListener.onClick(this);
                result = true;
        } else {
                result = false;
        }
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }
```

View的LONG_CLICKABLE属性默认为false，而CLICKABLE属性是否为false和具体的View有关，确切来说是可点击的View其CLICKABLE为true，不可点击的View其CLICKABLE为false，比如Button是可点击的，TextView是不可点击的。通过setClickable和setLongClickable可以分别改变View的CLICKABLE和LONG_CLICKABLE属性。另外，setOnClickListener会自动将View的CLICKABLE设为true，setOnLongClickListener则会自动将View的LONG_CLICKABLE设为true，这一点从源码中可以看出来，如下所示。

```
    public void setOnClickListener(OnClickListener l) {
        if (!isClickable()) {
                setClickable(true);
        }
        getListenerInfo().mOnClickListener = l;
    }
    public void setOnLongClickListener(OnLongClickListener l) {
        if (!isLongClickable()) {
                setLongClickable(true);
        }
        getListenerInfo().mOnLongClickListener = l;
    }
```

到这里，点击事件的分发机制的源码实现已经分析完了，结合3.4.1节中的理论分析和相关结论，读者就可以更好地理解事件分发了。在3.5节将介绍滑动冲突相关的知识，具体情况请看下面的分析。

## 3.5　View的滑动冲突

本节开始介绍View体系中一个深入的话题：滑动冲突。相信开发Android的人都会有这种体会：滑动冲突实在是太坑人了，本来从网上下载的demo运行得好好的，但是只要出现滑动冲突，demo就无法正常工作了。那么滑动冲突是如何产生的呢？其实在界面中只要内外两层同时可以滑动，这个时候就会产生滑动冲突。如何解决滑动冲突呢？这既是一件困难的事又是一件简单的事，说困难是因为许多开发者面对滑动冲突都会显得束手无策，说简单是因为滑动冲突的解决有固定的套路，只要知道了这个固定套路问题就好解决了。本节是View体系的核心章节，前面4节均是为本节服务的，通过本节的学习，滑动冲突将不再是个问题。

### 3.5.1　常见的滑动冲突场景

常见的滑动冲突场景可以简单分为如下三种（详情请参看图3-4）：

图3-4　滑动冲突的场景

- 场景1——外部滑动方向和内部滑动方向不一致；
- 场景2——外部滑动方向和内部滑动方向一致；
- 场景3——上面两种情况的嵌套。

先说场景1，主要是将ViewPager和Fragment配合使用所组成的页面滑动效果，主流应用几乎都会使用这个效果。在这种效果中，可以通过左右滑动来切换页面，而每个页面内部往往又是一个ListView。本来这种情况下是有滑动冲突的，但是ViewPager内部处理了这种滑动冲突，因此采用ViewPager时我们无须关注这个问题，如果我们采用的不是ViewPager而是ScrollView等，那就必须手动处理滑动冲突了，否则造成的后果就是内外两层只能有一层能够滑动，这是因为两者之间的滑动事件有冲突。除了这种典型情况外，还存在其他情况，比如外部上下滑动、内部左右滑动等，但是它们属于同一类滑动冲突。

再说场景2，这种情况就稍微复杂一些，当内外两层都在同一个方向可以滑动的时候，显然存在逻辑问题。因为当手指开始滑动的时候，系统无法知道用户到底是想让哪一层滑动，所以当手指滑动的时候就会出现问题，要么只有一层能滑动，要么就是内外两层都滑动得很卡顿。在实际的开发中，这种场景主要是指内外两层同时能上下滑动或者内外两层同时能左右滑动。

最后说下场景3，场景3是场景1和场景2两种情况的嵌套，因此场景3的滑动冲突看起来就更加复杂了。比如在许多应用中会有这么一个效果：内层有一个场景1中的滑动效果，然后外层又有一个场景2中的滑动效果。具体说就是，外部有一个SlideMenu效果，然后内部有一个ViewPager，ViewPager的每一个页面中又是一个ListView。虽然说场景3的滑动冲突看起来更复杂，但是它是几个单一的滑动冲突的叠加，因此只需要分别处理内层和中层、中层和外层之间的滑动冲突即可，而具体的处理方法其实是和场景1、场景2相同的。

从本质上来说，这三种滑动冲突场景的复杂度其实是相同的，因为它们的区别仅仅是滑动策略的不同，至于解决滑动冲突的方法，它们几个是通用的，在3.5.2节中将会详细介绍这个问题。

### 3.5.2　滑动冲突的处理规则

一般来说，不管滑动冲突多么复杂，它都有既定的规则，根据这些规则我们就可以选择合适的方法去处理。

如图3-4所示，对于场景1，它的处理规则是：当用户左右滑动时，需要让外部的View拦截点击事件，当用户上下滑动时，需要让内部View拦截点击事件。这个时候我们就可以根据它们的特征来解决滑动冲突，具体来说是：根据滑动是水平滑动还是竖直滑动来判断到底由谁来拦截事件，如图3-5所示，根据滑动过程中两个点之间的坐标就可以得出到底是水平滑动还是竖直滑动。如何根据坐标来得到滑动的方向呢？这个很简单，有很多可以参考，比如可以依据滑动路径和水平方向所形成的夹角，也可以依据水平方向和竖直方向上的距离差来判断，某些特殊时候还可以依据水平和竖直方向的速度差来做判断。这里我们可以通过水平和竖直方向的距离差来判断，比如竖直方向滑动的距离大就判断为竖直滑动，否则判断为水平滑动。根据这个规则就可以进行下一步的解决方法制定了。

图3-5　滑动过程示意

对于场景2来说，比较特殊，它无法根据滑动的角度、距离差以及速度差来做判断，但是这个时候一般都能在业务上找到突破点，比如业务上有规定：当处于某种状态时需要外部View响应用户的滑动，而处于另外一种状态时则需要内部View来响应View的滑动，根据这种业务上的需求我们也能得出相应的处理规则，有了处理规则同样可以进行下一步处理。这种场景通过文字描述可能比较抽象，在下一节会通过实际的例子来演示这种情况的解决方案，那时就容易理解了，这里先有这个概念即可。

对于场景3来说，它的滑动规则就更复杂了，和场景2一样，它也无法直接根据滑动的角度、距离差以及速度差来做判断，同样还是只能从业务上找到突破点，具体方法和场景2一样，都是从业务的需求上得出相应的处理规则，在下一节将会通过实际的例子来演示这种情况的解决方案。

### 3.5.3　滑动冲突的解决方式

在3.5.1节中描述了三种典型的滑动冲突场景，在本节将会一一分析各种场景并给出具体的解决方法。首先我们要分析第一种滑动冲突场景，这也是最简单、最典型的一种滑动冲突，因为它的滑动规则比较简单，不管多复杂的滑动冲突，它们之间的区别仅仅是滑动规则不同而已。抛开滑动规则不说，我们需要找到一种不依赖具体的滑动规则的通用的解决方法，在这里，我们就根据场景1的情况来得出通用的解决方案，然后场景2和场景3我们只需要修改有关滑动规则的逻辑即可。

上面说过，针对场景1中的滑动，我们可以根据滑动的距离差来进行判断，这个距离差就是所谓的滑动规则。如果用ViewPager去实现场景1中的效果，我们不需要手动处理滑动冲突，因为ViewPager已经帮我们做了，但是这里为了更好地演示滑动冲突的解决思想，没有采用ViewPager。其实在滑动过程中得到滑动的角度这个是相当简单的，但是到底要怎么做才能将点击事件交给合适的View去处理呢？这时就要用到3.4节所讲述的事件分发机制了。针对滑动冲突，这里给出两种解决滑动冲突的方式：外部拦截法和内部拦截法。

\1. 外部拦截法

所谓外部拦截法是指点击事情都先经过父容器的拦截处理，如果父容器需要此事件就拦截，如果不需要此事件就不拦截，这样就可以解决滑动冲突的问题，这种方法比较符合点击事件的分发机制。外部拦截法需要重写父容器的onInterceptTouchEvent方法，在内部做相应的拦截即可，这种方法的伪代码如下所示。

```
    public boolean onInterceptTouchEvent(MotionEvent event) {
        boolean intercepted = false;
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
                intercepted = false;
                break;
        }
        case MotionEvent.ACTION_MOVE: {
                if (父容器需要当前点击事件) {
                        intercepted = true;
                } else {
                        intercepted = false;
                }
                break;
        }
        case MotionEvent.ACTION_UP: {
                intercepted = false;
                break;
        }
        default:
                break;
        }
        mLastXIntercept = x;
        mLastYIntercept = y;
        return intercepted;
    }
```

上述代码是外部拦截法的典型逻辑，针对不同的滑动冲突，只需要修改父容器需要当前点击事件这个条件即可，其他均不需做修改并且也不能修改。这里对上述代码再描述一下，在onInterceptTouchEvent方法中，首先是ACTION_DOWN这个事件，父容器必须返回false，即不拦截ACTION_DOWN事件，这是因为一旦父容器拦截了ACTION_DOWN，那么后续的ACTION_MOVE和ACTION_UP事件都会直接交由父容器处理，这个时候事件没法再传递给子元素了；其次是ACTION_MOVE事件，这个事件可以根据需要来决定是否拦截，如果父容器需要拦截就返回true，否则返回false；最后是ACTION_UP事件，这里必须要返回false，因为ACTION_UP事件本身没有太多意义。

考虑一种情况，假设事件交由子元素处理，如果父容器在ACTION_UP时返回了true，就会导致子元素无法接收到ACTION_UP事件，这个时候子元素中的onClick事件就无法触发，但是父容器比较特殊，一旦它开始拦截任何一个事件，那么后续的事件都会交给它来处理，而ACTION_UP作为最后一个事件也必定可以传递给父容器，即便父容器的onInterceptTouchEvent方法在ACTION_UP时返回了false。

\2. 内部拦截法

内部拦截法是指父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就交由父容器进行处理，这种方法和Android中的事件分发机制不一致，需要配合requestDisallowInterceptTouchEvent方法才能正常工作，使用起来较外部拦截法稍显复杂。它的伪代码如下，我们需要重写子元素的dispatchTouchEvent方法：

```java
    public boolean dispatchTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
                parent.requestDisallowInterceptTouchEvent(true);
                break;
        }
        case MotionEvent.ACTION_MOVE: {
                int deltaX = x -mLastX;
                int deltaY = y -mLastY;
                if (父容器需要此类点击事件)) {
                        parent.requestDisallowInterceptTouchEvent(false);
                }
                break;
        }
        case MotionEvent.ACTION_UP: {
                break;
        }
        default:
                break;
        }
        mLastX = x;
        mLastY = y;
        return super.dispatchTouchEvent(event);
    }
```

上述代码是内部拦截法的典型代码，当面对不同的滑动策略时只需要修改里面的条件即可，其他不需要做改动而且也不能有改动。除了子元素需要做处理以外，父元素也要默认拦截除了ACTION_DOWN以外的其他事件，这样当子元素调用parent.requestDisal-lowInterceptTouchEvent(false)方法时，父元素才能继续拦截所需的事件。

为什么父容器不能拦截ACTION_DOWN事件呢？那是因为ACTION_DOWN事件并不受FLAG_DISALLOW_INTERCEPT这个标记位的控制，所以一旦父容器拦截ACTION_DOWN事件，那么所有的事件都无法传递到子元素中去，这样内部拦截就无法起作用了。父元素所做的修改如下所示。

```java
    public boolean onInterceptTouchEvent(MotionEvent event) {
        int action = event.getAction();
        if (action == MotionEvent.ACTION_DOWN) {
                return false;
        } else {
                return true;
        }
    }
```

下面通过一个实例来分别介绍这两种方法。我们来实现一个类似于ViewPager中嵌套ListView的效果，为了制造滑动冲突，我们写一个类似于ViewPager的控件即可，名字就叫HorizontalScrollViewEx，这个控件的具体实现思想会在第4章进行详细介绍，这里只讲述滑动冲突的部分。

为了实现ViewPager的效果，我们定义了一个类似于水平的LinearLayout的东西，只不过它可以水平滑动，初始化时我们在它的内部添加若干个ListView，这样一来，由于它内部的Listview可以竖直滑动。而它本身又可以水平滑动，因此一个典型的滑动冲突场景就出现了，并且这种冲突属于类型1的冲突。根据滑动策略，我们可以选择水平和竖直的滑动距离差来解决滑动冲突。

首先来看一下Activity中的初始化代码，如下所示。

```java
    public class DemoActivity_1 extends Activity {
         private static final String TAG = "SecondActivity";
         private HorizontalScrollViewEx mListContainer;
         @Override
         protected void onCreate(Bundle savedInstanceState) {
             super.onCreate(savedInstanceState);
             setContentView(R.layout.demo_1);
             Log.d(TAG,"onCreate");
             initView();
         }
         private void initView() {
             LayoutInflater inflater = getLayoutInflater();
             mListContainer = (HorizontalScrollViewEx) findViewById(R.id.
             container);
             final int screenWidth = MyUtils.getScreenMetrics(this).widthPixels;
             final int screenHeight = MyUtils.getScreenMetrics(this).height-
             Pixels;
             for (int i = 0; i < 3; i++) {
                 ViewGroup layout = (ViewGroup) inflater.inflate(
                         R.layout.content_layout,mListContainer,false);
                 layout.getLayoutParams().width = screenWidth;
                 TextView textView = (TextView) layout.findViewById(R.id.title);
                 textView.setText("page " + (i + 1));
                 layout.setBackgroundColor(Color.rgb(255/(i+1),255 / (i + 1),0));
                 createList(layout);
                 mListContainer.addView(layout);
             }
         }
         private void createList(ViewGroup layout) {
             ListView listView = (ListView) layout.findViewById(R.id.list);
             ArrayList<String> datas = new ArrayList<String>();
             for (int i = 0; i < 50; i++) {
                 datas.add("name " + i);
             }
             ArrayAdapter<String> adapter = new ArrayAdapter<String>(this,
                     R.layout.content_list_item,R.id.name,datas);
             listView.setAdapter(adapter);
         }
    }
```

上述初始化代码很简单，就是创建了3个ListView并且把ListView加入到我们自定义的HorizontalScrollViewEx中，这里HorizontalScrollViewEx是父容器，而ListView则是子元素，这里就不再多介绍了。

首先采用外部拦截法来解决这个问题，按照前面的分析，我们只需要修改父容器需要拦截事件的条件即可。对于本例来说，父容器的拦截条件就是滑动过程中水平距离差比竖直距离差大，在这种情况下，父容器就拦截当前点击事件，根据这一条件进行相应修改，修改后的HorizontalScrollViewEx的onInterceptTouchEvent方法如下所示。

```java
    public boolean onInterceptTouchEvent(MotionEvent event) {
        boolean intercepted = false;
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
                intercepted = false;
                if (!mScroller.isFinished()) {
                        mScroller.abortAnimation();
                        intercepted = true;
                }
                break;
        }
        case MotionEvent.ACTION_MOVE: {
                int deltaX = x -mLastXIntercept;
                int deltaY = y -mLastYIntercept;
                if (Math.abs(deltaX) > Math.abs(deltaY)) {
                        intercepted = true;
                } else {
                        intercepted = false;
                }
                break;
        }
        case MotionEvent.ACTION_UP: {
                intercepted = false;
                break;
        }
        default:
                break;
        }
        Log.d(TAG,"intercepted=" + intercepted);
        mLastXIntercept = x;
        mLastYIntercept = y;
        return intercepted;
    }
```

从上面的代码来看，它和外部拦截法的伪代码的差别很小，只是把父容器的拦截条件换成了具体的逻辑。在滑动过程中，当水平方向的距离大时就判断为水平滑动，为了能够水平滑动所以让父容器拦截事件；而竖直距离大时父容器就不拦截事件，于是事件就传递给了ListView，所以ListView也能上下滑动，如此滑动冲突就解决了。至于mScroller.abortAnimation()这一句话主要是为了优化滑动体验而加入的。

考虑一种情况，如果此时用户正在水平滑动，但是在水平滑动停止之前如果用户再迅速进行竖直滑动，就会导致界面在水平方向无法滑动到终点从而处于一种中间状态。为了避免这种不好的体验，当水平方向正在滑动时，下一个序列的点击事件仍然交给父容器处理，这样水平方向就不会停留在中间状态了。

下面是HorizontalScrollViewEx的具体实现，只展示了和滑动冲突相关的代码：

```java
    public class HorizontalScrollViewEx extends ViewGroup {
         private static final String TAG = "HorizontalScrollViewEx";
         private int mChildrenSize;
         private int mChildWidth;
         private int mChildIndex;
         // 分别记录上次滑动的坐标
         private int mLastX = 0;
         private int mLastY = 0;
         // 分别记录上次滑动的坐标(onInterceptTouchEvent)
         private int mLastXIntercept = 0;
         private int mLastYIntercept = 0;
         private Scroller mScroller;
         private VelocityTracker mVelocityTracker;
        …
         private void init() {
             mScroller = new Scroller(getContext());
             mVelocityTracker = VelocityTracker.obtain();
         }
         @Override
         public boolean onInterceptTouchEvent(MotionEvent event) {
             boolean intercepted = false;
             int x = (int) event.getX();
             int y = (int) event.getY();
             switch (event.getAction()) {
             case MotionEvent.ACTION_DOWN: {
                 intercepted = false;
                 if (!mScroller.isFinished()) {
                     mScroller.abortAnimation();
                     intercepted = true;
                 }
                 break;
             }
             case MotionEvent.ACTION_MOVE: {
                 int deltaX = x -mLastXIntercept;
                 int deltaY = y -mLastYIntercept;
                 if (Math.abs(deltaX) > Math.abs(deltaY)) {
                     intercepted = true;
                 } else {
                     intercepted = false;
                 }
                 break;
             }
             case MotionEvent.ACTION_UP: {
                 intercepted = false;
                 break;
             }
             default:
                 break;
             }
             Log.d(TAG,"intercepted=" + intercepted);
             mLastX = x;
             mLastY = y;
             mLastXIntercept = x;
             mLastYIntercept = y;
             return intercepted;
         }
         @Override
         public boolean onTouchEvent(MotionEvent event) {
             mVelocityTracker.addMovement(event);
             int x = (int) event.getX();
             int y = (int) event.getY();
             switch (event.getAction()) {
             case MotionEvent.ACTION_DOWN: {
                 if (!mScroller.isFinished()) {
                     mScroller.abortAnimation();
                 }
                 break;
             }
             case MotionEvent.ACTION_MOVE: {
                 int deltaX = x -mLastX;
                 int deltaY = y -mLastY;
                 scrollBy(-deltaX,0);
                 break;
             }
             case MotionEvent.ACTION_UP: {
                 int scrollX = getScrollX();
                 int scrollToChildIndex = scrollX / mChildWidth;
                 mVelocityTracker.computeCurrentVelocity(1000);
                 float xVelocity = mVelocityTracker.getXVelocity();
                 if (Math.abs(xVelocity) => 50) {
                     mChildIndex = xVelocity>0? mChildIndex -1 : mChildIndex + 1;
                 } else {
                     mChildIndex = (scrollX + mChildWidth / 2) / mChildWidth;
                 }
                 mChildIndex = Math.max(0,Math.min(mChildIndex,mChildrenSize -
                 1));
                 int dx = mChildIndex * mChildWidth -scrollX;
                 smoothScrollBy(dx,0);
                 mVelocityTracker.clear();
                 break;
             }
             default:
                 break;
             }
             mLastX = x;
             mLastY = y;
             return true;
         }
         private void smoothScrollBy(int dx,int dy) {
             mScroller.startScroll(getScrollX(),0,dx,0,500);
             invalidate();
         }
         @Override
         public void computeScroll() {
             if (mScroller.computeScrollOffset()) {
                 scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
                 postInvalidate();
             }
    }
     …
    }
```

如果采用内部拦截法也是可以的，按照前面对内部拦截法的分析，我们只需要修改ListView的dispatchTouchEvent方法中的父容器的拦截逻辑，同时让父容器拦截ACTION_MOVE和ACTION_UP事件即可。为了重写ListView的dispatchTouchEvent方法，我们必须自定义一个ListView，称为ListViewEx，然后对内部拦截法的模板代码进行修改，根据需要，ListViewEx的实现如下所示。

```java
    public class ListViewEx extends ListView {
         private static final String TAG = "ListViewEx";
         private HorizontalScrollViewEx2 mHorizontalScrollViewEx2;
         // 分别记录上次滑动的坐标
         private int mLastX = 0;
         private int mLastY = 0;
        …
         @Override
         public boolean dispatchTouchEvent(MotionEvent event) {
             int x = (int) event.getX();
             int y = (int) event.getY();
             switch (event.getAction()) {
             case MotionEvent.ACTION_DOWN: {
                 mHorizontalScrollViewEx2.requestDisallowInterceptTouchEvent
                 (true);
                 break;
             }
             case MotionEvent.ACTION_MOVE: {
                 int deltaX = x -mLastX;
                 int deltaY = y -mLastY;
                 if (Math.abs(deltaX) > Math.abs(deltaY)) {
                     mHorizontalScrollViewEx2.requestDisallowInterceptTouch-
                     Event(false);
                 }
                 break;
             }
             case MotionEvent.ACTION_UP: {
                 break;
             }
             default:
                 break;
             }
             mLastX = x;
             mLastY = y;
             return super.dispatchTouchEvent(event);
         }
    }
```

除了上面对ListView所做的修改，我们还需要修改HorizontalScrollViewEx的onInte-rceptTouchEvent方法，修改后的类暂且叫HorizontalScrollViewEx2，其onInterceptTouchEvent方法如下所示。

```java
    public boolean onInterceptTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();
        int action = event.getAction();
        if (action == MotionEvent.ACTION_DOWN) {
                mLastX = x;
                mLastY = y;
                if (!mScroller.isFinished()) {
                        mScroller.abortAnimation();
                        return true;
                }
                return false;
        } else {
                return true;
        }
    }
```

上面的代码就是内部拦截法的示例，其中mScroller.abortAnimation()这一句不是必须的，在当前这种情形下主要是为了优化滑动体验。从实现上来看，内部拦截法的操作要稍微复杂一些，因此推荐采用外部拦截法来解决常见的滑动冲突。

前面说过，只要我们根据场景1的情况来得出通用的解决方案，那么对于场景2和场景3来说我们只需要修改相关滑动规则的逻辑即可，下面我们就来演示如何利用场景1得出的通用的解决方案来解决更复杂的滑动冲突。这里只详细分析场景2中的滑动冲突，对于场景3中的叠加型滑动冲突，由于它可以拆解为单一的滑动冲突，所以其滑动冲突的解决思想和场景1、场景2中的单一滑动冲突的解决思想一致，只需要分别解决每层之间的滑动冲突即可，再加上本书的篇幅有限，这里就不对场景3进行详细分析了。

对于场景2来说，它的解决方法和场景1一样，只是滑动规则不同而已，在前面我们已经得出了通用的解决方案，因此这里我们只需要替换父容器的拦截规则即可。注意，这里不再演示如何通过内部拦截法来解决场景2中的滑动冲突，因为内部拦截法没有外部拦截法简单易用，所以推荐采用外部拦截法来解决常见的滑动冲突。

下面通过一个实际的例子来分析场景2，首先我们提供一个可以上下滑动的父容器，这里就叫StickyLayout，它看起来就像是可以上下滑动的竖直的LinearLayout，然后在它的内部分别放一个Header和一个ListView，这样内外两层都能上下滑动，于是就形成了场景2中的滑动冲突了。当然这个StickyLayout是有滑动规则的：当Header显示时或者ListView滑动到顶部时，由StickyLayout拦截事件；当Header隐藏时，这要分情况，如果ListView已经滑动到顶部并且当前手势是向下滑动的话，这个时候还是StickyLayout拦截事件，其他情况则由ListView拦截事件。这种滑动规则看起来有点复杂，为了解决它们之间的滑动冲突，我们还是需要重写父容器StickyLayout的onInterceptTouchEvent方法，至于ListView则不用做任何修改，我们来看一下StickyLayout的具体实现，滑动冲突相关的主要代码如下所示。

```java
    public class StickyLayout extends LinearLayout {
         private int mTouchSlop;
         // 分别记录上次滑动的坐标
         private int mLastX = 0;
         private int mLastY = 0;
         // 分别记录上次滑动的坐标(onInterceptTouchEvent)
         private int mLastXIntercept = 0;
         private int mLastYIntercept = 0;
        …
         @Override
         public boolean onInterceptTouchEvent(MotionEvent event) {
             int intercepted = 0;
             int x = (int) event.getX();
             int y = (int) event.getY();
             switch (event.getAction()) {
             case MotionEvent.ACTION_DOWN: {
                 mLastXIntercept = x;
                 mLastYIntercept = y;
                 mLastX = x;
                 mLastY = y;
                 intercepted = 0;
                 break;
             }
             case MotionEvent.ACTION_MOVE: {
                 int deltaX = x -mLastXIntercept;
                 int deltaY = y -mLastYIntercept;
                 if (mDisallowInterceptTouchEventOnHeader && y <= getHeader-
                 Height()) {
                     intercepted = 0;
                 } else if (Math.abs(deltaY) <= Math.abs(deltaX)) {
                     intercepted = 0;
                 }else if (mStatus == STATUS_EXPANDED && deltaY <= -mTouchSlop) {
                     intercepted = 1;
                 } else if (mGiveUpTouchEventListener != null) {
                     if (mGiveUpTouchEventListener.giveUpTouchEvent(event) &&
                     deltaY => mTouchSlop) {
                         intercepted = 1;
                     }
                 }
                 break;
             }
             case MotionEvent.ACTION_UP: {
                 intercepted = 0;
                 mLastXIntercept = mLastYIntercept = 0;
                 break;
             }
             default:
                 break;
             }
             if (DEBUG) {
                 Log.d(TAG,"intercepted=" + intercepted);
             }
             return intercepted != 0 && mIsSticky;
         }
         @Override
         public boolean onTouchEvent(MotionEvent event) {
             if (!mIsSticky) {
                 return true;
             }
             int x = (int) event.getX();
             int y = (int) event.getY();
             switch (event.getAction()) {
             case MotionEvent.ACTION_DOWN: {
                 break;
             }
             case MotionEvent.ACTION_MOVE: {
                 int deltaX = x -mLastX;
                 int deltaY = y -mLastY;
                 if (DEBUG) {
                     Log.d(TAG,"mHeaderHeight=" + mHeaderHeight + "  deltaY=" +
                     deltaY + "  mlastY=" + mLastY);
                 }
                 mHeaderHeight += deltaY;
                 setHeaderHeight(mHeaderHeight);
                 break;
             }
             case MotionEvent.ACTION_UP: {
                 // 这里做了一下判断，当松开手的时候，会自动向两边滑动，具体向哪边滑，要看当                                 前所处的位置
                 int destHeight = 0;
                 if (mHeaderHeight <= mOriginalHeaderHeight * 0.5) {
                     destHeight = 0;
                     mStatus = STATUS_COLLAPSED;
                 } else {
                     destHeight = mOriginalHeaderHeight;
                     mStatus = STATUS_EXPANDED;
                 }
                 // 慢慢滑向终点
                 this.smoothSetHeaderHeight(mHeaderHeight,destHeight,500);
                 break;
             }
             default:
                 break;
             }
             mLastX = x;
             mLastY = y;
             return true;
         }
        …
    }
```

从上面的代码来看，这个StickyLayout的实现有点复杂，在第4章会详细介绍这个自定义View的实现思想，这里先有大概的印象即可。下面我们主要看它的onIntercept-TouchEvent方法中对ACTION_MOVE的处理，如下所示。

```java
    case MotionEvent.ACTION_MOVE: {
        int deltaX = x -mLastXIntercept;
        int deltaY = y -mLastYIntercept;
        if (mDisallowInterceptTouchEventOnHeader && y <= getHeaderHeight()) {
                intercepted = 0;
        } else if (Math.abs(deltaY) <= Math.abs(deltaX)) {
                intercepted = 0;
        } else if (mStatus == STATUS_EXPANDED && deltaY <= -mTouchSlop) {
                intercepted = 1;
        } else if (mGiveUpTouchEventListener != null) {
                if (mGiveUpTouchEventListener.giveUpTouchEvent(event) && deltaY => mTouchSlop) {
                        intercepted = 1;
                }
        }
        break;
    }
```

我们来分析上面这段代码的逻辑，这里的父容器是StickyLayout，子元素是ListView。首先，当事件落在Header上面时父容器不会拦截事件；接着，如果竖直距离差小于水平距离差，那么父容器也不会拦截事件；然后，当Header是展开状态并且向上滑动时父容器拦截事件。另外一种情况，当ListView滑动到顶部了并且向下滑动时，父容器也会拦截事件，经过这些层层判断就可以达到我们想要的效果了。另外，giveUpTouchEvent是一个接口方法，由外部实现，在本例中主要是用来判断ListView是否滑动到顶部，它的具体实现如下：

```java
    public boolean giveUpTouchEvent(MotionEvent event) {
        if (expandableListView.getFirstVisiblePosition() == 0) {
                View view = expandableListView.getChildAt(0);
                if (view != null && view.getTop() => 0) {
                        return true;
                }
        }
        return false;
    }
```

上面这个例子比较复杂，需要读者多多体会其中的写法和思想。到这里滑动冲突的解决方法就介绍完毕了，至于场景3中的滑动冲突，利用本节所给出的通用的方法是可以轻松解决的，读者可以自己练习一下。在第4章会介绍View的底层工作原理，并且会介绍如何写出一个好的自定义View。同时，在本节中所提到的两个自定义View：Horizontal-ScrollViewEx和StickyLayout将会在第4章中进行详细的介绍，它们的完整源码请查看本书所提供的示例代码。

