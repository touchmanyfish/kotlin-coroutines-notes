# 桥接方法

```kotlin
fun main() {
    // 用接口引用指向实现类（泛型类型为 String）
    val transformer: Transformer<String> = StringTransformer()

    // 情况1：传入 String 类型，直接调用我们编写的 transform(String)
    println(transformer.transform("hello")) // 输出：HELLO

    // 情况2：通过反射强制传入 Object 类型（模拟泛型擦除后的调用）
    val method = Transformer::class.java.getMethod("transform", Any::class.java)
    val result = method.invoke(transformer, "world") // 调用桥接方法
    println(result) // 输出：WORLD
}

// 泛型接口：将输入类型 T 转换为 String
interface Transformer<T> {
    fun transform(input: T): String
}

// 实现类：指定 T 为 String 类型，转换为大写
class StringTransformer : Transformer<String> {
    // 重写方法：参数为具体类型 String
    override fun transform(input: String): String {
        return input.uppercase()
    }
}
```

```java
//反编译后java代码
public interface Transformer {
    @NotNull
    String transform(Object var1);
}

public final class StringTransformer implements Transformer {
    @NotNull
    public String transform(@NotNull String input) {
        Intrinsics.checkNotNullParameter(input, "input");
        String var10000 = input.toUpperCase(Locale.ROOT);
        Intrinsics.checkNotNullExpressionValue(var10000, "toUpperCase(...)");
        return var10000;
    }

    // $FF: synthetic method
    // $FF: bridge method
    public String transform(Object input) {
        return this.transform((String) input);
    }
}
```

### 泛型擦除的影响

在StringTransformer中期望重写Transformer中的：

```kotlin
override fun transform(input: String): String
```

但是编译时Transformer中的这个方法被擦除为:

```java
String transform(Object var1);
```

如果此时什么都不做，原本的擦除变成重载了。于是编译器生成一个桥接方法来让代“重载”关系得以保持:

```java

@NotNull
public String transform(@NotNull String input) {
    //...
    return var10000;
}

// $FF: synthetic method
// $FF: bridge method
// 这个方法编译器生成，让代“重载”关系得以保持
public String transform(Object input) {
    return this.transform((String) input);
}
```

### kotlin中编译器生成的状态机中的invoke()桥接方法

生成这个方法的原因也是如此