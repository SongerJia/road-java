## Synchronized & Monitor
synchronized基础->使用方式->锁的本质->对象头和锁标记->锁升级->锁优化->wait/notify机制->面试高频题
### synchronized基础
什么是synchronized?
synchronized是java内置的互斥锁，保证代码的可见性、有序性和原子性。
三种使用方式
```java
// 方式一：修饰实例方法 —— 锁的是当前对象（this）
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;  // 锁 this
    }
}

// 方式二：修饰静态方法 —— 锁的是 Class 对象
public class Counter {
    private static int total = 0;

    public static synchronized void incrementTotal() {
        total++;  // 锁 Counter.class
    }
}

// 方式三：修饰代码块 —— 可以指定锁对象
public class Counter {
    private int count = 0;
    private final Object lock = new Object();

    public void increment() {
        synchronized (lock) {  // 锁指定对象
            count++;
        }
    }

    // 也可以锁 this
    public void decrement() {
        synchronized (this) {
            count--;
        }
    }

    // 也可以锁 Class
    public static void reset() {
        synchronized (Counter.class) {
            // ...
        }
    }
}
```
锁的对象是什么？

|使用方式|锁的对象|
|---|---|
|实例方法|当前对象 `this`|
|静态方法|`Class` 对象|
|代码块|指定的对象|
```
// 关键：锁的是"对象"，不是代码
// 两个线程如果锁的是同一个对象 → 互斥
// 两个线程如果锁的是不同对象 → 不互斥

// 锁的是同一对象：
synchronized (lock1) { ... }  // 线程 A
synchronized (lock1) { ... }  // 线程 B —— 互斥！

// 锁的是不同对象：
synchronized (lock1) { ... }  // 线程 A
synchronized (lock2) { ... }  // 线程 B —— 不互斥，可并行！
```
### 锁的本质（monitor监视器）
monitor是什么？
Monitor：每个java对象都关联一个monitor锁，它是实现synchronized的核心机制。
```
// 每个对象天生自带一个 monitor
// 一个线程进入 synchronized 块，就是这个对象的 monitor 计数器 +1
// 退出时 -1
// monitor 计数为 0 时，其他线程可以进入

// 可重入：同一线程重复加锁，计数器累加
synchronized (lock) {
    synchronized (lock) {  // 可重入！计数器 1 → 2
        // ...
    }  // 计数器 2 → 1
}  // 计数器 1 → 0
```
字节码层面的实现
```
// 源码
synchronized (lock) {
    count++;
}

// 字节码（反编译）
// monitorenter  ← 进入监视器（计数器 +1）
//   aload_1
//   ...
//   iinc
// monitorexit   ← 退出监视器（计数器 -1）
// monitorexit   ← 异常路径也要退出（编译器自动生成）

// 关键：synchronized 块编译后
// 正常路径：monitorenter → ... → monitorexit
// 异常路径：编译器自动生成 try-catch，finally 中 monitorexit
// 所以异常时也能释放锁（JVM 保证）
```
### 面试高频

> **Q:** "synchronized 底层是怎么实现的？" 
> **A:** "synchronized 基于对象的 monitor 监视器实现。字节码层面是 monitorenter 和 monitorexit 指令，JVM 通过 monitor 的计数器实现可重入。Java 6 之后经过锁升级优化：无锁 → 偏向锁 → 轻量级锁 → 重量级锁。"
### 对象头与锁标记
对象的内存布局
```
// Java 对象在内存中由三部分组成：
// ① 对象头（Mark Word + 类型指针）
// ② 实例数据（字段）
// ③ 对齐填充

// 对象头（64 位 JVM）：
// Mark Word：8 字节 —— 存储 hashCode、GC 分代年龄、锁状态
// Klass Pointer：4/8 字节 —— 指向 Class 对象的指针
// 数组长度：4 字节（数组对象才有）
```
Mark Word的结构
```
Mark Word（64 位）：
//这个是字段占用的bit位，不是具体的值
无锁状态：
[ unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 ]

偏向锁：
[ thread:54 | epoch:2 | unused:1 | age:4 | biased_lock:1 | lock:2 ]

轻量级锁：
[ ptr_to_lock_record:62 | lock:2 ]  ← 指向栈中的锁记录

重量级锁：
[ ptr_to_heavyweight_monitor:62 | lock:2 ]  ← 指向 monitor 对象

GC 标记：
[ forwarding_ptr:62 | lock:2 ]
```
锁状态标记

