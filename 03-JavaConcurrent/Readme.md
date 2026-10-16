> **模块边界**：`CompletableFuture` / `parallelStream` / 响应式（`Flow` / Reactor）→ 新特性模块；类加载与线程上下文类加载器 → JVM 模块；集合的常规实现（非并发）→ 集合模块。本模块只讲 `java.util.concurrent` 及并发语义本身。

**① 并发基础**

1. 进程 vs 线程
2. 并发 vs 并行
3. 上下文切换及其开销
4. 线程的六种状态与生命周期
5. 线程创建方式：Thread / Runnable / Callable
6. Thread 常用方法：start / run / sleep / join / yield
7. 中断机制：interrupt / interrupted / isInterrupted、中断标志清除、InterruptedException 处理
8. 废弃的 stop / suspend / resume 及原因
9. 守护线程与用户线程
10. 线程优先级与调度

**② Java 内存模型（JMM）** 11. 主内存与工作内存 12. 并发三大特性：原子性、可见性、有序性 13. 指令重排序与 as-if-serial 14. happens-before 规则 15. 内存屏障 16. JMM 与 CPU 缓存、MESI 协议的对应 17. volatile 的语义与原理、double-checked locking 单例 18. final 的内存语义 19. JMM 的 8 种原子操作（了解）

**③ synchronized 与锁机制** 20. synchronized 的使用方式（方法 / 代码块） 21. synchronized 原理：monitor、monitorenter / monitorexit 22. 对象头与 Mark Word 23. 锁升级：偏向锁 → 轻量级锁 → 重量级锁（含 JDK 15 默认禁用、JDK 18 移除偏向锁的版本说明） 24. 偏向锁的撤销与锁膨胀细节 25. 锁优化：锁消除、锁粗化、自适应自旋 26. synchronized 的可重入 27. synchronized vs Lock 的对比

**④ 等待通知与线程通信** 28. Object 的 wait / notify / notifyAll 机制 29. 虚假唤醒与 while 循环判条件 30. Condition 与 wait / notify 的对比

**⑤ 锁的分类与死锁** 31. 乐观锁 vs 悲观锁 32. 公平锁 vs 非公平锁 33. 独占锁 vs 共享锁 34. 自旋锁与可重入锁 35. 死锁的四个必要条件与避免 36. 活锁与饥饿

**⑥ CAS 与原子类** 37. CAS 原理与 Unsafe 38. ABA 问题与 AtomicStampedReference 39. 原子基本类型与原子引用 40. 原子数组与字段更新器 41. LongAdder 分段累加 42. 伪共享（False Sharing）与 @Contended 43. 原子类的演进：Unsafe → VarHandle（JDK 9） 44. ThreadLocalRandom 与并发随机数

**⑦ AQS 与同步器** 45. LockSupport：park / unpark 46. AQS 原理：state + CLH 变体队列、Node waitStatus、入队与唤醒 47. 独占模式与共享模式 48. ReentrantLock：公平 / 非公平、可中断、超时、Condition、Condition 条件队列 49. ReentrantReadWriteLock 与锁降级 50. StampedLock 51. Semaphore（公平性与限流应用） 52. CountDownLatch（一次性） 53. CyclicBarrier（可循环、与 CountDownLatch 的对比） 54. Phaser / Exchanger

**⑧ 并发容器** 55. ConcurrentHashMap（JDK 7 分段锁 vs JDK 8 CAS + synchronized） 56. CopyOnWriteArrayList / CopyOnWriteArraySet 57. ConcurrentLinkedQueue 58. ConcurrentSkipListMap / Set 59. BlockingQueue 家族逐个对比（容量 / 锁粒度 / 是否排序或延迟） 60. 并发容器的弱一致性迭代器 61. 线程安全等级分类（不可变 / 绝对 / 相对 / 线程兼容 / 线程对立）

**⑨ 线程池** 62. Executor 框架总览 63. ThreadPoolExecutor 核心参数 64. 任务提交与执行流程 65. 四种拒绝策略 66. 线程池状态与线程数量变化 67. Worker 与线程复用机制（继承 AQS + getTask） 68. 线程池大小配置原则 69. 线程池任务队列的选择（有界 / 无界） 70. Executors 工厂方法与弊端 71. 线程池的优雅关闭（shutdown / shutdownNow / awaitTermination） 72. ScheduledThreadPoolExecutor 73. ForkJoinPool 与工作窃取 74. 线程池监控

**⑩ 线程级工具** 75. ThreadLocal 原理 76. ThreadLocal 内存泄漏与弱引用 77. InheritableThreadLocal 78. Thread.onSpinWait（JDK 9）

**⑪ 异步与并发模式** 79. Future / FutureTask 及其局限 80. 生产者-消费者模式 81. 两阶段终止与保护性暂停 82. 线程安全的实现手段（无状态、不可变、ThreadLocal）与安全发布 83. 死锁的运行时检测（ThreadMXBean.findDeadlockedThreads） 84. 并发问题排查（jstack、线程 dump） 85. 并发性能观察（上下文切换、CAS 自旋、锁竞争开销）｜**异步演进链与 CompletableFuture / parallelStream → 新特性模块**