[//]: # (title: Concurrent mutability)
[//]: # (auxiliary-id: Concurrent_Mutability)

When it comes to working with iOS, [Kotlin/Native's state and concurrency model](concurrency-overview.md) has [two simple rules](concurrency-overview.md#rules-for-state-sharing).

在 iOS 平台，[Kotlin/Native 的状态和并发模型](concurrency-overview.md)有[两个简单的规则](concurrency-overview.md#rules-for-state-sharing).

1. A mutable, non-frozen state is visible to only one thread at a time.
2. An immutable, frozen state can be shared between threads.

1. 一个可变的，未冻结的状态，同一时间只对一个线程可见。
2. 一个不可变的，冻结的状态，可以被所有线程共享。

The result of following these rules is that you can't change [global states](concurrency-overview.md#global-state), 
and you can't change the same shared state from multiple threads. In many cases, simply changing your approach to
how you design your code will work fine, and you don't need concurrent mutability. States were mutable from multiple threads in 
JVM code, but they didn't *need* to be.

由于以上两点，你不能修改 [全局状态](concurrency-overview.md#global-state)，也不能从多个线程修改一个被共享的状态。通常情况下，重新设计代码的实现即可，你其实并不需要并发可变性。
虽然 JVM 上的状态可以从多个线程修改，但他们其实*没必要*这样。

However, in many other cases, you may need arbitrary thread access to a state, or you may have _service_ objects that should be
 available to the entire application. Or maybe you simply don't want to go through the potentially costly exercise of 
redesigning existing code. Whatever the reason, _it will not always be feasible to constrain a mutable state to a single thread_.

不过，你可能需要从任意线程访问一个状态，或者你有一个 _service_ 对象，想要在整个应用范围内访问。又或者你只是不想承担重新实现现有代码的潜在成本。 _总之，将可变状态限制在单个线程上并不总是行得通的。_

There are various techniques that help you work around these restrictions, each with their own pros and cons:

以下几个方案可以帮你绕过这些限制，每个方案都各有各的利弊：

* [原子操作](#atomics)
* [线程隔离的状态](#thread-isolated-state)
* [底层能力](#low-level-capabilities)

<a id="atomics"></a>
## 原子操作

Kotlin/Native provides a set of Atomic classes that can be frozen while still supporting changes to the value they contain. 
These classes implement a special-case handling of states in the Kotlin/Native runtime. This means that you can change 
values inside a frozen state.

Kotlin/Native 提供一组原子操作工具类，可以在被冻结的状态下，修改它们所持有的值。
Kotlin/Native 运行时对这些类做了特殊处理，所以才可以修改一个冻结状态内部的值。

The Kotlin/Native runtime includes a few different variations of Atomics. You can use them directly or from a library.

Kotlin/Native 提供几种不同的原子操作，你可以直接使用或者引入一个库。

Kotlin provides an experimental low-level [`kotlinx.atomicfu`](https://github.com/Kotlin/kotlinx.atomicfu) library that is currently 
used only for internal purposes and is not supported for general usage. You can also use [Stately](https://github.com/touchlab/Stately), 
a utility library for multiplatform compatibility with Kotlin/Native-specific concurrency, developed by [Touchlab](https://touchlab.co). 

Kotlin 提供一个实验性的底层  [`kotlinx.atomicfu`](https://github.com/Kotlin/kotlinx.atomicfu) 库，目前只用于内部，还不适用于通用场景。还可以用 [Stately](https://github.com/touchlab/Stately)，是一个专用于 Kotlin/Native 并发的多平台兼容工具库，由 [Touchlab](https://touchlab.co) 开发。

### `AtomicInt`/`AtomicLong`
### `AtomicInt`/`AtomicLong`

The first two are simple numerics: `AtomicInt` and `AtomicLong`. They allow you to have a shared `Int` or `Long` that can be 
read and changed from multiple threads.

先看两个简单的数字类：`AtomicInt` 和 `AtomicLong`。可以从多个线程读写 `Int` 或 `Long`。

```kotlin
object AtomicDataCounter {
    val count = AtomicInt(3)
  
    fun addOne() {
        count.increment()
    }
}
```

The example above is a global `object`, which is frozen by default in Kotlin/Native. In this case, however, you can change the value of `count`. 
It's important to note that you can change the value of `count` _from any thread_.

上面是一个全局 `object` 的例子，在 Kotlin/Native 默认是被冻结的。然而在这里，你可以修改 `count` 的值。需要强调下，你可以  _在任意线程_ 修改 `count` 的值。

### `AtomicReference`
### `AtomicReference`

`AtomicReference` holds an object instance, and you can change that object instance. The object you put in `AtomicReference` 
must be frozen, but you can change the value that `AtomicReference` holds. For example, the following won't work in Kotlin/Native:

`AtomicReference` 持有一个对象实例，这个被引用的对象必须是被冻结的，不能修改，但是你可以修改
 `AtomicReference` 持有的引用。先看一个反例：

```kotlin
data class SomeData(val i: Int)

object GlobalData {
    var sd = SomeData(0)

    fun storeNewValue(i: Int) {
        sd = SomeData(i) // 这里不能修改 sd
    }
}
```

According to the [rules of global state](concurrency-overview.md#global-state), global `object` values are 
frozen in Kotlin/Native, so trying to modify `sd` will fail. You could implement it instead with `AtomicReference`:

根据 [全局状态规则](concurrency-overview.md#global-state)，全局 `object` 在 Kotlin/Native 内是被冻结的，所以尝试修改 `sd` 会失败。你应该使用 `AtomicReference` 去实现：


```kotlin
data class SomeData(val i: Int)

object GlobalData {
    val sd = AtomicReference(SomeData(0).freeze())

    fun storeNewValue(i: Int) {
        sd.value = SomeData(i).freeze()
    }
}
```

The `AtomicReference` itself is frozen, which lets it live inside something that is frozen. The data in the `AtomicReference` 
instance is explicitly frozen in the code above. However, in the multiplatform libraries, the data 
will be frozen automatically. If you use the Kotlin/Native runtime's `AtomicReference`, you *should* remember to call 
`freeze()` explicitly.

`AtomicReference` 本身是被冻结的，所以它内的状态也是被冻结的。上面的代码中我们显示的冻结了 `SomeData`，实际在 KMM 中，这个数据会被自动冻结。不过如果你使用 Kotlin/Native 运行时的 `AtomicReference`， *不要忘记*得手动调用 `freeze()`。

`AtomicReference` can be very useful when you need to share a state. There are some drawbacks to consider, however.
`AtomicReference` 在共享状态时很有用，但也有一些缺点。

Accessing and changing values in an `AtomicReference` is very costly performance-wise *relative to* a standard mutable state. 
If performance is a concern, you may want to consider using another approach involving a [thread-isolated state](#thread-isolated-state).

和一个标准的可变状态相比， 读写一个 `AtomicReference` 的性能消耗是非常大的。
对性能敏感的场景下，可以使用[线程隔离的状态](#thread-isolated-state)。

There is also a potential issue with memory leaks, which will be resolved in the future. In situations where the object 
kept in the `AtomicReference` has cyclical references, it may leak memory if you don't explicitly clear it out:

这里也有潜在的内存泄漏问题，会在后续版本解决。比如 `AtomicReference` 持有的对象有循环引用，不手动清除就会产生内存泄漏：

* If you have state that may have cyclic references and needs to be reclaimed, you should use a nullable type in the 
`AtomicReference` and set it to null explicitly when you're done with it.
* If you're keeping `AtomicReference` in a global object that never leaves scope, this won't matter (because the memory 
never needs to be reclaimed during the life of the process).

* 如果你有循环引用需要被回收，可以在 `AtomicReference` 的泛型使用 nullable 类型，需要清除引用时，手动设置为 null。
* 如果你是在全局对象使用 `AtomicReference`，就不需要考虑这个问题（因为在整个进程的生命周期内都不需要释放内存）。

```kotlin
class Container(a:A) {
    val atom = AtomicReference<A?>(a.freeze())

    /**
     * 不再需要 Container 时调用
     */
    fun clear(){
        atom.value = null
    }
}
```

Finally, there's also a consistency concern. Setting/getting values in `AtomicReference` is itself atomic, but if your 
logic requires a longer chain of thread exclusion, you'll need to implement that yourself. For example, if you have a 
list of values in an `AtomicReference` and you want to scan them first before adding a new one, you'll need to have some 
form of concurrency management that `AtomicReference` alone does not provide.

最后，还有一致性问题。读写 `AtomicReference` 操作是原子操作，但是如果你需要一个更长的线程独占期，就需要自己实现了。比如你有一个 `AtomicReference` list，想在遍历之后插入一个新值，你需要的是另一种并发管理机制，`AtomicReference` 并不提供。

The following won't protect against duplicate values in the list if called from multiple threads:

比如下面这个例子，如果从多线程调用这段代码，并不能保证没有重复 item：

```kotlin
object MyListCache {
    val atomicList = AtomicReference(listOf<String>().freeze())
    fun addEntry(s:String){
        val l = atomicList.value
        val newList = mutableListOf<String>()
        newList.addAll(l)
        if(!newList.contains(s)){
            newList.add(s)
        }
        atomicList.value = newList.freeze()
    }
}
```

You will need to implement some form of locking or check-and-set logic to ensure proper concurrency.
你需要锁或者 check-and-set 逻辑，保证并发正确性。

<a id="thread-isolated-state"></a>
## Thread-isolated state
## 线程隔离的状态

[Rule 1 of Kotlin/Native state](concurrency-overview.md#rule-1-mutable-state-1-thread) is that a mutable state is 
visible to only one thread. Atomics allow mutability from any thread. 
Isolating a mutable state to a single thread, and allowing other threads to communicate with that state, is an alternative 
method for achieving concurrent mutability.

[Kotlin/Native 状态规则 1](concurrency-overview.md#rule-1-mutable-state-1-thread) 要求可变状态只对单一线程可见。原子操作允许任意线程修改。还有一种方法可以实现并发可变性，将状态隔离在单线程内，允许其他线程与之通信。

To do this, create a work queue that has exclusive access to a thread, and create a mutable state that 
lives in just that thread. Other threads communicate with the mutable thread by scheduling _work_ on the work queue.

我们需要给线程创建一个独占访问权的任务队列，并在这个线程内维护一个可变状态。其他线程需要通过将 _任务_ 发送到这个任务队列，实现与可变线程通信。

Data that goes in or comes out, if any, needs to be frozen, but the mutable state hidden in the worker thread remains 
mutable. 

任何输入输出的数据都会被冻结，但是工作线程内的可变状态仍然是可变的。

Conceptually it looks like the following: one thread pushes a frozen state into the state worker, which stores it in 
the mutable state container. Another thread later schedules work that takes that state out.

概念上来说：一个线程将冻结的状态 push 到工作线程，后者将其储存在一个可变状态内。接着，另外的线程向工作线程请求这个状态。

![Thread-isolated state](isolated-state.png){animated="true"}

Implementing thread-isolated states is somewhat complex, but there are libraries that provide this functionality.

实现一个线程隔离的状态有些复杂，幸好已经有库为我们实现好了。

### `AtomicReference` vs. thread-isolated state
### `AtomicReference` vs. 线程隔离的状态

For simple values, `AtomicReference` will likely be an easier option. For cases with significant states, and potentially 
significant state changes, using a thread-isolated state may be a better choice. The main performance penalty is actually crossing 
over threads. But in performance tests with collections, for example, a thread-isolated state significantly outperforms a
mutable state implemented with `AtomicReference`.

对于简单的数据， `AtomicReference` 是一个方便的选择。对于重要的状态，尤其是可能需要修改的状态，线程隔离的状态更适用。主要的性能损耗产生在跨线程操作。但在使用集合的性能测试中，线程隔离状态的性能要显著优于 `AtomicReference`。

The thread-isolated state also avoids the consistency issues that `AtomicReference` has. Because all operations happen in the 
state thread, and because you're scheduling work, you can perform operations with multiple steps and guarantee consistency 
without managing thread exclusion. Thread isolation is a design feature of the Kotlin/Native state rules, and 
isolating mutable states works with those rules.

同时，线程隔离状态还避免了 `AtomicReference` 带来的一致性问题。因为所有操作都是在持有状态的线程上执行的，同时我们只是发送 event 到任务线程，不需要独占线程，就算使用多步操作，也能确保一致性。

The thread-isolated state is also more flexible insofar as you can make mutable states concurrent. 
You can use any type of mutable state, rather than needing to create complex concurrent implementations.

线程隔离的状态还更加灵活，比如你可以支持并发修改状态。
可以使用任何类型的状态，而不需要创建复杂的并发实现。

<a id="low-level-capabilities"></a>
## Low-level capabilities
## 底层能力

Kotlin/Native has some more advanced ways of sharing concurrent states. To achieve high performance, you may need to avoid 
the concurrency rules altogether. 

Kotlin/Native 还有一些更高级的共享并发状态的方法。想要更高的性能，你可能需要完全规避并发规则。

> This is a more advanced topic. You should have a deep understanding of how concurrency in Kotlin/Native works under 
> the hood, and you’ll need to be very careful when using this approach. Learn more about [concurrency](https://kotlinlang.org/docs/reference/native/concurrency.html).
>
{type="note"}

> 这是个更高级的话题。你需要深入理解 Kotlin/Native 并发的工作原理。
> 需要非常小心的使用这种方法。了解更多[关于并发的知识](https://kotlinlang.org/docs/reference/native/concurrency.html)。
{type="note"}

Kotlin/Native runs on top of C++ and provides interop with C and Objective-C. If you are running on iOS, you can also pass lambda 
arguments into your shared code from Swift. All of this native code runs outside of the Kotlin/Native state restrictions. 

Kotlin/Native 运行在 C++ 之上，对 C 和 Objective-C 提供的互操作。在 iOS平台上，你也可以从 Swift 传递 lambda 参数到共享代码。所有这些原生代码运行，都不受 Kotlin/Native 状态相关的约束。

That means that you can implement a concurrent mutable state in a native language and have Kotlin/Native talk to it.

所以你可以使用原生语言实现并发可变状态，再使用 Kotlin/Native 和它交互。

You can use [Objective-C interop](https://kotlinlang.org/docs/reference/native/c_interop.html) to access low-level code. 
You can also use Swift to implement Kotlin interfaces or pass in lambdas that Kotlin code can call 
from any thread.

可以使用 [Objective-C interop](https://kotlinlang.org/docs/reference/native/c_interop.html) 访问底层代码。你也可以用 Swift 实现 Kotlin 的接口，或者传递 lambda 给 Kotlin 代码执行，能在任意线程上被调用。

One of the benefits of a platform-native approach is performance. On the negative side, you'll need to manage concurrency on your own. 
Objective-C does not know about `frozen`, but if you store states from Kotlin in Objective-C structures, and share them 
between threads, the Kotlin states definitely need to be frozen. 
Kotlin/Native's runtime will generally warn you about issues, but it's possible 
to cause concurrency problems in native code that are very, very difficult to track down. It is also very easy to create 
memory leaks.

使用平台原生方法实现有一个好处就是性能。坏处就是你需要自己管理并发。
Objective-C 完全不知道 `frozen` 的概念，当你在 Kotlin 里以 Objective-C 结构储存状态，并在线程间共享时，Kotlin 的状态显然需要冻结。
Kotlin/Native 运行时通常只会给你一个警告，但可能会导致 native 代码里产生非常非常难以追踪的并发问题。也很容易产生内存泄漏。

Since in the KMM application you are also targeting the JVM, you'll need alternate ways to implement anything you use 
platform native code for. This will obviously take more work and may lead to platform inconsistencies.

KMM 应用同时需要 target 到 JVM，使用平台原生方式实现的逻辑还要给 JVM 准备一份。这显然会增加工作量， 还可能导致两端行为不一致。

_This material was prepared by [Touchlab](https://touchlab.co/) for publication by JetBrains._

