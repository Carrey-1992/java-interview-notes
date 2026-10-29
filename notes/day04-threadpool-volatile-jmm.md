# 第 4 天 · 10-29 · 线程池与 volatile/JMM

## 要点

### 线程池 ThreadPoolExecutor

- **7 个参数**：corePoolSize、maximumPoolSize、keepAliveTime + unit、workQueue、threadFactory、handler（拒绝策略）
- **ctl**：一个 `AtomicInteger`，高 3 位存运行状态（RUNNING / SHUTDOWN / STOP / TIDYING / TERMINATED），低 29 位存工作线程数
- **execute 流程**：线程数 < core 就新建 Worker；否则入队；入队失败（队列满）再尝试扩到 max；还失败就走拒绝策略。入队后会二次检查状态，线程数为 0 时补一个非核心 Worker
- **反直觉点**：先排队再扩容，所以用无界 `LinkedBlockingQueue` 时 maximumPoolSize 根本不生效；Tomcat 的线程池改写了这个顺序（先扩线程再排队）
- **Worker**：继承 AQS 做不可重入锁，用来区分“正在跑任务”和“空闲”，`shutdown` 只中断空闲线程；`runWorker` 循环 `getTask()`，超时拿不到任务的非核心线程被回收（`allowCoreThreadTimeOut` 可让核心线程也回收）
- **4 种拒绝策略**：AbortPolicy（默认，抛异常）、CallerRunsPolicy（调用方线程执行，起到反压作用）、DiscardPolicy、DiscardOldestPolicy
- **实战坑**：不要用 `Executors.newFixedThreadPool` / `newCachedThreadPool`（无界队列或无界线程数，可能 OOM）；`submit` 的异常被包进 Future，不 `get` 就看不到；线程命名要用自定义 ThreadFactory；ThreadLocal 在池化线程里要 remove；参数按 CPU 密集（约核数+1）与 IO 密集（压测定）区分，最好做成可动态调整（`setCorePoolSize` 等）并监控队列长度、活跃数、拒绝次数
- **虚拟线程（JDK 21，JEP 444）**：不要池化虚拟线程，用 `Executors.newVirtualThreadPerTaskExecutor()`，限流改用 Semaphore

### volatile 与 JMM

- **JMM 解决三件事**：可见性、有序性、原子性；核心是 happens-before 规则（程序次序、监视器锁、volatile、线程 start/join、传递性等）
- **volatile 保证**：可见性 + 禁止特定重排序（写前插 StoreStore、写后 StoreLoad，读后 LoadLoad + LoadStore），**不保证复合操作原子性**（`count++` 仍不安全，要用 AtomicInteger/LongAdder 或加锁）
- **DCL 单例**：`instance = new Singleton()` 可拆成分配内存、初始化、赋引用三步，可能重排成先赋引用，所以字段必须 `volatile`
- **硬件层面**：x86 上 volatile 写通常编译成带 `lock` 前缀的指令，起到 StoreLoad 屏障作用并让其他核缓存行失效（MESI 缓存一致性）
- **和 synchronized 对比**：volatile 轻量，只适合“一写多读”的状态标志、DCL 等；需要互斥和原子性时用锁或 CAS

## 面试题

1. 线程池 `execute` 提交一个任务后的完整执行流程是什么？为什么用 `LinkedBlockingQueue` 时最大线程数可能不起作用？生产上你会怎么设置参数和监控？
2. 线程池里任务抛了异常，`execute` 和 `submit` 的表现有什么不同？线程会怎样？怎么统一捕获？
3. volatile 能保证什么、不能保证什么？DCL 单例为什么必须加 volatile？从 happens-before 角度说明。

## 参考

- [ThreadPoolExecutor（Java SE 21 API 文档）](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)
- [JLS 第 17 章 Threads and Locks（Java 内存模型）](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
- [JSR 133 (Java Memory Model) FAQ](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
