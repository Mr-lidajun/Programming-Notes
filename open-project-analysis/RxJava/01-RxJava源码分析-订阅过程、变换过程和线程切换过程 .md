# 第1章 RxJava源码解析
## 1.1 RxJava的订阅过程
RxJava基本使用方法：
```java
Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override public void call(Subscriber<? super Integer> subscriber) {

    }
}).subscribe(new Subscriber<Integer>() {
    @Override public void onCompleted() {

    }

    @Override public void onError(Throwable e) {

    }

    @Override public void onNext(Integer integer) {

    }
});
```

之后查看Observable的create方法做了什么：
```java
public static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(RxJavaHooks.onCreate(f));
    }
```

继续查看RxJavaHooks.onCreate(f)方法：
```java
public static <T> Observable.OnSubscribe<T> onCreate(Observable.OnSubscribe<T> onSubscribe) {
    Func1<Observable.OnSubscribe, Observable.OnSubscribe> f = onObservableCreate;
    if (f != null) {
        return f.call(onSubscribe);
    }
    return onSubscribe;
}
```

继续查看onObservableCreate变量：
```java
onObservableCreate = new Func1<Observable.OnSubscribe, Observable.OnSubscribe>() {
    @Override
    public Observable.OnSubscribe call(Observable.OnSubscribe f) {
        return RxJavaPlugins.getInstance().getObservableExecutionHook().onCreate(f);
    }
};
```

继续查看RxJavaPlugins.getInstance().getObservableExecutionHook().onCreate(f);方法：
```java
public <T> OnSubscribe<T> onCreate(OnSubscribe<T> f) {
    return f;
}
```

在此可以看出，create方法主要就是创建了Observable类，其中，hook指的是RxJava ObservableExecutionHook。它的onCreate方法如上所示：
RxJavaObservableExecutionHook 的 onCreate 方法并没有做什么，只是返回了传进来的OnSubscribe。


现在来查看Observable的**subscribe**方法做了什么，如下所示：
```java
static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
    ...........
    subscriber.onStart();
    if (!(subscriber instanceof SafeSubscriber)) {
        subscriber = new SafeSubscriber<T>(subscriber); //1
    }
    try {
        RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);//2
        return RxJavaHooks.onObservableReturn(subscriber);
    } catch (Throwable e) {
        .........

        return Subscriptions.unsubscribed();
    }
}
```

在上面代码注释1处，如果传进来的Subscriber不是SafeSubscriber类型的，则将它封装成SafeSubscriber。SafeSubscriber 继承自 Subscriber，它在 Subscriber 的基础上添加了更加完善的处理。比如在onCompleted和onError方法调用时不会再调用onNext方法，并且保证onCompleted和onError方法只有一个被执行。在注释2处，onSubscribeStart方法如下所示：
```java
public static <T> Observable.OnSubscribe<T> onObservableStart(Observable<T> instance, Observable.OnSubscribe<T> onSubscribe) {
    Func2<Observable, Observable.OnSubscribe, Observable.OnSubscribe> f = onObservableStart;
    if (f != null) {
        return f.call(instance, onSubscribe);
    }
    return onSubscribe;
}
```

其中onObservableStart变量如下：
```java
onObservableStart = new Func2<Observable, Observable.OnSubscribe, Observable.OnSubscribe>() {
    @Override
    public Observable.OnSubscribe call(Observable t1, Observable.OnSubscribe t2) {
        return RxJavaPlugins.getInstance().getObservableExecutionHook().onSubscribeStart(t1, t2);
    }
};
```

它会返回OnSubscribe。因此，注释2处就是调用OnSubscribe.call（subscriber）来完成订阅的。