## 锁家族
ReentrantLock->ReentrantReadWriteLock->StampedLock->死锁与排查->锁对比总结
### ReentrantLock
基本使用
```
// ReentrantLock：基于 AQS 的可重入互斥锁
// 比 synchronized 更灵活

public class Counter {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock();  // 加锁
        try {
            count++;
        } finally {
            lock.unlock();  // 必须在 finally 中释放！
        }
    }
}
```
四大特性（对比Synchronnized的优势）
```
// ① 可中断
ReentrantLock lock = new ReentrantLock();
Thread t = new Thread(() -> {
    try {
        lock.lockInterruptibly();  // 可以被其他线程中断
    } catch (InterruptedException e) {
        System.out.println("被中断了，不再等待锁");
        return;
    }
});

// ② 可超时
if (lock.tryLock(2, TimeUnit.SECONDS)) {  // 最多等 2 秒
    try {
        // 拿到锁
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("2 秒内没拿到锁，放弃");
}

// ③ 公平锁
ReentrantLock fairLock = new ReentrantLock(true);  // 公平锁
ReentrantLock unfairLock = new ReentrantLock(false);  // 非公平（默认）

// ④ 多个条件变量（Condition）
ReentrantLock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();   // 条件 1
Condition notFull = lock.newCondition();    // 条件 2

lock.lock();
try {
    while (queue.isEmpty()) {
        notEmpty.await();  // 等待非空
    }
    // 消费
    notFull.signalAll();   // 唤醒生产者
} finally {
    lock.unlock();
}
```
### synchronized vs ReentrantLock

|对比|synchronized|ReentrantLock|
|---|---|---|
|**释放锁**|自动（JVM 保证）|手动（finally 中 unlock）|
|**可中断**|❌|✅ lockInterruptibly()|
|**超时**|❌|✅ tryLock(timeout)|
|**公平锁**|❌ 非公平|✅ 可配置|
|**条件变量**|wait/notify（一个）|Condition（多个）|
|**锁分组**|同一把锁|不同 Condition 分组唤醒|
|**性能**|已优化，差距很小|极端竞争下略好|
### ReentrantReadWriteLock（读写锁）
为什么需要读写锁？
```
// 场景：读多写少
// synchronized：读读互斥 —— 两个读线程也不能并行，太浪费
// 读写锁：读读共享，读写互斥，写写互斥 —— 提高并发度

public class Cache {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();    // 读锁
    private final Lock writeLock = rwLock.writeLock();  // 写锁
    private Map<String, Object> data = new HashMap<>();

    // 读操作 —— 多个线程可以同时读
    public Object get(String key) {
        readLock.lock();
        try {
            return data.get(key);
        } finally {
            readLock.unlock();
        }
    }

    // 写操作 —— 必须独占
    public void put(String key, Object value) {
        writeLock.lock();
        try {
            data.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```
锁的三种组合

|组合|读锁|写锁|是否允许|
|---|---|---|---|
|读-读|读锁|读锁|✅ 共享|
|读-写|读锁|写锁|❌ 互斥|
|写-写|写锁|写锁|❌ 互斥|
实现原理（基于AQS）
```
// 读写锁用 AQS 的 state 表示两种锁
// state 高 16 位：写锁的重入次数
// state 低 16 位：读锁的持有数量

// 例如：state = 0x00050003
// 高 16 位 = 5：写锁重入 5 次
// 低 16 位 = 3：3 个读锁持有

// 通过一个 int 变量管理两种锁 —— 巧妙
```
锁降级
```
// 锁降级：持有写锁 → 获取读锁 → 释放写锁
// 目的：保证数据的一致性（写锁释放前，读锁先拿到）

public void process() {
    writeLock.lock();
    try {
        // 修改数据
        data = compute();

        // 降级：先获取读锁
        readLock.lock();
    } finally {
        writeLock.unlock();  // 释放写锁，但仍持有读锁
    }

    // 此时持有读锁，保证 data 读取的一致性
    try {
        // 使用 data（其他线程不能写）
    } finally {
        readLock.unlock();
    }
}

// 注意：锁只能降级，不能升级！
// 读锁 → 写锁 会死锁（不推荐，可能阻塞等待）
```
### 面试高频

> **Q:** "什么是锁降级？为什么需要？" 
> **A:** "锁降级是写锁降级为读锁：先获取写锁，再获取读锁，最后释放写锁。目的是在释放写锁前保证数据的可见性——写锁释放后，其他线程可能立刻抢到写锁修改数据，而降级后持有的读锁保证了当前线程读到的数据是刚才写的版本。"

