# createCoroutineUnintercepted()

## 先调用createCoroutineUnintercepted()创建，然后在启动
```kotlin
// 带receiver
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
)

// 不带receiver
public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
)
```
两个方法唯一的区别就是receiver，其他都是一样的，接下来只分析带receiver版本。  

每次调用这个方法的时候，它返回一个Continuation<Unit>类型，它一个全新的状态机，后续调用它的continuation.resumeWith(Unit)方法可以让(suspend R.() -> T)对应的可被挂起的
计算执行起来。当可被挂起计算执行完毕以后：
````text
执行完毕表示：
1. 正常执行完毕
2. 抛出了未捕获异常
3. 执行过程中拿到后续的续体，但是恢复续体时调用resumeWith(Result.failure(exception))
````
参数completion的resumeWith()方法会被回调。receiver参数则会被不同方式传递给这个可被挂起的计算，好让它在其中可以被使用。  

我们可以直接调用createCoroutineUnintercepted()方法返回的Continuation<Unit>类型也可以间接调用它

### 源码分析
```kotlin
// IntrinsicsJvm.kt
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(receiver, probeCompletion)
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}
```
```kotlin
val probeCompletion = probeCoroutineCreated(completion)
```
probeCompletion有两种情况：
1. probeCompletion === completion
2. Continuation<T>类型的包装类，当调用它的resumeWith()方法时，后续会调用到completion.resumeWith()  

probeCoroutineCreated()主要起一个探测作用，不影响主要逻辑

(suspend R.() -> T)分为3种情况:
1. 指向BaseContinuationImpl类型的状态机
2. 指向函数引用
3. 指向一个java代码实现的挂起方法

createCoroutineUnintercepted()方法内部会将1和2/3分开处理
#### 指向BaseContinuationImpl类型的状态机
```kotlin
// block在运行时指向一个BaseContinuationImpl类型的状态机
val block: suspend String.() -> String = {
    println("run block000")
    "block000"
}
```
这种情况直接通过create(receiver, probeCompletion)方法创建一个状态机并返回
```text
//可被挂起计算被调用的路径

// 直接或者间接调用       
impl.resumeWith(Result.success(Unit))
       |
       V
impl.invokeSuspend(Result.success(Unit))     
       |
       V
(suspend R.() -> T)对应的可被挂起的计算         
```
查看反编译后的字节码，create(receiver, probeCompletion)中receiver参数可以在lambda中通过this关键字被引用到。
```text
//reciver的赋值流程

//1 创建状态机时
createCoroutineUnintercepted(receiver,competion)
       |
       V
create(receiver,probeCompletion) // 返回stateMachine  
       |
       V
stateMachine.L$0 = receiver // 保存receiver到编译器生成的字段中

//2. 执行可被挂起计算时
stateMachine.resumeWith(Result.success(Unit))
       |
       V
stateMachine.invokeSuspend(Result.success(Unit))
       |
       V
String var2 = (String)this.L$0; // 通过L$0引用receiver
System.out.println("receiver " + var2); //  对应编译前的代码：println("run block000")            
```
接下来看看completion参数如何被使用
```text
//1 在创建状态机时保存到BaseContinuationImpl.completion字段中

val probeCompletion = probeCoroutineCreated(completion)     
//  1. probeCompletion === completion
//  2. probeCompletion包装completion
       |
       V 
createCoroutineUnintercepted(receiver,completion)
       |
       V     
create(receiver,probeCompletion)
       |
       V
状态机实际类型(value:Continuation)
       |
       V
// 因为看不到代码所以不确定，调用哪个
// 不过如果probeCompletion不为空则一定调用的是第一个
Suspendlambda(arity,probeCompletion)或者(arity)
       |
       V    
ContinuationImpl(probeCompletion,_context)          
       |
       V 
BaseContinuationImpl(probeCompletion)    
       |
       V    
create(receiver,probeCompletion)执行完毕，返回状态机stateMachine

//2 当可被挂起计算执行完毕时，调用它的resumeWith()方法的调用路径 
可被挂起计算执行完毕
       |
       V   
stateMachine.completion.resumeWith()
// stateMachine.completion实际是probeCompletion
       |                       
       V                      
1.probeCompletion包装了completion，后续调用completion.resumeWith()
2.probeCompletion === completion,completion.resumeWith()被直接调用到
```

