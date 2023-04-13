## Chapter 11. Concurrency（并发）

# 第八十节: Executor、task、流优于直接使用线程

本书的第一版包含一个简单工作队列的代码 [Bloch01, Item 49]。这个类允许客户端通过后台线程为异步处理排队。当不再需要工作队列时，客户端可以调用一个方法，要求后台线程在完成队列上的任何工作后优雅地终止自己。这个实现只不过是一个玩具，但即便如此，它也需要一整页的代码，如果你做得不对，就很容易出现安全和活性失败。幸运的是，没有理由再编写这种代码了。

当这本书的第二版出版时，`java.util.concurrent` 已经添加到 Java 中。这个包有一个 Executor 框架，它是一个灵活的基于接口的任务执行工具。创建一个工作队列，它在任何方面都比在这本书的第一版更好，只需要一行代码：

```
ExecutorService exec = Executors.newSingleThreadExecutor();

Here is how to submit a runnable for execution:
exec.execute(runnable);

And here is how to tell the executor to terminate gracefully (if you fail to do this,it is likely that your VM will not exit):
exec.shutdown();
```

你可以使用 executor 服务做更多的事情。例如，你可以等待一个特定任务完成（使用 get 方法，参见 [Item-79](../Chapter-11/Chapter-11-Item-79-Avoid-excessive-synchronization)，319 页），你可以等待任务集合中任何或全部任务完成（使用 invokeAny 或 invokeAll 方法），你可以等待 executor 服务终止（使用 awaitTermination 方法），你可以一个接一个检索任务，获取他们完成的结果（使用一个 ExecutorCompletionService），还可以安排任务在特定时间运行或定期运行（使用 ScheduledThreadPoolExecutor），等等。

如果希望多个线程处理来自队列的请求，只需调用一个不同的静态工厂，该工厂创建一种称为线程池的不同类型的 executor 服务。你可以使用固定或可变数量的线程创建线程池。`java.util.concurrent.Executors` 类包含静态工厂，它们提供你需要的大多数 executor。但是，如果你想要一些不同寻常的东西，你可以直接使用 ThreadPoolExecutor 类。这个类允许你配置线程池操作的几乎每个方面。

为特定的应用程序选择 executor 服务可能比较棘手。对于小程序或负载较轻的服务器，`Executors.newCachedThreadPool` 通常是一个不错的选择，因为它不需要配置，而且通常「做正确的事情」。但是对于负载沉重的生产服务器来说，缓存的线程池不是一个好的选择！在缓存的线程池中，提交的任务不会排队，而是立即传递给线程执行。如果没有可用的线程，则创建一个新的线程。如果服务器负载过重，所有 CPU 都被充分利用，并且有更多的任务到达，就会创建更多的线程，这只会使情况变得更糟。因此，在负载沉重的生产服务器中，最好使用 `Executors.newFixedThreadPool`，它为你提供一个线程数量固定的池，或者直接使用 ThreadPoolExecutor 类来实现最大限度的控制。

你不仅应该避免编写自己的工作队列，而且通常还应该避免直接使用线程。当你直接使用线程时，线程既是工作单元，又是执行它的机制。在 executor 框架中，工作单元和执行机制是分开的。关键的抽象是工作单元，即任务。有两种任务：Runnable 和它的近亲 Callable（与 Runnable 类似，只是它返回一个值并可以抛出任意异常）。执行任务的一般机制是 executor 服务。如果你从任务的角度考虑问题，并让 executor 服务为你执行这些任务，那么你就可以灵活地选择合适的执行策略来满足你的需求，并在你的需求发生变化时更改策略。本质上，Executor 框架执行的功能与 Collections 框架聚合的功能相同。

在 Java 7 中，Executor 框架被扩展为支持 fork-join 任务，这些任务由一种特殊的 Executor 服务（称为 fork-join 池）运行。由 ForkJoinTask 实例表示的 fork-join 任务可以划分为更小的子任务，由 ForkJoinPool 组成的线程不仅处理这些任务，而且还从其他线程「窃取」任务，以确保所有线程都处于繁忙状态，从而提高 CPU 利用率、更高的吞吐量和更低的延迟。编写和调优 fork-join 任务非常棘手。并行流（[Item-48](../Chapter-7/Chapter-7-Item-48-Use-caution-when-making-streams-parallel)）
是在 fork 连接池之上编写的，假设它们适合当前的任务，那么你可以轻松地利用它们的性能优势。

对 Executor 框架的完整处理超出了本书的范围，但是感兴趣的读者可以在实践中可以参阅《Java Concurrency in Practice》 [Goetz06]。
