# launch()方法block参数的receiver分析

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
)


```

### 分析
```kotlin
// ①
GlobalScope.launch {
    println("launch 111111 this=${this}")
}
```
根据block参数的类型：suspend CoroutineScope.() -> Unit可知，此时的this就是block的receiver，所以this的值是什么即可。  
  
launch()方法会先生成一个StandaloneCoroutine实例(②)，然后调用start()方法启动它。
继续执行代码代码到createCoroutineUnintercepted()方法：
```kotlin
//IntrinsicsJvm.kt
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        // ③
        create(receiver, probeCompletion)
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}
```
①处launch()方法的参数block会被编译成SuspendLambda(BaseContinualtionImpl的子类之一)，所以会进入这个if分支。  
  
根据源代码，此处的receiver的值为前面StandaloneCoroutine实例(②)，那这个receiver是否是寻找的那个receiver?  
查看①的反编译代码:
```java
public final class Coro2Kt {
    public static final void main() {
        BuildersKt.launch$default((CoroutineScope)GlobalScope.INSTANCE, (CoroutineContext)null, (CoroutineStart)null, new Function2((Continuation)null) {
            int label;
            // $FF: synthetic field
            private Object L$0;

            public final Object invokeSuspend(Object $result) {
                IntrinsicsKt.getCOROUTINE_SUSPENDED();
                switch (this.label) {
                    case 0:
                        ResultKt.throwOnFailure($result);

                        // 由这里可知this的值为L$0
                        CoroutineScope $this$launch = (CoroutineScope)this.L$0;
                        System.out.println("launch 111111 this=" + $this$launch);
                       
                        return Unit.INSTANCE;
                    default:
                        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                }
            }

            public final Continuation create(Object value, Continuation $completion) {
                Function2 var3 = new <anonymous constructor>($completion);
                // 这个create()方法就是③处调用的create()方法
                // 将第一个参数value赋值给了L$0
                
                var3.L$0 = value;
                return (Continuation)var3;
            }

           // .............
        }, 3, (Object)null);
    }
    // ...........
}

```

由反编译代码可知:
```
//赋值流程
create()参数value            -> L$0,
L$0                         -> this(要找的receiver)
```
所以现在需要找出value的值是什么，而此处的create()方法在前面③处被调用，所以value的值就是StandaloneCoroutine实例(②)。完善一下赋值流程:
```
StandaloneCoroutine实例(②)  -> create()参数value
create()参数value           ->  L$0,
L$0                        ->  this(要找的receiver)
```

所以①处launch()方法block参数的receiver为StandaloneCoroutine实例。






