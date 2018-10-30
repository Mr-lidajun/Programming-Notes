# 第7章 Android动画深入分析
> Android的动画可以分为三种：View动画、帧动画和属性动画，其实帧动画也属于View动画的一种，只不过它和平移、旋转等常见的View动画在表现形式上略有不同而已。
## 7.1 View动画
View动画的作用对象是View，它支持4种动画效果，分别是平移动画、缩放动画、旋转动画和透明度动画。除了这四种典型的变换效果外，帧动画也属于View动画。
### 7.1.1 View动画的种类
表7-1 View动画的四种变换
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/T7-1.jpg)
### 7.1.2 自定义View动画
自定义动画是一件既简单又复杂的事情，说它简单，是因为派生一种新动画只需要继承Animation这个抽象类，然后重写它的initialize和applyTransformation方法，在initialize方法中做一些初始化工作，在applyTransformation中进行相应的矩阵变换即可，很多时候需要采用Camera来简化矩阵变换的过程。说它复杂，是因为自定义View动画的过程主要是矩阵变换的过程，而矩阵变换是数学上的概念，如果对这方面的知识不熟悉的话，就会觉得这个过程比较复杂了。

这个例子来自于Android的ApiDemos中的一个自定义View动画Rotate3dAnimation。Rotate3dAnimation可以围绕y轴旋转并且同时沿着z轴平移从而实现一种类似于3D的效果，它的代码如下：略
### 7.1.3 帧动画
帧动画是顺序播放一组预先定义好的图片，类似于电影播放。
## 7.2 View动画的特殊使用场景
### 7.2.1 LayoutAnimation
LayoutAnimation作用于ViewGroup，为ViewGroup指定一个动画，这样当它的子元素出场时都会具有这种动画效果。
### 7.2.2 Activity的切换效果
overridePendingTransition这个方法必须位于startActivity或者finish的后面，否则动画效果将不起作用。
我们可以通过FragmentTransaction中的setCustomAnimations()方法来添加切换动画。
## 7.3 属性动画
属性动画可以对任何对象做动画，甚至还可以没有对象。
### 7.3.1 使用属性动画
属性动画可以对任意对象的属性进行动画而不仅仅是View，动画默认时间间隔300ms，默认帧率10ms/帧。其可以达到的效果是：在一个时间间隔内完成对象从一个属性值到另一个属性值的改变。
### 7.3.2 理解插值器和估值器
**TimeInterpolator中文翻译为时间插值器，它的作用是根据时间流逝的百分比来计算出当前属性值改变的百分比**，系统预置的有LinearInterpolator（线性插值器：匀速动画）、AccelerateDecelerateInterpolator（加速减速插值器：动画两头慢中间快）和Decelerate-Interpolator（减速插值器：动画越来越慢）等。**TypeEvaluator的中文翻译为类型估值算法，也叫估值器，它的作用是根据当前属性改变的百分比来计算改变后的属性值**，系统预置的有IntEvaluator（针对整型属性）、FloatEvaluator（针对浮点型属性）和ArgbEvaluator（针对Color属性）。**属性动画中的插值器（Interpolator）和估值器（TypeEvaluator）很重要，它们是实现非匀速动画的重要手段。**

如图7-1所示，它表示一个匀速动画，采用了线性插值器和整型估值算法，在40ms内，View的x属性实现从0到40的变换。
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Android/Android开发艺术探索/img/7-1.jpg)
图7-1　插值器的工作原理（注：此图来自Android官方文档）

由于动画的默认刷新率为10ms/帧，所以该动画将分5帧进行，我们来考虑第三帧（x=20，t=20ms），当时间t=20ms的时候，时间流逝的百分比是0.5（20/40=0.5），意味着现在时间过了一半，那x应该改变多少呢？这个就由插值器和估值算法来确定。拿线性插值器来说，当时间流逝一半的时候，x的变换也应该是一半，即x的改变是0.5，为什么呢？因为它是线性插值器，是实现匀速动画的，下面看它的源码：

