### 异步流

挂起函数总是异步的返回单个值，但是怎样才能异步返回过个计算好的值呢？这就是`Kotlin`流(`Flow`)的用武之地

#### 多个值的表示

多值可用用Collection表示，ru

```kotlin
fun foo():List<Int> = listOf(1,2,3)
fun main= foo.forEach{value ->println(value)}
```

#### 序列-Sequence

如果在计算一些消耗CPU的数字计算(每次计算要100ms)，可以使用Sequence代表这些数字

```kotlin
fun foo():Sequence<Int> = sequence{//sequece builder
  	for(i in 1..3){
      Thread.sleep(100)
      yield(i)
    }
}
fun main() = foo().forEach{value-> println(value)}
```