#### 指向函数引用
```kotlin
@OptIn(InternalCoroutinesApi::class)
fun main() {
    val block1: suspend String.() -> String = String::getImage
    val block2: suspend String.() -> String = String::getText

    // ...
}

// 挂起函数,带receiver
public suspend fun String.getImage(): String {
    return suspendCoroutineUninterceptedOrReturn {
        thread {
            it.resumeWith(Result.success("getImage000"))
        }
        COROUTINE_SUSPENDED
    }
}

// 挂起函数，有一个参数
public suspend fun getImage2(id:String): String {
    return suspendCoroutineUninterceptedOrReturn {
        thread {
            it.resumeWith(Result.success("getImage000"))
        }
        COROUTINE_SUSPENDED
    }
}

// 非挂起函数
public suspend fun String.getText():String{
    return "getText000"
}
```
如果(suspend R.() -> T)不是BaseContinuationImpl类型，则会创建一个ContinuationImpl的包装类，在这个包装类的invokeSuspend()方法中将(suspend R.() -> T)
强转为Function2<R,Continuation<T>,Any?>然后调用它的invoke()方法，开始执行(suspend R.() -> T)对应的可被挂起的计算。
```text
// 可被挂起计算被调用的流程

// 直接或者间接调用       
包装类.resumeWith(Result.success(Unit))
       |
       V
包装类.invokeSuspend(result)     
       |
       V
Function2<R,Continuation<T>,Any?>.invoke(result) 
       |
       V     
(suspend R.() -> T)对应的可被挂起的计算 
```
根据compeltion.context会选择2个包装类中的一个，这2个包装类的流程大致一样，ai说是性能优化。  

createCoroutineUnintercepted(receiver,completion)中的receiver参数回先被捕获到闭包中，然后在调用Function2.invoke(receiver,completion)
时传递给可被挂起计算
```text
// 1. receiver的捕获
createCoroutineUnintercepted(receiver,completion)
       |
       V 
receiver被捕获到Function2.invoke()所在的闭包中  

// 2. 将receiver传递给可挂起计算
 包装类.invokeSuspend(result)
       |
       V 
捕获receiver的闭包(result)
       |
       V
(this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
       |
       V
    ??????
       |
       V
可被挂起计算中可以引用到这个receiver                                          
```
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
其中"??????"部分不知道发生了什么。接下来看看completion参数：
```text
// completion的赋值

//1 在创建状态机时保存到BaseContinuationImpl.completion字段中

val probeCompletion = probeCoroutineCreated(completion)     
//  1. probeCompletion === completion
//  2. probeCompletion包装completion
       |
       V 
createCoroutineFromSuspendFunction(probeCompletion,block)
       |
       V     
// 或者ContinuationImpl(probeCompletion)       
RestrictedContinuationImpl(probeCompletion)
       |
       V
// 赋值给了completion字段       
BaseContinuationImpl(probeCompletion)
```    
createCoroutineUnintercepted(receiver,completion)的completion被被直接或者包装成probeCompletion赋值给包装类的completion字段。

````text
// 调用到completion.resumeWith()的路径

可被挂起计算执行完毕
       |
       V
包装类.compeletion.resumeWith()
//如果probeCoroutineCreated()直接返回的completion
//这里直接就调用createCoroutineUnintercepted()的compltion的resumeWith()方法
       |
       V      
completion.resumeWith()      
````
当可被挂起计算执行完毕，回调用包装类.compeletion.resumeWith(),这回让createCoroutineUnintercepted()的compltion的resumeWith()方法被直接
或者间接的调用到。  

为什么可以强转为Function2<R,Continuation<T>,Any?>呢？可以通过block::class.java反射来查看实际类型，目前可以推测到如下程度：
```kotlin
// 泛型参数T目前不知道怎么推测
Function2<R,Continuation,Any?>
```
  
为什么这种情况下需要将(suspend R.() -> T)包装到一个状态机呢？因为kotlin框架内部实现了续体的拦截机制：
```kotlin
public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this
```
显然这种拦截机制的前提是续体必须是ContinuationImpl子类型，但是此时(suspend R.() -> T)指向的实际类型并不是ContinuationImpl子类型(通过反射得知)所以为了让这种情况下
它对应的可挂起计算中也没有办法能拿到支持挂起的ContinuationImpl子类型续体，于是就将它包装到了一个状态机中:
```kotlin
public suspend fun String.getImage3(): String {
    return suspendCoroutineUninterceptedOrReturn {
        // 
        val intercepted = it.intercepted()
        
        "getImage3333"
    }
}
```
你也可以直接将一个ContinuationImpl子类型的续体直接传递给Function2<String,Continuation<String>,Any?>.invke(receiver,completion)，
这样对应可被挂起计算中就能拿到这个子类型来进行拦截了。

#### 指向一个java代码实现的挂起方法
例子参考[java中实现的挂起方法引用](协程的启动.md#3java中实现的挂起方法引用)

### 启动

````kotlin
import kotlin.coroutines.intrinsics.createCoroutineUnintercepted
import kotlin.coroutines.intrinsics.intercepted

fun main() {
    val block: suspend String.() -> String = {
        println("run block000")
        "block000"
    }

    val receiver = "block000receiver"
    val completion = object : Continuation<String> {
        override val context: CoroutineContext
            get() = EmptyCoroutineContext
            // 如果要使用intercepted()方法
            // get() = Dispatchers.IO

        override fun resumeWith(result: Result<String>) {
            println("completion resule $result")
        }

    }

    val coroutine = block.createCoroutineUnintercepted(receiver, completion)

    // 直接调用
    // coroutine.resumeWith(Result.success(Unit))

    // 间接调用例子
    coroutine.intercepted().resumeCancellableWith(Result.success(Unit))

    while (true) {
    }
}
````









