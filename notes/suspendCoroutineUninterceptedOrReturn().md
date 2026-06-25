# suspendCoroutineUninterceptedOrReturn()

```kotlin
public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(crossinline block: (Continuation<T>) -> Any?): T {
// ...
}

```

### 背景
当协程挂起以后，如果要恢复执行，需要拿到后续执行对应的续体。

### 作用
参考suspendCoroutineUninterceptedOrReturn()文档

### 原理
suspendCoroutineUninterceptedOrReturn()方法的调用处编译器会：
```text
1. 将block中的代码inline到调用处
2. 将当前续体赋值给block中所有用到它的地方，此时block内的代码就拿到了续体，如果block中的代码返回COROUTINE_SUSPENDED挂起以后，后续可以通
过调用这个续体的resume()系列方法
来恢复后续的执行
3. 检查block返回值，如果是COROUTINE_SUSPENDED则调用：DebugProbesKt.probeCoroutineSuspended($continuation)
```
需要说明的是suspendCoroutineUninterceptedOrReturn()不同的调用位置拿到的续体类型也不一样：
```text
1.在带suspend关键字的lambda中执行，则block中代码直接拿到这个lambda
2.如果在非inline方法末尾执行，则拿到编译器在编译时期插入到挂起方法末尾的续体
3.如果如果在非inline方法非末尾执行，这个挂起方法末尾有其他有效调用，编译器会在编译时期插在挂起方法内部插入一段生成ContinuationImpl续体的代码。
  在block中可以拿到这个续体。
```
### 例子
```kotlin
private suspend fun like() {
    suspendCoroutineUninterceptedOrReturn<String> {
        thread {
            it.resume("hah")
        }
        COROUTINE_SUSPENDED
    }
}
```
```java
private static final Object like(final Continuation $completion) {
    int var2 = 0;
    ThreadsKt.thread$default(false, false, (ClassLoader) null, (String) null, 0, new Function0() {
        public final void invoke() {
            Result.Companion var10001 = Result.Companion;
            // suspendCoroutineUninterceptedOrReturn()在末尾调用，挂起like()方法内部不生成创建续体的代码
            // 直接使用传递给挂起方法的续体 
            $completion.resumeWith(Result.constructor - impl("hah"));
        }

        public Object invoke() {
            this.invoke();
            return Unit.INSTANCE;
        }
    }, 31, (Object) null);

    // 检查返回值
    Object var10000 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
    if (var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED()) {
        // 挂起则调用
        DebugProbesKt.probeCoroutineSuspended($completion);
    }

    return var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED() ? var10000 : Unit.INSTANCE;
}
```

当在挂起lambda内部调用suspendCoroutineUninterceptedOrReturn()时的情况类似。

