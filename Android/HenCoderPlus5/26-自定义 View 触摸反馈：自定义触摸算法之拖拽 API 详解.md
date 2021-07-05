## 自定义触摸算法之拖拽 API 详解

### OnDragListener

* 通过 startDrag() 来启动拖拽
* 用 setOnDragListener() 来监听
  * OnDragListener 内部只有一个方法：onDrag()
  * onDragEvent() 方法也会收到拖拽回调（界面中的每个 View 都会收到）

### ViewDragHelper

* 需要创建一个 ViewDragHelper 和 Callback()
* 需要写在 ViewGroup 里面，重写 onIntercept() 和 onTouchevent()

### 为什么要这两个东西，而不是一个？

* OnDragListener
  * API 11 加入的工具类，用于拖拽操作。
  * 使用场景：用户的「拖起 -> 放下」操作，重在内容的移动。可以附加拖拽数据
  * 不需要写自定义 View，使用 startDrag() / startDragAndDrop() 手动开启拖拽
  * 拖拽的原理是创造出一个图像在屏幕的最上层，用户的手指拖着图像移动
* ViewDragHelper
  * 2015 年的 support v4 包中新增的工具类，用于拖拽操作。
  * 使用场景：用户拖动 ViewGroup 中的某个子 View
  * 需要应用在自定义 ViewGroup 中调用 ViewDragHelper.shouldInterceptTouchEvent() 和 processTouchEvent()，程序会自动开启拖拽
  * 拖拽的原理是实时修改被拖拽的子 View 的 mLeft, mTop, mRight, mBottom 值

