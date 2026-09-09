## JMM & Volatile
JMM是什么->内存模型三特性->volatile原理->happens-before规则->volatile vs synchronized->面试高频题目
### JMM是什么
定义：
JMM Java Memory Model java内存模型：Java规范中定义的一套规则，规定多线程下共享变量的读写如何保证可见性、有序性和原子性。
为什么需要JMM？
```
// 问题：CPU 缓存导致可见性问题
// 线程 A 修改了变量，但线程 B 看不到！

// 硬件层面：
// CPU 有三级缓存（L1/L2/L3），数据先写缓存，再刷回主内存
// 多核 CPU 各自有缓存，缓存之间数据不一致

// Java 层面抽象：
// 每个线程有"工作内存"（抽象概念，对应 CPU 缓存/寄存器）
// 共享变量存在"主内存"（对应物理内存）
// 线程操作变量：先从主内存拷贝到工作内存 → 修改 → 刷回主内存
// 线程间不能直接访问对方的工作内存
```
JMM抽象结构图
```
┌─────────────────────────────────────┐
│            主内存（Main Memory）      │
│   共享变量：count、flag、list...      │
└─────────────────────────────────────┘
        ↑          ↑            ↑
     拷贝/回写    拷贝/回写      拷贝/回写
        │          │            │
┌───────┴───┐  ┌───┴───────┐  ┌─┴────────┐
│线程A工作内存│  │线程B工作内存│  │线程C工作内存│
│  count=0  │  │  count=0  │  │  count=0 │
└───────────┘  └───────────┘  └──────────┘
      │             │             │
  线程 A 操作      线程 B 操作     线程 C 操作
```
经典问题演示
```java
// 可见性问题
public class VisibilityDemo {
    private static boolean flag = true;  // 没加 volatile

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            while (flag) {
                // 死循环 —— 永远看不到 flag 被改成 false！
            }
            System.out.println("线程结束");
        });
        t.start();

        Thread.sleep(1000);
        flag = false;  // 主线程修改 flag
        // ❌ 子线程可能永远看不到这个修改（无限循环）
    }
}

// 加 volatile 后
private static volatile boolean flag = true;  // ✅ 立即可见
```
### 内存模型三特性

| 特性  | 说明               | 保证手段                       |
| --- | ---------------- | -------------------------- |
| 原子性 | 操作不可分割，要么全做，要么不做 | synchronized、lock、CAS      |
| 可见性 | 一个线程修改，其他线程理解可见  | volatile、synchronized、lock |
| 有序性 | 指令执行顺序符合预期       | volatile、sychronized       |
#### 原子性
```
// 什么是原子操作？
// i = 1;            ✅ 原子（赋值）
// int x = i;        ✅ 原子（读取）
// i++;              ❌ 非原子！三步：读 → 加 → 写
// i += 1;           ❌ 非原子！

// 为什么 i++ 非原子？
// ① 从主内存读取 i 的值到工作内存
// ② 在工作内存中加 1
// ③ 把结果写回主内存
// 三步之间可能被其他线程插队 → 丢失更新

// 解决方案：
// synchronized 包裹
// AtomicInteger 类（CAS）
```
#### 可见性
```
// volatile 保证可见性
// synchronized 进入和退出时也会同步（锁释放时刷新到主内存）

// 但普通变量不保证：
private boolean flag = true;
// 线程 A 修改，线程 B 可能看不到（读的是自己工作内存的旧值）
```
#### 有序性
```
// 指令重排：编译器和 CPU 为了优化，可能调整指令执行顺序

// 例子：双重检查锁单例
public class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {              // ① 检查
            synchronized (Singleton.class) {
                if (instance == null) {      // ② 二次检查
                    instance = new Singleton();  // ③ 创建
                }
            }
        }
        return instance;
    }
}

// 问题：③ 不是原子操作
// ① 分配内存
// ② 初始化对象
// ③ 把引用赋值给 instance
// 指令重排后可能变成：① → ③ → ②
// 其他线程拿到未初始化完成的对象！

// 解决：instance 加 volatile，禁止重排
private static volatile Singleton instance;
```
### volatile原理
volatile的两个保证
```
// ① 可见性：修改立即刷新到主内存
// ② 有序性：禁止指令重排（内存屏障）

// volatile 不保证原子性！
volatile int count = 0;
// count++ 依然不是原子的，多线程下依然会丢更新
```
Memory Barrier 内存屏障
```
// volatile 的实现：插入内存屏障指令

// 写 volatile 变量时：
// ① 插入 StoreStore 屏障 —— 禁止前面的普通写与 volatile 写重排
// ② 插入 StoreLoad 屏障 —— 禁止 volatile 写与后面读重排，且强制刷新到主内存

// 读 volatile 变量时：
// ① 插入 LoadLoad 屏障 —— 禁止 volatile 读与后面普通读重排
// ② 插入 LoadStore 屏障 —— 禁止 volatile 读与后面普通写重排

// 效果：
// 写：写屏障强制把工作内存的修改刷回主内存
// 读：读屏障强制从主内存重新读取
```
volatile的实现级别

