> **模块边界**：JMM / happens-before / volatile 语义 / 锁的内存语义 → 并发模块；`javac` 编译与语言语法 → 基础模块；类加载器在 Tomcat 等容器中的具体层次 → 对应中间件模块。本模块只讲 HotSpot 的运行时机制。

**① JVM 概述与体系结构**

1. JVM 是什么：规范与实现（HotSpot / OpenJ9 / GraalVM）
2. JVM 整体架构（类加载子系统 / 运行时数据区 / 执行引擎 / 本地接口）
3. JVM 生命周期
4. HotSpot 的执行模式（解释器 + JIT）
5. JVM 内存模型 ≠ Java 内存模型（概念边界澄清）
6. **JVM 版本演进主线（PermGen → Metaspace、CMS → G1 → ZGC）**

**② 运行时数据区** 7. 程序计数器 8. 虚拟机栈与栈帧组成（局部变量表 / 操作数栈 / 动态链接 / 返回地址 / 异常表） 9. 本地方法栈 10. 堆的结构（新生代 / 老年代） 11. 方法区与永久代 / 元空间的关系（规范 vs 实现） 12. 常量池三兄弟：Class 文件常量池 / 运行时常量池 / 字符串常量池 13. 直接内存 14. 各区域线程私有 vs 共享、OOM 情况

**③ 对象** 15. 对象创建过程 16. 内存分配方式（指针碰撞 / 空闲列表） 17. TLAB 与并发分配 18. 对象内存布局与对象头细节（Mark Word：hashCode / GC 分代年龄 / 锁状态标志） 19. 对象访问定位 20. 堆分配策略：Eden / S0 / S1、晋升与年龄阈值、大对象直接进老年代、动态年龄判定、空间分配担保 21. 对象存活判定（引用计数 / 可达性分析） 22. GC Roots 有哪些 23. 引用计数为何不被采用（循环引用） 24. 引用类型（强 / 软 / 弱 / 虚） 25. 逃逸分析与栈上分配、标量替换

**④ 垃圾回收** 26. GC 基础与 STW 27. 垃圾回收算法：标记-清除 / 复制 / 标记-整理 28. 分代收集理论与跨代引用（Remembered Set / Card Table） 29. Serial / ParNew / Parallel Scavenge 30. Serial Old / Parallel Old 31. CMS 四阶段、缺点、浮动垃圾与 Concurrent Mode Failure 32. G1：Region、Remembered Set、Card Table、SATB、Young / Mixed / Full 回收模式 33. ZGC：染色指针与读屏障；Shenandoah 对比 34. JVM 各版本默认 GC 变迁（JDK 8 Parallel → JDK 9+ G1） 35. Minor / Major / Full GC 36. 三色标记法与并发标记问题（漏标 / 错标） 37. 安全点与安全区

**⑤ 类加载机制** 38. 类加载过程：加载 / 验证 / 准备 / 解析 / 初始化 39. 类加载器分类 40. 双亲委派模型 41. 打破双亲委派（SPI / Tomcat） 42. 类加载时机（主动 / 被动引用） 43. 类的卸载

**⑥ 字节码与执行引擎** 44. Class 文件结构 45. 字节码指令集 46. 解释器与 JIT（C1 / C2 / 分层编译） 47. JIT 热点探测（方法调用计数器 + 回边计数器） 48. 方法调用指令 49. 编译优化技术：方法内联及触发条件、逃逸分析、去虚拟化、公共子表达式消除、循环优化 50. GraalVM 与 AOT（生态演进）

**⑦ 性能监控与调优** 51. JVM 参数分类（标准 / -X / -XX）与常用参数 52. 监控工具全集（jps / jstat / jmap / jstack / jinfo / jcmd / jhat / jconsole + Arthas） 53. OOM 全类型与各自根因 54. 堆 dump 获取与分析（jmap -dump + MAT） 55. CPU 100% 排查流程 56. GC 日志与统一日志（-Xlog，JDK 9+） 57. 调优目标分层（吞吐量 / 低延迟 / 内存）与调优实践 58. 容器感知（JDK 10+ UseContainerSupport）