## 调优与排查
JVM常用参数->常用工具->OOM类型->OOM排查流程->内存泄漏排查->GC日志分析->调优实战->面试高频题
### JVM常用参数
堆参数
```
// 内存参数
-Xms2g                  // 初始堆大小
-Xmx2g                  // 最大堆大小（生产建议 -Xms = -Xmx）
-Xmn1g                  // 新生代大小
-Xss256k                // 栈大小（默认 1MB）
-XX:MetaspaceSize=256m  // 元空间初始
-XX:MaxMetaspaceSize=256m  // 元空间最大

// 新生代比例
-XX:SurvivorRatio=8     // Eden:S0:S1 = 8:1:1
-XX:NewRatio=2          // 新生代:老年代 = 1:2
```
GC参数
```
// 收集器选择
-XX:+UseParallelGC      // 吞吐量优先（JDK 8 默认）
-XX:+UseG1GC            // G1（JDK 9+ 默认）
-XX:+UseZGC             // ZGC（JDK 15+）

// G1 专用
-XX:MaxGCPauseMillis=200    // 停顿目标
-XX:G1HeapRegionSize=2m     // Region 大小

// GC 日志（JDK 9+ 新语法）
-Xlog:gc*:gc.log        // 输出 GC 日志到文件
-Xlog:gc*:stdout        // 输出到控制台
```
内存溢出参数
```
// 堆溢出时自动 dump
-XX:+HeapDumpOnOutOfMemoryError   // OOM 时自动生成 dump 文件
-XX:HeapDumpPath=/tmp/dump.hprof  // dump 文件位置

// 打印参数
-XX:+PrintGCDetails     // JDK 8 打印 GC 详情（JDK 9+ 用 -Xlog:gc*）
-XX:+PrintGCDateStamps  // 打印 GC 时间
```
生产推荐配置
```
# 生产 JVM 参数示例（JDK 17 + G1）
-Xms4g -Xmx4g                        # 堆大小固定
-XX:+UseG1GC                          # G1 收集器
-XX:MaxGCPauseMillis=200             # 停顿目标
-XX:+HeapDumpOnOutOfMemoryError      # OOM 自动 dump
-XX:HeapDumpPath=/data/logs/dump/    # dump 位置
-Xlog:gc*:/data/logs/gc.log:time     # GC 日志
-XX:+DisableExplicitGC               # 禁用 System.gc()
```
### 常用排查工具
命令行工具

