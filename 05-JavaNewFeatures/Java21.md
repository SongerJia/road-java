## Java21
Java21总览->虚拟线程原理->虚拟线程使用->适用场景->与平台线程对比->vs Reactive->结构化并发->面试高频题
### Java21总览
```
// 核心特性：
// ① 虚拟线程（Virtual Threads）—— 最重磅！
// ② 结构化并发（预览）
// ③ 记录模式（正式）
// ④ switch 模式匹配（正式）
// ⑤ 有序集合（Sequenced Collections）

// 其他：
// ⑥ 分代 ZGC
// ⑦ 字符串模板（预览）
// ⑧ 向量 API（孵化）
```
记录模式正式版
```
// 记录模式（Record Patterns）：解构 Record

record Point(int x, int y) {}
record Line(Point start, Point end) {}

// 嵌套解构：
Object obj = new Line(new Point(1, 2), new Point(3, 4));
if (obj instanceof Line(Point(int x1, int y1), Point(int x2, int y2))) {
    System.out.println("起点: " + x1 + "," + y1);
    System.out.println("终点: " + x2 + "," + y2);
}
// 一行解构出所有字段！
```
switch模式匹配正式
```
// Java 21 正式
Object obj = "hello";
String result = switch (obj) {
    case null -> "null";
    case String s -> "字符串: " + s;
    case Integer i -> "整数: " + i;
    default -> "其他";
};
```
有序集合
```
// SequencedCollection：统一的首尾操作
List<String> list = new ArrayList<>();
list.getFirst();    // 第一个（之前 list.get(0)）
list.getLast();     // 最后一个
list.addFirst("A");
list.addLast("B");
list.removeFirst();
list.reversed();    // 反转视图

// Set、Map 也有对应方法
```
### 虚拟线程原理
平台线程的问题
```
// 平台线程（普通线程）：1:1 映射 OS 线程
// 痛点：
// ① 创建成本高（内核调用）
// ② 内存占用大（栈默认 1MB）
// ③ 数量有限（几万就到顶）
// ④ 阻塞时浪费资源（IO 等待时线程闲着）

// 场景：100 万并发 IO 任务
// 平台线程需要 100 万线程 → 内存 1TB + 创建爆炸 → 不可能
```
虚拟线程是什么？
```
// 虚拟线程（Virtual Thread）：
// JVM 管理的轻量级线程（不映射 OS 线程）
// 由 JVM 调度，跑在少量的载体线程（Carrier Thread）上

// 关键：
// ① 创建成本极低（普通对象）
// ② 内存占用小（栈自动伸缩，KB 级）
// ③ 数量可以百万级
// ④ 阻塞时不占资源（自动让出载体线程）
```
原理图
```
虚拟线程（百万级）：
┌──────────┐ ┌──────────┐ ┌──────────┐
│  VT-1    │ │  VT-2    │ │  VT-3    │  ...
└──────────┘ └──────────┘ └──────────┘
      │           │           │
      └───────────┼───────────┘
                  ▼
载体线程（Carrier Thread，= CPU 核数）：
┌──────────┐ ┌──────────┐ ┌──────────┐
│  CT-1    │ │  CT-2    │ │  CT-3    │  ← 映射到 OS 线程
└──────────┘ └──────────┘ └──────────┘

// 虚拟线程阻塞（IO）时：
// ① 从载体线程上卸载（挂起）
// ② 载体线程去跑其他虚拟线程
// ③ IO 完成后重新调度回来

// 所以：阻塞不浪费资源！
```
虚拟线程的实现
```
// 内部实现（JDK 21）：
// ① 虚拟线程是 M:N 模型（M 个虚拟线程跑 N 个载体线程）
// ② 阻塞点自动切换：synchronized/IO/Thread.sleep 等
// ③ JVM 内置调度器：ForkJoinPool（并行度 = CPU 核数）
// ④ 延续（Continuation）：保存/恢复执行状态

// 注意：虚拟线程的调度在 JVM 用户态
// 比 OS 线程切换快得多
```
### 虚拟线程的使用
创建方式
```
// ① Thread.ofVirtual
Thread vThread = Thread.ofVirtual()
    .name("my-virtual")
    .start(() -> System.out.println("虚拟线程"));

// ② 批量创建（推荐）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // 每个任务一个虚拟线程
    for (int i = 0; i < 100000; i++) {
        executor.submit(() -> handleRequest());
    }
}
// 百万任务轻松搞定！

// ③ 检查是否是虚拟线程
Thread.currentThread().isVirtual();  // true/false
```
对比平台线程创建
```
// 平台线程：
ExecutorService pool = Executors.newFixedThreadPool(200);
// 线程池上限 200，任务多了排队

// 虚拟线程：
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // 任务再多也不怕，每任务一线程
}

// 语义：虚拟线程场景下"每个任务一个线程"是正确姿势
// 不需要池化（创建成本极低）
```
### 适用场景
适用IO密集型
```
// 大量阻塞等待的场景，收益巨大：
// ① HTTP 请求处理（Web 服务器）
// ② 数据库查询
// ③ 远程调用（RPC、外部 API）
// ④ 文件读写
// ⑤ 消息消费

// 例子：Web 服务器并发
// 平台线程：200 线程处理 200 并发请求
// 虚拟线程：百万虚拟线程处理百万并发请求

// Spring Boot 3.2 + Tomcat 支持虚拟线程：
spring.threads.virtual.enabled=true
```
不适用 CPU密集型
```
// 虚拟线程不提升 CPU 计算能力
// 计算密集型：性能取决于 CPU 核数
// 虚拟线程不会更快（甚至略慢，调度开销）

// 应使用：平台线程数 = CPU 核数
```
注意点
```
// ① synchronized 会"钉住"载体线程（JDK 21 部分解决）
// 虚拟线程在 synchronized 块中阻塞 → 载体线程也被占住
// 高频 synchronized 场景用 ReentrantLock 替代

// ② 不要池化虚拟线程
// 创建成本低，池化没必要

// ③ 线程局部变量（ThreadLocal）慎用
// 虚拟线程太多，ThreadLocal 内存放大

// ④ 使用前确认 JDK 21+
```
### 虚拟线程 vs 平台线程
| 对比       | 平台线程      | 虚拟线程        |
| -------- | --------- | ----------- |
| **映射**   | 1:1 OS 线程 | M:N（JVM 调度） |
| **创建成本** | 高（内核）     | 低（对象）       |
| **栈内存**  | ~1MB      | 自动伸缩（KB 级）  |
| **最大数量** | 几千~几万     | 百万级         |
| **阻塞时**  | 占用 OS 线程  | 让出载体线程      |
| **调度**   | OS 调度     | JVM 用户态调度   |
| **适用**   | 通用、CPU 密集 | IO 密集、高并发   |
| **池化**   | 必须（线程池）   | 不需要         |
**虚拟线程是 JVM 管理的轻量级线程，M:N 映射到少量载体线程。创建成本极低、支持百万级并发，IO 阻塞时自动让出载体线程，大幅提升 IO 密集型应用的并发能力。JDK 21 正式发布，适合替代传统线程池处理高并发 IO 场景。**
### 虚拟线程 vs Reactive线程
```
// 之前解决"高并发 IO"的方案：
// ① 响应式编程（Reactive）—— WebFlux
// ② 异步回调 —— CompletableFuture
// ③ 协程 —— 虚拟线程（新）

// 虚拟线程的目标：用"同步阻塞"的写法，达到"异步"的性能
```

