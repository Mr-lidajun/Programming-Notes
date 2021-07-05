## 多点触控的三种类型

* 接力型

* 协作型/配合型

* 各自为战型，互不干扰的 - 画板

### 前置知识

每一个事件序列是针对View的。对于同一个View的多个手指触摸，这些事件会在同一个序列里面

MotionEvent.getX()、getY()不是属于一个事件序列，也不是属于一个事件，而是属于一个Pointer(手指)

```kotlin
ACTION_DOWN p(x, y, index, id)
ACTION_MOVE p(x, y, index, id)
ACTION_MOVE p(x, y, index, id)
ACTION_MOVE p(x, y, index, id)
ACTION_POINTER_DOWN p(x, y, index, id) p(x, y, index, id)
ACTION_MOVE p(x, y, index, id) p(x, y, index, id)
ACTION_MOVE p(x, y, index, id) p(x, y, index, id)
ACTION_POINTER_DOWN
ACTION_MOVE
ACTION_POINTER_UP
ACTION_MOVE
ACTION_POINTER_UP
ACTION_MOVE
ACTION_UP
```

index: 哪个手指，每一次事件当中，可能是会变化的 - 用来做遍历

id: 手指按下时，指派的id，只要手指不抬起，id就不会改变 - 用来做追踪

findPointerIndex(id):  得到这个id所对应的index在此次事件中的index

获取某个id所对应的Pointer的坐标：

```kotlin
event.getX(event.findPointerIndex(downId))
```

**总结：**

**触摸事件是针对整个View的**

getX()、getY()，不是取的产生事件的手指的坐标，而是`index`为0的手指的坐标。getX()等价于getX(0)，getY()等价于getY(0)

对于多点触摸来说，getX()、getY()这两个方法是废的，没价值

如果是多点触摸中，想获取某个手指的坐标，应该使用带参数的版本：getX(index), getY(index)

如果想获取落下的那根手指（导致事件发生的那根手指）的坐标，或者是手指移动的坐标，导致这次move事件发生的手指的坐标，分两种情况来看：

* 第1种：ACTION_POINTER_DOWN、ACTION_POINTER_UP

  * 第1根手指按下，第1根手指抬起，直接使用getX()和getY()即可

  * 非第1根手指按下，非最后一根手指抬起，这种情况有多个pointer，需要想办法拿到按下和抬起的那个pointer的index。使用 `event.getActionIndex()`，通过 `event.getX(event.getActionIndex())` 就能获取到正在按下(刚刚按下)的手指的坐标，也能获取到刚刚抬起的那根手指的坐标。

* 第2种：ACTION_MOVE

  * 两根手指都放在界面上，先滑动左手手指，然后滑动右手手指，他们每时每刻都在一起移动。对于移动事件，所谓的正在移动的手指，或者导致`ACTION_MOVE`发生的那根手指，这些概念都是无意义的，因为所有手指都在不停的移动，所有手指都在导致`ACTION_MOVE`的发生，所以对于 `ACTION_MOVE` 来说，并没有所谓的 ACTION_POINTER，所以也没 `actionIndex`，该方法在ACTION_MOVE中调用总是会返回 0，并不是代表总数index为0的手指在移动，而是因为并没有所谓的唯一在移动的手指。`actionIndex`只有在ACTION_POINTER_DOWN、ACTION_POINTER_UP中才有意义。