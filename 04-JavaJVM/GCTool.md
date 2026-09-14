## 垃圾收集器
收集器总览->Serial/ParNew->Parallel->CMS->G1->ZGC->收集器选择->面试高频题
### 垃圾收集器总览
```
收集器全景图
新生代：                     老年代：
Serial ────────────────→ Serial Old（标记-整理）
ParNew ────────────────→ CMS（标记-清除）
Parallel Scavenge ─────→ Parallel Old（标记-整理）

G1：统一管理新生代 + 老年代
ZGC：超低延迟，不分代（JDK 15+ 分代）
```
各收集器定位

|收集器|新生代/老年代|算法|线程|目标|
|---|---|---|---|---|
|Serial|新生代|复制|单线程|简单、客户端|
|ParNew|新生代|复制|多线程|配 CMS|
|Parallel Scavenge|新生代|复制|多线程|吞吐量|
|Serial Old|老年代|整理|单线程|配 Serial|
|Parallel Old|老年代|整理|多线程|配 Parallel|
|**CMS**|老年代|清除|多线程|低延迟|
|**G1**|不分代（Region）|复制+整理|多线程|可预测停顿|
|**ZGC**|不分代|复制|多线程|超低延迟|
### Serial/ParNew/Parallel Scavenge
Serial（串行收集器）
```
// 特点：单线程 GC，全程 STW
// 客户端默认收集器（JDK 8 前）

// 使用：-XX:+UseSerialGC

// 流程：
// ① Stop The World（暂停所有业务线程）
// ② 单线程回收（新生代复制）
// ③ 恢复业务线程

// 优点：简单、单线程无切换开销
// 缺点：STW 时间长
// 适用：单核 CPU、客户端应用、小堆
```
ParNew（并行新生代收集器）
```
// 特点：Serial 的多线程版（新生代）
// 唯一能和 CMS 搭配的新生代收集器

// 使用：-XX:+UseConcMarkSweepGC 自动配 ParNew

// 和 Serial 区别：多线程并行回收（但也是 STW）
// 适合：多核 CPU + CMS 场景
```
Parallel Scavenge（吞吐量优先收集器）
```
// 特点：目标是"高吞吐量"（CPU 利用率高）

// 使用：-XX:+UseParallelGC（JDK 8 默认）

// 关键参数：
// -XX:MaxGCPauseMillis=100   最大停顿时间（尽力而为）
// -XX:GCTimeRatio=99          吞吐量比例（默认 99% 时间用于业务）
// -XX:+UseAdaptiveSizePolicy  自适应调节（自动调堆大小、比例）

// 对比：
// CMS：追求"停顿小"（牺牲吞吐量）
// Parallel：追求"吞吐量高"（接受较长的停顿）

// 适合：后台计算、批处理任务
```
### CMS（并发标记清除）
```
// CMS（Concurrent Mark Sweep）：老年代收集器
// 目标：最短停顿时间（响应快）

// 使用：-XX:+UseConcMarkSweepGC（JDK 9 标记废弃，JDK 14 移除）

// ① 初始标记（Initial Mark）—— STW
// 标记 GC Roots 直接可达的对象
// 停顿很短（只扫根）

// ② 并发标记（Concurrent Mark）
// 从初始标记对象出发，标记所有可达对象
// 和业务线程并发执行
// 用三色标记 + 增量更新

// ③ 重新标记（Remark）—— STW
// 修正并发期间变化的引用
// 停顿较短

// ④ 并发清除（Concurrent Sweep）
// 清除垃圾对象（标记-清除算法）
// 和业务线程并发执行

// 总 STW = 初始标记 + 重新标记（都很短）
// 其他时间并发 → 停顿小
```
CMS的缺点
```
// ① 内存碎片（标记-清除产生碎片）
// 碎片多了 → 大对象分配失败 → 触发 Full GC（Serial Old 整理）
// 参数：-XX:CMSFullGCsBeforeCompaction 整理频率

// ② 并发失败（Concurrent Mode Failure）
// 并发标记期间，老年代被填满
// 没空间分配新对象 → 退化为 Serial Old 全量回收（STW 很长！）

// ③ 浮垃圾（浮动垃圾）
// 并发清除期间产生的垃圾，本次清不掉，下次再清

// ④ 占用 CPU 资源
// 并发阶段和业务线程抢 CPU
// -XX:+CMSInitiatingOccupancyFraction=70
// 老年代占用 70% 时提前开始 CMS（避免并发失败）

// 总结：CMS 已过时，被 G1 取代
```
### G1（Garbage First）
```
// G1（JDK 7 实验，JDK 9 默认）：全堆收集器
// 目标：可预测的停顿时间（软实时）
// 把堆划分为大小相等的 Region，按"回收收益"优先回收

// 使用：-XX:+UseG1GC（JDK 9+ 默认）
// 参数：-XX:MaxGCPauseMillis=200（默认停顿目标 200ms）
```
Region结构
```
G1 的堆（不分代，只分 Region）：

┌────┬────┬────┬────┬────┬────┬────┬────┐
│ E  │ E  │ S  │ O  │ O  │ H  │ E  │ O  │
├────┼────┼────┼────┼────┼────┼────┼────┤
│ E  │ E  │ E  │ O  │ O  │ H  │ O  │ S  │
├────┼────┼────┼────┼────┼────┼────┼────┤
│ E  │ O  │ E  │ O  │ S  │ E  │ E  │ E  │
└────┴────┴────┴────┴────┴────┴────┴────┘

E = Eden（新生代）   S = Survivor（幸存区）
O = Old（老年代）    H = Humongous（大对象区，>Region 50%）
```
Region特点
```
// ① 堆被分成 ~2048 个 Region，每个 1MB~32MB
// ② 新生代/老年代不再是连续区域，而是逻辑集合
// ③ Region 之间可以动态调整（E 不够 → 更多 Region 变 E）
// ④ H 区：大对象直接放多个连续 H Region
// ⑤ 每个 Region 有记忆集（Remembered Set）记录引用

// 参数：
// -XX:G1HeapRegionSize=2m：Region 大小
// -XX:MaxGCPauseMillis=200：停顿目标
```
G1的回收过程
```
// ① 新生代回收（Young GC）
// 回收所有 Eden + Survivor
// 存活对象复制到新的 Survivor 或 Old

// ② 混合回收（Mixed GC）
// 新生代 + 部分老年代（按收益排序选 Region）
// 用"停顿预测模型"选择回收哪些老年代 Region
// 保证总停顿不超目标

// ③ Full GC（退化的兜底）
// 并发标记失败 → Full GC（STW，用 Serial 算法）
// 应该避免
```
G1vsCMS

