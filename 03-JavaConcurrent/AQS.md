## AQS
AQS是什么->核心数据结构->加锁流程->解锁流程->公平锁vs非公平锁->AQS的应用->高频面试题目
### AQS是什么
AbstractQueuedSychronizer 抽象队列同步器：JUC并发包的基石，提供了一套实现阻塞锁和同步器的框架。
AQS的核心思想
AQS=一个状态变量state+一个FIFO等待队列（CLH队列变体）+钩子方法（让子类实现）
```
AQS
┌─────────────────────────────────────────┐
│  state（volatile int）—— 同步状态        │
│  锁：0=未持有，1=已持有，>1=可重入次数    │
│  信号量：剩余许可数量                      │
│  计数门闩：剩余待计数数量                  │
├─────────────────────────────────────────┤
│  CLH 等待队列（FIFO）                     │
│  head → Node1 → Node2 → Node3 → tail    │
│  获取锁失败的线程 → 包装成 Node → 入队     │
├─────────────────────────────────────────┤
│  钩子方法（子类实现）                      │
│  tryAcquire()    —— 尝试获取              │
│  tryRelease()    —— 尝试释放              │
│  tryAcquireShared() —— 尝试获取（共享）   │
│  tryReleaseShared() —— 尝试释放（共享）   │
│  isHeldExclusively() —— 是否独占         │
└─────────────────────────────────────────┘
```
JUC全局桶与AQS关系：
```
// 都继承/使用了 AQS：
// ReentrantLock       → 基于 AQS 实现互斥锁
// ReentrantReadWriteLock → 基于 AQS 实现读写锁
// Semaphore           → 基于 AQS 实现信号量
// CountDownLatch      → 基于 AQS 实现倒数门闩
// ThreadPoolExecutor  → Worker 类使用 AQS
// SynchronousQueue / DelayQueue → 内部使用了 AQS
```
### 核心数据结构
state状态变量
```
// 核心：一个 volatile 的 int 变量
private volatile int state;

// 提供三个方法操作 state（CAS 保证线程安全）
protected final int getState() {
    return state;
}

protected final void setState(int newState) {
    state = newState;
}

// CAS 原子更新
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

// state 的含义由子类决定：
// ReentrantLock：0=没人持有锁，1=有人持有，N=可重入 N 次
// Semaphore：剩余许可数量
// CountDownLatch：剩余待计数数量
```
Node节点
```
// 等待队列的节点
static final class Node {
    // 节点状态
    static final int CANCELLED = 1;   // 已取消（超时或中断）
    static final int SIGNAL = -1;     // 后继节点需要唤醒
    static final int CONDITION = -2;  // 等待在条件队列
    static final int PROPAGATE = -3;  // 共享模式传播

    volatile int waitStatus;    // 节点状态
    volatile Node prev;         // 前驱节点
    volatile Node next;         // 后继节点
    volatile Thread thread;     // 当前线程
    Node nextWaiter;            // 条件队列中的下一个节点

    // 模式
    static final Node SHARED = new Node();  // 共享模式（信号量、门闩）
    static final Node EXCLUSIVE = null;     // 独占模式（互斥锁）
}

// 节点状态流转：
// 初始：0
// 入队后：前驱节点设为 SIGNAL（-1）
// 超时/中断：CANCELLED（1）
// 条件等待：CONDITION（-2）
```
队列结构
```
AQS 等待队列（CLH 变体，双向链表）：

head ──→ Node1 ──→ Node2 ──→ Node3 ──→ tail
 │         │         │         │
thread   thread   thread   thread
(占位)   (等待中)  (等待中)  (等待中)

head 是哨兵节点（不存线程）
获取锁失败的线程 → 包装成 Node → 尾插法入队 → 阻塞
```
### 加锁流程
acquire流程
```
// 独占模式获取（ReentrantLock.lock() 的底层）
public final void acquire(int arg) {
    // ① tryAcquire：尝试获取锁（子类实现）
    //   成功 → 直接返回
    if (!tryAcquire(arg) &&
        // ② 失败 → acquireQueued：加入等待队列
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // ③ 如果中断了，恢复中断标志
        selfInterrupt();
}
```
addWaiter：入队
```
// 把当前线程包装成 Node，加入队列尾部
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);

    // 快速入队：先尝试一次 CAS
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        // CAS 设置尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // CAS 失败 → 用自旋重试入队
    enq(node);
    return node;
}

// 自旋入队
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
            // 队列为空 → 初始化哨兵节点（CAS）
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {  // CAS 成功 → 入队完成
                t.next = node;
                return t;
            }
        }
    }
}
```
acquireQueued：排队等待
```
// 入队后，循环检查自己是否是第一个等待节点
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();  // 前驱节点

            // 如果前驱是 head（自己是第一个等待的）
            if (p == head && tryAcquire(arg)) {
                // 尝试获取锁成功！
                setHead(node);  // 把自己设为新的 head
                p.next = null;  // 帮助 GC
                failed = false;
                return interrupted;
            }

            // 没获取到 → 检查并阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);  // 异常 → 取消
    }
}
```
shouldParkAfterFailedAcquire：决定是否阻塞
```
// 检查前驱节点状态，决定是否阻塞当前线程
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;

    if (ws == Node.SIGNAL) {
        // 前驱已设置 SIGNAL → 可以安心阻塞（前驱会唤醒我）
        return true;
    }
    if (ws > 0) {
        // 前驱已取消 → 跳过它
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 前驱是 0 或 PROPAGATE → 设置 SIGNAL，下次再检查
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;  // 不阻塞，再试一次（避免遗漏唤醒）
}
```
parkAndCheckInterrupt：阻塞
```
private final boolean parkAndCheckInterrupt() {
    // 阻塞当前线程（LockSupport.park —— 不释放 CPU）
    LockSupport.park(this);
    // 唤醒后返回是否被中断
    return Thread.interrupted();
}
```
加锁流程总结
```
lock() → acquire()
    │
    ├── tryAcquire() 成功？ → 拿到锁 ✅
    │
    ├── 失败 → addWaiter() 入队（CAS + 自旋）
    │
    ├── acquireQueued() 循环
    │   ├── 前驱是 head 且 tryAcquire 成功？ → 拿锁，成为新 head ✅
    │   ├── 否则 → shouldParkAfterFailedAcquire
    │   │   ├── 前驱 SIGNAL？ → park() 阻塞
    │   │   └── 前驱取消？ → 跳过，重试
    │   └── 唤醒后 → 重新循环检查
```
### 解锁流程
release流程
```
// 独占模式释放（ReentrantLock.unlock() 的底层）
public final boolean release(int arg) {
    // ① tryRelease：尝试释放（子类实现）
    if (tryRelease(arg)) {
        Node h = head;
        // ② 头节点不为空且需要唤醒
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);  // 唤醒后继节点
        return true;
    }
    return false;
}
```
unparkSuccessor：唤醒后继
```
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        // 清空 SIGNAL 状态
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        // 后继为空或已取消 → 从尾部往前找最近的有效节点 prev是可信的，入队时先设置，next是不可靠的，入队时后设置，可能暂时为空，不直接使用prev的原因，next是最快获取的，prev是兜底，当next为空时，使用prev。
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);  // 唤醒线程
}
```
解锁流程总结
```
unlock() → release(1)
    │
    ├── tryRelease() 成功（state 减到 0）
    │   ├── head 存在且 waitStatus != 0？
    │   │   ├── 是 → unparkSuccessor() 唤醒后继节点
    │   │   └── 否 → 不需要唤醒（没有等待者）
    │
    └── 唤醒的线程从 park() 返回 → acquireQueued 循环 → 尝试抢锁
```
### 公平锁vs非公平锁
非公平锁 ReentrantLock默认
```
// 非公平：新线程先尝试抢锁，不管队列里有没有等待者
final void lock() {
    // ① 先直接 CAS 抢一次锁（不管队列）
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());  // 抢到！
    else
        // ② 抢不到才走 AQS 队列
        acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    // 非公平版：直接尝试获取
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 直接 CAS，不检查队列
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 可重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        setState(nextc);
        return true;
    }
    return false;
}
```
公平锁
```
// 公平：必须排队，新线程不能插队
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 关键区别：先检查队列有没有人排队
        if (!hasQueuedPredecessors() &&   // ← 公平锁的核心
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 可重入逻辑相同
    ...
}

// hasQueuedPredecessors：队列中是否有比当前线程更早的等待者
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    // 队列不为空，且第一个等待节点不是当前线程
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
对比

|对比|公平锁|非公平锁|
|---|---|---|
|**获取顺序**|FIFO 严格排队|可以插队|
|**吞吐量**|低|高（减少上下文切换）|
|**饥饿**|不会饿死|可能饿死（但概率低）|
|**实现**|tryAcquire 前检查队列|先 CAS 抢一次|
|**默认**|—|ReentrantLock 默认|
### 面试高频

> **Q:** "公平锁和非公平锁的区别？为什么默认非公平？" 
> **A:** "公平锁严格按 FIFO 排队，非公平锁允许新线程插队。
> 默认用非公平锁是因为：① 吞吐量更高，插队减少了线程唤醒的上下文切换开销；
> ② 线程持有锁时间通常很短，插队抢到的概率高，减少排队。代价是可能造成饥饿，但实际概率很低。"
### AQS的应用
ReentrantLock的完整流程
```
// ReentrantLock 使用 AQS 的方式：
// ① 定义内部类 Sync 继承 AQS
// ② 实现 tryAcquire / tryRelease 钩子方法

