# 第3章 View的事件体系
## 3.1 View基础知识
### 3.1.1 什么是view
### 3.1.2 View的位置参数
View的位置主要由它的四个顶点来决定，分别对应于View的四个属性：top、left、right、bottom，其中top是左上角纵坐标，left是左上角横坐标，right是右下角横坐标，bottom是右下角纵坐标。需要注意的是，这些坐标都是相对于View的父容器来说的，因此它是一种相对坐标。在Android中，x轴和y轴的正方向分别为右和下，这点不难理解，不仅仅是Android，大部分显示系统都是按照这个标准来定义坐标系的。
从Android3.0开始，View增加了额外的几个参数：x、y、translationX和translationY，其中x和y是View左上角的坐标，而translationX和translationY是View左上角相当于父容器的偏移量。这几个参数也是相对于父容器的坐标，并且translationX和translationY的默认值是0，和View的四个基本的位置参数一样，View也为它们提供了get/set方法，这几个参数的换算关系如下所示。

```java
x = left + translationX
y = left + translationY
```

需要注意的是，View在平移的过程中，top和left表示的是原始左上角的位置信息，其值并不会发生改变，此时发生改变的是x、y、translationX和translationY这个四个参数。
### 3.1.3 MotionEvent和TouchSlop
1. MotionEvent
通过MotionEvent对象我们可以得到点击事件发生的x和y坐标。为止，系统提供了两组方法：getX/getY和getRawX/getRawY。它们的区别其实很简单，getX/getY返回的是相对于当前View左上角的 x 和 y 坐标，而getRawX/getRawY返回的是相对于手机屏幕左上角的 x 和 y 坐标。
2. TouchSlop
TouchSlop是系统所能识别出的被认为是滑动的最小距离。这是一个常量，和设备有关，在不同设备上这个值可能是不同的，通过如下方式即可获取这个常量：ViewConfiguration.get(getContext()).getScaledTouchSlop()。可以在源码中找到这个常量的定义，在frameworks/base/core/res/res/values/config.xml文件中。

```xml
<!-- Base"touch slop" value used by ViewConfiguration as a movement threshold where scrolling should begin. -->
<dimen name="config_viewConfigurationTouchSlop">8dp</dimen>
```

### 3.1.4 VelocityTracker、GestureDetector和Scroller
1. VelocityTracker
    速度追踪，用于追踪手指在滑动过程中的速速，包括水平和竖直方法的速度。
2. GestureDetector
    手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。
3. Scroller
    弹性滑动对象，用于实现View的弹性滑动。我们知道，当使用View的scrollTo/scrollBy方法来进行滑动时，其过程是瞬间完成的，这个没有过渡效果的滑动用户体验不好。这个时候就可以使用Scroller来实现有过渡效果的滑动，其过程不是瞬间完成的，而是在一定的时间间隔内完成的。Scroller本身无法让View弹性滑动，它需要和View的computeScroll方法配合使用才能共同完成这个功能。那么如何使用Scroller呢？它的典型代码是固定的，代码略了~，至于它为什么能实现弹性滑动，这个在3.2节中会进行详细介绍。
## 3.2 View的滑动
在Android设备上，滑动几乎是应用的标配，不管是下拉刷新还是SlidingMenu，它们的基础都是滑动。从另外一方面来说，Android手机由于屏幕比较小，为了给用户呈现更多的内容，就需要使用滑动来隐藏和显示一些内容。通过三种方式可以实现View的滑动：第一种是通过View本身提供的scrollTo/scrollBy方法来实现滑动；第二种是通过动画给View施加平移效果来实现滑动；第三种是通过改变View的LayoutParams使得View重新布局从而实现滑动。
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

从源代码可以看出，scrollBy实际上也是调用了scrollTo方法，它实现了基于当前位置的相对滑动，而scrollTo则实现了基于所传递参数的绝对滑动，这个不难理解。但是我们要明白滑动过程中View内部的两个属性mScrollX和mScrollY的改变规则，这两个属性可以通过getScrollX和getScrollY方法分别得到。这里先简要概括一下：在滑动过程中，mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离，而mScrollY的值总是等于View上边缘和View内容上边缘在竖直方向的距离。View边缘是指View的位置，由四个顶点组成，而View内容边缘是指View中的内容的边缘，scrollTo和scrollBy只能改变View内容的位置而不能改变View在布局中的位置。mScrollX和mScrollY的单位为像素，并且当View左边缘在View内容左边缘的右边时，mScrollX为正值，反之为负值；当View上边缘在View内容上边缘的下边时，mScrollY为正值，反之为负值。换句话说，如果从左向右滑动，那么mScrollX为负值，反之为正值；如果从上往下滑动，那么mScrollY为负值，反之为正值。
    注意：使用scrollTo和scrollBy来实现View的滑动，只能将View的内容进行移到，并不能将View本身进行移到，也就是说，不管怎么滑动，也不可能将当前View滑动到附近View所在的区域，这个需要仔细体会一下。
### 3.2.2 使用动画
通过动画我们能够让一个View进行平移，而平移就是一种滑动。使用动画来移到View，主要是操作View的translationX和translationY属性，既可以采用传统的View动画，也可以采用属性动画。这里需要注意传统的View动画的一个弊端：传统的View动画并不能真正改变View的位置，这位带来一个很严重的问题。比如我们通过View动画将一个Button向右移到100px，并且这个View设置的有单击事件，然后你会惊奇地发现，单击新位置无法触发onClick事件，而单击原始位置仍然可以触发onClick事件，尽管Button已经不在原始位置了。这个问题带来的影响是致命的，但是它却又是可以理解的，因为不管Button怎么做变换，但是它的位置信息（四个顶点和宽/高）并不会随着动画而改变，因此在系统眼里，这个Button并没有发生任何改变，它的真身仍然在原始位置。在这种情况下，单击新位置当然不会触发onClick事件了，因为Button的真身并没有发生改变，在新位置上只是View的影像而已。
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

















