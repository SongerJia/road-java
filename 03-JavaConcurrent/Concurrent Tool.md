## 并发工具类
三大同步工具->CAS原理->ThreadLocal->对比总结
### 三大同步工具
CountDownLanch
```java
// 场景：一个线程等待其他 N 个线程都完成后再继续
// 应用：主线程等所有子任务完成

// 示例：等待 3 个任务都完成
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        // 计数 3
        CountDownLatch latch = new CountDownLatch(3);

        for (int i = 1; i <= 3; i++) {
            int taskId = i;
            new Thread(() -> {
                try {
                    System.out.println("任务 " + taskId + " 执行中...");
                    Thread.sleep(1000);
                    System.out.println("任务 " + taskId + " 完成");
                } catch (InterruptedException e) {
                } finally {
                    latch.countDown();  // 计数 -1（必须放 finally）
                }
            }).start();
        }

        System.out.println("主线程等待...");
        latch.await();  // 阻塞，直到计数归 0
        System.out.println("所有任务完成，主线程继续");
    }
}
//核心特点
一次性（计数归 0 后不能复用）
`countDown()` 计数 -1
`await()` 阻塞等待计数归 0
基于 AQS 共享模式实现
```
CycliBarrier
```java
// 场景：N 个线程互相等待，都到达屏障后才一起继续
// 应用：并发计算，等所有线程到齐后汇总

public class CyclicBarrierDemo {
    public static void main(String[] args) {
        int threadCount = 3;
        // 屏障：3 个线程到齐后，执行 barrierAction
        CyclicBarrier barrier = new CyclicBarrier(threadCount, () -> {
            System.out.println("=== 所有线程到齐，汇总结果 ===");
        });

        for (int i = 1; i <= threadCount; i++) {
            int taskId = i;
            new Thread(() -> {
                try {
                    System.out.println("线程 " + taskId + " 开始计算...");
                    Thread.sleep(1000);
                    System.out.println("线程 " + taskId + " 计算完成，等待其他线程");
                    barrier.await();  // 等待其他线程到达
                    System.out.println("线程 " + taskId + " 继续执行后续");
                } catch (Exception e) {
                }
            }).start();
        }
    }
}
//核心特点
可循环复用（计数归零后自动重置）
`await()` 等待其他线程到达
基于 ReentrantLock + Condition 实现（不是 AQS 共享）
```
Semaphore
```java
// 场景：控制同时访问资源的线程数量（限流）
// 应用：连接池、限流控制

public class SemaphoreDemo {
    public static void main(String[] args) {
        // 最多 2 个线程同时访问
        Semaphore semaphore = new Semaphore(2);

        for (int i = 1; i <= 5; i++) {
            int taskId = i;
            new Thread(() -> {
                try {
                    semaphore.acquire();  // 获取许可（没有则阻塞）
                    System.out.println("线程 " + taskId + " 进入，剩余许可: "
                        + semaphore.availablePermits());
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                } finally {
                    semaphore.release();  // 释放许可
                    System.out.println("线程 " + taskId + " 离开");
                }
            }).start();
        }
        // 输出：同一时刻最多 2 个线程进入
    }
}
//核心特点
控制并发数（限流）
`acquire()` 获取许可（无许可则阻塞）
`release()` 释放许可
基于 AQS 共享模式实现
公平/非公平可选
```
三者对比

|对比|CountDownLatch|CyclicBarrier|Semaphore|
|---|---|---|---|
|**语义**|等 N 个任务完成|N 个线程互相等待|控制并发数|
|**方向**|等待者 → 被等待者|互相等待|竞争许可|
|**复用**|❌ 一次性|✅ 可循环|✅ 可复用|
|**计数**|只减不增|到达即减，到 0 重置|许可数量|
|**实现**|AQS 共享|ReentrantLock + Condition|AQS 共享|
|**典型应用**|主线程等子任务|分治计算汇总|连接池限流|
面试高频
> **Q:** "CountDownLatch 和 CyclicBarrier 的区别？" 
> **A:** " ① 等待关系：CountDownLatch 是一个线程等 N 个线程（1 等 N）；CyclicBarrier 是 N 个线程互相等（N 等 N） ② 复用：CountDownLatch 一次性；CyclicBarrier 可循环 ③ 触发动作：CyclicBarrier 构造器可传 barrierAction（到齐后执行）；CountDownLatch 没有 ④ 实现：CountDownLatch 基于 AQS 共享；CyclicBarrier 基于 ReentrantLock + Condition "

