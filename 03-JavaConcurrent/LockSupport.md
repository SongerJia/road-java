## LockSupport
LockSupport是什么->核心方法->park/unpark原理->与wait/notify对比->中断处理->应用场景->面试高频题
### LockSupport是什么？
LockSupport：线程阻塞与唤醒的底层工具类，基于Unsafe实现。AQS的阻塞park和唤醒unpark基础。
为什么重要
```
// AQS 的 acquireQueued 中：
LockSupport.park(this);   // 阻塞等待锁 ← 就是这个
LockSupport.unpark(s.thread);  // 唤醒后继线程 ← 就是这个

// 所以 LockSupport 是 AQS 的地基
// 面试问 AQS 深了，一定会问到 park/unpark
```
核心方法
```
// 阻塞当前线程
LockSupport.park();                    // 无限期阻塞
LockSupport.parkNanos(long nanos);     // 阻塞指定纳秒
LockSupport.parkUntil(long deadline);  // 阻塞到指定时间戳

// 唤醒指定线程
LockSupport.unpark(Thread thread);     // 唤醒指定线程
```
### park/unpark原理
和wait/notify的区别
```
wait/notify：基于"对象监视器"（monitor），必须 synchronized
park/unpark：基于"许可证"（permit），不需要锁

关键：unpark 可以先于 park 执行！
```
许可证机制
```
// 每个线程有一个"许可证"（permit），类似信号量，但只有 0/1

// park() 逻辑：
// 如果有许可证（permit=1）→ 消耗许可证，直接返回（不阻塞）
// 如果没有许可证（permit=0）→ 阻塞

// unpark() 逻辑：
// 发放许可证（permit=1）
// 如果线程正在 park 阻塞 → 唤醒它
// 如果线程还没 park → 许可证被记住，下次 park 直接通过
```
先unpark后park
```
// 这是 LockSupport 最大的特点：unpark 可以先执行！

Thread t = new Thread(() -> {
    LockSupport.park();  // ② 因为许可证已经在，直接通过，不阻塞！
    System.out.println("线程执行完成");
});

LockSupport.unpark(t);  // ① 先发放许可证
t.start();              // ② 启动线程，park 发现许可证存在 → 不阻塞

// wait/notify 做不到这个：
// 如果先 notify 后 wait → 会丢失通知，永远阻塞！
```
对比wait/notify

|对比|wait/notify|park/unpark|
|---|---|---|
|**前置条件**|必须在 synchronized 中|不需要锁|
|**唤醒顺序**|notify 必须在 wait 之后|unpark 可先于 park|
|**阻塞方式**|释放 monitor 锁|不涉及锁|
|**超时**|wait(timeout)|parkNanos/parkUntil|
|**唤醒单个/全部**|notify/notifyAll|unpark（指定线程）|
|**实现**|monitor 机制|Unsafe（许可证）|
|**精确唤醒**|notify 随机唤醒一个|✅ 精确唤醒指定线程|
### 中断处理
park对中断的响应
```
// 关键：park 阻塞时被 interrupt，不会抛异常！
// 只是"默默地返回"（像被唤醒一样）

Thread t = new Thread(() -> {
    System.out.println("线程开始 park");
    LockSupport.park();          // 阻塞在这里
    System.out.println("park 返回了");
    // 注意：没有 InterruptedException！
    System.out.println("中断标志: " + Thread.currentThread().isInterrupted());  // true
});

t.start();
Thread.sleep(1000);
t.interrupt();  // 中断线程

// 输出：
// 线程开始 park
// park 返回了
// 中断标志: true
```
和sleep/wait的区别
```
// sleep/wait 被中断 → 抛 InterruptedException（清除中断标志）
// park 被中断 → 静默返回（不清除中断标志，标志保留）

// 所以 AQS 中：
LockSupport.park(this);
// park 返回后，通过 Thread.interrupted() 检查中断标志
// 决定是"继续排队"还是"响应中断"
```
### AQS中的应用
```
// AQS 中 park 的使用（acquireQueued）：
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();

            // 抢到锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null;
                failed = false;
                return interrupted;
            }

            // 抢不到 → park 阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;  // park 被中断唤醒
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// parkAndCheckInterrupt：
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);   // 阻塞当前线程
    return Thread.interrupted();  // 返回并清除中断标志
}

// AQS 中 unpark 的使用（release）：
private void unparkSuccessor(Node node) {
    ...
    LockSupport.unpark(s.thread);  // 唤醒后继线程
}
```
执行流程
```
线程抢锁失败 → 入队 → park 阻塞
    ↓ 锁释放
unlock → unpark 后继线程 → park 返回
    ↓
线程醒来 → 重新尝试抢锁 → 抢到 → 出队
```
### 应用场景
#### 场景1：AQS锁
```
ReentrantLock lock = new ReentrantLock();
lock.lock();   // 底层：抢不到锁 → park
lock.unlock(); // 底层：unpark 唤醒等待者
```
#### 场景2：自定义阻塞队列
```
// 用 park/unpark 实现一个简单的阻塞队列
public class MyBlockingQueue<E> {
    private final LinkedList<E> queue = new LinkedList<>();
    private final int capacity;

    public MyBlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    public void put(E e) {
        synchronized (queue) {
            while (queue.size() == capacity) {
                // 队列满 → 阻塞自己
                LockSupport.park();
            }
            queue.addLast(e);
            // 唤醒所有等待的消费者
            queue.notifyAll();
        }
    }

    public E take() throws InterruptedException {
        synchronized (queue) {
            while (queue.isEmpty()) {
                // 队列空 → 等生产者唤醒
                queue.wait();  // 这里用 wait，因为需要释放锁
            }
            return queue.removeFirst();
        }
    }
}
```
#### FIFO精准唤醒
```
// park/unpark 可以精准唤醒指定线程
// 比 notifyAll（唤醒所有）效率更高

Thread t1 = new Thread(() -> {
    LockSupport.park();
    System.out.println("t1 被唤醒");
});

Thread t2 = new Thread(() -> {
    LockSupport.park();
    System.out.println("t2 被唤醒");
});

t1.start();
t2.start();

LockSupport.unpark(t1);  // 精准唤醒 t1，t2 继续阻塞
// 输出：只有 t1 被唤醒
```
### 面试高频题
#### 题目1：park和wait的区别？
```
// ① 前置条件：wait 需要 synchronized；park 不需要
// ② 唤醒顺序：unpark 可以先于 park；notify 必须先于 wait
// ③ 中断：wait 抛 InterruptedException；park 静默返回
// ④ 唤醒方式：notify 随机；unpark 精准
// ⑤ 实现：monitor vs 许可证
```
#### unpark先与park会怎么样？
```
// 不会阻塞！
// 许可证被记住，park 时发现许可证存在 → 直接通过
// 这就是"先放行后检查"的机制
```
#### park会响应中断吗？
```
// 会返回，但不抛异常
// 中断标志保留（不清除）
// 调用方通过 Thread.interrupted() 检测
```
#### Locksupport和Condition的关系？
```
// Condition 的 await/signal 底层也是 park/unpark
// ConditionObject 中：
// await() 最后会 LockSupport.park(this)
// signal() 会 LockSupport.unpark(线程)
```

