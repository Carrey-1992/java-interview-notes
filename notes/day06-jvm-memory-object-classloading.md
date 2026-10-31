# 第 6 天 · 10-31 · JVM：内存区域、对象创建与布局、类加载

## 要点

### 1. 运行时数据区（按 JVM 规范 + HotSpot 实现）
- **线程私有**：程序计数器（唯一不会 OOM 的区域）、虚拟机栈（栈帧 = 局部变量表 + 操作数栈 + 动态链接 + 返回地址；递归过深抛 `StackOverflowError`，`-Xss` 控制）、本地方法栈（HotSpot 与虚拟机栈合一）。
- **线程共享**：堆（对象实例和数组，GC 主战场，`-Xms/-Xmx`）、方法区（类元数据、运行时常量池）。
- **方法区的演进**：JDK 7 把字符串常量池、静态变量挪到堆；JDK 8 移除永久代（JEP 122），类元数据放到本地内存的 **Metaspace**（`-XX:MaxMetaspaceSize`，默认不设上限，只受物理内存约束）。
- **直接内存**：不属于运行时数据区，NIO `ByteBuffer.allocateDirect`、Netty 常用，受 `-XX:MaxDirectMemorySize` 限制，泄漏时堆看着正常但 RSS 一直涨。
- **各类 OOM 对应**：`Java heap space`（堆）、`Metaspace`（动态代理/CGLIB/Groovy 脚本反复生成类、类加载器泄漏）、`Direct buffer memory`、`unable to create native thread`（线程数或系统限制）、`GC overhead limit exceeded`。
- **实战**：容器里用 `-XX:MaxRAMPercentage` 而不是写死 `-Xmx`；线上加 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=...`；排查 RSS 涨用 `-XX:NativeMemoryTracking=summary` + `jcmd <pid> VM.native_memory`。

### 2. 对象创建流程（`new` 指令）
1. **类加载检查**：常量池里的符号引用对应的类是否已加载、链接、初始化，没有就先走类加载。
2. **分配内存**：大小在类加载后就确定。
   - 规整堆（Serial、ParNew 等带压缩的收集器）用**指针碰撞**；不规整（CMS 标记清除）用**空闲列表**。
   - 并发安全：**TLAB**（每个线程在 Eden 预先划一小块，默认开启 `-XX:+UseTLAB`），TLAB 用完再 CAS + 失败重试。
3. **零值初始化**：字段先有默认值，所以不赋值也能用。
4. **设置对象头**：Mark Word、类型指针（Klass Pointer）。
5. **执行 `<init>`**：构造方法、实例初始化块。
- 注意 3/4 和 5 之间可以被重排，这就是 DCL 单例必须加 `volatile` 的原因（第 4 天讲过）。
- **逃逸分析**：对象不逃逸时 JIT 可做标量替换（不在堆上分配）、锁消除；HotSpot 并没有真正的“栈上分配”，是靠标量替换实现的效果。

### 3. 对象内存布局（64 位 HotSpot）
- **对象头**：Mark Word 8 字节（哈希、GC 年龄 4 bit、锁状态，第 3 天讲过）+ 类型指针（开启压缩类指针时 4 字节）；数组再加 4 字节长度。
- **实例数据**：字段按宽度重排（long/double → int/float → short/char → byte/boolean → 引用），父类字段在前。
- **对齐填充**：对象大小是 8 字节的倍数。
- **经典算账**：`new Object()` 在默认开启压缩指针时 = 12 字节头 + 4 字节填充 = **16 字节**。
- **压缩指针**：`-XX:+UseCompressedOops` 在堆小于约 32GB 时默认开启，引用 4 字节；超过 32GB 自动失效，所以“堆设 31G 可能比 33G 能放更多对象”。
- **新变化**：JDK 25 的 Compact Object Headers（JEP 519，转正；JDK 24 是实验特性 JEP 450）把对象头压到 8 字节，`-XX:+UseCompactObjectHeaders` 开启，小对象多的服务能省不少堆。
- 用 **JOL**（`ClassLayout.parseInstance(obj).toPrintable()`）验证布局，面试时可以说“我用 JOL 看过”。

### 4. 类加载
- **生命周期**：加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载（验证、准备、解析合称链接）。
  - **准备**：给 static 变量分配内存并赋零值；`static final` 编译期常量此时直接赋真值。
  - **初始化**：执行 `<clinit>`（static 赋值 + static 块），JVM 保证加锁只执行一次，所以静态内部类单例线程安全；`<clinit>` 里死锁或抛异常会导致 `NoClassDefFoundError`。
  - **主动使用才初始化**：new、读写非常量 static 字段、调 static 方法、反射、初始化子类先初始化父类、main 所在类。被动引用不触发：通过子类引用父类 static 字段、定义数组 `new A[10]`、引用编译期常量。
- **类加载器（JDK 9+）**：Bootstrap → Platform（原 Extension）→ Application；JDK 9 模块化（JEP 261）后委派规则变成先看包属于哪个模块。
- **双亲委派**：先交给父加载器，父加载不了才自己加载。好处是核心类不会被篡改、同一个类不会被重复加载。判断“同一个类”= 全限定名 + 加载它的类加载器。
- **打破双亲委派的场景**：
  - SPI（JDBC `DriverManager`、`ServiceLoader`）用**线程上下文类加载器**让 Bootstrap 加载的接口找到应用层实现；
  - Tomcat 的 WebappClassLoader 先自己加载，实现应用之间隔离；
  - OSGi、热部署、Arthas 等；
  - 写法上重写 `loadClass` 就是打破，只重写 `findClass` 是遵守。
- **类卸载**：该类所有实例被回收 + 加载它的 ClassLoader 被回收 + Class 对象无引用。所以自定义类加载器泄漏是 Metaspace OOM 的常见根因（如反复热部署、Groovy 脚本每次 new GroovyClassLoader）。

## 面试题
1. 一个线上服务堆监控正常，但容器 RSS 一直涨直到被 OOMKilled。可能是哪些区域？你会怎么一步步排查？
2. `Object o = new Object();` 从字节码到对象可用经历了哪些步骤？这个对象在 64 位 JVM 上占多少字节？堆从 31G 调到 33G 会发生什么？
3. 什么是双亲委派？JDBC 的 SPI 和 Tomcat 分别为什么、怎么打破它？如果线上出现 Metaspace OOM，你会怀疑什么？

## 参考
- [JVM 规范第 2 章：运行时数据区](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html)
- [JVM 规范第 5 章：加载、链接与初始化](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html)
- [JEP 122：移除永久代](https://openjdk.org/jeps/122)
- [JEP 261：模块系统](https://openjdk.org/jeps/261)
- [JEP 450：紧凑对象头（实验）](https://openjdk.org/jeps/450)
- [JEP 519：紧凑对象头（正式）](https://openjdk.org/jeps/519)
- [JOL：Java Object Layout](https://github.com/openjdk/jol)