> **Q:** "Semaphore 怎么实现限流？" 
> **A:** "Semaphore 维护一个许可计数（AQS 的 state），acquire 时 CAS 减 1（不够就阻塞），release 时 CAS 加 1 并唤醒等待线程。这样同时执行的线程数不超过许可数。比如数据库连接池设置 Semaphore(10)，最多 10 个连接同时使用。"
### CAS原理
CAS：Compare And Swap 比较并交换，一种无锁原子操作，比较内存中的值和期望值，如果相同则更新，否则不操作。
```
// 伪代码：
boolean compareAndSwap(int* addr, int expected, int newValue) {
    if (*addr == expected) {   // 比较
        *addr = newValue;      // 交换
        return true;
    }
    return false;
}

// Java 中的实现：Unsafe 类            对象  偏移量    期望值        目标值
Unsafe.getUnsafe().compareAndSwapInt(obj, offset, expectedValue, newValue);
```
CAS的三要素

|要素|说明|
|---|---|
|**内存地址**|要操作的变量位置|
|**期望值**|认为当前的值（expected）|
|**新值**|要更新的值（update）|
硬件层面的实现
```
// CAS 是 CPU 指令级别的原子操作
// x86 平台：CMPXCHG 指令（带 lock 前缀）
// 不需要加锁，性能高

// 乐观锁思想：
// 不加锁，假设不会冲突
// 冲突时重试（自旋）
```
AtomicInteger
```
// 核心类：AtomicInteger
AtomicInteger count = new AtomicInteger(0);

// ① 原子自增
count.incrementAndGet();  // 返回 +1 后的值

// 源码（自旋 CAS）：
public final int incrementAndGet() {
    for (;;) {
        int current = get();           // 读取当前值
        int next = current + 1;        // 计算新值
        if (compareAndSet(current, next))  // CAS 更新
            return next;               // 成功返回
        // 失败 → 循环重试（自旋）
    }
}
```
ABA问题
```
// 问题：CAS 只比较值，不关心值被改过几次
// 线程 A 读到 值为 5
// 线程 B 把 5 改为 6，再改回 5
// 线程 A 的 CAS 比较：还是 5 → 成功！
// 但中间被改过两次 —— A 不知道

// 实际危害：
// 共享变量是对象引用时，中间被换过对象
// 虽然最终值一样，但对象内容可能变了

// 示例（链表栈）：
// 栈顶 A → B → C
// 线程 A 读到栈顶 A
// 线程 B 弹出 A，压入 D → 栈顶 D → A?（引用可能被复用）
// 线程 A 的 CAS 发现栈顶还是 A 的引用 → 更新成功
// 但此时栈已经不是原来的栈了！

// 解决方案：版本号
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(5, 0);
// 每次修改版本号 +1
// CAS 时同时比较值和版本号

// 类似方案：AtomicMarkableReference（boolean 标记）
```
CAS优缺点