```java
public class LinearInterpolator implements Interpolator, NativeInterpolatorFactory {

    public LinearInterpolator() {
    }
    
    public LinearInterpolator(Context context, AttributeSet attrs) {
    }
    
    public float getInterpolation(float input) {
        return input;
    }

    /** @hide */
    @Override
    public long createNativeInterpolator() {
        return NativeInterpolatorFactoryHelper.createLinearInterpolator();
    }
}
```

很显然，线性插值器的返回值和输入值一样，因此插值器返回的值是0.5，这意味着x的改变是0.5，这个时候插值器的工作就完成了。具体x变成了什么值，这个需要估值算法来确定，我们来看看整型估值算法的源码：

```java
public class IntEvaluator implements TypeEvaluator<Integer> {

    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startInt = startValue;
        return (int)(startInt + fraction * (endValue - startInt));
    }
}
```

上述算法很简单，evaluate的三个参数分别表示估值小数、开始值和结束值，对应于我们的例子就分别是0.5、0、40。根据上述算法，整型估值返回给我们的结果是20，这就是（x=20，t=20ms）的由来。

**属性动画要求对象的该属性有set方法和get方法（可选）。**
具体一点就是：自定义插值器需要实现Interpolator或者TimeInter-polator，自定义估值算法需要实现TypeEvaluator。另外就是如果要对其他类型（非int、float、Color）做动画，那么必须要自定义类型估值算法。

### 7.3.3 属性动画的监听器

```java
public static interface AnimatorListener {
    void onAnimationStart(Animator animation);
    void onAnimationEnd(Animator animation);
    void onAnimationCancel(Animator animation);
    void onAnimationRepeat(Animator animation);
}
```

同时为了方便开发，系统还提供了AnimatorListenerAdapter这个类，它是Animator-Listener的适配器类

### 7.3.4 对任意属性做动画
首先提出一个问题：给Button加一个动画，让这个Button的宽度从当前宽度增加到500px。
我们用属性动画来做：

```java
private void performAnimate() {
    ObjectAnimator.ofInt(mButton,"width",500).setDuration(5000).start();
}
@Override
public void onClick(View v) {
    if (v == mButton) {
        performAnimate();
    }
}
```

上述代码运行后发现没效果，其实没效果是对的，如果随便传递一个属性过去，轻则没动画效果，重则程序直接Crash。

下面**分析属性动画的原理**：
属性动画要求动画作用的对象提供该属性的get和set方法，属性动画根据外界传递的该属性的初始值和最终值，以动画的效果多次去调用set方法，每次传递给set方法的值都不一样，确切来说是随着时间的推移，所传递的值越来越接近最终值。总结一下，**我们对object的属性abc做动画，如果想让动画生效，要同时满足两个条件**：

- （1）object必须要提供setAbc方法，如果动画的时候没有传递初始值，那么还要提供getAbc方法，因为系统要去取abc属性的初始值（如果这条不满足，程序直接Crash）。
- （2）object的setAbc对属性abc所做的改变必须能够通过某种方法反映出来，比如会带来UI的改变之类的（如果这条不满足，动画无效果但不会Crash）。

以上条件缺一不可。那么为什么我们对Button的width属性做动画会没有效果？这是因为Button内部虽然提供了getWidth和setWidth方法，但是这个setWidth方法并不是改变视图的大小，它是TextView新添加的方法，View是没有这个setWidth方法的，由于Button继承了TextView，所以Button也就有了setWidth方法。下面看一下这个getWidth和setWidth方法的源码：


```java
/**
 * Makes the TextView exactly this many pixels wide.
 * You could do the same thing by specifying this number in the
 * LayoutParams.
 *
 * @see #setMaxWidth(int)
 * @see #setMinWidth(int)
 * @see #getMinWidth()
 * @see #getMaxWidth()
 *
 * @attr ref android.R.styleable#TextView_width
 */
@android.view.RemotableViewMethod
public void setWidth(int pixels) {
    mMaxWidth = mMinWidth = pixels;
    mMaxWidthMode = mMinWidthMode = PIXELS;

    requestLayout();
    invalidate();
}
```