|对比|CMS|G1|
|---|---|---|
|**堆结构**|分代连续|Region|
|**算法**|标记-清除|复制+整理|
|**碎片**|有|无（复制整理）|
|**停顿**|尽量短（不保证）|可预测（软实时）|
|**大对象**|直接进老年代|H Region|
|**默认**|JDK 8 可选|JDK 9+ 默认|
|**记忆集**|全局|每 Region 一个|
G1的特点
```
// ① 记忆集维护开销大（每个 Region 都有）
// ② 大对象分配需要连续 H Region
// ③ 小堆下（<4GB）可能不如 CMS/Parallel
// ④ 写屏障开销（维护记忆集）
```
### ZGC
```
// ZGC（JDK 11 实验，JDK 15 正式）：超低延迟收集器
// 目标：停顿时间不超过 10ms（无论堆多大！）

// 使用：-XX:+UseZGC
// JDK 21：-XX:+UseZGC -XX:+ZGenerational（分代模式）

// 原理：
// ① 染色指针（Colored Pointer）：对象标记存在指针中
// ② 读屏障：读取对象时检查/修正指针
// ③ 并发转移：对象移动和业务线程并发
```
染色指针（Colored Pointer）
```
// 64 位指针中，用 4 位存储标记信息：
// | 46位 地址 | 4位 颜色标记 | 14位 未使用 |

// 4 位颜色：
// Finalizable：可终结
// Remapped：已重映射
// Marked0：标记 0
// Marked1：标记 1

// 好处：
// ① 标记信息不用存在对象头（省内存）
// ② 标记和移动并发进行
// ③ 通过指针状态即可判断对象状态
```
ZGC vs G1

|对比|G1|ZGC|
|---|---|---|
|**停顿**|~200ms 目标|<10ms|
|**实现**|三色标记+SATB|染色指针+读屏障|
|**对象移动**|STW 时移动|并发移动|
|**适用**|通用|超大堆、低延迟|
|**代价**|记忆集开销|读屏障开销（CPU）|
**ZGC 用染色指针 + 读屏障实现并发标记和并发转移，停顿时间稳定在 10ms 以内，适合超大堆低延迟场景（如金融交易）。代价是读屏障带来少量 CPU 开销。**
### 收集器选择
```
JDK 8 默认：Parallel（吞吐量）
JDK 9+ 默认：G1（平衡）

小堆（<4GB）？        → Parallel 或 Serial
大堆 + 低延迟？        → G1（默认）或 ZGC
超大堆 + 超低延迟？     → ZGC
吞吐量优先？           → Parallel
```
各JDK默认GC

|JDK|默认收集器|
|---|---|
|JDK 8|Parallel Scavenge + Parallel Old|
|JDK 9~16|G1|
|JDK 17+|G1（ZGC 可选）|
### 面试高频题
#### 题目1：G1为什么叫Garbage First?
```
// 因为 G1 优先回收"垃圾最多的 Region"（回收收益最大）
// 通过停顿预测模型，选择收益最高的 Region 集合回收
// 用有限的停顿换取最大的回收量
```
#### 题目2：G1和CMS的区别？
```
// ① 结构：Region vs 分代连续
// ② 算法：复制+整理（无碎片）vs 标记-清除（有碎片）
// ③ 停顿：可预测 vs 尽量短
// ④ 回收：按收益选 Region vs 全老年代
```
#### 题目2：什么是并发失败
```
// CMS：并发标记期间老年代满 → 退化为 Serial Old（长 STW）
// G1：并发标记期间堆满 → Full GC
// 避免：提前触发 GC（调占用率阈值）
```
#### 题目3：ZGC为什么停顿那么小？
```
// ① 染色指针：标记信息在指针里，不用停业务线程
// ② 读屏障：业务线程读取时自行修正指针（并发转移）
// ③ 大部分工作并发完成，STW 只剩极小部分
```