|优点|缺点|
|---|---|
|无锁，性能高|ABA 问题|
|不会死锁|自旋消耗 CPU（竞争激烈时）|
|无阻塞|只能保证一个变量的原子性|
|乐观，冲突少时快|不能保证代码块的原子性|
使用场景
```
// ✅ 适合：单变量简单操作
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();

// ❌ 不适合：复合操作、多变量、代码块
// 用 synchronized 或 Lock
```
### ThreadLocal
ThreadLocal：线程局部变量。每个线程都有自己独立的变量副本，互不干扰。
```
// 基本使用
ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

// 线程 A 设置
threadLocal.set(100);

// 线程 B 设置（和 A 互不影响）
threadLocal.set(200);

// 线程 A 读取 → 100（自己的副本）
// 线程 B 读取 → 200（自己的副本）
```
底层结构
```
// ThreadLocal 的实现原理：
// 每个 Thread 内部有一个 ThreadLocalMap
// ThreadLocal 作为 key，值作为 value

// Thread 类内部：
public class Thread {
    // 每个线程自己的 ThreadLocalMap
    ThreadLocal.ThreadLocalMap threadLocals;
}

// ThreadLocalMap 结构：
// Entry[] table
// Entry：key = ThreadLocal（弱引用！），value = 值（强引用）

// 查找流程：
threadLocal.get()
  → 获取当前线程 Thread.currentThread()
    → 从当前线程的 threadLocals 中查找
      → key 是当前 ThreadLocal → 返回 value
```
ThreadLocal的内存泄漏
```
// 问题：key 是弱引用，value 是强引用

// 引用链：
// Thread → ThreadLocalMap → Entry（key=ThreadLocal 弱引用，value=强引用）

// 如果 ThreadLocal 对象不再被外部引用：
// key（ThreadLocal）被 GC 回收 → key 变为 null
// 但 value 还被 Entry 强引用 → value 无法回收 → 内存泄漏！

// 什么时候泄漏最严重？
// 线程池中的线程长期存活
// ThreadLocal 使用后没 remove()
// → Thread → ThreadLocalMap → Entry(null key) → value 一直无法回收
```
解决方案
```
// ① 使用后必须 remove()
ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
try {
    threadLocal.set(100);
    // 业务逻辑
} finally {
    threadLocal.remove();  // ✅ 必须移除，防止内存泄漏
}

// ② 但 ThreadLocal 源码也做了兜底：
// get()/set() 时，会顺带清理 key 为 null 的 Entry（expungeStaleEntry）
// 但这只是"顺带"，不是"保证"
// 如果不调用 get()/set()，依然泄漏
```
ThreadLocal的典型应用
```
// ① 数据库连接/事务管理（Spring 的 @Transactional）
public class ConnectionHolder {
    private static final ThreadLocal<Connection> CONN = new ThreadLocal<>();

    public static Connection getConnection() {
        Connection conn = CONN.get();
        if (conn == null) {
            conn = dataSource.getConnection();
            CONN.set(conn);
        }
        return conn;
    }

    public static void close() {
        Connection conn = CONN.get();
        if (conn != null) {
            conn.close();
            CONN.remove();  // 必须 remove！
        }
    }
}

// ② 用户信息传递（登录上下文）
public class UserContext {
    private static final ThreadLocal<User> USER = new ThreadLocal<>();

    public static void set(User user) { USER.set(user); }
    public static User get() { return USER.get(); }
    public static void clear() { USER.remove(); }
}
// 登录后：UserContext.set(user)
// 业务代码任意地方：UserContext.get() —— 不用层层传参

// ③ SimpleDateFormat 线程安全问题  
public class DateUtils {
    // ❌ 静态的 SimpleDateFormat 线程不安全  把计算结果存到了静态实例变量calendar中
    // private static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

    // ✅ 用 ThreadLocal 每个线程一份
    private static final ThreadLocal<SimpleDateFormat> SDF =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

    public static String format(Date date) {
        return SDF.get().format(date);
    }
}
建议使用 DateTimeFormatter 
不可变对象，所有字段fianl
格式化时将中间结果存在局部变量或方法栈里
不修改任何共享状态
```
### 面试高频

> **Q:** "ThreadLocal 的原理？" 
> **A:** "每个 Thread 内部有一个 ThreadLocalMap，ThreadLocal 作为 key、值作为 value。操作当前线程自己的 map，所以线程间互不干扰。"

> **Q:** "ThreadLocal 为什么会内存泄漏？怎么解决？" 
> **A:** "Entry 的 key 是弱引用，value 是强引用。ThreadLocal 被回收后 key 变 null，但 value 还强引用着，无法回收。尤其是线程池场景更严重。解决：使用后必须 remove()。"

> **Q:** "ThreadLocal 应用场景？" 
> **A:** "数据库连接、用户上下文、SimpleDateFormat、Spring 事务管理等线程隔离场景。"
### 并发工具对比
```
Semaphore：控制并发数（限流）
CountDownLatch：主线程等 N 个任务（1 等 N，一次性）
CyclicBarrier：N 个线程互相等待（N 等 N，可循环）
CAS：无锁原子操作（乐观锁，有 ABA 问题）
ThreadLocal：线程隔离变量（防泄漏要 remove）
```
选型场景
```
需要限流？              → Semaphore ✅
需要等待任务完成？        → CountDownLatch ✅
需要线程互相等待汇合？     → CyclicBarrier ✅
需要原子计数？           → AtomicInteger（CAS）✅
需要线程隔离数据？        → ThreadLocal ✅（记得 remove）
需要保证代码块原子性？     → synchronized / Lock（不是这些）
```