从上述源码可以看出，getWidth的确是获取View的宽度的，而setWidth是TextView和其子类的专属方法，它的作用不是设置View的宽度，而是设置TextView的最大宽度和最小宽度的，这个和TextView的宽度不是一个东西。具体来说，TextView的宽度对应XML中的android:layout_width属性，而TextView还有一个属性android:width，这个android:width属性就对应了TextView的setWidth方法。总之，TextView和Button的setWidth、getWidth干的不是同一件事情，通过setWidth无法改变控件的宽度，所以对width做属性动画没有效果。对应于属性动画的两个条件来说，本例中动画不生效的原因是只满足了条件1而未满足条件2。

**针对上述问题，官方文档上告诉我们有3种解决方法：**

- 给你的对象加上get和set方法，如果你有权限的话；
- 用一个类来包装原始对象，间接为其提供get和set方法；
- 采用ValueAnimator，监听动画过程，自己实现属性的改变。

1. **给你的对象加上get和set方法，如果你有权限的话**
如果你有权限的话，加上get和set就搞定了。但是很多时候我们没权限去这么做。

2. **用一个类来包装原始对象，间接为其提供get和set方法**
很有用的解决办法，直接举例说明。


```java
private void performAnimate() {
    ViewWrapper wrapper = new ViewWrapper(mButton);
    ObjectAnimator.ofInt(wrapper, "width", 500).setDuration(5000).start();
}

@Override 
public void onClick(View v) {
    if (v == mButton) {
        performAnimate();
    }
}

private static class ViewWrapper {
    private View mTarget;

    public ViewWrapper(View target) {
        mTarget = target;
    }

    public int getWidth() {

        return mTarget.getLayoutParams().width;
    }

    public void setWidth(int width) {
        mTarget.getLayoutParams().width = width;
        mTarget.requestLayout();
    }
}
```

上述代码在5s内让Button的宽度增加到了500px，为了达到这个效果，我们提供了ViewWrapper类专门用于包装View，具体到本例是包装Button。然后我们对ViewWrapper的width属性做动画，并且在setWidth方法中修改其内部的target的宽度，而target实际上就是我们包装的Button。这样一个间接属性动画就搞定了，上述代码同样适用于一个对象的其他属性。

- **采用ValueAnimator，监听动画过程，自己实现属性的改变**
首先说说什么是ValueAnimator，ValueAnimator本身不作用于任何对象，也就是说直接使用它没有任何动画效果。它可以对一个值做动画，然后我们可以监听其动画过程，在动画过程中修改我们的对象的属性值，这样也就相当于我们的对象做了动画。下面用例子来说明：


```java
private void performAnimate(final View target, final int start, final int end) {
    ValueAnimator valueAnimator = ValueAnimator.ofInt(1, 100);
    valueAnimator.addUpdateListener(new AnimatorUpdateListener() {
        // 持有一个IntEvaluator对象，方便下面估值的时候使用
        private IntEvaluator mEvaluator = new IntEvaluator();

        @Override 
        public void onAnimationUpdate(ValueAnimator animator) {
            // 获得当前动画的进度值，整型，1~100之间
            int currentValue = (Integer) animator.getAnimatedValue();

            Log.d(TAG, "current value: " + currentValue);
            // 获得当前进度占整个动画过程的比例，浮点型，0~1之间
            float fraction = animator.getAnimatedFraction();
            // 直接调用整型估值器，通过比例计算出宽度，然后再设给Button
            target.getLayoutParams().width = mEvaluator.evaluate(fraction, start, end);
            target.requestLayout();
        }
    });
    valueAnimator.setDuration(5000).start();
}

@Override 
public void onClick(View v) {
    if (v == mButton) {
        performAnimate(mButton, mButton.getWidth(), 500);
    }
}
```

