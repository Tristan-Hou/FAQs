### 协程

#### 1. suspend关键字

- 被suspend关键字所修饰的函数叫挂起函数
- 挂起函数可以普通函数一样用在协程中，不过他的一个特性在于可以使用其他挂起函数
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

#### 2. 守护线程

- 全局协程类似于守护线程
- 使用GlobalScope启动的协程并不会保持进程的声明，他们就像是守护线程一样

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

#### 3. 