> **Q:** "读写锁适合什么场景？" 
> **A:** "读多写少的场景，比如缓存、配置加载。读读共享提高了并发度。缺点是写锁饥饿（读线程多时写线程很难拿到锁），JDK 提供了非公平模式缓解。"
### StampedLock
邮戳锁
为什么需要StampedLock
```
// ReentrantReadWriteLock 的问题：
// 读锁加锁/解锁都有 CAS 开销
// 读多时写线程可能饥饿

// StampedLock 的改进：
// ① 乐观读 —— 读不加锁，性能极高
// ② 乐观读失败再升级为悲观读锁

// 适用：读多写极少，且对性能要求极高的场景
```
三种模式
```java
public class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // ① 写锁（独占）
    public void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();  // 获取写锁，返回版本戳
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);  // 释放写锁
        }
    }

    // ② 悲观读锁
    public double distanceFromOrigin() {
        long stamp = sl.readLock();  // 悲观读锁
        try {
            return Math.sqrt(x * x + y * y);
        } finally {
            sl.unlockRead(stamp);
        }
    }

    // ③ 乐观读（核心特性！）
    public double optimisticRead() {
        // 获取版本戳（无锁，不阻塞）
        long stamp = sl.tryOptimisticRead();

        // 读取数据（不加锁）
        double currentX = x;
        double currentY = y;

        // 验证：读取期间有没有写操作
        if (!sl.validate(stamp)) {
            // 有写操作 → 数据可能不一致 → 升级为悲观读锁
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```
乐观读原理
```
tryOptimisticRead() 获取版本戳
    ↓
读取数据（无锁，无 CAS，无阻塞 —— 极快）
    ↓
validate(stamp) 验证版本戳
    ├── 有效 → 数据没被修改 → 直接用 ✅
    └── 无效 → 数据可能被修改 → 升级悲观读锁重新读
```
三种锁对比：

|对比|读锁|写锁|乐观读|
|---|---|---|---|
|**加锁**|CAS 开销|CAS 开销|无锁|
|**阻塞**|会阻塞|会阻塞|不阻塞|
|**数据一致性**|强一致|强一致|可能不一致（需验证）|
|**性能**|中|中|最高|
|**使用**|读多写少|写操作|读极多写极少|
注意
```
// StampedLock 不可重入！
// 不支持 Condition！
// 悲观锁获取失败会阻塞，可用 tryXxxLock 避免
// 不是基于 AQS 实现的（内部是 CLH 队列自旋）
```
### 死锁与排查
死锁的四个必要条件
```
// ① 互斥：资源一次只能被一个线程使用
// ② 持有并等待：持有资源的同时等待其他资源
// ③ 不可剥夺：资源不能被强制夺走
// ④ 循环等待：多个线程形成环路等待

// 破坏任意一个条件，死锁就不会发生
```
死锁示例
```
public class DeadlockDemo {
    private static final Object resourceA = new Object();
    private static final Object resourceB = new Object();

    public static void main(String[] args) {
        // 线程 1：先拿 A，再拿 B
        Thread t1 = new Thread(() -> {
            synchronized (resourceA) {
                System.out.println("线程1: 拿到 A");
                try { Thread.sleep(100); } catch (Exception e) {}
                synchronized (resourceB) {  // 等待 B —— B 被线程 2 持有
                    System.out.println("线程1: 拿到 B");
                }
            }
        });

        // 线程 2：先拿 B，再拿 A
        Thread t2 = new Thread(() -> {
            synchronized (resourceB) {
                System.out.println("线程2: 拿到 B");
                try { Thread.sleep(100); } catch (Exception e) {}
                synchronized (resourceA) {  // 等待 A —— A 被线程 1 持有
                    System.out.println("线程2: 拿到 A");
                }
            }
        });

        t1.start();
        t2.start();
        // 结果：两个线程互相等待，永远卡死！
        // 线程1: 拿到 A + 线程2: 拿到 B → 死锁
    }
}
```
死锁排查
```
# ① 找到 Java 进程 PID
jps -l
# 输出：12345 com.example.DeadlockDemo

# ② 用 jstack 分析线程栈
jstack 12345

# ③ 找到死锁信息
# Found one Java-level deadlock:
# =============================
# "Thread-1":
#   waiting to lock monitor 0x0000000002a1b008 (object 0x0000000780a8c2d8, ...)
#   which is held by "Thread-0"
# "Thread-0":
#   waiting to lock monitor 0x0000000002a1b2a0 (object 0x0000000780a8c2e8, ...)
#   which is held by "Thread-1"
```
避免死锁的策略
```
// 策略一：按固定顺序加锁（最有效）
public void transfer(Account from, Account to, double amount) {
    // 按账户 ID 排序，保证所有线程加锁顺序一致
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;

    synchronized (first) {
        synchronized (second) {
            // 转账逻辑
        }
    }
}

// 策略二：超时放弃（tryLock）
ReentrantLock lock1 = new ReentrantLock();
ReentrantLock lock2 = new ReentrantLock();

if (lock1.tryLock(1, TimeUnit.SECONDS)) {
    try {
        if (lock2.tryLock(1, TimeUnit.SECONDS)) {
            try {
                // 业务逻辑
            } finally {
                lock2.unlock();
            }
        }
    } finally {
        lock1.unlock();
    }
}

// 策略三：减少锁持有时间（缩小同步块）

// 策略四：避免嵌套锁（一个线程尽量只持有一把锁）
```
### 锁对比总结

|对比|synchronized|ReentrantLock|ReentrantReadWriteLock|StampedLock|
|---|---|---|---|---|
|**互斥性**|独占|独占|读写分离|读/写/乐观|
|**可重入**|✅|✅|✅|❌|
|**公平锁**|❌|✅|✅|✅|
|**可中断**|❌|✅|✅|✅|
|**超时**|❌|✅|✅|✅|
|**条件变量**|wait/notify|Condition|Condition|❌|
|**乐观读**|❌|❌|❌|✅|
|**实现**|monitor|AQS|AQS|CLH 自旋|
|**适用**|通用|灵活锁|读多写少|读极多写极少|
选型建议
```
通用场景？              → synchronized（简单）或 ReentrantLock（灵活）
读多写少（缓存）？       → ReentrantReadWriteLock
读极多写极少（高性能）？  → StampedLock 乐观读
只是简单计数/标志？      → volatile / AtomicInteger（更轻）
```

