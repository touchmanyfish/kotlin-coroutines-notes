# kotlin中调用java中实现的挂起函数

### 挂起方法编译以后反编译成的java代码特点
```kotlin
// kotlin代码
suspend fun show0(): Int {}
suspend fun show1(name: String): String {}
suspend fun String.showR(label: String): Float {}
```
```java
// 反编译以后的java代码
public static final Object show0(Continuation $completion) {}
public static final Object show1(String name,  Continuation $completion) {}
public static final Object showR(String $this$showR,  String label,  Continuation $completion) {}
```
观察上述代码发现:
1. 挂起函数反编译成java代码以后的返回值类型总是“Object”
2. 函数参数一一对应，但是编译器自动在参数列表最后插入“Continuation $completion”
3. 如果挂起方法为扩展函数，则第一个参数对应receiver

### 在java中实现挂起函数
如果需要实现某个挂起函数类型：
1. 只需将这个挂起函数类型对应函数反编译成的java代码复制过来删掉方法体
2. 如果需要挂起当前执行则返回CoroutineSingletons.COROUTINE_SUSPENDED，然后在后续调用completion.resumeWith()方法恢复执行。
3. 如果不需要挂起当前执行则直接返回结果

### 将java中的实现赋值给kotlin中的挂起函数类型变量
```kotlin
val showRRef: suspend String.() -> Float =
        SuspendFromJava::showR as suspend String.() -> Float
```
1. 使用“::”操作符引用java方法
2. 使用as suspend String.() -> Float转换对应类型
3. 赋值

```java
// java中实现
public class SuspendFromJava {

    @Nullable
    public static Object show0(Continuation continuation) {
        new Thread(() -> {
            // 在子线程上恢复协程
            continuation.resumeWith(100000000);
        }).start();

        // 挂起协程
        return CoroutineSingletons.COROUTINE_SUSPENDED;
    }

    public static Object show1(String name, Continuation continuation) {
        new Thread(() -> {
            // 有异常产生，在子线程上恢复协程
            // 在java中如何创建success/failure的Result
            // 估计可以先用kt将Result包装一下然后再在java中调用
            Result.Failure failure = new Result.Failure(new RuntimeException());
            continuation.resumeWith(failure);
        }).start();

        // 挂起协程
        return CoroutineSingletons.COROUTINE_SUSPENDED;
    }

    public static Object showR(String name, Continuation continuation) {
        // 不挂起直接返回结果
        return 55555F;
    }

}
```

```kotlin
// kotlin中调用
private fun invokeJavaSuspendCase() {
    // 调用java中的show0实现
    val show0Ref: suspend () -> Int =
        SuspendFromJava::show0 as suspend () -> Int

    // 调用java中的show1实现
    val show1Ref: suspend (String) -> String =
        SuspendFromJava::show1 as suspend (String) -> String

    // 调用java中的showR实现
    val showRRef: suspend String.() -> Float =
        SuspendFromJava::showR as suspend String.() -> Float

    runBlocking {
        println("show0 ${show0Ref()}")
        println("show1 ${show1Ref("tom")}")
        println("showR ${"label".showRRef()}")
    }

}
```
### resumeWith()问题

就算在show0()方法中尝试调用:
```java
completion.resumeWith("string");
```
恢复执行，以下代码也不会报错：
```kotlin
val show0Ref: suspend () -> Int =
        SuspendFromJava::show0 as suspend () -> Int
```
这种情况下调用resumeWith()时传递的值已经和show0Ref函数类型标识的返回类型不匹配了
