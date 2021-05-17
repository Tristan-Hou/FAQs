### 协程

#### 1. 什么是协程

- 将复杂性放入库中解决复杂性，程序的逻辑顺序表达，由底层库为我们解决异步性

- 该库可将用户代码相关部分包装为回调、订阅事件、不同线程（或机器）上调度执行，而代码则保持如同顺序执行一样简单

- 协程`像`是非常轻量级的线程（仅仅是像）

- Kotlin使用协程需要引入coroutines-core包

#### 2. 协程的重要概念

- CoroutineScope: 协程本身，包含CoroutineContext

- CoroutineContext: 协程上下文，包含job、coroutineDispatcher，代表一个协程场景

- EmptyCoroutineContext：空的协程上下文

- CoroutineDispatcher：分配器（调度器），决定了协程所在的线程或线程池，它可以指定协程运行于特定的一个线程、线程池或者不指定任何线程（此时允许于当前线程）

- Job：任务，封装了协程中需要执行的代码和逻辑，可以取消，并且有简单的生命周期（new-active-completing-canclling-canclled-completed)完成时没有返回值

- Deferred：Jod的子类，有返回值（`public interface Deferred<out T> : Job`)

- CoroutineScope.launch：属于协程构建起coroutine builders，是一个创建协程的函数

- CoroutineScope.launch{}：最常用的协程构造器，不阻塞当前线程，后台创建新协程

- runBlocking{}：创建新协程，同时阻塞当前线程，直到协程结束（主要用于test）

- withContext()：不创建新协程，在指定协程上运行挂起的代码块，并挂起该协程知道代码块运行结束

- CoroutineScope.Async{}：效果与CoroutineScope.launch{}一样，区别是有返回值的（返回类型为deferred）

#### 3. 协程构建器

- 本节内容最好复制到自己在编译器中亲自观察运行结果

1）`GlobalScope.launch`: 实现了[CoroutineScope接口](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html)，不阻塞当前线程（上例main线程），并创建新协程（launch后代码块）。

```
①
fun main() {
    GlobalScope.launch {
        delay(1000)
        println("kotlin coroutine")
    }
    println("hello")
    Thread.sleep(2000)
    println("world")
}
```
输出：
```
hello
kotlin coroutine
world
```
- 类似于线程执行逻辑:`GlobalScope.launch创建协程`->`delay`->`main线程输出hello`->`sleep 2s`->`1s后协程时间执行kotlin coroutine`->`2s后main打印world`

```
②
fun main() {
    GlobalScope.launch {
        delay(1000)
        println("kotlin coroutine")
    }
    println("hello")
    Thread.sleep(500)  // 时间由2000改为500
    println("world")
}
```
输出：
```
hello
world
```
- `GlobalScope.launch创建协程`->`delay`->`main线程输出hello`->`sleep 0.5s`->`main打印world`->`主线程结束，协程亦停止`

2） kotlin中的[`thread`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.concurrent/thread.html)：封装了java的Thread类
```
fun main() {
    thread {
        Thread.sleep(1000)
        println("kotlin coroutine")
    }
    println("hello")
    Thread.sleep(2000)
    println("world")
}
```
执行结果同1）①

- 本质就是创建了一个子线程

3） `runBlocking`: 创建新协程，同时阻塞当前线程直到runBlocking创建的协程结束
```
fun main() {
    GlobalScope.launch {
        delay(1000)
        println("Kotlin Coroutines")
    }
    println("hello")
    runBlocking {
        delay(2000)
    }
    println("world")
}
```
输出同1）①

- `GlobalScope.launch创建协程` -> `delay 1s`->`主线程输出hello`->`runBlocking创建另一个协程`->`delay 2s`->`main线程被block`->`1s后第一个协程输出Kotlin Coroutines`->`2s后第二协程结束，主线程继续执行`->`主线程输出world`

```
fun main() = runBlocking {
    GlobalScope.launch {
        delay(1000)
        println("Kotlin coroutines")
    }
    println("hello")
    delay(2000)
    println("world")
}
```
输出同1）①

- `runBlocking创建协程并阻塞main线程`->`GlobalScope.launch创建新后台协程`->`delay 1s`->`打印hello`->`delay 2s`->`1s后打印Kotlin coroutines`->`2s后打印world`->`协程结束`->`main线程退出`
```
fun main() = runBlocking {
    GlobalScope.launch {
        delay(1000)
        println("Kotlin coroutines")
    }
    println("hello")
    delay(500)
    println("world")
}
```
输出同1）②

上面的代码都需要死死的让主线程的delay时间>协程时间，否则协程的内容不会输出。
但其实协程也有类似于java中join的方法，使得主线程等待协程执行完成之后继续执行，这需要使用到[`Job`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html)接口：
```
fun main() = runBlocking {
    val myJob : Job = GlobalScope.launch {
        delay(1000)
        println("Kotilin coroutines")
    }
    println("Hello")
    myJob.join()
    println("world")
}
```
输出同1）①

- `创建新协程`->`delay`->`main线程输出hello`->`main线程join`->`1s后协程输出内容`->`主线程输出world`

但其实每一个协程构建器（包括runBlocking）都会向其他代码块作用域添加一个CoroutineScope实例，我们可以在该作用域中启动协程，而无需显式将其join到一起，这是因为外部协程（下面实例中就是runBlocking）会等待该作用域中的所有启动的协程全部完成后才会完成

```
fun main() = runBlocking {
    launch {
        delay(1000)
        println("Kotlin Coroutines")
    }
    println("hello")
}
```
输出：
```
hello
Kotlin Coroutines
```

- 与上面不同，此例中runBlocking等待了协程中代码的运行。runBlocking内部自动有一个CoroutineScope实例，因此launch也是这个CoroutineScope调用的。所以runBlocking会等待launch中的协程执行。
- 前例中由于是主动创建了一个CoroutineScope，该实例与runBlocking没有什么关系，因此runBlocking不会等待launch逻辑
- 即使此处添加`println("hello")`、`delay(500)`、`println("world")`依然正常运行
       
4) coroutineScope: 自己声明协程作用域，同时会等待所有的子协程全部完成后才会完成

```
fun main() = runBlocking {
    launch {
        delay(1000)
        println("my job1")
    }
    println("person")

    coroutineScope {
        launch {
            delay(10000)
            println("my job2")
        }
        delay(5000)
        println("hello world")
    }
    println("welcome")
}
```
输出：
```
person
my job1
hello world
my job2
welcome
```

#### 4. 协程是轻量级的

- 线程由底层操作系统进行管理和创建，因此每个操作系统的线程数量是又上限的
- 协程的数量是无上限的，比线程更轻

1) 多次创建协程

```
fun main() = runBlocking {
    repeat(100) {
        launch {
            delay(100)
            println("A")
        }
    }
    println("Hello world")
}
```
结果：
```
Hello world
A
...
A
```

- 输出100次A

使用Thread替换协程：

```
fun main() {
    repeat(100) {
        thread {
            Thread.sleep(100)
            println("A")
        }
    }
    println("Hello world")
}
```

- 输出结果相同

如果分别将上述2段代码的repeat值改为10000，那么协程正常运行，而线程则可能抛出outofMemeryException（不同电脑结果不同，有的电脑线程也可以输出结果，但是时间要比协程慢很久，同时卡机）




