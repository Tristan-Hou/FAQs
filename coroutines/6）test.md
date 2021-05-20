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

#### 2. 什么是Flow

第1节中发现3种方法都不能实现即不阻塞，返回值也分次输出的代码

- 如果使用lisOf()，返回List<String>结果类型，通过不同方式1）或3）可以解决阻塞问题，但是输出流只能一次性返回
  
- 如果使用Sequence<String>类型，输出流可以每次计算立即返回，但它是同步计算的，会阻塞线程
  
- 要想能够实现可以异步计算的流式的值，也可以每次立即返回计算结果，我们就可以使用Flow<String>类型，它非常类似于Sequence<String>类型，不同的它是异步计算的

```
private fun myMethod(): Flow<Int> = flow {
    for (i in 1..4) {
//        delay(100)                           // (1)
        Thread.sleep(100)                      // (2)
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

    myMethod().collect { println(it) }
}
```






  

**********************************