public class ReentrantLock implements Lock {
    private final Sync sync;

    // 内部类：继承 AQS
    abstract static class Sync extends AbstractQueuedSynchronizer {
        // 实现钩子方法
        abstract void lock();

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
    }

    // 公平版
    static final class FairSync extends Sync {
        protected final boolean tryAcquire(int acquires) {
            // 公平获取（检查队列）
        }
    }

    // 非公平版
    static final class NonfairSync extends Sync {
        protected final boolean tryAcquire(int acquires) {
            // 非公平获取（先抢）
        }
    }
}
```
Semaphore的AQS应用
```
// 信号量：共享模式
public class Semaphore {
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) {
            setState(permits);  // state = 剩余许可数
        }

        // 尝试获取许可
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||  // 许可不够
                    compareAndSetState(available, remaining))  // CAS 扣减
                    return remaining;
            }
        }

        // 释放许可
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (compareAndSetState(current, next))  // CAS 增加
                    return true;
            }
        }
    }
}
```
CountDownLanch的AQS应用
```
// 倒数门闩：共享模式
public class CountDownLatch {
    private final Sync sync;

    Sync(int count) {
        setState(count);  // state = 待计数数量
    }

    // 计数 -1
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;  // 计数到 0 → 允许通过
    }

    // 释放一个计数
    protected boolean tryReleaseShared(int releases) {
        for (;;) {
            int c = getState();
            if (c == 0) return false;  // 已经是 0
            int nextc = c - 1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;  // 减到 0 → 唤醒所有等待者
        }
    }
}
```
### 共享模式 vs 独占模式

| 对比     | 独占模式          | 共享模式                     |
| ------ | ------------- | ------------------------ |
| **获取** | tryAcquire    | tryAcquireShared         |
| **释放** | tryRelease    | tryReleaseShared         |
| **唤醒** | 唤醒一个后继        | 唤醒所有后继（传播）               |
| **代表** | ReentrantLock | Semaphore、CountDownLatch |
| **语义** | 一个线程独享        | 多个线程共享                   |
### 面试高频题目
#### 题目1：AQS的核心是什么？
```
// 三个核心：
// ① state（volatile int）—— 同步状态
// ② CLH 变体等待队列 —— 获取失败的线程排队
// ③ CAS —— 原子操作 state 和队列
// ④ 模板方法模式 —— 子类实现 tryAcquire/tryRelease 钩子
```
#### 题目2：AQS为什么使用CLH队列？
```
// CLH 队列优点：  CLH：基于链表实现的，用于实现自旋锁的FIFO等待队列。
// ① FIFO 公平性保障
// ② 入队/出队都是 O(1)
// ③ 无锁化（CAS 入队）
// ④ 每个节点只依赖前驱状态（前驱唤醒后继）

// AQS 用的是 CLH 的变体：  CLH队列+阻塞（park/unpark）+状态机
// 原版 CLH 自旋，AQS 改为阻塞（park/unpark）
// 原版单向链表，AQS 改为双向
```
#### 题目3：AQS的state在可重入锁着怎么用？
```
// ReentrantLock 中 state 的语义：
// state = 0：锁未被持有
// state = 1：锁被某个线程持有 1 次
// state = N：同一线程可重入 N 次

// 加锁：state++（CAS）
// 解锁：state--，减到 0 才真正释放
```
#### 题目4：AQS中断是怎么处理的？
```
// lock() 不响应中断（acquire 忽略中断）
// lockInterruptibly() 响应中断（acquireInterruptibly 抛异常）

// acquireInterruptibly:
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);  // 中断时抛异常，不继续等
}
```
