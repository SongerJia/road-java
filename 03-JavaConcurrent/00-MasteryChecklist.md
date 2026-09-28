# 精通清单 · Java 并发

> **使用说明**
> 1. 合上资料，录音回答每题 → 对照三级标准自评（及格 / 良好 / 精通）
> 2. 未达「精通」的题 → 回锚点材料补 → **3 天后重答**
> 3. 单题标准：**及格** = 核心结论 + 主干要点；**良好** = 及格 + 能解释「为什么」+ 一个应用/坑；**精通** = 良好 + 源码级细节 + 连环追问不卡壳
>
> **本板块通关线**：6 题全部达到「精通」档（同见根目录全局标准）

---

## 1. `volatile` 保证什么、不保证什么？为什么？

**锚点**：JLS §17.4 / 《Java 并发编程的艺术》第 3 章

- [ ] **及格**：保证**可见性**（修改立即刷主内存）+ **有序性**（禁止指令重排）；**不保证原子性**（`count++` 依然不安全）
- [ ] **良好**：及格 + 能解释实现原理——**内存屏障**（写：StoreStore + StoreLoad；读：LoadLoad + LoadStore）+ 双重检查锁单例为什么必须加 volatile（`new` 三步可能重排，需要禁止）+ 适用场景（状态标志、发布不可变对象）
- [ ] **精通**：良好 + 能应对追问——字节码层面 `ACC_VOLATILE` 标志、硬件层面 MESI 缓存一致性协议、**volatile 数组**的问题（数组引用 volatile 不代表元素 volatile）、`long`/`double` 的原子性（JVM 规范保证 8 字节原子读写，但实际建议 volatile）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 2. `synchronized` 底层原理？锁升级过程？

**锚点**：HotSpot 源码 / 《深入理解 Java 虚拟机》第 13 章

- [ ] **及格**：基于 **Monitor（监视器）** 实现；可修饰方法/代码块；**可重入**；Java 6 后有锁升级
- [ ] **良好**：及格 + 能说出锁升级路径——**无锁 → 偏向锁 → 轻量级锁（自旋）→ 重量级锁** + `Mark Word` 存锁状态 + 重量级锁依赖 OS 互斥量（会阻塞、有上下文切换开销）
- [ ] **精通**：良好 + 能应对追问——**为什么要有偏向锁**（无竞争时省 CAS）、**自旋的代价**（占 CPU，适合临界区短）、锁消除/锁粗化（JIT 优化）、`synchronized` vs `ReentrantLock` 区别（是否可中断、是否公平、Condition）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 3. AQS 的核心是什么？加锁/解锁流程？

**锚点**：`AbstractQueuedSynchronizer.java` 源码

- [ ] **及格**：核心三件套——**state（volatile int）+ CLH 变体等待队列 + 钩子方法**（`tryAcquire`/`tryRelease`，子类实现）；加锁流程（尝试获取 → 失败入队 → 阻塞），解锁流程（释放 → 唤醒后继）
- [ ] **良好**：及格 + 能说出入队细节（`addWaiter` CAS 入队 + `enq` 自旋）、阻塞细节（`acquireQueued` 循环 + `shouldParkAfterFailedAcquire` + `LockSupport.park`）、公平 vs 非公平锁区别（`hasQueuedPredecessors`）、共享 vs 独占模式
- [ ] **精通**：良好 + 能应对追问——AQS 为什么用 CLH **变体**（原版自旋，AQS 改 park/unpark 阻塞；原版单向，AQS 双向）、`unparkSuccessor` 为什么从**尾部往前**找（`next` 不可靠，`prev` 是兜底）、`state` 在 Semaphore（剩余许可）/ CountDownLatch（待计数）中的语义

**自评记录**：___（档位） 日期：___ 复答：___

---

## 4. 线程池 7 大参数？执行流程？拒绝策略？

**锚点**：`ThreadPoolExecutor.java` 源码

- [ ] **及格**：7 大参数——`corePoolSize`、`maximumPoolSize`、`keepAliveTime`、`unit`、`workQueue`、`threadFactory`、`handler`；执行流程——**核心线程 → 阻塞队列 → 最大线程 → 拒绝策略**
- [ ] **良好**：及格 + 能解释**为什么是这个顺序**（先复用核心、再排队缓冲、再临时扩线程，最后拒绝）+ 四种拒绝策略（Abort/CallerRuns/Discard/DiscardOldest）+ 线程池状态（RUNNING/SHUTDOWN/STOP/TIDYING/TERMINATED）
- [ ] **精通**：良好 + 能应对追问——`execute` vs `submit`（submit 封装 Future，异常吞进返回结果）、核心线程会不会回收（默认不会，`allowCoreThreadTimeOut` 可开）、**如何配置参数**（CPU 密集 vs IO 密集的经验公式）、线程池里异常怎么处理、为什么队列要选阻塞队列（生产者-消费者解耦）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 5. happens-before 8 条规则？

**锚点**：JLS §17.4.5 / 《Java 并发编程的艺术》

- [ ] **及格**：能默写出 8 条——① 程序顺序 ② 监视器锁 ③ volatile 变量 ④ 传递性 ⑤ 线程启动 ⑥ 线程终止 ⑦ 线程中断 ⑧ 对象终结
- [ ] **良好**：及格 + 能解释每条的含义（如「锁规则」：解锁 happens-before 后续加锁）+ 用一个实际例子演示（如 volatile 单例为什么安全：写 happens-before 读）
- [ ] **精通**：良好 + 能应对追问——**为什么需要 happens-before**（定义 JMM 的可见性契约，让并发代码有可推理的顺序）、和 `as-if-serial` 的关系（单线程语义 vs 跨线程语义）、happens-before 是**偏序关系**、`Thread.join()` 属于哪条规则（线程终止）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 6. CAS 原理？ABA 问题？

**锚点**：`Unsafe.java` / `AtomicStampedReference.java` 源码

- [ ] **及格**：CAS = Compare And Swap，比较并交换；三个值（内存地址 / 期望值 / 新值）；由 `Unsafe.compareAndSwapInt` 实现，**硬件层面原子**；ABA 问题 = 值从 A 变 B 再变回 A，CAS 无法察觉
- [ ] **良好**：及格 + 能解释 CAS 的应用（`AtomicInteger`、并发容器）+ 自旋重试模式 + ABA 的解决（`AtomicStampedReference` 带版本号/`AtomicMarkableReference`）+ ABA 实际危害场景（如链表头指针被复用）
- [ ] **精通**：良好 + 能应对追问——CAS 的**开销**（自旋占 CPU、伪共享 false sharing）+ `LongAdder` 为什么比 `AtomicLong` 快（Cell 数组分散热点，最终求和）+ CAS 与 `synchronized` 的取舍（无竞争 CAS 快，高竞争 synchronized 更稳）

**自评记录**：___（档位） 日期：___ 复答：___

---

> **通关检查**：本板块 6 题是否全部达到「精通」档？是 → 进入下一板块；否 → 标记未达标的题号：___
