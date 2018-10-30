# 第8章 理解Window和WindowManager
> **Window表示一个窗口的概念**，Window是一个抽象类，它的具体实现是PhoneWindow。**通过WindowManager创建一个Window**。WindowManager是外界访问Window的入口，Window的具体实现位于WindowManagerService中，**WindowManager和WindowManagerService的交互是一个IPC过程**。Android中所有的视图都是通过Window来呈现的，不管是Activity、Dialog还是Toast，它们的视图实际上都是附加在Window上的，因此**Window实际是View的直接管理者**。从第4章中所讲述的View的事件分发机制也可以知道，单击事件由Window传递给DecorView，然后再由DecorView传递给我们的View，就连Activity的设置视图的方法setContentView在底层也是通过Window来完成的。

## 8.1 Window和WindowManager
如何使用WindowManager添加一个Window？

```java
mFloatingButton = new Button(this);
mFloatingButton.setText("button");
mLayoutParams =new WindowManager.LayoutParams(
LayoutParams.WRAP_CONTENT,LayoutParams.WRAP_CONTENT,0,0,PixelFormat.TRANSPARENT);
mLayoutParams.flags =LayoutParams.FLAG_NOT_TOUCH_MODAL |LayoutParams.FLAG_NOT_FOCUSABLE |LayoutParams.FLAG_SHOW_WHEN_LOCKED;
mLayoutParams.gravity =Gravity.LEFT |Gravity.TOP;
mLayoutParams.x =100;
mLayoutParams.y =300;
mWindowManager.addView(mFloatingButton,mLayoutParams);
```

上述代码可以将一个Button添加到屏幕坐标为（100，300）的位置上。

- Flags参数表示Window的属性，这些选项可以控制Window的显示特性：

    **FLAG_NOT_FOCUSABLE**
    表示Window不需要获取焦点，也不需要接收各种输入事件，此标记会同时启用FLAG_NOT_TOUCH_MODAL，最终事件会直接传递给下层的具有焦点的Window。
    
    **FLAG_NOT_TOUCH_MODAL**
    在此模式下，系统会将当前Window区域以外的单击事件传递给底层的Window，当前Window区域以内的单击事件则自己处理。这个标记很重要，一般来说都需要开启此标记，否则其他Window将无法收到单击事件。
    
    **FLAG_SHOW_WHEN_LOCKED**
    开启此模式可以让Window显示在锁屏的界面上。

- Type参数表示Window的类型，Window有三种类型，分别是应用Window、子Window和系统Window。
    应用类Window对应着一个Activity。子Window不能单独存在，它需要附属在特定的父Window之中，比如常见的一些Dialog就是一个子Window。系统Window是需要声明权限在能创建的Window，比如Toast和系统状态栏这些都是系统Window。

- Window是分层的
    每个Window都有对应的z-ordered，层级大的会覆盖在层级小的Window的上面，这和HTML中的z-index的概念是完全一致的。
    应用Window:1~99













