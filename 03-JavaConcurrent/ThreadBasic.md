## 线程基础
进程VS线程->线程模型->线程生命周期->线程创建方式->面试高频题
### 进程VS线程
核心定义

| 概念  | 定义         | 说明              |
| --- | ---------- | --------------- |
| 进程  | 资源调度的最小单位  | 有独立的内存空间、文件句柄等  |
| 线程  | CPU调度的最小单位 | 共享进程的资源，有独立的执行栈 |
共享与私有
```
// 进程 vs 线程的存储视角
// 一个进程包含多个线程

// 线程共享（进程级资源）：
// ① 堆（方法区中的对象）
// ② 方法区/元空间（类信息、常量）
// ③ 打开的文件、网络连接

// 线程私有（线程级资源）：
// ① 程序计数器（PC）—— 记录当前执行的指令地址
// ② 虚拟机栈 —— 每个线程一个栈
// ③ 本地方法栈
// ④ ThreadLocal 变量（逻辑私有）
```
切换成本对比
```
// 线程切换（轻量）：
// ① 保存/恢复寄存器状态
// ② 保存/恢复程序计数器
// ③ 不需要切换内存映射

// 进程切换（重量）：
// ① 切换页目录（Page Table）→ 刷新 TLB  快表/地址变换高速缓存
// ② CPU 缓存（Cache）失效
// ③ 需要陷入内核态

// 所以：线程切换比进程切换快 10~100 倍
```
通信方式
```
// 进程间通信（IPC）：
// 管道、消息队列、共享内存、信号量、Socket

// 线程间通信：
// 共享内存 + 锁（synchronized/Lock）
// volatile
// wait/notify
// BlockingQueue
```
### 面试高频

> **Q:** "进程和线程有什么区别？" 
> **A:** "进程是资源分配的最小单位，线程是 CPU 调度的最小单位。进程有独立的地址空间，线程共享进程的地址空间。线程切换比进程切换轻量——进程切换需要切换页表、刷新 TLB，线程切换只需要保存恢复寄存器。"
### 线程模型
三种线程模型

| 模型    | 描述               | 特点               | 代表           |
| ----- | ---------------- | ---------------- | ------------ |
| 1:1模型 | 一个java线程=一个OS线程  | 简单，但创建成本高        | HotSpot JVM  |
| N:1模型 | 多个java线程映射一个OS线程 | 用户态调度，切换快但无法利用多核 | 早期绿色线程       |
| M:N模型 | M个java线程映射N个OS线程 | 灵活但实现复杂          | Go goroutine |
HotSpot 1:1 模型
```
// Java 线程创建时，JVM 调用操作系统的线程创建
// Linux：pthread_create
// Windows：CreateThread

// 流程：
new Thread() 
  → JVM 创建 JavaThread 对象
    → 调用 pthread_create（Linux）
      → 内核分配 TCB（线程控制块）
        → 分配线程栈空间
          → 加入调度队列

// 所以 Java 线程的创建/销毁成本不低
// 这也是为什么需要线程池
```
虚拟线程 java21
```
// Java 21 引入虚拟线程（Virtual Thread）—— 解决 1:1 模型的痛点
// 虚拟线程是 M:N 模型，由 JVM 调度

// 创建虚拟线程
Thread virtualThread = Thread.ofVirtual()
    .name("my-virtual")
    .start(() -> {
        System.out.println("运行在虚拟线程");
    });

// 或者用 Executors
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> System.out.println("虚拟线程任务"));
}

// 适用场景：IO 密集型任务（大量阻塞等待）
// 不适用：CPU 密集型任务
```
### 面试高频

> **Q:** "Java 的线程和操作系统线程是一一对应的吗？" 
> **A:** "HotSpot 虚拟机是 1:1 模型，一个 Java 线程对应一个 OS 线程。所以 Java 线程的创建成本并不低，生产环境要用线程池。Java 21 引入的虚拟线程则是 M:N 模型，由 JVM 调度，能支持海量并发。"
### 线程生命周期
状态总览
```
// Java 线程的 6 种状态（Thread.State 枚举）
NEW          // 新建：new Thread() 后，还没 start()
RUNNABLE     // 就绪/运行：调用了 start()，可能正在运行或等待 CPU
BLOCKED      // 阻塞：等待进入 synchronized 代码块（竞争锁）
WAITING      // 等待：无限期等待（wait()/join() 等）
TIMED_WAITING// 限时等待：带超时（sleep()/wait(timeout) 等）
TERMINATED   // 终止：run() 执行完或抛出异常
```
状态流转图
```
      new Thread()
              │
              ▼
           NEW ──────→ start()
              │            │
              │            ▼
              │       RUNNABLE ←──────┐
              │         │     │       │
              │   获得锁失败   sleep/等待
              │         │     │       │
              │         ▼     ▼       │
              │      BLOCKED  TIMED_WAITING
              │         │     │       │
              │     获得锁  超时/唤醒   │
              │         │     │       │
              │         └──┬──┘       │
              │            │          │
              │        WAITING ←──────┘
              │         │
              │      被唤醒/通知
              │         │
              │         ▼
              │      RUNNABLE
              │         │
              │    run() 结束/异常
              │         │
              │         ▼
              │    TERMINATED
              └──────────┘
```
触发状态的API
```
// NEW → RUNNABLE
Thread t = new Thread(runnable);
t.start();

// RUNNABLE → BLOCKED（竞争 synchronized 锁失败）
synchronized (lock) {
    // 另一个线程持有锁时，这里会 BLOCKED
}

// RUNNABLE → WAITING
lock.wait();          // 无限期等待（需要在 synchronized 中）
t.join();             // 等待 t 线程结束
LockSupport.park();   // 暂停

// RUNNABLE → TIMED_WAITING
Thread.sleep(1000);       // 睡眠 1 秒
lock.wait(1000);          // 等待 1 秒（需要在 synchronized 中）
t.join(1000);             // 等 t 最多 1 秒
LockSupport.parkNanos(1000);

// 任何状态 → TERMINATED
// run() 正常结束 或 抛异常
```
### 面试高频