| 层面     | 机制                  |
| ------ | ------------------- |
| Java层面 | volatile关键字         |
| 字节码层面  | 变量访问标记 ACC_VOLATILE |
| JVM层面  | 插入内存屏障（使用lock前缀指令）  |
| 硬件层面   | CPU缓存一致性协议 MESI     |
volatile使用场景
```java
// 场景一：状态标志
public class FlagDemo {
    private volatile boolean running = true;

    public void stop() {
        running = false;  // 写 volatile
    }

    public void run() {
        while (running) {  // 读 volatile
            // 执行任务
        }
    }
}

// 场景二：双重检查锁单例
private static volatile Singleton instance;

// 场景三：发布安全（发布不可变对象）
private volatile List<String> cachedList;
```
volatile不可用的场景
```java
// ❌ 不能用于"复合操作"（不保证原子性）
volatile int count = 0;

// 多线程并发 count++ —— 依然会丢更新！
// 因为 i++ = 读 → 加 → 写，三步之间可能被打断

// ✅ 用 AtomicInteger 替代
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```
### happens-before规则
happens-before：如果操作A happens-before 操作B，那么A的结果对B是可见的，且A的操作顺序在B之前。
8条规则
```
// ① 程序顺序规则
// 一个线程内，按代码顺序，前面的操作 happens-before 后面的
int a = 1;     // A
int b = a + 1; // B —— A happens-before B

// ② 监视器锁规则
// 解锁 happens-before 后续对同一锁的加锁
synchronized (lock) {
    x = 10;   // 解锁
}
// 另一个线程
synchronized (lock) {  // 加锁 —— 能看到 x = 10
    // ...
}

// ③ volatile 变量规则
// 对 volatile 变量的写 happens-before 后续对它的读
volatile boolean flag = false;
flag = true;         // 写
// 其他线程读 flag —— 能看到 true

// ④ 传递性
// A happens-before B，B happens-before C → A happens-before C

// ⑤ 线程启动规则
// Thread.start() happens-before 该线程的每个动作

// ⑥ 线程终止规则
// 线程的所有操作 happens-before 其他线程检测到该线程终止（join/return）

// ⑦ 线程中断规则
// interrupt() 调用 happens-before 被中断线程检测到中断（isInterrupted/异常）

// ⑧ 对象终结规则
// 对象初始化的完成 happens-before finalize() 开始
```
实际应用
```
// 利用 happens-before 分析：volatile 单例为什么安全？
// ① volatile 写（instance = new Singleton()）happens-before volatile 读（其他线程读 instance）
// ② 所以：其他线程读 instance 时，能看到创建过程中的所有操作
// ③ 即使有指令重排，volatile 屏障保证了创建的顺序
```
### volatile vs synchronized

| 对比维度 | volatile  | synchronized |
| ---- | --------- | ------------ |
| 原子性  | 不保证       | 保证           |
| 可见性  | 保证        | 保证           |
| 有序性  | 禁止重排      | 保证           |
| 锁    | 无锁        | 重量级          |
| 性能   | 快（内存屏障）   | 慢（锁竞争）       |
| 阻塞   | 不阻塞       | 可能阻塞         |
| 使用场景 | 单一变量的状态标志 | 复合操作、代码块     |
### 面试高频题目
#### 题目1：volatile能保证原子性吗
```
// 不能！
volatile int count = 0;

// 多线程 count++ 依然不安全
// 因为 count++ 是三步：读 → 加 → 写
// volatile 只保证了"读"和"写"的可见性
// 但"读-改-写"之间可能被其他线程插队

// 用 AtomicInteger 或 synchronized 解决
```
#### 题目2：为什么要用volatile修饰双重检查锁的instance？
```
// 因为 new Singleton() 不是原子操作：
// ① 分配内存
// ② 初始化对象
// ③ 引用赋值
// 可能重排为 ① → ③ → ②

// 线程 A 执行到 ③（引用已赋值但对象未初始化）
// 线程 B 检查 instance != null → 直接用 → 拿到未初始化对象！

// volatile 禁止重排 → 保证 ③ 在 ② 之后
```
#### 题目3：内存屏障有哪些类型？
```
// ① LoadLoad —— 禁止两个读重排
// ② StoreStore —— 禁止两个写重排
// ③ LoadStore —— 禁止读后写重排
// ④ StoreLoad —— 禁止写后读重排（最重，全屏障）
```
#### 题目4：volatile适用场景示例？
```
// ① 状态标志（running）
// ② 双重检查锁单例
// ③ 发布不可变对象
// ④ 替代锁的轻量级同步（单个变量）
```