|锁状态|lock 位（2 bit）|biased_lock（1 bit）|
|---|---|---|
|无锁|01|0|
|偏向锁|01|1|
|轻量级锁|00|—|
|重量级锁|10|—|
|GC 标记|11|—|
### 锁升级
锁升级总览
```
无锁
  ↓ 第一个线程获取锁
偏向锁（Biased Locking）
  ↓ 有竞争（第二个线程来抢）
轻量级锁（Lightweight Locking）—— 自旋
  ↓ 自旋失败/竞争激烈
重量级锁（Heavyweight Locking）—— 阻塞
```
偏向锁
```
// 场景：只有一个线程访问 synchronized 块（无竞争）
// 目标：消除同步开销，第一次加锁后，后续加锁无需 CAS

// 原理：
// 第一次获取锁时：CAS 把线程 ID 写入 Mark Word
// 后续加锁：检查 Mark Word 里的线程 ID 是不是自己
//   是 → 直接进入，无开销
//   否 → 升级

// 适用：单线程反复访问（如 ArrayList 的同步代码）
// 劣势：有其他线程竞争时，撤销偏向锁有开销

// JVM 参数：
// -XX:+UseBiasedLocking  开启（默认）
// -XX:BiasedLockingStartupDelay=0  启动后立即开启（默认延迟 4 秒）
```
轻量级锁
```
// 场景：有竞争，但竞争不激烈（线程交替执行）
// 目标：用 CAS 代替操作系统互斥量（重量级操作）

// 原理：
// 加锁：在线程栈中创建锁记录（Lock Record）
//       CAS 把对象头 Mark Word 拷贝到锁记录，对象头指向锁记录
//       成功 → 持有轻量级锁
//       失败 → 自旋重试（最多 10 次）
// 解锁：CAS 恢复 Mark Word

// 自旋：不阻塞线程，忙等（占用 CPU）
// 适用：锁持有时间短，线程切换开销 > 自旋开销
```
重量级锁
```
// 场景：竞争激烈，自旋也抢不到
// 目标：线程阻塞，让出 CPU

// 原理：
// 未获取到锁的线程 → 阻塞（BLOCKED 状态）
// 靠操作系统互斥量实现（进入内核态）
// 获取锁的线程释放时，通知阻塞线程唤醒

// 开销：用户态 → 内核态切换，性能最差
// 适用：锁持有时间长，竞争激烈
```
锁升级完整流程
```
线程进入 synchronized
    │
    ├── 无锁状态
    │     ↓ 第一个线程获取
    ├── 偏向锁（线程 ID 写入对象头）
    │     ↓ 其他线程访问
    ├── 偏向锁撤销 → 升级
    │     ↓
    ├── 轻量级锁（CAS + 自旋）
    │     ↓ 自旋失败（10 次）
    ├── 升级
    │     ↓
    └── 重量级锁（monitor 阻塞）
```
### 面试高频