> **Q:** "sleep 和 wait 的区别？" 
> **A:** " ① 释放锁：sleep 不释放锁；wait 释放锁 
> ② 使用条件：sleep 可以任意使用；wait 必须在 synchronized 块中 
> ③ 所属类：sleep 是 Thread 的静态方法；wait 是 Object 的方法 
> ④ 恢复方式：sleep 到时间自动恢复；wait 需要 notify/notifyAll 唤醒
> ⑤ 状态：sleep 进入 TIMED_WAITING；wait 进入 WAITING（或 TIMED_WAITING） "

> **Q:** "BLOCKED 和 WAITING 有什么区别？" 
> **A:** "BLOCKED 是竞争 synchronized 锁失败被阻塞，等锁释放后自动恢复；WAITING 是主动调用 wait/join 等待，需要被 notify 唤醒或等线程结束才能恢复。"
### 线程的创建方式
#### 方式一：继承Thread
```
// ① 继承 Thread，重写 run()
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("线程执行: " + Thread.currentThread().getName());
    }
}

// 使用
new MyThread().start();

// 缺点：
// ① Java 单继承，继承了 Thread 就不能继承其他类
// ② 任务和线程耦合在一起，不利于复用
// ③ 拿不到返回值
```
#### 方式二：实现Runnable
```
// ② 实现 Runnable（推荐）
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("任务执行: " + Thread.currentThread().getName());
    }
}

// 使用
Thread t = new Thread(new MyTask());
t.start();

// 优点：
// ① 不占用继承位（可以实现其他接口）
// ② 任务和线程分离，任务可以复用
// ③ 配合线程池使用更方便

// 缺点：拿不到返回值
```
#### 方式三：实现Callable接口+FutureTask
```java
// ③ 实现 Callable（能返回结果）
class CallableTask implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        Thread.sleep(1000);
        return 42;  // 返回计算结果
    }
}

// 使用：FutureTask 包装
FutureTask<Integer> futureTask = new FutureTask<>(new CallableTask());
Thread t = new Thread(futureTask);
t.start();

// 获取结果（阻塞，直到任务完成）
Integer result = futureTask.get();  // 42

// 优点：能拿到返回值，能抛异常
// 缺点：get() 会阻塞
```
#### 方式四：线程池 ExecutorService
```java
// ④ 线程池（生产环境首选）
ExecutorService executor = Executors.newFixedThreadPool(10);

// 提交任务（不需要返回结果）
executor.execute(() -> System.out.println("任务执行"));

// 提交任务（需要返回结果）
Future<Integer> future = executor.submit(new CallableTask());
Integer result = future.get();

// 关闭线程池
executor.shutdown();

// 优点：
// ① 复用线程，避免频繁创建销毁
// ② 统一管理线程数量
// ③ 提供异步结果获取（Future）
```
### 面试高频
> **Q:** "创建线程有哪几种方式？生产环境用哪种？" 
> **A:** "四种：继承 Thread、实现 Runnable、实现 Callable（配 FutureTask）、线程池。生产环境**一定用线程池**，因为 Java 线程是 1:1 映射 OS 线程，频繁 new Thread() 创建销毁成本太高，线程池能复用线程、控制并发数、统一管理。"

> **Q:** "Runnable 和 Callable 的区别？" 
> **A:** " ① 返回值：Runnable 的 run() 无返回值；Callable 的 call() 有返回值 ② 异常：Runnable 不能抛受检异常；Callable 可以抛 Exception ③ 使用：Runnable 可以直接给 Thread；Callable 要包一层 FutureTask ④ 方法名：run() vs call() "
### 面试高频题目
#### 题目1：线程有哪些状态
```
NEW → RUNNABLE → BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```
#### 题目2：为什么多线程看起来并行
```
// 单核 CPU：时间片轮转，看起来并行，实际串行
// 多核 CPU：真正并行

// 时间片：约 10~100ms，用完就让出 CPU
```
#### 题目3：线程安全的前提条件
```
// 多线程不安全三要素：
// ① 多个线程同时访问共享资源
// ② 至少一个线程在写
// ③ 没有同步机制

// 消除任一个，就安全了：
// ① 不共享（ThreadLocal、局部变量）
// ② 只读（不可变对象）
// ③ 加锁同步（synchronized/Lock）
```
#### 线程的优雅停止
```java
// ❌ 不推荐：stop()（已废弃，直接杀死线程，可能破坏数据）
// ❌ 不推荐：interrupt() 直接强制

// ✅ 推荐：协作式停止
class Worker extends Thread {
    private volatile boolean running = true;  // volatile 保证可见性

    @Override
    public void run() {
        while (running) {  // 检查标志
            // 执行任务
        }
    }

    public void stopWorker() {
        running = false;  // 协作式停止
    }
}

// ✅ 推荐：利用 interrupt 标志
class Worker2 implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // 恢复中断标志   传递中断标志
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
}
```
