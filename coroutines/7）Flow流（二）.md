### 协程 —— Flow流（二）

#### 8. Flow上下文原理

- Flow的收集动作总是发生在调用协程的上下文当中，这个特性叫做上下文保留（Context Preservation）

- 通过withContext切换线程

- 通过flowOn可以让Flow在emit发射元素时所在的上下文与collect收集（终止操作）时所处的上下文是不同的，flowOn运算符本质上会改变上下文中的CoroutineDispatcher，并且为上游的flow创建另一个协程

```
private fun log(msg: String) = println("${Thread.currentThread().name}, $msg")

private fun myMethod() : Flow<Int> = flow {
    log("started")

    for (i in 1..3) {
        emit(i)
    }
}

fun main() = runBlocking {
    myMethod().collect { log("Collected: $it") }
}
```

输出
```
main @coroutine#1, started
main @coroutine#1, Collected: 1
main @coroutine#1, Collected: 2
main @coroutine#1, Collected: 3
```

- flow的执行线程和协程都是在runBlocking所在的main线程中执行的

- 如果上述flow中的代码比较耗时，那么就会阻塞main线程

```
private fun myMethod() : Flow<Int> = flow {
    withContext(Dispatchers.Default) {
        for (i in 1..4) {
            Thread.sleep(100)
            emit(i)
        }
    }
}

fun main() = runBlocking {
    myMethod().collect { println(it) }
}
```
结果
```
Exception in thread "main" java.lang.IllegalStateException: Flow invariant is violated:
		Flow was collected in [BlockingCoroutine{Active}@f5f6978, BlockingEventLoop@bce72],
		but emission happened in [DispatchedCoroutine{Active}@5590f4ab, Dispatchers.Default].
		Please refer to 'flow' documentation or use 'flowOn' instead
```

- 结果异常，显示Flow invariant is violated，Flow的不变性被违反了

- flow的collect是在runBlocking协程当中，而emit通过withContext切换了CoroutineContext上下文到Dispatchers.Default中，二者上下文不一致，这是不允许的

- 异常提示使用`flowOn`代替flow

```
// -Dkotlinx.coroutines.debug
private fun log(msg: String) = println("${Thread.currentThread().name}, $msg")

private fun myMethod(): Flow<Int> = flow {
    for (i in 1..4) {
        Thread.sleep(100)
        log("emit: $i")
        emit(i)
    }
}.flowOn(Dispatchers.Default)

fun main() = runBlocking {
    myMethod().collect { log("Collected: $it") }
}
```
结果
```
DefaultDispatcher-worker-1 @coroutine#2, emit: 1
main @coroutine#1, Collected: 1
DefaultDispatcher-worker-1 @coroutine#2, emit: 2
main @coroutine#1, Collected: 2
DefaultDispatcher-worker-1 @coroutine#2, emit: 3
main @coroutine#1, Collected: 3
DefaultDispatcher-worker-1 @coroutine#2, emit: 4
main @coroutine#1, Collected: 4
```
- 值得注意的是：flowOn运算符改变了Flow本身默认的顺序性

- 现在，collect收集操作实际上是发生在一个协程当中，而emit发射操作是发生在另外一个协程当中。

**********************************

#### 8. Flow的换冲Buffer

- 对于流的emit发射和collect收集处理都有时间损耗时，适合使用buffer

- 可以视flow在逻辑上存在两个端，一个用于发送信息，一个用于接收信息。执行emit就是从发射端发送到接收端，而执行collect就是将接收端数据处理掉。
    - `信息生成`->`emit`->`信息接收`->`collect处理`

- buffer的主要作用在于对发射的换冲，减少等待时间
    - `信息生成`->`emit`->`信息接收（可缓存多个信息）`->`collect处理`同时`emit继续发送`

- buffer与flowOn之间存在一定的关系

- 实际上，flowOn运算符在本质上在遇到需要改变CoroutineDispatcher时也会使用同样的缓冲机制，只不过该实例没有改变执行上下文而已

