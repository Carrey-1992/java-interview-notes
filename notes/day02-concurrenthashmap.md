# 第 2 天 · 10-27 · ConcurrentHashMap

## 要点

| 对比 | JDK7 | JDK8 |
| --- | --- | --- |
| 加锁方式 | Segment 分段锁（继承 ReentrantLock） | CAS + synchronized 锁桶头节点 |
| 粒度 | Segment 级，并发度默认 16 | 单个桶 |
| 结构 | Segment 数组 + HashEntry 链表 | Node 数组 + 链表 + 红黑树 |

- put：槽位为空用 CAS 放入；遇到 ForwardingNode 先协助扩容；否则 synchronized 锁头节点后插入
- get 不加锁：Node 的 `val` 和 `next` 是 volatile，读到的总是最新值
- sizeCtl：-1 表示正在初始化，其他负数表示正在扩容，正数是下次扩容阈值
- 协助扩容：迁移完的桶放 ForwardingNode，其他线程 put 时遇到会领一段区间一起迁移
- 计数：baseCount + CounterCell 数组，思路类似 LongAdder，减少 CAS 竞争
- key 和 value 都不允许 null：并发下无法区分“不存在”和“值为 null”
- computeIfAbsent 坑：映射函数里再修改同一个 map 可能死循环或抛 IllegalStateException（JDK8 有 bug，JDK9 起检测递归更新）

## 面试题

1. JDK7 和 JDK8 的 ConcurrentHashMap 实现有什么区别？为什么 JDK8 放弃分段锁？
2. ConcurrentHashMap 的 get 为什么不需要加锁？
3. 多线程是怎么协助扩容的？size 是怎么统计的？
