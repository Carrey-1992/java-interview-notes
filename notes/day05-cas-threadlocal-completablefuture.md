# 第 5 天 · 10-30 · CAS/Atomic/LongAdder、ThreadLocal 与 CompletableFuture

## 要点

### CAS 与原子类

- **CAS**（Compare-And-Swap）：比较内存值是否等于期望值，相等才写入新值，整个动作由 CPU 指令保证原子（x86 上是 `lock cmpxchg`）。Java 里早期靠 `Unsafe`，JDK 9 起推荐 `VarHandle`
- **AtomicInteger/AtomicLong**：`value` 用 volatile 保证可见性，`incrementAndGet` 失败就自旋重试，无锁但不是无代价
- **CAS 三个问题**：ABA（用 `AtomicStampedReference` 加版本号，或 `AtomicMarkableReference`）；高并发下自旋失败多，空耗 CPU；只能保证单个变量，多个变量要合成一个对象用 `AtomicReference` 或者加锁
- **LongAdder**：分段思想，低竞争时只 CAS `base`；竞争时按线程哈希落到 `Cell[]` 里不同槽位各自累加，`Cell` 用 `@Contended` 填充避免伪共享；`sum()` 把 base 和所有 Cell 相加，**并发更新时不是精确快照**
- **选型**：高并发计数（QPS、监控指标）用 LongAdder；需要精确读、`compareAndSet` 或做序列号时用 AtomicLong；自定义累加规则用 `LongAccumulator`

### ThreadLocal

- **结构**：数据不在 ThreadLocal 里，而在每个 `Thread` 的 `threadLocals`（一个 `ThreadLocalMap`）里，key 是 ThreadLocal 实例，value 是值；用开放地址法（线性探测）解决冲突，哈希增量是 `0x61c88647`，让槽位分布均匀
- **Entry 的 key 是弱引用，value 是强引用**：ThreadLocal 对象没人引用时 key 被 GC 回收，但 value 还被“线程 → Map → Entry → value”强引用着；`get/set/remove` 时会顺带清理 key 为 null 的过期条目，但不保证清干净
- **内存泄漏与串数据**：线程池里的线程长期存活，不 `remove` 就会泄漏，还会让下一个任务读到上一个用户的数据（比如登录用户上下文）。规范写法是 `try { set } finally { remove() }`
- **为什么 key 用弱引用**：如果 key 是强引用，ThreadLocal 对象本身也回收不了；弱引用至少保证 key 能回收，并给清理过期条目留了机会
- **跨线程传递**：`InheritableThreadLocal` 只在创建子线程时复制一次，线程池复用线程时会传错或传旧值；实战常用阿里开源的 TransmittableThreadLocal（TTL）包装线程池，或者手动在提交任务时拷贝上下文（MDC 的 traceId 透传同理）
- **虚拟线程**：虚拟线程数量可能上百万，每个都带 ThreadLocal 成本很高；JDK 25 正式提供 `ScopedValue`（JEP 506），不可变、有作用域，更适合传上下文

### CompletableFuture

- **创建**：`supplyAsync/runAsync` 不传线程池时默认用 `ForkJoinPool.commonPool()`（并行度通常是 CPU 核数减 1），所有地方共享；**IO 型业务一定要传自定义线程池**，否则一个慢接口就能拖垮别的异步任务
- **编排**：`thenApply`（转换结果）、`thenCompose`（串联另一个异步任务，避免嵌套 Future）、`thenCombine`（两个结果合并）、`allOf/anyOf`（等全部或任意一个）
- **线程**：不带 Async 的回调可能在完成任务的线程或调用线程里执行；带 Async 的回调交给指定或默认线程池
- **异常**：`exceptionally`（兜底返回默认值）、`handle`（同时拿结果和异常）、`whenComplete`（只观察不改结果）；异常会被包成 `CompletionException`，`join` 抛未检查异常，`get` 抛 `ExecutionException`
- **超时**：JDK 9 起有 `orTimeout`（超时就异常完成）和 `completeOnTimeout`（超时就给默认值）；JDK 8 只能用 `get(timeout)` 或自己调度
- **典型场景**：商品详情页并行调用价格、库存、评价等多个下游，`allOf` 汇总，每路单独超时和降级，整体再设一个总超时

## 面试题

1. AtomicLong 和 LongAdder 的实现原理有什么区别？为什么高并发下 LongAdder 更快？它的 `sum()` 准确吗？什么场景你仍然选 AtomicLong？顺带说说 CAS 的 ABA 问题怎么解决。
2. ThreadLocal 为什么会内存泄漏？Entry 的 key 为什么设计成弱引用？在线程池里用 ThreadLocal 存用户上下文会有哪些坑，怎么把 traceId 这类上下文传到异步线程里？
3. 一个接口要并行调用 3 个下游服务再合并结果，用 CompletableFuture 怎么写？如何给每路设置超时和降级？为什么不建议用默认线程池？

## 参考

- [LongAdder（Java SE 21 API 文档）](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/LongAdder.html)
- [VarHandle（Java SE 21 API 文档）](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html)
- [ThreadLocal（Java SE 21 API 文档）](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html)
- [CompletableFuture（Java SE 21 API 文档）](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506)
- [alibaba/transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local)
