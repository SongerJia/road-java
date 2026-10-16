## 一、Java 语言基础（11 篇专题）

**A1 面向对象**（`Object Oriented.md`）

- 封装的语义与实现手段
- 继承：extends → 构造器链（父类构造器调可重写方法的危险）→ 重写规则 → 继承 vs 组合 → 继承破坏封装
- 多态：重载 vs 重写 → 向上转型 → 虚方法表 → 静态绑定 vs 动态绑定 → 多态在框架中的应用

**A2 接口 vs 抽象类**（`Interface&Abstract.md`）

- 语法对比矩阵（方法/变量/修饰符，JDK8 default·static、JDK9 private）
- 设计意图（can-do vs is-a）
- default 方法菱形冲突的解决规则
- 设计模式中的运用（模板方法等）+ 高频抉择题

**A3 异常体系**（`Exception.md`）

- 异常分类（Throwable → Error / Exception → 受检 / 非受检）
- try-with-resources（AutoCloseable，JDK7 前后写法对比）
- 异常链与性能（initCause、fillInStackTrace 开销、重写跳过栈扫描）
- 最佳实践 + finally return 陷阱等高频题

**A4 泛型**（`Generics.md`）

- 基础语法
- 类型擦除（为什么擦、运行时无泛型）
- 桥接方法（JVM 重写规则与语言层重写的差异、协变返回）
- 通配符与 PECS 原则
- 泛型的限制与项目实战

**A5 反射**（`Reflection.md`）

- 反射基础与核心 API（Class / Method / Field / Constructor）
- 应用场景（JDK 动态代理、框架底层）
- 性能问题与优化（Method 缓存、MethodHandle）
- 安全问题（setAccessible 绕过检查）

**A6 注解**（`Annotation.md`）

- 注解基础与五种元注解（@Target / @Retention / @Repeatable）
- 自定义注解与注解处理（APT 生成代码）
- 框架中的注解
- 注解 vs 其他方案

**A7 Object 类**（`Object.md`）

- 方法总览
- equals（重写规则、5 条性质）
- hashCode（约定、只重写 equals 的后果）
- clone（浅拷贝默认语义、深拷贝三种实现、Cloneable 缺陷）
- finalize（弃用与 Cleaner 替代）

**A8 基本类型与包装类**（`BasicType&PackageClass.md`）

- 8 种基本类型
- 自动装箱拆箱
- Integer 缓存池（-128~127、valueOf 源码）
- 性能对比与高频陷阱（== 比较装箱对象、自动拆箱 NPE）

**A9 枚举**（`Enum.md`）

- 枚举本质（final class 继承 Enum，反编译视角）
- 枚举的字段、方法与抽象方法式多态
- EnumMap / EnumSet
- 枚举单例（JVM 保证反序列化不 new → 序列化安全）
- 面试高频题

**A10 字符串**（`String.md`）

- String 不可变性
- 字符串常量池（JDK6/7/8 位置演变、intern 行为）
- StringBuilder vs StringBuffer（线程安全与性能）
- 字符串拼接优化
- 高频题（new String("a") 创建对象数等）

**A11 内部类**（`Inner Class.md`）

- 四种内部类语法
- 字节码视角（独立 class 文件、隐式持外部引用）
- 为什么需要内部类
- 内存泄漏问题
- 实际应用场景

---

## 二、集合框架（14 篇专题）

**B1 Collection 体系**（`Collection.md`）

- Iterable → Collection → List / Set / Queue 总纲，与 Map 并行的体系结构

**B2 HashMap**（`HashMap.md`）

- 底层数据结构（数组 + 链表 + 红黑树）
- 哈希算法与扰动函数（h ^ (h>>>16)）
- put / get 流程（懒加载、树化双条件 8 + 64）
- resize 扩容（2 的幂的原因、高低位链拆分 e.hash & oldCap）
- 树化与红黑树（泊松分布、退化阈值 6）
- JDK1.7 vs 1.8 对比（头插法死循环）
- 线程安全问题

**B3 ConcurrentHashMap**（`ConcurrentHashMap.md`）

- 1.7 Segment 分段锁 → 1.8 CAS + synchronized 锁桶头
- 读操作无锁（volatile）
- size 并发统计（CounterCell）、多线程协助扩容（ForwardingNode）

**B4 HashTable vs HashMap vs CHM**（同名文件）

- 设计思想 / 数据结构 / 锁机制三重对比
- null key 支持差异与选型结论

**B5 ArrayList vs LinkedList**（同名文件）

- 底层结构（动态数组 vs 双向链表）
- 扩容机制（1.5 倍 oldCap + oldCap>>1）
- 内存占用
- RandomAccess 标记接口
- 差异化结论

**B6 TreeMap vs TreeSet**（同名文件）

- 红黑树底层数据结构
- 排序规则（Comparable vs Comparator）
- TreeMap 特有功能（范围查询）、TreeSet 底层即 TreeMap

**B7 LinkedHashMap**（`LinkedHashMap.md`）

