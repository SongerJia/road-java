## 条件变量
Condition是什么->基本使用->多个条件分组唤醒->原理->与wait/notify对比->应用场景->面试高频题
### Condition是什么？
Condition：ReentrantLock的条件变量，实现线程等待与唤醒。类似synchronized的wait/notify,但支持多个条件、精准唤醒、可中断和可超时。
怎么创建？
```
// 必须先有 Lock
ReentrantLock lock = new ReentrantLock();

// 一个 Lock 可以创建多个 Condition
Condition notFull = lock.newCondition();   // 条件 1：队列不满
Condition notEmpty = lock.newCondition();  // 条件 2：队列不空
```
核心方法
```
// 等待（必须在 lock 内调用）
condition.await();             // 等待（释放锁）
condition.await(timeout, unit); // 限时等待
condition.awaitNanos(nanos);
condition.awaitUninterruptibly();  // 不可中断等待

// 唤醒
condition.signal();      // 唤醒一个等待线程
condition.signalAll();   // 唤醒所有等待线程
```
### 基本使用
标准使用方式
```
// 使用模板（和 wait/notify 类似）：
// ① 必须持有 lock
// ② await 前用 while 检查条件
// ③ await 会释放锁
// ④ 唤醒后重新检查条件

public class BoundedQueue<E> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    private final Object[] items;
    private int count, putIndex, takeIndex;

    public BoundedQueue(int capacity) {
        items = new Object[capacity];
    }

    // 生产者
    public void put(E e) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();  // 队列满 → 等待"不满"条件
            }
            items[putIndex] = e;
            putIndex = (putIndex + 1) % items.length;
            count++;
            notEmpty.signal();   // 唤醒一个等待"不空"的消费者
        } finally {
            lock.unlock();
        }
    }

    // 消费者
    public E take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 队列空 → 等待"不空"条件
            }
            E item = (E) items[takeIndex];
            takeIndex = (takeIndex + 1) % items.length;
            count--;
            notFull.signal();   // 唤醒一个等待"不满"的生产者
        } finally {
            lock.unlock();
        }
    }
}
```
为什么用while不用if?
```
// 和 wait/notify 一样的原因：
// ① 虚假唤醒（spurious wakeup）
// ② 唤醒后条件可能又变了（多个线程竞争）
// 必须用 while 循环重新检查条件
```
### 多个条件分组唤醒
解决wait/notify做不到精准分组唤醒
```
// ❌ synchronized 只有一个等待队列
// 所有等待的线程挤在一起，notify 随机唤醒一个
// 可能唤醒错误类型的线程

// 场景：生产者-消费者
// 消费者等"非空"，生产者等"非满"
// 如果用 wait/notify：
// - 队列满，生产者 wait
// - 消费者取走数据后 notify —— 可能唤醒另一个消费者（没用！）
// 只能 notifyAll，唤醒所有，让错误的线程再 wait 回去

// 这就是"惊群效应"：无效唤醒，浪费性能
```
Condition精准唤醒
```
// 一个 Lock 创建两个 Condition
// 生产者等在 notFull 上
// 消费者等在 notEmpty 上

// 生产者入队后：notEmpty.signal() —— 只唤醒消费者 ✅
// 消费者出队后：notFull.signal() —— 只唤醒生产者 ✅

// 效果：
// ① 精准唤醒，无惊群效应
// ② 不同条件互不干扰
// ③ 唤醒效率高
```
对比图
```
wait/notify（一个等待队列）：
┌───────────────────────┐
│      等待队列          │
│  生产者① 消费者① 生产者② │  ← 混在一起
└───────────────────────┘
     notify() → 随机唤醒一个（可能是错误类型）

Condition（多个等待队列）：
┌─────────────────┐  ┌─────────────────┐
│ notFull 条件队列 │  │ notEmpty 条件队列│
│  生产者① 生产者②  │  │  消费者① 消费者② │
└─────────────────┘  └─────────────────┘
   notFull.signal()      notEmpty.signal()
   → 只唤醒生产者          → 只唤醒消费者
```
### 原理
Condition的实现
```
// Condition 的实现类是 AQS 内部类 ConditionObject
public class AbstractQueuedSynchronizer {

    public class ConditionObject implements Condition {
        // 条件队列的头尾（单向链表）
        private transient Node firstWaiter;
        private transient Node lastWaiter;
        // ...
    }
}
```
await流程
```
// ① 把当前线程从"同步队列"移到"条件队列"
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();

    // ① 包装成 Node，加入条件队列
    Node node = addConditionWaiter();

    // ② 释放持有的锁（否则死锁！）
    int savedState = fullyRelease(node);

    int interruptMode = 0;

    // ③ 如果不在同步队列中 → park 阻塞
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);  // 阻塞
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }

    // ④ 被 signal 唤醒后，重新竞争锁
    if (acquireQueued(node, savedState) && interruptMode != INTERRUPTED)
        interruptMode = INTERRUPTED;
    // ⑤ 清理条件队列
    // ...
}

// 关键：await 会"释放锁 + 阻塞"，保证不占锁等待
```
signal流程
```
// 唤醒条件队列中的一个线程
public final void signal() {
    // 必须持有锁才能调用
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();

    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

// 把节点从条件队列移到同步队列
private void doSignal(Node first) {
    do {
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&   // 移到同步队列
             (first = firstWaiter) != null);
}

// transferForSignal：用 CAS 把节点状态改为 SIGNAL
// 然后 unpark 唤醒线程
final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // 加入同步队列
    Node p = enq(node);
    int ws = p.waitStatus;
    // 唤醒线程
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
两个队列的流转
```
                await()
