# 通过startCoroutineUninterceptedOrReturn()创建并启动协程
```kotlin
public actual inline fun <T> (suspend () -> T).startCoroutineUninterceptedOrReturn(
    completion: Continuation<T>
): Any?

internal fun <T> (suspend () -> T).wrapWithContinuationImpl(
    completion: Continuation<T>
): Any?

public actual inline fun <R, T> (suspend R.() -> T).startCoroutineUninterceptedOrReturn(
    receiver: R,
    completion: Continuation<T>
): Any?

internal actual inline fun <R, P, T> (suspend R.(P) -> T).startCoroutineUninterceptedOrReturn(
    receiver: R,
    param: P,
    completion: Continuation<T>
): Any?
```
也是3个版本，不过主要流程都差不多。

这个方法有如下特点：
1. 如果可挂起计算中没有挂起点挂起，并且没有未捕获异常，方法直接返回结果,后续compeletion不会被回调
2. 如果可挂起计算中有挂起点挂起，并且没有未捕获异常，方法直接返回COROUTINE_SUSPENDED，后续compeletion会被回调
3. 如果可挂起计算中没有挂起点挂起,但是有未捕获的异常，这个异常可以被包住的trycatch块捕获，后续compeletion不会被回调
4. 如果可挂起计算中有挂起点挂起,但是在执行到第一个挂起点挂起之前有未捕获的异常，这个异常可以被包住的trycatch块捕获，后续compeletion不会被回调

TODO:为什么有这种特性的分析

注意！只有当startCoroutineUninterceptedOrReturn()直接返回COROUTINE_SUSPENDED以后，后续compeletion才会被回调！而receiver(使用带receiver的版本)则和
createCoroutineUnintercepted(receiver,completion)的receiver类似：可以在对应的可被挂起计算中被引用到。


例子参考[挂起函数方法引用](协程的启动.md#2挂起函数方法引用)

另外，如果成功的执行到了挂起点返回COROUTINE_SUSPENDED时，那么这段执行和startCoroutineUninterceptedOrReturn()调用者在同一线程。
```kotlin
val block: suspend String.() -> String = {
    suspendCoroutineUninterceptedOrReturn {
        thread(name = "thread1") {
            //thread1
            println("before resume ${Thread.currentThread().name}")
            it.resumeWith(Result.success("block111"))
        }

        // abcdef main
        // 执行到这里时为止都是和startCoroutineUninterceptedOrReturn()的调用者在同一线程
        COROUTINE_SUSPENDED
    }

    // thread1
    println("after resume ${Thread.currentThread().name}")
    "block000"
}

block.startCoroutineUninterceptedOrReturn(receiver, completion)
```

接下来只分析(suspend R.() -> T)对应的版本

### 源码分析 
```kotlin
public actual inline fun <R, T> (suspend R.() -> T).startCoroutineUninterceptedOrReturn(
    receiver: R,
    completion: Continuation<T>
): Any? =
    // Wrap with ContinuationImpl, otherwise the coroutine will not be interceptable. See KT-55869
    if (this !is BaseContinuationImpl) wrapWithContinuationImpl(receiver, completion)
    else (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, completion)
```
(suspend R.() -> T)分为3种情况:
1. 指向BaseContinuationImpl类型的状态机
2. 指向函数引用
3. 指向一个java代码实现的挂起方法

#### 指向BaseContinuationImpl类型的状态机
如果(suspend R.() -> T)指向一个BaseContinuationImpl类型的状态机，则启动流程为:
```text
stateMachine.startCoroutineUninterceptedOrReturn(receiver,completion)
       |
       V 
(stateMachine as Function2<R, Continuation<T>, Any?>).invoke(receiver, completion)
       |
       V 
stateMachine.create(receiver,completion).invokeSuspend(Result.success(Unit))       
```
可以看到实质是create()一个新的状态机然后立马调用它的invokeSuspend()方法开始执行可被挂起的计算。

为什么可以强转为Function2类型然后调用invoke/receiver如何传递给可被挂起的计算/completion如何被回调的参考[协程的启动](协程的启动.md)
```kotlin
create(receiver,completion).invokeSuspend(Result.success(Unit))
```
#### 指向函数引用
如果(suspend R.() -> T)指向一个非BaseContinuationImpl类型，则启动流程为：
```text
(this as Function2<R, Continuation<T>, Any?>).invoke(receiver, completion)
       |
       V 
可被挂起计算就被执行起来了       
```
在可挂起计算中可以引用到receiver:
```kotlin
// 在可被挂起计算中引用这个receiver
public suspend fun String.getImage(): String {
   // 通过this引用
}

// 挂起函数，有一个参数
public suspend fun getImage2(id:String): String {
    // 通过第一个参数引用
    // 这里是"id"
}

public suspend fun String.getText():String{
    // 通过this引用
}
```
如果startCoroutineUninterceptedOrReturn()返回过一次COROUTINE_SUSPENDED，后续当可被挂起计算执行完毕以后startCoroutineUninterceptedOrReturn(receiver,completion)
方法的completion参数会被调用。
````text
可被挂起计算执行完毕
       |
       V 
//包装类，可能为如下情况，不过这2种情况主流程没区别
//1.RestrictedContinuationImpl
//2.ContinuationImpl       
newCompletion.resumeWith()
       |
       V
1. probeCompletion.resumeWith() -> completion.resumeWith() // probeCoroutineCreated()返回了包装类
2. completion.resumeWith() //probeCoroutineCreated()直接返回completion     
````

#### 指向一个java代码实现的挂起方法
例子参考[java中实现的挂起方法引用](协程的启动.md#3java中实现的挂起方法引用)

