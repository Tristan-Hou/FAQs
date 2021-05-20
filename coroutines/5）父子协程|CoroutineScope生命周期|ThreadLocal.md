### 协程 —— 父子协程|CoroutineScope生命周期|ThreadLocal

#### 1. 协程上下文和Job

协程的Job是归属于其上下文（Context）的一部分，Kotlin为我们提供了一种简洁的手段来通过协程上下文获取到协程自身的Job对象

```
//-Dkotlinx.coroutines.debug

fun main() = runBlocking<Unit> {
    val job : Job? = coroutineContext[Job]
    println(job)
}
```

结果
```
"coroutine#1":BlockingCoroutine{Active}@23ab930d
```

- `runBlocking`的block参数是`suspend CoroutineScope.() -> T`类型，而`coroutineContext`是`CoroutineScope`的内置属性

- `coroutineContext[Job]`以表达式形式访问上下文中的Job对象（`CoroutineContext.get()`函数通过operator和泛型重载了get()，而`Job`中也定义了伴生对象`Key`以实现这种功能）

- `BlockingCoroutine`是Builders.kt中的实现，它是在runBlocking中生成的，最终帮助代码生成协程并运行代码块中逻辑。

**********************************

#### 2. 父子协程关系

- 当一个协程是通过另外一个协程启动的，那么这个协程就会通过CoroutineScope.coroutineContext来继承其上下文信息

- 新协程的Job就会成为父协程Job的一个孩子（子协程）

- 当父协程被取消时，该父协程的所有孩子都会通过递归的方式一并取消执行

- 特例：当我们使用GlobalScope来启动协程时，对于启动的新协程来说，其Job是没有父Job的。因此它就不会绑定到其所启动的那个范围上，所以它可以独立执行

- 对于父子协程来说，父协程总是会等待期所有子协程的完成。

- 对于父协程来说，它不必显示地去追踪由它所自动的所有子协程，同时也不必调用Job.join方法来等待子协程的完成


```
fun main() = runBlocking {
    val request = launch {                // (0)
        GlobalScope.launch {              // (1)
            println("job1: hello")        // (3)
            delay(1000)
            println("job1: world")        // (4)
        }
        launch {                          // (2)
            delay(100)
            println("job2: hello")        // (5)
            delay(1000)
            println("job2: world")        // (6)
        }
    }

    delay(500)
    request.cancel()                      // (7)

    delay(1000)
    println("welcome")
}
```
输出
```
job1: hello
job2: hello
job1: world
welcome
```

- 代码(0)处创建了一个协程，代码(2)创建了(0)协程的子协程，而代码(1)是通过GlobalScope启动的，因此它与(0)协程没有什么关系，可视为独立协程

- 因此由于delay时间存在，(2)协程被取消，代码(6)并没有被执行，而协程(1)由于不受外层协程控制，(3)和(4)都会执行

```
fun main() = runBlocking {
    val request = launch {
        repeat(3) {
            launch {
                delay((it + 1) * 100L)
                println("Coroutine $it finish")
            }
        }
        println("hello")
    }

    request.join()                                  // (1)
    println("world")
}
```
输出
```
hello
Coroutine 0 finish
Coroutine 1 finish
Coroutine 2 finish
world
```

- 父协程request里面创建了3个子协程，由于调用了request.join父协程等待所有子协程完成之后才会继续执行，输出world

- 如果注释掉代码(1)，会发生什么？


**********************************

#### 3. CoroutineScope生命周期

- 当对象到达生命周期被销毁时，与该对象相关的协程会自动销毁

```
class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {     // 委托CoroutineScope()工厂函数
    fun destroy() {
        cancel()
    }

    fun doSomething() {
        repeat(8) {
            launch {
                delay((it + 1) * 300L)
                println("Coroutine $it is finished")
            }
        }
    }
}

fun main() = runBlocking {
    val activity = Activity()
    activity.doSomething()

    println("start coroutines")
    delay(1300)

    println("destroy activity")
    activity.destroy()

    delay(5000)
}
```

结果
```
start coroutines
Coroutine 0 is finished
Coroutine 1 is finished
Coroutine 2 is finished
Coroutine 3 is finished
destroy activity
```

- 通过`by`将CoroutineScope的创建委托给CoroutineScope()工厂

- [CoroutineScope(context: CoroutineContext)](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html)是工厂函数，用于生成CoroutineScope实例

- 调用cancel之后，所有协程被取消，因此没有后续输出（本来有8个协程，最终只输出4行）

**********************************

#### 3. ThreadLocal

- 协程可以在不同的线程中切换，而ThreadLocal在每个线程中保存不同的值，因此协程在不同线程中切换时可以任意使用当前线程里ThreadLocal对应的不用值

```
val threadLocal = ThreadLocal<String>()

fun main() = runBlocking<Unit> {                                                                                        // (1)
    threadLocal.set("hello")
    println("pre main, current thread: ${Thread.currentThread()}, thread local value: ${threadLocal.get()}")            // (2)

    val  job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "world")) {                            // (3)
        println("launch start, current thread: ${Thread.currentThread()}, thread local value: ${threadLocal.get()}")    // (4)
        yield()
        println("after yield, current thread: ${Thread.currentThread()}, thread local value: ${threadLocal.get()}")     // (5)
    }
    job.join()                                                                                                          // (6)

    println("pre main, current thread: ${Thread.currentThread()}, thread local value: ${threadLocal.get()}")             // (7)
}
```
输出
```
pre main, current thread: Thread[main @coroutine#1,5,main], thread local value: hello
launch start, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: world
after yield, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: world
pre main, current thread: Thread[main @coroutine#1,5,main], thread local value: hello
```
- (1)处创建协程，其运行线程为main，如(2)处输出显示

- (3)处通过launch的Dispatchers.Default参数创建了一个运行于默认线程池里某一个线程的新协程，[asContextElement()](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/java.lang.-thread-local/as-context-element.html)是ThreadLocal的拓展函数，在此功能为用value = ”world“参数将threadLocal变量在新线程里保存的值赋值为value，因此(4)和(5)处输出值为value

- (7)处代码又会到了main线程，因此此处threadLocal.get()的值为hello