同步队列 ──────────────────────→ 条件队列
(竞争锁的线程)                    (等待条件的线程)
   ↑                                  │
   │             signal()             │
   └──────────────────────────────────┘
   回到同步队列，重新竞争锁

同步队列：FIFO，所有抢锁失败的线程
条件队列：非 FIFO，await 的线程，signal 时从队头唤醒
```
### 与wait/notify对比
|对比|wait/notify|Condition|
|---|---|---|
|**前置条件**|synchronized|Lock（lock/unlock）|
|**等待队列**|一个（monitor）|多个（每个 Condition 一个）|
|**分组唤醒**|❌ 随机唤醒|✅ 精准分组|
|**惊群效应**|有（notifyAll）|无|
|**可中断**|wait 可中断|await 可中断（还有不可中断版）|
|**超时**|wait(timeout)|await(timeout)/awaitNanos/awaitUntil|
|**多个条件**|❌ 不支持|✅ 支持|
|**公平唤醒**|❌ 随机|✅ 队列顺序（signal 队头）|
|**实现**|monitor|AQS ConditionObject|
### 面试高频

> **Q:** "Condition 和 wait/notify 的区别？" 
> **A:** " ① 前置条件：wait/notify 需要 synchronized；Condition 需要 Lock ② 条件队列：wait/notify 只有一个；Condition 可以有多个，支持分组精准唤醒 ③ 惊群效应：notifyAll 唤醒所有导致惊群；Condition.signal 精准唤醒 ④ 超时：Condition 有更丰富的超时方法 ⑤ 实现：monitor vs AQS "

> **Q:** "为什么需要多个 Condition？" 
> **A:** "生产者-消费者场景：生产者等'队列不满'，消费者等'队列不空'。用两个 Condition 分组，生产者入队只唤醒消费者，消费者出队只唤醒生产者，避免 notifyAll 的惊群效应。"
### 应用场景
#### 场景1：阻塞队列内部
```
// ArrayBlockingQueue 内部就用了两个 Condition
public class ArrayBlockingQueue<E> {
    final ReentrantLock lock;
    private final Condition notEmpty;  // 等待"非空"
    private final Condition notFull;   // 等待"非满"

    public void put(E e) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();  // 满则等
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }

    private void enqueue(E x) {
        // 入队后
        notEmpty.signal();  // 唤醒一个消费者
    }
}
```
#### 场景2：限流器
```
// 场景：限流 —— 超过阈值就等待
public class RateLimiter {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition permit = lock.newCondition();
    private int permits = 10;

    public void acquire() throws InterruptedException {
        lock.lock();
        try {
            while (permits <= 0) {
                permit.await();  // 没有许可 → 等待
            }
            permits--;
        } finally {
            lock.unlock();
        }
    }

    public void release() {
        lock.lock();
        try {
            permits++;
            permit.signal();  // 唤醒一个等待者
        } finally {
            lock.unlock();
        }
    }
}
```
### 面试高频题
#### 题目1：await会释放锁吗
```
// 会！必须释放，否则其他线程无法进入临界区，永远无法唤醒
// await 内部：fullyRelease(node) —— 完全释放锁
// 唤醒后：acquireQueued 重新获取锁
```
#### 题目2：signal和signalAll的区别
```
// signal：唤醒条件队列的头节点（一个）
// signalAll：唤醒所有等待线程

// 什么时候用 signalAll？
// 条件可能满足多个线程时（如多个消费者都等"数据"）
// 但通常 signal 就够（唤醒一个，让队列流转）
```
#### 题目3：await为什么必须配合while使用？
```
// ① 虚假唤醒：await 可能无故返回
// ② 多个线程竞争：唤醒后条件可能已不满足
// 必须 while 循环重新检查
```
#### 题目4：Condition可以脱离Lock单独使用吗？
```
// 不可以！
// Condition 必须由 Lock.newCondition() 创建
// 必须持有对应 Lock 才能调用 await/signal
// 否则抛 IllegalMonitorStateException
```
题目5：await与awaitNanos区别？
```
// await()：无限等待，直到 signal
// awaitNanos(nanos)：等待指定时间，超时自动返回
// 返回值：剩余时间（负数表示超时）
```