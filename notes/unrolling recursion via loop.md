# unrolling recursion via loop

```kotlin
// BaseContinuationImpl.kt
public final override fun resumeWith(result: Result<Any?>) {
    // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
    var current = this
    var param = result
    while (true) {
        // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
        // can precisely track what part of suspended callstack was already resumed
        probeCoroutineResumed(current)
        with(current) {
            val completion = completion!! // fail fast when trying to resume continuation without completion
            val outcome: Result<Any?> =
                try {
                    val outcome = invokeSuspend(param)
                    if (outcome === COROUTINE_SUSPENDED) return
                    Result.success(outcome)
                } catch (exception: Throwable) {
                    Result.failure(exception)
                }
            releaseIntercepted() // this state machine instance is terminating
           
            // A1,这里为什么要这样？
            if (completion is BaseContinuationImpl) {
                // unrolling recursion via loop
                current = completion
                param = outcome
            } else {
                // top-level completion reached -- invoke and return
                completion.resumeWith(outcome)
                return
            }
        }
    }
}
```
### 作用
将递归调用转成循环避免深度递归导致的栈溢出或不必要的开销。

### 原理
构造一个状态机分布：
```text
// C表示completion
// X，Y，Z为BaseContinuationImpl子类的状态机
// top completion为非BaseContinuationImpl子类

top completion
         <--C-- X
               <--C-- Y
                     <--C-- Z
```
假设没有这种优化，则当Z.resumeWith()被调用，依次恢复状态机执行，直到(top completion).resumeWith()被调用时，调用栈为：
```text
Z.resumeWith() 
    --> Y.resumeWith() 
    --> X.resumeWith() 
    --> top completion.resumeWith() 
```
可以看到，随着状态机的增多，调用栈会越来越深，性能消耗越来越大！  

启用优化，当Z.resumeWith()被调用时后续执行情况为：

```text
// 调用栈：Z.resumeWith() --> Z.invokeSuspend()
1. 执行Z.invokeSuspend()

// 调用栈：Z.resumeWith() 
2. 进入“unrolling ”分支，局部变量completion赋值为Y
  
// 调用栈：Z.resumeWith() --> Y.invokeSuspend()
3. 执行Y.invokeSuspend()

// 调用栈：Z.resumeWith() 
4. 进入“unrolling ”分支，局部变量completion赋值为X 
  
// 调用栈：Z.resumeWith() --> X.invokeSuspend()
5. 执行X.invokeSuspend()

// 调用栈：Z.resumeWith() --> (top completion).reumeWith() 
6. 进入“top completion”分支，调用(top completion).reumeWith()
```
可以看到不管有多少状态机，相比没有优化调用栈大大减少。