上述代码的效果图和采用ViewWrapper是一样的。关于这个ValueAnimator要再说一下，拿上面的例子来说，它会在5000ms内将一个数从1变到100，然后动画的每一帧会回调onAnimationUpdate方法。在这个方法里，我们可以获取当前的值（1～100）和当前值所占的比例，我们可以计算出Button现在的宽度应该是多少。比如时间过了一半，当前值是50，比例为0.5，假设Button的起始宽度是100px，最终宽度是500px，那么Button增加的宽度也应该占总增加宽度的一半，总增加宽度是500－100=400，所以这个时候Button应该增加的宽度是400×0.5=200，那么当前Button的宽度应该为初始宽度 + 增加宽度（100+200=300）。上述计算过程很简单，其实它就是整型估值器IntEvaluator的内部实现，所以我们不用自己写了，直接用吧。


### 7.3.5 属性动画的工作原理
属性动画要求动画作用的对象提供该属性的set方法，属性动画根据你传递的该属性的初始值和最终值，以动画的效果多次去调用set方法。每次传递给set方法的值都不一样，确切来说是随着时间的推移，所传递的值越来越接近最终值。如果动画的时候没有传递初始值，那么还要提供get方法，因为系统要去获取属性的初始值。

源码分析：略，P288
**属性动画需要运行在有Looper的线程中**
calculateValue方法就是计算每帧动画所对应的属性的值

在初始化的时候，如果属性的初始值没有提供，则get方法将会被调用，请看Property-ValuesHolder的setupValue方法，可以发现get方法是通过反射来调用的，如下所示。


```java
private void setupValue(Object target, Keyframe kf) {
    if (mProperty != null) {
        Object value = convertBack(mProperty.get(target));
        kf.setValue(value);
    }
    try {
        if (mGetter == null) {
            Class targetClass = target.getClass();
            setupGetter(targetClass);
            if (mGetter == null) {
                // Already logged the error - just return to avoid NPE
                return;
            }
        }
        Object value = convertBack(mGetter.invoke(target));
        kf.setValue(value);
    } catch (InvocationTargetException e) {
        Log.e("PropertyValuesHolder", e.toString());
    } catch (IllegalAccessException e) {
        Log.e("PropertyValuesHolder", e.toString());
    }
}
```

当动画的下一帧到来的时候，PropertyValuesHolder中的setAnimatedValue方法会将新的属性值设置给对象，调用其set方法。从下面的源码可以看出，set方法也是通过反射来调用的：


```java
void setAnimatedValue(Object target) {
    if (mProperty != null) {
        mProperty.set(target, getAnimatedValue());
    }
    if (mSetter != null) {
        try {
            mTmpValueArray[0] = getAnimatedValue();
            mSetter.invoke(target, mTmpValueArray);
        } catch (InvocationTargetException e) {
            Log.e("PropertyValuesHolder", e.toString());
        } catch (IllegalAccessException e) {
            Log.e("PropertyValuesHolder", e.toString());
        }
    }
}
```

## 7.4 使用动画的注意事项
1. OOM问题
这个问题主要出现在帧动画中，当图片数量较多且图片较大时就极易出现OOM，这个在实际的开发中要尤其注意，尽量避免使用帧动画。

2. 内存泄露
在属性动画中有一类无限循环的动画，这类动画需要在Activity退出时及时停止，否则将导致Activity无法释放从而造成内存泄露，通过验证后发现View动画并不存在此问题。

3. 兼容性问题
    3.0以下系统
    
4. View动画的问题
View动画是对View的影像做动画，并不是真正地改变View的状态，因此有时候会出现动画完成后View无法隐藏的现象，即setVisibility(View.GONE)失效了，这个时候只要调用view.clearAnimation()清除View动画即可解决此问题。

5. 不要使用px
在进行动画的过程中，要尽量使用dp，使用px会导致在不同的设备上有不同的效果。

6. 动画元素的交互
将view移动（平移）后，在Android 3.0以前的系统上，不管是View动画还是属性动画，新位置均无法触发单击事件，同时，老位置仍然可以触发单击事件。尽管View已经在视觉上不存在了，将View移回原位置以后，原位置的单击事件继续生效。从3.0开始，属性动画的单击事件触发位置为移动后的位置，但是View动画仍然在原位置。

7. 硬件加速
使用动画的过程中，建议开启硬件加速，这样会提高动画的流畅性。