> **Q:** "synchronized 的锁升级过程？" 
> **A:** "Java 6 优化后，锁有四个状态：无锁 → 偏向锁 → 轻量级锁 → 重量级锁，只升不降。偏向锁：单线程访问无开销；有竞争就撤销偏向锁升级为轻量级锁（CAS+自旋）；自旋失败或竞争激烈升级为重量级锁（阻塞）。"
### 锁优化
锁消除
```
// JIT 编译器检测到锁对象不会被其他线程访问 → 消除锁

// 例子：StringBuffer 的方法加了 synchronized
public String concat(String a, String b) {
    StringBuffer sb = new StringBuffer();  // 局部变量，不会逃逸
    sb.append(a);       // 加锁了
    sb.append(b);       // 加锁了
    return sb.toString();
}
// JIT 分析：sb 是局部变量，不会被其他线程访问
// → 消除 synchronized 锁（逃逸分析 + 锁消除）
```
锁粗化
```
// 连续加锁/解锁同一对象 → 合并为一次加锁

// 例子：
public void method() {
    synchronized (lock) { /* 操作1 */ }
    synchronized (lock) { /* 操作2 */ }  // 相邻的锁合并
    synchronized (lock) { /* 操作3 */ }
}
// JIT 优化为：一次加锁，三个操作连续执行
```
自适应自旋
```
// 自旋次数不固定
// JVM 根据前一次自旋获取锁的成功率动态调整
// 上次成功 → 这次多自旋几次
// 上次失败 → 这次少自旋甚至不自旋

// 比固定 10 次更智能
```
### wait/notify机制
为什么必须在synchronized块中？
```
// wait/notify 必须持有对象的 monitor 锁
// 因为要保证：
// ① 检查条件 → 进入等待 这两个操作是原子的
// ② 防止"错过通知"问题

// ❌ 错误用法
public void waitForData() {
    // 没有 synchronized
    lock.wait();  // ❌ IllegalMonitorStateException！
}

// ✅ 正确用法
public void waitForData() throws InterruptedException {
    synchronized (lock) {
        while (data == null) {  // 条件不满足
            lock.wait();        // 释放锁，进入等待
        }
        // 条件满足，继续执行
    }
}
```
wait/notify的完整流程
```
// 生产者-消费者模型
class DataBox {
    private int data;
    private boolean hasData = false;

    // 生产者
    public synchronized void put(int value) throws InterruptedException {
        while (hasData) {      // 已有数据，等待消费者消费
            wait();            // 释放锁，进入 WAITING
        }
        data = value;
        hasData = true;
        notifyAll();           // 唤醒消费者  唤醒所有等待消费者
    }

    // 消费者
    public synchronized int get() throws InterruptedException {
        while (!hasData) {     // 没有数据，等待生产者生产
            wait();            // 释放锁，进入 WAITING
        }
        hasData = false;
        notifyAll();           // 唤醒生产者
        return data;
    }
}
```
为什么用while而不是if?
```
// 正确：while —— 防止"虚假唤醒"和"过早唤醒"
while (condition) {  //还需要等待的条件
    wait();  // 唤醒后重新检查条件
}

// 错误：if —— 唤醒后不再检查条件
if (condition) {
    wait();  // ❌ 唤醒后条件可能又变了
}

// 虚假唤醒：即使没有 notify，wait 也可能返回（操作系统原因）
// 所以必须用 while 循环重新检查条件（还需要等待的条件）
```
为什么定义在Object而不是Thread中
```
// 因为锁是对象级别的！
// synchronized 锁的是"对象"，不是"线程"
// 任何对象都可以作为锁
// 所以 wait/notify 应该属于所有对象（Object）
// 而不是 Thread

// 如果定义在 Thread：
// 一个线程锁了对象 A，另一个线程锁了对象 B
// 怎么用"线程"来唤醒？无法定位
```
### 高频面试题目
#### 题目1：synchronized是可重入的吗？
```
// 是的！monitor 计数器实现
public class ReentrantDemo {
    public synchronized void methodA() {
        methodB();  // 重入，计数器 1 → 2
    }

    public synchronized void methodB() {
        // 计数器 2 → 1
    }
}
```
#### 构造器能加synchronized吗？
```
// 语法不允许！构造器不能加 synchronized
public class Demo {
    // public synchronized Demo() {}  // ❌ 编译错误
}
// 因为构造器执行期间，对象还没完全创建，锁没有意义
```
#### 静态方法的class对象和实例方法锁的this互斥吗？
```
// 不互斥！
public class Demo {
    public synchronized void instanceMethod() { }     // 锁 this
    public static synchronized void staticMethod() { }  // 锁 Demo.class

    // 两个方法可以并行执行（锁的不是同一个对象）
}
```
#### synchronized和lock的区别
| 对比       | synchronized | ReentrantLock         |
| -------- | ------------ | --------------------- |
| **锁释放**  | 自动（JVM 保证）   | 手动 unlock()（finally）  |
| **可中断**  | ❌ 不可中断       | ✅ lockInterruptibly() |
| **超时**   | ❌            | ✅ tryLock(timeout)    |
| **公平**   | ❌ 非公平        | ✅ 可配置公平               |
| **条件变量** | wait/notify  | Condition（多个）         |
| **性能**   | 已优化，接近 Lock  | 略好（极端竞争）              |