```
private fun myMethod() : Flow<Int> = flow {
    for (i in 1..4) {
        delay(100)                                    // 模拟耗时操作，耗时操作之后才开始流元素的发射
        emit(i)
    }
}

fun main() = runBlocking {
    val time = measureTimeMillis {
        myMethod()/*.buffer()*/.collect {             // (1)
            delay(200)
            println(it)
        }
    }

    println(time)
}
```

结果
```
1
2
3
4
1237
```

- emit和collect都需要耗时操作，因此总共需要时间 > (100 + 200) * 4 = 1200

- 如果将代码(1)中的注释去掉，执行buffer()，那么collect可以缓存emit发送来的数据，emit可以持续发送消息，而不必等待collect处理上一条消息后在工作。(执行时间大约900~1000)

**********************************

#### 9. Flow组合

- 将不同的流组合成一个

```
fun main() = runBlocking {
    val nums = (1..5).asFlow()
    val strs = flowOf("one", "two", "three", "four", "five")
    nums.zip(strs) { a, b -> "$a -> $b"}.collect { println(it) }
}
```

输出
```
1 -> one
2 -> two
3 -> three
4 -> four
5 -> five
```

- zip()将两个流组成成一个，并从过collect处理

**********************************

#### 10. Flow的flatten

- 将多层的流打平成一层（`Flow<Flow<Int>> -> Flow<Int>`）

- `flatMapConcat`中间操作可以将二维flow打平成一维flow
  
```
private fun myMethod(i : Int): Flow<String> = flow {
    emit("$i: First")
    delay(500)
    emit("$i: Second")
}

fun main() = runBlocking {
    val startTime = System.currentTimeMillis()

    (1..3).asFlow().onEach { delay(100) }
        .flatMapConcat { myMethod(it) }
        .collect { value ->
            println("$value at ${System.currentTimeMillis() - startTime} ms")
        }
}
```
输出
```
1: First at 131 ms
1: Second at 631 ms
2: First at 733 ms
2: Second at 1236 ms
3: First at 1339 ms
3: Second at 1842 ms
```

**********************************

#### 11. Flow的异常

- try-cacht将捕获流元素发射、中间操作和最终终止操作中所有的异常

```
private fun myMethod(): Flow<String> =
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i)
        }
    }.map { value ->
        check(value <= 1) {
            "Crash on $value"
        }
        "string $value"
    }

fun main() = runBlocking {
    try {
        myMethod().collect { println(it) }
    } catch (e: Throwable) {
        println("Caught $e")
    }
}
```

结果
```
Emitting 1
string 1
Emitting 2
Caught java.lang.IllegalStateException: Crash on 2
```
- check检查大于1的数字，如果发现则抛出message，message作为异常的信息打印出来

**********************************

#### 12. Flow的完成方式

- Kotlin提供两种方式实现Flow的完成：`命令式`、`声明式`

- 命令式：通过try-finally来实现，无论如何finally都会调用

- 声明式：这是Kotlin的flow独有的，Flow提供了一个名为onCompletion`中间操作`
    - 该操作会在Flow完成收集之后才会调用
    - onCompletion中间操作的一个优势在于它有一个可空的Throwable参数，可用作确定Flow的收集操作是正常完成的还是异常完成的

命令式：
```
private fun myMethod() : Flow<Int> = (1..3).asFlow()

fun main() = runBlocking {
    try {
        myMethod().collect { println(it) }
    } finally {
        println("finally")
    }
}
```
结果
```
1
2
3
finally
```

声明式：
```
private fun myMethod() : Flow<Int> = (1..3).asFlow()

fun main() = runBlocking {
    myMethod().onCompletion { println("onCompletion") }
        .collect { println(it) }
}
```
结果
```
1
2
3
onCompletion
```

- 虽然onCompletion是中间操作，但它却是在collect都执行完之后才执行

声明式（产生异常时）：

```
private fun myMethod(): Flow<Int> = flow {
    emit(1)
    throw RuntimeException()
}

fun main() = runBlocking {
    myMethod().onCompletion { cause -> if (cause != null) println("Flow Completed Exceptionally") }
        .catch { cause -> println("catch entered") }
        .collect { println(it) }

}
```
结果
```
1
Flow Completed Exceptionally
catch entered
```



