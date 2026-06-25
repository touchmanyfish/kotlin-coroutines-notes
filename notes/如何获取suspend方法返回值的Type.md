## 原理
通过反射method.genericReturnType()获取挂起方法的返回值类型总是“java.lang.Object”。编译器会给挂起方法的参数列表最后面自动插入一个参数：
```kotlin
kotlin.coroutines.Continuation<? super 挂起方法的返回值>

//例如返回值是String则插入
kotlin.coroutines.Continuation<? super String>
```
所以可以通过反射挂起方法的参数列表来获取返回值类型的相关信息，注意，这里的类型信息只能获取到一个范围，并不能精确获取。

## 完整例子
```kotlin
interface Api {
    suspend fun doSomething(): String
}

fun main() {
    // 反射“doSomething”方法
    val method = Api::class.java.getMethod("doSomething", Continuation::class.java)

    // 类型为“class java.lang.Object”
    val returnType = method.genericReturnType
    print(returnType)
    
    // 获取编译器自动插入的参数kotlin.coroutines.Continuation<? super java.lang.String>
    // 显然这个参数是一个ParameterizedType类型
    val continuationParamType = method.genericParameterTypes[0] as ParameterizedType
    // 获取“? super java.lang.String”
    val actualType = continuationParamType.actualTypeArguments[0] as WildcardType
    // 获取下界“String”
    val lowerBounds = actualType.lowerBounds

    print(lowerBounds)
}
```