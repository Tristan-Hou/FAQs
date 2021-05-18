### 协程

#### 1. suspend关键字

- 被suspend关键字所修饰的函数叫挂起函数
- 挂起函数可以像普通函数一样用在协程中，不过它的一个特性在于可以使用其他挂起函数
- 挂起函数只能用在协程或者另外一个挂起函数中

```
fun main() = runBlocking {
    launch {
        world()
    }
    println("welcome")
}

suspend fun hello() {
    delay(1000)
    println("hello world")
}

/*suspend*/ fun world() {   // 编译错误，hello是suspend函数，此处必须同为suspend函数
    hello()
}
```

#### 2. 全局协程

- 全局协程类似于守护线程
- 使用GlobalScope启动的协程并不会保持进程的生命，它们就像是守护线程一样，跟随主程序一起退出

```
fun main() {
    GlobalScope.launch {
        repeat(100) {
            println("i am sleeping $it")
            delay(400)
        }
    }
    Thread.sleep(2000)
}
```

结果：
```
i am sleeping 0
i am sleeping 1
i am sleeping 2
i am sleeping 3
i am sleeping 4
```
- `创建后台协程`->`main线程sleep`->`每0.4s输出一次`->`2s后主线程退出`->`后台协程退出，不再执行`

#### 3. 协程的取消

1） kotlinx.coroutines包下的所有挂起函数都是可以取消的，它们会检查协程的取消状态，当取消时就会抛出CancellationException异常

```
fun main() = runBlocking {
    val myJob = GlobalScope.launch {
        repeat(100) {i ->
            println("hello $i")
            delay(500)
        }
    }
    delay(1100)                                         // (1)
    println("hello world")
//    myJob.cancel(CancellationException("just a try"))
//    myJob.join()
    myJob.cancelAndJoin()                               // (2)
    println("welcome")
}
```

结果：
```
hello 0
hello 1
hello 2
hello world
welcome
```
- `主线程cancel`->`协程取消`

2） 如果协程正在处于某个计算过程当中，并且没有检查取消状态，那么它就是无法被取消的

```
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime

        var i = 0
        while (i < 20) {                                           // （1）
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I am sleepint ${i++}")
                nextPrintTime += 500L
            }
        }
    }

    delay(1300)
    println("hello world")

    job.cancelAndJoin()
    println("welcome")
}
```
结果：
```
job: I am sleepint 0
job: I am sleepint 1
job: I am sleepint 2
hello world
job: I am sleepint 3

    (ignore...)

job: I am sleepint 19
welcome
```
- `main线程cancel协程`->`协程中（1）处做while循环，并未检查取消状态`->`协程未被取消`->`直到执行结束`

3） 有两种方式可以让计算代码变为可取消的

- 周期性调用一个挂起函数，改挂起函数会检查取消状态，比如使用yield函数

- 显示的检查取消状态 —— [isActive扩扎属性](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html)

```
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        (ignore...)
        while (isActive) {                                     // (1)
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I am sleepint ${i++}")
                nextPrintTime += 500L
            }
        }
    }
    (ignore...)
}
```
结果：
```
job: I am sleepint 0
job: I am sleepint 1
job: I am sleepint 2
hello world
welcome
```

- 上例中仅仅将2）的示例代码中while的判定值改为isActive，其余不变

- isActive 用于在长时间计算的逻辑里检查取消状态

- `协程后台运行`->`main取消协程`->`协程检查isActive状态，退出while循环`

4） 为何1）中的cancel可以直接取消协程，而2）就不可以，必须改为3）才能取消呢？

- 在1）的代码中，（2）处实际在内部设置了isActive的值

- 在1）的代码中，（2）处可以感知到isActive的变化（具体原理暂时未知，个人猜测通过观察者模式或者类似线程睡眠挂起和激活的过程实现）

5） 使用finally来关闭资源，join与cancelAndJoin都会等待所有清理动作完成才会继续往下执行。

- finally代码块中使用挂起函数会抛出CancellationException异常，原因是该代码块协程已经被取消了。

- 如果非要在一个已经取消的协程中使用挂起函数，可以将相应的代码块放到withContext(NonCancellable){}中，使用withContext与NonCancellable上下文

```
fun main() = runBlocking {
    val myJob = launch {
        try {
            repeat(100) { i ->
                println("job: I am sleeping ${i}")
                delay(500)
            }
        } finally {
            withContext(NonCancellable) {           // (1)
                println("run finally block")   
                delay(1000)
                println("after delay")
            }                                       // (2)
        }
    }

    delay(1300)
    println("hello world")

    myJob.cancelAndJoin()
    println("welcome")
}
```
输出：
```
job: I am sleeping 0
job: I am sleeping 1
job: I am sleeping 2
hello world
run finally block
after delay
welcome
```

- 可以将（1）、（2）注释掉，对比前后输出结果

#### 4. 协程和超时

协程的取消很多时候是由于超时。前面我们是通过引用协程返回的Job引用来取消协程的。不过Kotlin提供了内建的函数帮助我们又快又好的做到这一点:[withTimeout](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout.html) 和 [withTimeoutOrNull](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout-or-null.html)

- withTimeout: 抛出TimeoutCancellationException，可以放入try...catch块中
```
fun main() = runBlocking {
    withTimeout(1900) {
        repeat(1000) { i ->
            println("hello, $i")
            delay(400)
        }
    }
}
```
输出
```
hello, 0
hello, 1
hello, 2
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1900 ms
(ignore...)
```
- withTimeoutOrNull: 类似于withTimeout，不过超时时直接返回null

```
fun main() = runBlocking {
    val result = withTimeoutOrNull(1900) {
        repeat(1000) { i ->
            println("hello, $i")
            delay(900)
        }
        "hello world"
    }
    println("result is $result")
}
```
输出
```
hello, 0
hello, 1
hello, 2
result is null
```







