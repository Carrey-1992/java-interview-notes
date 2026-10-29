# 第 3 天 · 10-28 · synchronized 与 AQS

## 要点

- **Mark Word**：对象头里存哈希码、GC 年龄、锁标志位；锁状态变化就是 Mark Word 的变化
- **锁升级**：无锁 → 偏向锁 → 轻量级锁（CAS + 自旋，锁记录在栈帧）→ 重量级锁（Monitor，线程阻塞），一般只升不降
- **JEP 374**：JDK 15 默认关闭并废弃偏向锁，后续版本已移除，所以新版本面试要说明这一点
- **虚拟线程 pinning**：JDK 21 中虚拟线程在 synchronized 内阻塞会钉住载体线程；JEP 491（JDK 24）解决了这个问题

- AQS：`volatile int state` + CLH 变体的 FIFO 双向等待队列，子类实现 `tryAcquire` / `tryRelease`
- ReentrantLock：state 计重入次数；公平锁先看队列有没有人（`hasQueuedPredecessors`），非公平锁直接 CAS 抢
- 相比 synchronized：可中断、可超时 `tryLock`、可公平、多个 Condition
- Condition：`await` 释放锁进入条件队列，`signal` 把节点转回同步队列重新抢锁；必须在持锁时调用

## 面试题

1. synchronized 的锁升级过程是怎样的？偏向锁现在还存在吗？
2. AQS 的核心原理是什么？ReentrantLock 的公平锁和非公平锁区别在哪？
3. synchronized 和 ReentrantLock 怎么选？虚拟线程场景下有什么要注意的？

## 参考

- [JEP 374: Deprecate and Disable Biased Locking](https://openjdk.org/jeps/374)
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
