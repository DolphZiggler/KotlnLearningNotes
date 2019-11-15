### 异步流

挂起函数总是异步的返回单个值，但是怎样才能异步返回过个计算好的值呢？这就是`Kotlin`流(`Flow`)的用武之地

#### 多个值的表示

多值可用用Collection表示，如

```kotlin
fun foo():List<Int> = listOf(1,2,3)
fun main= foo.forEach{value ->println(value)}//一次打印所有数值
```

#### 序列-Sequence

如果在计算一些消耗CPU的数字计算(每次计算要100ms)，可以使用Sequence代表这些数字。

和集合不同的是：集合拿到所有值之后才能返回，序列可以拿到一个返回一个。

```kotlin
fun foo():Sequence<Int> = sequence{//sequece builder
  	for(i in 1..3){
      Thread.sleep(1000)
      yield(i)
    }
}
fun main() = foo().forEach{value-> println(value)}//每隔一秒才能打印一个
```

#### 挂起函数

上述的计算都会阻塞主线程。当这些计算通过`suspend`修饰符标记的异步代码执行时，可不阻塞线程返回结果。

```kotlin
suspend fun foo():List<Int>{
  delay(1000L)
  return listOf(1,2,3)
} 
fun main()= runBlock<Unit>{foo().forEach{v->println(v)}}
```

#### 流

使用`List<Int>`作为返回结果，意味着一次返回所有的值。可以使用`Flow<Int>`以流的形式表示异步计算的结果，就像是`Sequence<Int>` 类似。

```kotlin
fun foo():Flow<Int> = flow{
  for (i in 1..3){
    delay(100L)
    emit(i)
  }
}
fun main() = runBlocking<Unit>{
  launch{
    for(k in 1..3){
      println("I'm not blocked $k")
      delay(100L)
    }
  }
  foo().collect(v->println(v))
}
```

#### 流是冷启动

Flows和Sequences一样都是冷启动的：在流builder里面的代码直到流被收集的时候才执行。既调用`emit`、`yield`方法后才运行。

#### 取消流

流的取消依附于对应的协程的取消，而没有引入新的取消点，它是完全透明的。

```kotlin
fun foo():Flow<Int> = flow{
  for(i in 1..3){
    delay(100L)
    println("Emitting $i")
    emit(1)
  }
}
fun main() = runBlocking<Unit>{
  withTimeOutOrNull(250){
    foo().collect{value -> println(value)}
  }
  println("Done")
}
```

#### 构建流

`flow{....}`、`flowOf`、`asFlow()`

#### 操作符过渡流

流可以通过操作符转换。过渡操作符(Intermediate operators)将上行流装换为下行流(upstream flow to downstream flow)。所有的操作符都是冷启动的。 对操作符的调用本身不是一个挂起函数，它快速运行，返回新的转换过的流。

基础的操作符和`map`、`filter`命名类似。对比序列，它最大的不同是，调用操作符会产生挂起函数。

```kotlin
suspend fun performRequest(req:Int):String{
  delay(1000L)
  return "response $req"
}
fun main() = runBlocking{
  (1..3).asFlow().map{value -> performRequest(value)}.collect{println(it)}
}
```

#### 转换操作符 -transform

在所有操作符中，最通用就是`transform`。它可以用来模拟`map`、`filter`，也可以实现更复杂的转换。使用`transform`可以发射任意类型任意次数的数据。

```kotlin
(1..3).asFlow()
	.transform{req->
   	emit("Making request $req")
    emit(performRequest(req))         
   }.collect{response -> println(response)}
```

#### 数量限制操作符

`take`类似操作符在流超出限制时取消流。协程的取消总是通过抛出异常，所以常用到`try {...}finally{...}`

```kotlin
fun numbers():Flow<Int> = flow{
  try{
    emit(1)
		emit(2)
    println("This line will not be executed")
    emit(3)
  }finally{
    println("Finally in numbers")
  }
}
fun main()=runBlocking<Unit>{
  numbers().take(2).collect{println(it)}
}
```

#### 终端操作符

流的终端操作符是开始收集流数据的挂起函数。`collect`是最基本的一个，除此外还有:

- 转换成集合的`toList`、`toSet`
- 获取第一个值`first` ，限制流值发射单个数据`single`
- 减少流`reduce`，`fold`

```kotlin
val sum() = (1..5).asFlow().map{it*it}.reduce{a,b-> a+b}
```

#### 流是序列化的

除非特殊的操作符作用每个流的收集都是顺序执行的。收集工作在调用终端操作符的协程里面完成。默认不会有新协程启动。每一个转换操作符发射的数据从upstream到downstream，然后传递到终端操作符。

```kotlin
(1..5).asFlow().filter{it%2==0}.map{"string $it"}.collect{println{it}}
```

#### 流的上下文

流的收集工作总在调用者协程的上下文中。称为 **上下文保存**

```
withContext(context){
		foo.collect{println(it)}
}
```

#### 错误使用`withContext`发射

长时间消耗CPU的代码应在`Dispatchers.Default`执行，UI更新代码应该在`Dispatchers.Main`中执行。

通常使用`withContext`改变协程的上下文，但是流构建者应为`context preservation`特性不允许在不同的上下文发射数据。

