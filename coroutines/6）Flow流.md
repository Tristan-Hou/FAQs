### 协程 —— Flow流

#### 1. 流式输出数据

- 阻塞√，一次性返回√：集合本身是一次性返回给调用端的，即集合中的全部元素均已经获取到后才一同返回给调用端，同时调用集合时会阻塞

- 阻塞√，一次性返回×：序列中的数据并非像集合那样一次性返回给调用端，而是计算完一个数据后就返回一个数据。序列中的计算过程会阻塞线程

- 阻塞×，一次性返回√：协程计算结果会一次性返回给调用函数，但不会阻塞主线程

- 阻塞×，一次性返回×：？？？

首先对比3个代码例子，输出一个list或sequence中的内容
  - 1）直接方式
  ```
  private fun myMethod() : List<String> = listOf("hello", "world", "hello world")

  fun main() {
      myMethod().forEach{println(it)}
  }
  ```
  输出
  ```
  hello
  world
  hello world
  ```
  - 2）使用Sequence序列
  ```
  private fun myMethod() : Sequence<Int> = sequence {
    for (i in 100..105) {
        Thread.sleep(1000)                                 // 模拟耗时阻塞
        yield(i)
    }
  }

  fun main() {
      myMethod().forEach { println(it) }
  }
  ```
  输出
  ```
  100
  101
  102
  103
  104
  105
  ```
  - 3）使用协程
  ```
  private suspend fun myMethod() : List<String> {
    delay(1000)
    return listOf("hello", "world", "hello world")
  }

  fun main() = runBlocking {
      myMethod().forEach { println(it) }
  }
  ```
  输出
  ```
  hello
  world
  hello world
  ```
  - listOf是集合，代码1）会阻塞主线程，并将结果一次性返回，这里返回了1次结果

  - Sequence是序列，代码2）会阻塞主线程，但是每计算完一次结果就返回一次结果给调用函数，这里返回了6次结果
  
  - runBlocking是协程，代码3）不会阻塞主线程，并将结果一次性返回，这里返回了1次结果

  - 有没有什么机制可以实现既不阻塞，计算结果也可以每次计算完立刻返回（而不是等待所有计算都完成后一起返回）


**********************************


#### 2. 什么是Flow

第1节中发现3种方法都不能实现即不阻塞，返回值也分次输出的代码

- 如果使用lisOf()，返回List<>结果类型，通过不同方式1）或3）可以解决阻塞问题，但是输出流只能一次性返回
  
- 如果使用Sequence<>类型，输出流可以每次计算立即返回，但它是同步计算的，会阻塞线程
  
- 要想能够实现可以异步计算的流式的值，也可以每次立即返回计算结果，我们就可以使用Flow<>类型，它非常类似于Sequence<>类型，不同的它是异步计算的

Flow的使用：

- FLow构建器是通过flow来实现的

- 位于flow{ }构建器中的代码是可以挂起suspend的

- 构建器方法(如下myMethod())方法无需再使用suspend标识符，值是通过emit函数来发射出来的

- Flow里面的值是通过collect方法来收集的

```
private fun myMethod(): Flow<Int> = flow {
    for (i in 1..4) {
        delay(100)                                // (1)
//        Thread.sleep(100)                       // (2)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    launch {
        for (i in 1..4) {
            println("hello $i")
            delay(200)
        }
    }
//    delay(1000)                                 // (3)
    myMethod().collect { println(it) }
}
```
输出
```
hello 1
1
hello 2
2
3
hello 3
4
hello 4
```
- myMethod()方法流输出与launch协程内部的输出是交替进行的，说明flow<>做到了异步执行和单次结果立即返回的功能，（如果是同步的那么会阻塞主线程，结果不会不会交替执行(Sequence<>))

- 如果将(1)处代码注释掉，而执行(2)处代码，结果会如何？
    ```
    1
    2
    3
    4
    hello 1
    hello 2
    hello 3
    hello 4
    ```
  - myMethod()内部使用的是阻塞代码Thread.sleep()，因此代码执行到此处时会阻塞，因此即使launch生成了一个协程但并不会执行，直到myMethod()执行完为止。（有时候也会先执行完luanch，再执行myMethod()代码）
  - 如果此时将(3)处代码反注释，与(2)一起执行，那么结果就是先执行launch的代码，再执行myMethod()代码

**********************************

#### 3. Flow的执行时机

- 当我们调用了构建器方法(如下myMethod())方法后，它实际上是立刻返回的，并不会去执行其中的代码。

- 只有当调用了Flow对象上的终止操作（如collect)之后，Flow才会真正执行

