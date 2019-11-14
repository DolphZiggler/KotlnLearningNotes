### 协程上下文和调度器

协程总是在`CoroutineContext`实例的上下文里运行，该类型定义在kotlin标准库中。

协程上下文是一组不同的元素集合。主要的元素就是协程的`job`和`dispatcher`

#### 调度器和线程

协程上下文包含一个协程调度器，用来决定对应的协程在哪个线程执行。调度器可以限制协程运行在某个线程、线程池或不加限制。

所有的协程构建者如`launch`和`async` 都可以接收一个可选的`CoroutineContext`参数用来明确指定新协程或其他元素的调度器

* 默认，不写参数。继承所在CoroutineContext中的调度器

* Dispatchers.Unconfied

* Dispatchers.Default

  从`GlobalScope`中启动的协程默认调度器为`Dispatcher.Default`，使用一个线程池运行任务。`GlobalScope.launch{...}`与`launch(Dispatchers.Default){...}`效果一样。

* newSingleThreadContext

  一个专用线程是非常消耗资源的。使用中在不需要时要及时释放，或者放在应用级别重复使用。

#### Unconfined  vs Confined 调度器

调度器`Dispatcher.Unconfined`在调用线程开启一个新协程，但它只运行到第一个挂起点`suspension point`。而挂起之后回到协程中的动作完全由被挂起的函数决定。无限制的调度器适用于在指定线程执行消耗CPU时间或者更新共享数据(eg:UI)的协程

另一方面，调度器默认继承外层的`CoroutineContext`。特别是`runBlocking`协程，默认限制到调用者线程，所以继承它有限制任务在这个线程按FIFO的调度方式执行。

所以，继承自`runBlocking`的Context 的协程，默认在main线程执行任务。而unconfined协程在delay方法调用后有回到默认的执行线程。

#### 调试协程和线程

协程可以在一个线程上挂起在另一个线程上恢复。即使使用单线程调度器，也很难指出协程在哪里，什么时候，做什么。一般的调试方法就是在日志文件里的每一次log都打印出线程名字。但在使用协程的时候，线程名不能代表`CoroutineContext`的全部信息。

#### 线程之间跳转

```kotlin
newSingleThreadContext("Ctx1").use { ctx1 ->
    newSingleThreadContext("Ctx2").use { ctx2 ->
        runBlocking(ctx1) {
            log("Started in ctx1")
            withContext(ctx2) {
                log("Working in ctx2")
            }
            log("Back to ctx1")
        }
    }
}
```

####  上下文中的`Job`

Job是CoroutineContext中的一部分，可以通过`coroutineContext[Job]`获得

注意coroutineScope中的`isActive`仅仅是`coroutineContext[Job]?.isActive == true`的简写

#### 协程的子协程

Coroutine Builder： async，launch

Scoping Function：coroutineScope，withContext

当从一个协程的Scope下启动另一个协程，子协程会通过`CoroutineScope.coroutineContext`继承Context，子协程的Job变成父协程Job的子Job。当父协程取消的时候，所有的子协程都会被递归取消。

使用`GlobalScope.launch`启动一个协程，该协程的Job没有parent，所以没有绑定到启动它的Scope，是独立运行的。

#### 父协程职责

父协程总是等待所有子协程结束执行。父协程不必要显示跟踪所有它启动的子协程，不必使用`Job.join`来等他们结束。

```kotlin
@Test
fun test10() = runBlocking {
	val request = launch {
	repeat(3){
		launch {
			delay((it+1)*200L)
			println("Coroutin $it is done")
		}
	}
  // don't explicitly call join but still active until all children complete
  println("request: I'm done and I don't explicitly join my children 
  that are still active")
  }
  println("Main process complete")
 }
```

#### 协程命名

```kotlin
val v1 = async(CoroutineName("v1coroutine")) {
    delay(500)
    log("Computing v1")
    252
}
```

#### 上下文元素的组合

有时需要为一个协程上下文定义多个元素，可以使用`+`operator完成。 

```kotlin
launch(Dispatchers.Default+CoroutineName("test")){
  println("I'm working in thread ${Thread.currentThread().name}")
}
```

#### 协程作用域

把协程上下文，子协程，Job放在一起。假设我们的应用有一个拥有生命周期但不是协程的对象。比如，在Android应用中，在Activity中启动多个协程执行异步任务来获取数据执行动画等。所有的协程必须在Activity销毁的时候取消来避免内存泄露。我们可以人为的把上下文和Job都与Activity生命周期关联，但`kotlinx.coroutines`提供一种封装`ConroutineScope`。

通过管理`CoroutineScope`的生命周期来管理协程的周期，只需关联`CoroutineScope`和Activity的生命周期。一个`CoroutineScope`可通过`CoroutingScope()`或`MainScope()`工厂方法创建。前一个创建一个普通的Scope，后者创建一个专门的UI Scope，并使用`Dispatchers.Main`作为默认dispatcher

```kotlin
class Activity{
  private val mainScope = MainScope()
  fun destory{
    super.destory()
    mainScope.cancel()
  }
}
```

另外，我们可以让Activity实现`CoroutineScope`接口来达到目的。这种方法的最好实现就是使用默认的工厂方法作为代理。然后可以在Activity中使用coroutine不用显示指定它们的上下文

```kotlin
class Activity: CoroutineScope by CoroutineScope(Dispatchers.Default){
  fun doSomething(){
    repreat(10){i->
    	launch{
        delay((i+1)*100L)
        println("Coroutine $i is done")
      }           
    }
  }
}
```

#### 线程本地数据Tread-Local Data