```kotlin
fun foo():Flow<Int> = flow{
  withContext(Dispatchers.Default){
    for(i in 1..3){
      Thread.sleep(100)
      emit(1)
    }
  }
}
fun main() = runBlocking{
  foo().collect{println(it)}
}
/**
		Flow was collected in [CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@641109d1, BlockingEventLoop@6d0b88d2],
		but emission happened in [CoroutineId(1), "coroutine#1":DispatchedCoroutine{Active}@6e80c69c, DefaultDispatcher].
		Please refer to 'flow' documentation or use 'flowOn' instead
**/    
```

#### 操作符`flowOn`

上述异常可以参考`flowOn`操作符，可用来切换流发射的上下文。

```kotlin
fun foo():Flow<Int> = flow{
  for(i in 1..3){
    Thread.sleep(100)
    emit(i)
  }
}.flowOn(Dispatchers.Default)

fun main() = runBlocking{
  foo().collect{println{it}}
}
```

#### Buffering缓存

将流的不同部分运行在不同的协程从总共花销的时间来看是有用的，特别是在长时间有异步运行的情况。

```kotlin
fun foo():Flow<Int> = flow{
  for(i in 1..3){
    delay(100)
    emit(1)
  }
}
fun main() = runBlocking{
  val time = measureTimeMillis{
    foo()
    	.buffer()
  		.collect{
      		delay(300)
      		println(it)
      }
  }
  println("Collected in $time ms")
}
```

不加buffer的话，要运行最少 （100+300）*3 = 1200ms，

加上buffer，最少只需要等300 *3 + 1 *3 = 1000ms，在获取流的时候只在第一个时间才有等待时间(在处理数据的时候，流继续发射数据缓存在buffer中，处理完直接从buffer中拿数据) 。

#### 融合Conflation

当流代表操作符结果或者状态更新的结果，也许不必要对每一个值进行处理，而只需要处理最近的一些。`confalte`操作符可以用来跳过一些中间值，当collector处理数据较慢时。

```kotlin
val time = measureTimeMillis{
  foo()
		.conflate()
  	.collect{
      delay(300)
      println(it)//只打印1，3
    }
}
```

#### 处理最新的值

当发射器和处理器都很慢时，融合是一种加速处理的方法。它丢掉一些发射的值。

另一种法法就是当每个新值被发射时，取消掉非常慢的收集器然后重启它。有一系列`xxxLatest`操作符执行必要的`xxx`逻辑，当有新值来的时候取消旧值的运算。

```kotlin
val time = measureTimeMillis{
  foo()
  	.collectLatest{
      println("Collectiong $it")
      delay(300)
      println("Done $it")
    }
}
//Collecting 1
//Collecting 2
//Collecting 3
//Done 3
```

## 组合多个流

### zip

和标准库中的`Sequence.zip`扩展方法一样，流也有zip操作符来组合对应的两种流：

```kotlin
val nums = (1..3).asFlow()
val strs = flowOf("one"，"tow"，"three")
nums.zip(strs){a,b-> "$a -> $b"}.collect{println(it)}
```

#### combine

需要执行的计算如果需要各流对应的最新值，并且如果某个流有新值时就要重新计算，需要`combine`系列操作符

```kotlin
val nums = (1..3).asFlow().onEach{dealy(300)}
val strs = flowOf("one","two","thred").onEach{delay(400)}
nums.combine(strs){a,b -> "$a -> $b"}.collect{println(it)}
```

使用zip每次计算都保证所有流都拿到的是新的未计算过的数据。

使用combine每次计算只保证至少有一个流拿到新数据。

### 展平流

流代表着异步获取到的一些列数据，所以常遇到一个值对应一个请求其他值的请求。如

```kotlin
fun requsetFlow(i:Int):Flow<String> = flow{
  emit("$i:First")
  delay(500)
  emit("$i:Second")
}
(1..3).asFlow().map{requestFlow(it)}
```

以上操作可以用一系列展平操作符来完成。

#### flatMapConcat

串联模式通过`flatMapConcat`和`flattenConcat`操作符完成。它们等待一次内嵌流完成才开始收集下一个外层流发射的数据。

```kotlin
val startTime = System.currentTimeMillis()
(1..3).asFlow().onEach{delay(100)}
			.flatMapConcat({requestFlow(it)})
			.collect{println(it)}
1: First at 121 ms from start
1: Second at 622 ms from start
2: First at 727 ms from start
2: Second at 1227 ms from start
3: First at 1328 ms from start
3: Second at 1829 ms from start
```

#### flatMapMerge

另一种展平方式就是并发的收集所有内嵌流，然后把它们组合成一个流。通过`flatMapMerge`和`flattenMerge`实现。它们接受一个`concurrency`参数来限制并发流的数量。

```kotlin
val startTime = System.currentTimeMillis()
(1..3).asFlow().onEach{delay(100)}
		.flatMapMerge(requestFlow(it))
		.collect{println(it)}
1: First at 136 ms from start
2: First at 231 ms from start
3: First at 333 ms from start
1: Second at 639 ms from start
2: Second at 732 ms from start
3: Second at 833 ms from start
```

