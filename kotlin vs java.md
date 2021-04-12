### 参考资料：[Kotlin语言特性与Java语言对比](https://www.bilibili.com/video/BV1rE411k7Xe?p=1) 

#### 一、基本语法详解

1. Kotlin中可以不再类中编写函数，但为了使其符合Java字节码标准，编译器在编译后都会将它们转化为在类当中，类名由编译器自动生成，格式为“文件名+Kt.class”；
   - ` HelloKotlin.kt`中的函数编译后在`HelloKotlinKt.class`中
3. Kotlin文件同Java一样，可以打包成jar包；
   - `kotlinc HelloKotlin.kt -include-runtime -d HelloKotlin.jar` 其中`-include-runtime`用来引入kotlin资源包，`-d`用于打包成jar包。
3. Kotlin函数必须带有fun关键字，格式：fun ‘name'(): 返回值， 参数类型在参数名后面：参数名：参数类型，结尾没有;
   - `fun sum2(a: Int, b: Int) = a + b`
4. val类似于Java中的final，自定不可修改，而var定义的变量可修改；
5. Kotlin具有类型推断功能，很多时候编译器可以自动推断出变量类型
   - `val a: Int = 3` 中`Int`可以省略：`val a = 3`
   - ```
     if(a is String)
         a.toUpperCase()    // Java中需要类型转换(String)a.toxxx();
     ```
6. Java中基本数据类型在Kotlin中对应着`Int`/`Long`/`Float`/`Byte`等，类似于Java中的`Integer`。**因此在Kotlin中这些类型之间的转换会相比在Java中会更加严格**。**仅仅在Jvm中可以将Int等类型视为Java的int等类型，但Kotlin可以在许多平台上使用，不同平台有不同特性，因此其他情况下需要特别注意**
   - `Java`
     ```
     int a = 1; 
     int b = 2;
     int c = 3;
     a = b;         // 对
     b = a          // 错
     b = (byte) a;  // 对，byte不可以直接转为int，需要cast
     a = c;         // 对
     ```
   - `Kotlin`  
     ```
     var x = 10
     var y = 20          // int类型
     var z : Byte = 30   // byte类型
     x = y
     x = z.toInt()
     ```
7. Kotlin中也需要import，同时Kotlin也可以类似Python一样为import取别名
   - `import com.xxx.xxx.MyClass as OtherNameClass`后，当前类中`MyClass`可以替代为`OtherNameClass`
8. Kotlin中增加了很多灵活的语法糖，简化代码
   - `val max = if(x > y) x else y` 相当于max的值为if的返回值
   - 亦可以用于代码块中，代码块最后一个值作为返回值：
     ```
      val max = if(x > y) {
          printf(x)
          x
      } else {
          printf(y)
          y
      }
     ```
9. `?`表示可能为空
   - `Int?`表示该值可能为一个Int，也可能为null
10. `Any`是可以代表任何类型，是所有类的顶层类，类似Java中的Object
11. `is` 可视为`instanceof`
12. 数组：`IntArray`
    - 在JVM中相当于int[]
     ```
     val array : IntArray = intArrayOf(1, 2, 3)
     for(item: Int in array) {
         printf(item)
     }
     for(item : Int in array.indices) {
         printf(item)
     }
     ```
    - 其他类型类似：CharArray/ShotrArray/ByteArray
13.  when关键字
     ```
     when(str) {
         "hello" -> return "hello"
         "world" -> return "world"
         else -> return "error"
     }
     
     fun myMethod(str: String): String = 
         when(str) {
              "hello" -> "hello"
              "world" -> "world"
              else -> "error"
         }
         
     fun myTest(a: Int) : Int {
         var result = when(a) {
             1 -> {
                 print("1")
                 10
             }
             2 -> {
                 print("2")
                 20
             }
             3,4,5 -> {
                 print("3,4,5")
                 30
             }
             in 6..10 -> {
                 print("10")
                 40
             }
             else -> {
                 print("else")
                 50
             }
         }
         return result
     }
     ```
14. 

#### 二、包、函数、变量、表达式、控制语句

#### 三、基本类型

#### 四、类、对象

#### 五、类与继承、属性与字段

#### 六、可见性

#### 七、扩展

#### 八、数据类与密封类

#### 九、kotlin泛型

#### 十、when关键字

#### 十一、open、final、abstract修饰符

#### 十二、对象构建过程

#### 十三、Any类型

#### 十四、Kotlin反射

#### 十五、对象表达式与声明

#### 十六、函数与高阶函数

#### 十七、Kotlin内联函数

#### 十八、Lamdba表达式

#### 十九、Kotlin函数式编程与java函数式编程

#### 二十、Kotlin类型检查与类型转换

#### 二十一、空安全的实现与Optional

#### 二十二、异常与注解

#### 二十三、枚举

#### 二十四、嵌套类与匿名类

#### 二十五、Kotlin集合

#### 二十六、不变集合与可变集合

#### 二十七、运算符重载

#### 二十八、java与kotlin互操作实例剖析

#### 二十九、协程与Kotlin协程