```
private fun myMethod() : Flow<Int> = flow {
    println("myMethod executed")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking {
    println("hello")
    val flow = myMethod()
    println("world")

//    flow.collect { println(it) }              // (1)
//    println("welcome")                        // (2)
//    flow.collect { println(it) }              // (3)
}
```
输出
```
hello
world
```
- 可见flow代码块内部的逻辑并没有被执行

- 而如果反注释掉(1)、(2)、(3)的代码，那么flow的代码会执行，而且会执行两次
    ```
    hello
    world
    myMethod executed
    1
    2
    3
    welcome
    myMethod executed
    1
    2
    3
    ```

**********************************

#### 4. Flow的取消

- low的取消实际上与协程的取消之间是一种协同关系
- 
- 对于Flow来说，它自身并没有引入任何新的取消点这样的概念，它对于取消时完全透明的

- FLow的收集操作是可以取消的，前提是Flow在一个可取消的挂起函数（如delay）中被挂起了，除此之外，我们无法通过其他方式来取消Flow的执行

```
private fun myMethod() : Flow<Int> = flow {
    for (i in 1..4) {
        delay(100)
        println("Emit: $i")
        emit(i)
    }
}

fun main() = runBlocking {
    withTimeoutOrNull(200) {                          // (1)
        myMethod().collect { println(it) }            // (2)
    }
    println("Finished")
}
```

结果
```
Emit: 1
1
Finished
```
- (1)处withTimeoutOrNull是一个suspend函数，因此(2)处myMethod()方法依附于withTimeoutOrNull()挂起函数，可以被取消

**********************************


#### 5. Flow的构建器

- flow是最经常被使用的一种流构建器

- flowOf构建器可以用于定义能够发射固定数量值的流

- 对于各种集合与序列来说，它们都提供了asFlow()扩展方法来将自身转换为Flow

```
fun main() = runBlocking {
    (1..10).asFlow().collect { println(it) }
    println("~~~~~~~~~~")
    flowOf(10, 20, 30).collect { println(it) }
}
```

输出

```
1
(ignore...)
9
10
~~~~~~~~~~
10
20
30
```

**********************************


#### 6. Flow中间运算符/中间操作

- Flow的中间运算符思想与Java Stream完全一致

- Flow与Sequence之间在中间运算符上的重要区别在于：对于Flow来说，这些中间运算符内的代码块是可以调用suspend挂起函数的

`filter`、`map`:

```
private suspend fun myExecution(input : Int) : String {
    delay(1000)
    return "output: $input"
}

fun main() = runBlocking {
    (1..10).asFlow().filter { it > 5 }.map { myExecution(it) }.collect { println(it) }
}
```

结果
```
output: 6
output: 7
output: 8
output: 9
output: 10
```
- filter过滤大于5的元素、map对每个元素操作传递到下一层

`transform`（比filter/map更加灵活）：
```
private suspend fun myExecution(input : Int) : String {
    delay(1000)
    return "output: $input"
}

fun main() = runBlocking {
    (1..10).asFlow().transform {
        emit("my input: $it")
        emit((myExecution(it)))
        emit("hello world")
    }.collect { println(it) }
}
```
结果
```
my input: 1
output: 1
hello world
(ignore...)
my input: 10
output: 10
hello world
```
- transform中可做任意操作，共有3次emit

`take`（限定数量的中间操作，本质是通过异常实现的）:
```
fun myNumbers() : Flow<Int> = flow {
    emit(1)
    println("hello world")
    emit(2)
    emit(3)
}

fun main() = runBlocking {
    myNumbers().take(2).collect { println(it) }
}
```
结果
```
1
hello world
2
```
- take参数为2，因此myNumbers中emit 2次后结束。

**********************************

#### 7. Flow终止操作

- Flow是顺序执行的

- 对于Flow的收集操作来说，它是运行在调用了终止操作的那个协程上。

- 默认情况下，它是不会启动新的协程的。

- 每个emit的元素值都会由所有的中间操作进行处理，最后再由终止操作进行处理。本质上，就是由上游进入到了下游

`reduce`（聚合结果）：它是一个终止操作，同样的终止操作还有toList与toSet，它们的返回值不再是Flow，而是其他具体响应类型

```
fun main() = runBlocking {
    val result = (1..4).asFlow().map { it * it }.reduce { a, b -> a + b }
    println(result)
}
```
结果
```
30
```

**********************************

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

#### 8. Flow组合

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

#### 8. Flow的flatten

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

#### 9. Flow的异常

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

#### 9. Flow的完成方式

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