|工具|作用|常用命令|
|---|---|---|
|**jps**|查看 Java 进程|`jps -l`|
|**jstack**|线程栈（死锁排查）|`jstack <pid>`|
|**jmap**|堆内存快照|`jmap -heap <pid>`、`jmap -histo <pid>`|
|**jstat**|GC 统计|`jstat -gcutil <pid> 1000`|
|**jinfo**|JVM 参数查看/修改|`jinfo -flags <pid>`|
|**jcmd**|综合诊断|`jcmd <pid> help`|
jps —— 查看进程
```
jps -l
# 输出：
# 12345 com.example.JudgeApplication
# 23456 org.jetbrains.jps.cmdline.Launcher
```
jstack —— 线程栈
```
# 查看线程状态
jstack 12345

# 输出示例：
# "judge-worker-1" #12 prio=5 os_prio=0 tid=0x0000 nid=0x2a34
#    java.lang.Thread.State: WAITING (parking)
#         at jdk.internal.misc.Unsafe.park(Native Method)
#         at java.util.concurrent.locks.LockSupport.park(LockSupport.java:194)
#         ...
#         at com.example.JudgeService.process(JudgeService.java:123)

# 死锁检测：输出末尾有
# Found one Java-level deadlock:
# =============================
# "Thread-1":
#   waiting to lock ... which is held by "Thread-0"
# "Thread-0":
#   waiting to lock ... which is held by "Thread-1"

# CPU 打满排查：
top -Hp 12345    # 找 CPU 最高的线程号（如 28842）
printf "%x\n" 28842  # 转 16 进制：70da
jstack 12345 | grep -A 20 "70da"  # 定位到代码
```
jmap —— 堆内存（重点）
```
# ① 查看堆概况
jmap -heap 12345
# 输出：堆配置、各代使用情况

# ② 查看对象统计（内存泄漏排查第一步）
jmap -histo 12345
# 输出对象数量排行：
# num     #instances         #bytes  class name
# 1:        1234567     987654321  [B          ← 字节数组最多
# 2:         234567      76543210  java.util.HashMap$Node
# 3:          12345       4321098  com.example.JudgeTask  ← 可疑类

# ③ 生成堆快照（配合 MAT 分析）
jmap -dump:format=b,file=/tmp/dump.hprof 12345
```
jstat —— GC 统计
```
# 每秒输出一次 GC 信息
jstat -gcutil 12345 1000
# 输出：
# S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
# 0.00  100.00  45.2   60.3  88.1  92.0   1234    15.23    12     3.45   18.68
# S0/S1：幸存区使用率
# E：Eden 使用率    O：老年代使用率
# YGC：Minor GC 次数  FGC：Full GC 次数
# 观察：FGC 频繁 → 老年代有问题
```
### OOM的类型
各种OOM的场景
```
// ① OutOfMemoryError: Java heap space
// 堆溢出
// 场景：对象过多、内存泄漏、堆太小

// ② OutOfMemoryError: Metaspace
// 元空间溢出
// 场景：动态生成类太多（反射、代理、热部署）

// ③ OutOfMemoryError: unable to create native thread
// 无法创建线程
// 场景：线程数太多（超过系统限制）
// 排查：ulimit -u、线程池配置

// ④ OutOfMemoryError: Direct buffer memory
// 直接内存溢出
// 场景：NIO 缓冲区太多

// ⑤ StackOverflowError（不是 OOM）
// 栈溢出
// 场景：无限递归
```
OOM常见原因
```
// ① 内存泄漏：对象无法回收（ThreadLocal 没 remove、静态集合）
// ② 内存不足：堆太小（-Xmx 配小了）
// ③ 并发太高：对象瞬间暴增
// ④ 大数据加载：一次加载过多数据到内存
// ⑤ 连接泄漏：数据库/HTTP 连接不关闭
```
### OOM排查流程
```
# ① 查看进程
jps -l

# ② 看 GC 情况（FGC 是否频繁）
jstat -gcutil <pid> 1000

# ③ 看堆使用
jmap -heap <pid>

# ④ 看对象分布（找可疑对象）
jmap -histo <pid>

# ⑤ 生成堆快照（OOM 会自动生成）
jmap -dump:format=b,file=dump.hprof <pid>
# 或等待 -XX:+HeapDumpOnOutOfMemoryError 自动生成

# ⑥ 用 MAT（Eclipse Memory Analyzer）分析 dump
# - 打开 dump.hprof
# - 查看 "Leak Suspects Report"（泄漏嫌疑报告）
# - 看对象引用链，定位泄漏根因
```
MAT分析要点
```
// MAT 核心视图：
// ① Leak Suspects：自动分析泄漏嫌疑
// ② Dominator Tree：支配树（找大对象）
// ③ Path to GC Roots：对象的 GC Root 引用链
//    —— 看对象为什么没被回收
// ④ Top Consumers：内存占用排行

// 排查思路：
// ① 找到占用内存最大的对象
// ② 看它的 GC Root 引用链
// ③ 找到持有它不放的"根"（静态集合、ThreadLocal 等）
```
面试回答模版
```
面试官："你遇到过 OOM 吗？怎么排查的？"

"遇到过，排查步骤是：

① jps 找到进程，jstat -gcutil 看 GC 情况
   发现 Full GC 非常频繁，老年代一直满

② jmap -histo 看对象分布
   发现某个业务对象（比如 JudgeTask）实例数异常多

③ 用 -XX:+HeapDumpOnOutOfMemoryError 拿到堆快照
   用 MAT 打开分析

④ MAT 的 Leak Suspects 发现：
   任务对象被一个静态的 ConcurrentHashMap 持有
   只往里面 put 不 remove，导致任务对象无法回收

⑤ 修复：任务完成后从 map 移除 + 限制 map 大小
   问题解决，Full GC 恢复正常"
```
### 内存泄漏排查
内存泄漏vs内存溢出
```
// 内存溢出（OOM）：内存不够用了（结果）
// 内存泄漏（Leak）：对象无法回收（原因）

// 关系：内存泄漏 → 占用越来越多 → 最终 OOM

// 泄漏的特征：
// ① 内存占用持续上升，不回落
// ② GC 后内存占用不下降
// ③ 最终 OOM
```
常见泄漏原因
```
// ① ThreadLocal 没 remove（线程池中严重）
// ② 静态集合无限 add（HashMap 当缓存不清理）
// ③ 连接/流没关闭（IO、数据库、HTTP）
// ④ 监听器/回调没移除
// ⑤ 内部类持有外部类引用（成员内部类）
// ⑥ 缓存不清理（WeakHashMap 可缓解）
// ⑦ String.intern() 用太多（JDK 7+ 在堆，可能泄漏）
```
排查命令
```
# ① 观察内存是否持续增长
jstat -gcutil <pid> 1000 500   # 连续看 500 秒

# ② 两次 dump 对比（找增长对象）
jmap -dump:format=b,file=1.hprof <pid>   # 第一次
# ...运行一段时间...
jmap -dump:format=b,file=2.hprof <pid>   # 第二次
# MAT 打开两个 dump，对比哪些对象增长了

# ③ 用 MAT 的对比功能
# Compare Base Lines → 找新增/增长的对象
```
### GC日志分析
怎么看GC日志
```
# JDK 8 参数：
-XX:+PrintGCDetails -XX:+PrintGCDateStamps
# JDK 9+：
-Xlog:gc*:gc.log:time

# GC 日志示例（G1）：
# [GC pause (G1 Evacuation Pause) (young) 512M->180M(1024M), 0.032s]
#   ↑ 新生代回收：512MB → 180MB，堆总量 1024MB，耗时 32ms
#
# [GC pause (G1 Humongous Allocation) ...]
#   ↑ 大对象分配触发的 GC
#
# [Full GC ...]
#   ↑ 完整回收（要避免，说明老年代满了）

# 关键指标：
# ① FGC 次数：Full GC 越少越好（理想 0 次）
# ② 停顿时间：是否超过预期
# ③ 堆使用率：是否持续高位
```
### 调优实战
```
// ① 明确目标：
// 吞吐量优先？延迟优先？还是内存优先？

// ② 收集现状（GC 日志、监控）
// 先量测，再调整，不拍脑袋

// ③ 定位问题：
// FGC 频繁？→ 老年代压力大（调大堆/检查泄漏）
// 停顿长？→ 调 GC 参数或换收集器
// CPU 高？→ 检查死循环/锁竞争/线程数

// ④ 调整 + 验证：
// 一次只改一个参数，A/B 对比验证

// ⑤ 沉淀配置：
// 稳定后固化为标准配置
```
常见问题调优
```
# 问题 1：FGC 频繁
# 原因：老年代频繁满
# 方案：
# ① 调大堆（-Xmx）
# ② 检查内存泄漏（重点！）
# ③ 调整新生代/老年代比例

# 问题 2：停顿时间长
# 方案：换 G1/ZGC，设 MaxGCPauseMillis

# 问题 3：CPU 打满
# 方案：jstack 定位死循环/锁竞争
```
### 面试高频题
#### 题目1：OOM有哪些类型
```
// Heap space（堆）、Metaspace（元空间）、
// unable to create native thread（线程）、
// Direct buffer memory（直接内存）
```
#### 题目2：排查OOM的步骤
```
// jps → jstat → jmap -histo → dump → MAT 分析 → 定位修复
```
#### 题目3：内存泄漏和内存溢出的区别？
```
// 泄漏是原因（对象无法回收）
// 溢出是结果（内存不够）
// 泄漏累积到一定程度 → 溢出
```
#### 题目4：CPU打满怎么排查
```
// top -Hp <pid> 找 CPU 最高线程
// 转 16 进制
// jstack <pid> | grep 线程号
// 定位到代码行
```
#### 题目5：生产环境JVM怎么配？
```
// -Xms=-Xmx（固定堆，避免伸缩）
// G1 + MaxGCPauseMillis
// HeapDumpOnOutOfMemoryError
// GC 日志开启
// DisableExplicitGC（禁用 System.gc）
```

