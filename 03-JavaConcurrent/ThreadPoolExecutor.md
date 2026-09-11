## 线程池
为什么需要线程池->七大参数->执行流程->四种拒绝策略->线程五种状态->Executors工厂->参数优化->项目实战
### 为什么需要线程池
直接new Thread的问题
```
// ① 创建/销毁开销大
// Java 线程是 1:1 映射 OS 线程，创建涉及内核调用
// 100 万次 new Thread() → 创建 100 万个 OS 线程

// ② 无控制并发数
// 并发太高 → 线程过多 → 内存耗尽、频繁上下文切换

// ③ 无法统一管理
// 无法复用、无法监控、无法优雅关闭

// 所以：生产环境必须用线程池
```
线程池的好处

|好处|说明|
|---|---|
|**复用线程**|减少创建销毁开销|
|**控制并发数**|防止资源耗尽|
|**统一管理**|监控、拒绝、关闭|
|**任务缓冲**|队列排队，削峰填谷|
### 七大参数
```
public ThreadPoolExecutor(int corePoolSize,     // ① 核心线程数
                          int maximumPoolSize,  // ② 最大线程数
                          long keepAliveTime,   // ③ 空闲存活时间
                          TimeUnit unit,        // ④ 时间单位
                          BlockingQueue<Runnable> workQueue,  // ⑤ 任务队列
                          ThreadFactory threadFactory,         // ⑥ 线程工厂
                          RejectedExecutionHandler handler)    // ⑦ 拒绝策略
```
参数详解
```
// ① corePoolSize 核心线程数
// 即使空闲也保留的线程数
// 默认：核心线程创建后不回收（除非 allowCoreThreadTimeOut）

// ② maximumPoolSize 最大线程数
// 线程池允许的最大线程数
// 条件：队列满且核心线程都在忙，才创建到最大

// ③ keepAliveTime 空闲存活时间
// 非核心线程空闲多久后回收
// 前提：线程数 > corePoolSize

// ④ unit 时间单位
TimeUnit.SECONDS

// ⑤ workQueue 任务队列
// 核心线程全忙时，任务放入队列

// ⑥ threadFactory 线程工厂
// 创建线程的工厂，可以自定义线程名、优先级等

// ⑦ handler 拒绝策略
// 队列满且线程达到 maximumPoolSize 时，怎么处理新任务
```
参数关系图
```
提交任务
    │
    ├── 线程数 < corePoolSize？ → 创建核心线程执行 ✅
    │
    ├── 核心线程全忙 → 放入 workQueue 排队 ✅
    │
    ├── 队列满 → 线程数 < maximumPoolSize？ → 创建非核心线程 ✅
    │
    └── 线程数 == maximumPoolSize？ → 拒绝策略处理 ❌
```
### 执行流程
```java
// 任务提交的完整流程
public void execute(Runnable command) {
    int c = ctl.get();

    // ① 工作线程数 < 核心线程数
    // → 创建新线程执行（不管核心线程是否空闲！）
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // ② 线程数 >= 核心线程数
    // → 尝试放入任务队列
    if (isRunning(c) && workQueue.offer(command)) {
        // 入队成功
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command))
            reject(command);  // 线程池关闭了 → 拒绝
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);  // 没有线程 → 创建
    }

    // ③ 队列满了
    // → 尝试创建非核心线程
    else if (!addWorker(command, false))
        reject(command);  // 线程数达到最大值 → 拒绝
}

// 注意：核心线程数判断的是"线程数量"，不是"是否空闲"
// 即使核心线程空闲，只要线程数 < corePoolSize，也会创建新线程！
```
执行流程总结
```
execute(task)
    │
    ├── workerCount < corePoolSize
    │   └── 创建核心线程执行（不排队）
    │
    ├── workerCount >= corePoolSize
    │   ├── 队列未满 → 放入队列
    │   └── 队列已满 → 尝试创建非核心线程
    │       ├── workerCount < maximumPoolSize → 创建
    │       └── workerCount == maximumPoolSize → 拒绝
```
### 四种拒绝策略
```
// ① AbortPolicy（默认）—— 抛异常
// 新任务直接抛 RejectedExecutionException
ThreadPoolExecutor.AbortPolicy

// ② CallerRunsPolicy —— 调用者执行
// 谁提交的任务谁执行（不丢弃，但不阻塞线程池）
// 适合：希望任务一定执行，且可以降速
ThreadPoolExecutor.CallerRunsPolicy

// ③ DiscardPolicy —— 直接丢弃
// 新任务静默丢弃（不报错）
ThreadPoolExecutor.DiscardPolicy

// ④ DiscardOldestPolicy —— 丢弃最旧
// 丢弃队列中最旧的任务，然后重试新任务
ThreadPoolExecutor.DiscardOldestPolicy
```
对比