|对比|虚拟线程|Reactive（WebFlux）|
|---|---|---|
|**代码风格**|同步（简单）✅|响应式（复杂）❌|
|**学习成本**|低（就是普通线程）|高（Mono/Flux）|
|**调试**|简单（栈正常）✅|难（回调地狱）❌|
|**并发能力**|百万级 ✅|百万级 ✅|
|**生态**|新（Spring 支持）|成熟|
|**底层**|JVM 调度|Netty + 事件循环|
结论
```
// 虚拟线程出现后，Reactive 的优势被削弱：
// ① 虚拟线程用同步写法就能达到类似并发能力
// ② 代码可读性、可维护性大幅提升
// ③ 现有同步代码几乎不用改（换 Executor 即可）

// 所以业界趋势：
// 简单场景：虚拟线程（更简单）
// 复杂场景（背压、流式）：Reactive 仍有价值
```
### 面试高频

> **Q:** "虚拟线程和 Reactive 编程怎么选？" 
> **A:** "虚拟线程用同步写法实现高并发，代码简单、易调试、学习成本低，适合大多数 IO 密集场景；Reactive 学习成本高、调试难，但背压和流式处理更强。虚拟线程出现后，一般场景优先选虚拟线程，Spring 3.2 也已支持。"
### 结构化并发
```
// 结构化并发（Structured Concurrency）：
// 把并发的任务组织成"结构化"的作用域
// 类似 try-with-resources 管理线程生命周期

// 目标：解决线程生命周期管理混乱
// 当前：线程各自为政，超时/取消/错误处理麻烦
```
示例
```
// 预览 API（StructuredTaskScope）：
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    // 并行提交两个任务
    Future<String> user = scope.fork(() -> fetchUser());
    Future<String> order = scope.fork(() -> fetchOrder());

    // 等待所有任务完成（或一个失败）
    scope.join();
    scope.throwIfFailed();

    // 所有任务都在 scope 作用域内
    // scope 关闭时，未完成的任务自动取消
    String result = user.resultNow() + order.resultNow();
}
// 异常传播、超时取消都自动处理！
```
好处
```
// ① 任务生命周期与作用域绑定（自动清理）
// ② 错误传播（一个失败，整体失败）
// ③ 取消传播（scope 关闭，子任务取消）
// ④ 代码结构清晰（哪里开始、哪里结束）
```
### 面试高频题
#### 题目1：Java21有哪些特性
```
// 虚拟线程（最重要）、结构化并发（预览）、
// 记录模式（正式）、switch 模式匹配（正式）、有序集合
```
#### 题目2：虚拟线程的原理
```
// JVM 管理、M:N 映射载体线程、
// IO 阻塞自动让出载体线程、用户态调度
```
#### 题目3：虚拟线程适用什么场景
```
// IO 密集型：高并发请求、数据库访问、远程调用
// 不适用：CPU 密集型
```
#### 题目4：虚拟线程需要池化吗
```
// 不需要！创建成本极低，每任务一线程
```
#### 题目5：虚拟线程代替线程池吗
```
// IO 密集场景：可以替代传统线程池
// CPU 密集场景：还是平台线程 + 线程池
```
#### 题目6：虚拟线程和WebFlux对比
```
// 同步写法 + 高并发 vs 响应式 + 高并发
// 虚拟线程更简单，一般场景优先
```