- 双向链表挂接 HashMap 桶
- 两种顺序模式（插入序 / 访问序 accessOrder）
- LRU 缓存实现（removeEldestEntry）
- 与 HashMap 对比

**B8 阻塞队列**（`BlockingQueue.md`）

- 核心 API（put/take 阻塞 vs offer/poll 超时）
- 7 种实现逐个详解（Array / Linked / Priority / Delay / Synchronous / LinkedTransfer / 双端）
- 应用场景与选型结论

**B9 CopyOnWriteArrayList**（同名文件）

- COW 写时复制原理
- 读无锁 / 写加锁复制的核心源码
- 迭代器快照特性（弱一致）
- 读多写少选型

**B10 HashSet vs LinkedHashSet vs TreeSet**（同名文件）

- 底层实现（分别包装 HashMap / LinkedHashMap / TreeMap）
- 顺序特性、null 支持、性能对比
- 各自特有功能与选型

**B11 优先级队列**（`PriorityQueue.md`）

- 堆的概念与数组实现（父子下标公式）
- offer / poll 的 siftUp / siftDown 流程
- 扩容、特性与限制（非线程安全）
- TopK 等应用场景

**B12 ArrayDeque vs LinkedList**（同名文件）

- Deque 概念
- 循环数组 vs 链表的底层结构与核心操作
- 作为栈 / 队列使用（Stack 继承 Vector 的设计缺陷）
- 选型结论

**B13 Iterator 与 fail-fast**（`Iterator 与 fail-fast.md`）

- 迭代器概念与 for-each 本质（语法糖反编译）
- fail-fast 机制（modCount / expectedModCount 源码）
- fail-safe 对比（COW 快照）
- 遍历中安全删除的正确姿势

**B14 Collections 工具类**（`Collections.md`）

- 排序与查找
- 不可变集合（视图语义）
- 同步包装
- 空集合与单元素集合、差异化结论

---

## 三、并发编程（13 篇专题）

**C1 线程基础**（`ThreadBasic.md`）

- 进程 vs 线程
- 线程模型（用户态 / 内核态、1:1 内核映射）
- 线程生命周期 6 状态及流转
- 创建方式（Thread / Runnable / Callable+Future）

**C2 JMM & volatile**（`JMM & Volatile.md`）

- JMM 是什么（主内存 / 工作内存抽象）
- 三特性（原子性 / 可见性 / 有序性）
- volatile 原理（内存屏障、硬件层协议）
- happens-before 规则（8 条）
- volatile vs synchronized

**C3 Synchronized & Monitor**（同名文件）

- synchronized 基础（三种用法、monitorenter/exit）
- 锁的本质（monitor 监视器）
- 对象头与锁标记（Mark Word 布局）
- 锁升级（偏向 → 轻量级 → 重量级）
- 锁优化（锁消除、锁粗化、自适应自旋）
- wait/notify 机制

**C4 AQS**（`AQS.md`）

- 核心数据结构（state + CLH 变体队列）
- 加锁 / 解锁全流程（CAS 入队、park 阻塞、从尾向前唤醒）
- 公平锁 vs 非公平锁
- 共享模式 vs 独占模式
- AQS 的应用（ReentrantLock / Semaphore / CountDownLatch 的实现方式）

**C5 锁家族**（`Lock.md`）

- ReentrantLock（公平 / 可中断 / tryLock 超时）
- synchronized vs ReentrantLock
- ReentrantReadWriteLock（读写互斥、锁降级）
- StampedLock（乐观读）
- 死锁与排查（jstack 定位 Found one Java-level deadlock）

**C6 并发工具类**（`Concurrent Tool.md`）

- 三大同步工具（CountDownLatch / CyclicBarrier / Semaphore）对比
- CAS 原理（Unsafe、ABA、AtomicStampedReference）
- ThreadLocal（ThreadLocalMap、弱引用与内存泄漏、remove 必要性）

**C7 线程池**（`ThreadPoolExecutor.md`）

- 七大参数
- 执行流程（核心线程 → 队列 → 最大线程 → 拒绝）
- 四种拒绝策略
- 五种状态流转
- Executors 工厂及弊端（为何阿里规范禁用）
- 参数调优（CPU 密集 N+1 / IO 密集 2N）
- 项目实战

**C8 异步**（`Async.md`）

- Future 的痛点（get 阻塞）
- CompletableFuture 核心（supplyAsync / thenApply / thenCompose / allOf）
- 异常处理（exceptionally / handle）
- ForkJoinPool（工作窃取算法）

**C9 原子类**（`AtomicClass.md`）

- 原子类总览（基本 / 引用 / 数组 / FieldUpdater 四族）
- CAS 底层实现
- 使用陷阱与工具方法

**C10 高并发计数器**（`LongAdder.md`）

- AtomicLong 的痛点（高竞争 CAS 自旋）
- LongAdder 原理（base + Cell[] 热点分离）
- 伪共享问题（缓存行、@Contended）
- LongAdder vs AtomicLong 取舍、LongAccumulator

**C11 LockSupport**（`LockSupport.md`）