|策略|行为|适用场景|
|---|---|---|
|**AbortPolicy**|抛异常|默认，任务不能丢时|
|**CallerRunsPolicy**|调用者执行|任务必须执行，可降速|
|**DiscardPolicy**|静默丢弃|允许丢任务（日志等）|
|**DiscardOldestPolicy**|丢弃最旧|要最新的任务|
自定义拒绝测试
```
// 生产环境常用：记录日志 + 降级处理
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 8, 60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100),
    new ThreadPoolExecutor.AbortPolicy() {  // 或自定义
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            // 记录日志
            log.error("任务被拒绝: {}", r);
            // 降级处理：写入 MQ 或数据库，稍后重试
            saveToRetryQueue(r);
        }
    });
```
### 线程池的五种状态
```
// 用 int 的高 3 位表示状态，低 29 位表示线程数
// ctl = 状态 + 线程数（用一个变量存两个信息）

private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

// 五种状态：
RUNNING    // 运行中：可以接收新任务，也可以处理队列任务
SHUTDOWN   // 关闭：不接收新任务，但处理队列中的任务
STOP       // 停止：不接收新任务，不处理队列，中断正在执行的任务
TIDYING    // 整理：所有任务终止，workerCount = 0，执行 terminated()
TERMINATED // 终止：terminated() 执行完
```
状态流转
```
RUNNING → SHUTDOWN（调用 shutdown()）
RUNNING → STOP（调用 shutdownNow()）
SHUTDOWN → TIDYING（队列空 + 线程数 0）
STOP → TIDYING（线程数 0）
TIDYING → TERMINATED（terminated() 执行完）
```
shutdown vs shutdownNow
```
// shutdown()：优雅关闭
// ① 不接收新任务
// ② 继续处理队列中的任务
// ③ 等所有任务完成后停止

// shutdownNow()：立即关闭
// ① 不接收新任务
// ② 停止处理队列任务（返回未执行的任务列表）
// ③ 中断正在执行的任务

ThreadPoolExecutor executor = new ThreadPoolExecutor(...);

// 优雅关闭
executor.shutdown();
try {
    // 等待所有任务完成（最多等 60 秒）
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow();  // 超时强制关闭
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```
### Executors工厂
四种预置线程池
```
// ① FixedThreadPool —— 固定线程数
ExecutorService fixed = Executors.newFixedThreadPool(4);
// 内部：corePoolSize = maximumPoolSize = 4
// 队列：LinkedBlockingQueue（无界！）

// ② CachedThreadPool —— 弹性线程数
ExecutorService cached = Executors.newCachedThreadPool();
// 内部：corePoolSize = 0，maximumPoolSize = Integer.MAX_VALUE
// 队列：SynchronousQueue（容量 0，直接交付）
// keepAlive：60 秒

// ③ SingleThreadExecutor —— 单线程
ExecutorService single = Executors.newSingleThreadExecutor();
// 内部：corePoolSize = maximumPoolSize = 1
// 队列：LinkedBlockingQueue（无界！）

// ④ ScheduledThreadPool —— 定时任务
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(4);
// 内部：DelayedWorkQueue
```
为什么禁止Executors
```
// ❌ 问题 1：FixedThreadPool 和 SingleThreadExecutor
// 用无界队列 LinkedBlockingQueue（Integer.MAX_VALUE）
// 任务堆积 → OOM！

// ❌ 问题 2：CachedThreadPool
// maximumPoolSize = Integer.MAX_VALUE
// 无限创建线程 → OOM！

// ✅ 正确做法：手动创建 ThreadPoolExecutor
// 用有界队列，自定义拒绝策略
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                          // 核心线程
    8,                          // 最大线程
    60, TimeUnit.SECONDS,       // 空闲回收
    new ArrayBlockingQueue<>(100),  // 有界队列！
    r -> new Thread(r, "judge-pool-" + r.hashCode()),  // 自定义线程名
    new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
);
```
### 参数调优
核心公式
```
// CPU 密集型任务：核心线程数 = CPU 核数 + 1（或 N+1）
// 因为几乎没有等待，多一个线程利用 CPU 的空闲周期

// IO 密集型任务：核心线程数 = CPU 核数 * 2（或 2N）
// 因为大量时间在等待 IO，需要更多线程填补等待

// 更精确的公式（《Java 并发编程实战》）：
// 线程数 = N * (1 + W/C)
// N = CPU 核数
// W = 等待时间
// C = 计算时间

// 获取 CPU 核数
int cores = Runtime.getRuntime().availableProcessors();
```
调优步骤
```
// ① 先按类型估算
// CPU 密集（编译+运行）→ 核心线程 = 核数 + 1

// ② 压测验证
// 用 JMeter/自写压测 → 观察指标：
// - CPU 使用率（目标 60%~80%）
// - 队列积压情况
// - 任务响应时间

// ③ 根据指标调整
// CPU 空闲 → 加大线程数
// 队列堆积 → 加大队列或加大线程数
// CPU 打满 → 减线程数（避免上下文切换）

// ④ 核心线程数可以动态调整
executor.setCorePoolSize(8);
executor.setMaximumPoolSize(16);
```
队列怎么选

|队列|特点|适用|
|---|---|---|
|**ArrayBlockingQueue**|有界数组|需要限制积压（推荐）|
|**LinkedBlockingQueue**|可选有界|默认用，注意设容量|
|**SynchronousQueue**|容量 0|直接交付（CachedThreadPool）|
|**PriorityBlockingQueue**|优先级|任务有优先级|
### 项目实战
```
面试官："你项目里线程池怎么配的？为什么这么配？"

"我的XXXX模块用的线程池：

① 核心参数：
   - 核心线程数 = CPU 核数 + 1（XXX是 CPU 密集型）
   - 最大线程数 = CPU 核数 * 2
   - 队列 = ArrayBlockingQueue(1000)（有界，防 OOM）
   - 拒绝策略 = 降级到重试队列（任务不能丢）

② 为什么有界队列：
   用无界队列（Executors 默认）任务会无限堆积导致 OOM
   有界队列 + 拒绝降级是最佳实践

③ 自定义线程名：
   judge-worker-1 这种名字，线上用 jstack 一眼看出问题

④ 压测验证：
   最开始核心线程设 4，压测发现队列积压
   根据 CPU 密集公式调到 N+1，效果明显
```