- park / unpark 原理（Permit 许可证语义）
- 中断处理
- 在 AQS 中的应用（阻塞队列的底层支撑）

**C12 条件变量**（`Condition.md`）

- Condition 是什么与基本使用
- 多个条件分组唤醒
- 原理（条件队列与同步队列的节点转移）
- 与 wait/notify 对比

**C13 Phaser & Exchanger 等**（`Phaser & Exchanger.md`，四合一）

- Phaser 阶段屏障（可复用、按阶段推进）
- Exchanger 交换器（两线程成对交换）
- ThreadLocalRandom（种子线程隔离）
- CopyOnWriteArraySet（基于 COW List）

---

## 四、JVM（9 篇专题）

**D1 运行时数据区**（`RunDataArea.md`）

- 内存区域总览（线程私有 / 共享）
- 线程私有区（PC / 虚拟机栈 / 本地方法栈）
- 线程共享区（堆 / 方法区 → 元空间）
- 堆结构详解（分代 Eden / S0 / S1）
- 直接内存
- 各区域异常汇总（StackOverflowError / OOM）

**D2 类加载机制**（`ClassLoan.md`）

- 类加载过程五步（加载 / 验证 / 准备 / 解析 / 初始化）
- 类加载器层级（Bootstrap / Platform / App）
- 双亲委派模型与两个好处
- 双亲委派的破坏（SPI / Tomcat 隔离 / 热部署）
- 自定义类加载器（继承 ClassLoader 重写 findClass）

**D3 对象创建与分配**（`CreatePartner.md`）

- 对象创建流程（检查 → 分配 → 零值 → 设头 → init）
- 对象内存布局（对象头 / 实例数据 / 对齐填充）
- TLAB 线程本地分配缓冲
- 内存分配策略（Eden → Survivor → 晋升、大对象直入老年代）

**D4 引用类型与 GC 判定**（`ReferenceType.md`）

- GC 判定方法（引用计数 vs 可达性分析）
- GC Roots 有哪些
- 四种引用类型（强 / 软 / 弱 / 虚）与 ReferenceQueue
- 软 / 弱引用的实际应用（缓存场景）

**D5 GC 算法**（`GC Arch.md`）

- 三大基础算法（标记-清除 / 复制 / 整理）
- 分代收集理论
- 三色标记（并发标记核心）
- 解决漏标的两种方案（增量更新 vs SATB）
- GC 完整流程（以 CMS 为例）

**D6 垃圾收集器**（`GCTool.md`）

- 收集器总览与演进线（Serial → Parallel → CMS → G1 → ZGC）
- Serial / ParNew / Parallel Scavenge
- CMS（四阶段、并发失败模式）
- G1（Region、Mixed GC、停顿预测）
- ZGC（染色指针、毫秒级停顿）
- 收集器选择

**D7 逃逸分析**（`EscapeAnalysis.md`）

- 逃逸的三种程度（不逃逸 / 方法逃逸 / 线程逃逸）
- 三大优化（栈上分配 → 标量替换、锁消除、同步省略）
- 实际效果与参数验证

**D8 JIT 编译与执行引擎**（`JIT.md`）

- 解释器 vs JIT（混合模式）
- C1 / C2 与分层编译
- 方法内联（最核心优化）
- 热点探测（调用计数器 / 回边计数器、衰减）
- OSR 栈上替换

**D9 调优与排查**（`Tuning&Check.md`）

- JVM 常用参数（-Xms / -Xmx / -XX，生产参数示例）
- 排查工具（jps / jstack / jmap / jstat，含 CPU 打满排查流程）
- OOM 的类型与排查流程（HeapDumpOnOutOfMemoryError → MAT Leak Suspects）
- 内存泄漏排查（两次 dump 对比定位增长对象）
- GC 日志分析（JDK8 vs 9+ 参数、FGC / 停顿 / 堆使用率关键指标）
- 调优实战案例

---

## 五、Java 新特性（6 篇专题）

**E1 版本节奏 & Java8**（`Java8.md`）

- 版本节奏（6 个月一版、LTS 概念）
- Lambda 表达式与函数式接口
- Stream API（惰性执行、并行流）
- Optional（正确使用姿势）
- 新日期 API（java.time，不可变线程安全）
- 接口变化（default / static 方法）

**E2 Java9**（`Java9.md`）

- JPMS 模块化
- JShell
- 集合工厂方法 List.of()

**E3 Java10-11**（`Java10-11.md`）

- var 局部变量类型推断
- HTTP Client 标准化
- 其他特性（String 新方法等）

**E4 Java12-14**（`Java12-14.md`）

- switch 表达式
- Records 记录类型
- instanceof 模式匹配
- NPE 精准提示

**E5 Java17**（`Java17.md`）

- Sealed Classes 密封类
- switch 模式匹配
- 强封装 JDK 内部 API

**E6 Java21**（`Java21.md`）

- 虚拟线程原理（载体线程、挂载与卸载）
- 虚拟线程 vs 平台线程 vs Reactive
- 适用场景（IO 密集）
- 结构化并发