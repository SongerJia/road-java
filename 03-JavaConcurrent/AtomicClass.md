## 原子类
原子类总览->基本类型原子类->引用原子类->数组原子类->FieldUpdater->工具方法->面试高频题
### 原子类总览
什么是原子类？
java.util.concurrent.atomic（原子类）：基于CAS实现的线程安全类，不需要加锁就能保证原子性，性能高于synchronized。
原子类的分类：
```
java.util.concurrent.atomic 包
  ├── 基本类型
  │   ├── AtomicInteger
  │   ├── AtomicLong
  │   └── AtomicBoolean
  │
  ├── 引用类型
  │   ├── AtomicReference
  │   ├── AtomicStampedReference（带版本号，解决 ABA）
  │   └── AtomicMarkableReference（带布尔标记）
  │
  ├── 数组
  │   ├── AtomicIntegerArray
  │   ├── AtomicLongArray
  │   └── AtomicReferenceArray
  │
  ├── 字段更新器
  │   ├── AtomicIntegerFieldUpdater
  │   ├── AtomicLongFieldUpdater
  │   └── AtomicReferenceFieldUpdater
  │
  └── 高性能累加器（Java 8）
      ├── LongAdder
      ├── LongAccumulator
      ├── DoubleAdder
      └── DoubleAccumulator
```
### 基本类型原子类
AtomicInteger
```
// 常用方法
AtomicInteger count = new AtomicInteger(0);

count.get();               // 0 —— 获取当前值
count.set(10);             // 直接设置
count.getAndSet(20);       // 返回旧值 10，设置为 20
count.compareAndSet(20, 30);  // 期望 20，是则更新为 30 → true

count.incrementAndGet();   // 自增并返回新值（++count）
count.getAndIncrement();   // 返回旧值并自增（count++）
count.decrementAndGet();   // 自减并返回新值（--count）
count.addAndGet(5);        // 加 5 并返回新值

count.updateAndGet(x -> x * 2);  // Java 8：函数式更新
count.accumulateAndGet(10, Integer::sum);  // Java 8：累加

// 使用场景：计数器
public class Counter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();  // 线程安全，无需加锁
    }

    public int get() {
        return count.get();
    }
}
```
AtomicBoolean
```
// 使用场景：开关标志（只允许一个线程设置成功）
public class FlagManager {
    private final AtomicBoolean flag = new AtomicBoolean(false);

    // 只允许第一个线程设置成功，返回 true
    public boolean tryAcquire() {
        return flag.compareAndSet(false, true);
    }

    // 释放
    public void release() {
        flag.set(false);
    }
}
```
AtomicLong
```
// 和使用 Integer 类似
// 场景：全局唯一 ID 生成器
public class IdGenerator {
    private static final AtomicLong sequence = new AtomicLong(0);

    public static long nextId() {
        return sequence.incrementAndGet();
    }
}
```
源码核心
```
// AtomicInteger 的 incrementAndGet
public final int incrementAndGet() {
    for (;;) {
        int current = get();              // 读当前值
        int next = current + 1;           // 计算新值
        if (compareAndSet(current, next)) // CAS 更新
            return next;                  // 成功
        // 失败 → 自旋重试
    }
}

// compareAndSet 底层
public final boolean compareAndSet(int expect, int update) {
    // Unsafe 类的 CAS 指令（CPU 原子操作）
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```
### 引用原子类
AtomicReference
```
// 对"引用"进行原子操作
public class User {
    private String name;
    private int age;
    // getter/setter...
}

// 原子更新对象引用
AtomicReference<User> userRef = new AtomicReference<>(new User("张三", 25));

// 原子替换整个对象
User newUser = new User("李四", 30);
userRef.compareAndSet(oldUser, newUser);

// 场景：并发下的配置热更新
public class ConfigHolder {
    private static final AtomicReference<Config> config =
        new AtomicReference<>(Config.load());

    // 原子替换配置，其他线程读取时要么旧要么新，不会读到中间态
    public static void update(Config newConfig) {
        config.set(newConfig);
    }

    public static Config get() {
        return config.get();
    }
}
```
解决ABA问题的版本号引用
```
// AtomicStampedReference —— 带版本号
// 同时比较"引用"和"版本号"，解决 ABA 问题

AtomicStampedReference<User> ref = new AtomicStampedReference<>(user, 0);

int[] stampHolder = new int[1];
User current = ref.get(stampHolder);  // 获取引用和版本号
int stamp = stampHolder[0];

// 更新时比较引用 + 版本号
boolean success = ref.compareAndSet(
    current, newUser,   // 期望引用、新引用
    stamp, stamp + 1    // 期望版本号、新版本号
);

// AtomicMarkableReference —— 带布尔标记
AtomicMarkableReference<User> ref2 = new AtomicMarkableReference<>(user, false);
// 只关心"是否被修改过"，不关心修改次数
```
### 数组原子类
```
// 数组中的每个元素都是原子操作的
// 注意：数组引用本身不可变，但元素可以原子更新

// AtomicIntegerArray
AtomicIntegerArray array = new AtomicIntegerArray(10);

array.get(0);                // 读取下标 0 的元素
array.set(0, 100);           // 设置下标 0
array.incrementAndGet(0);    // 下标 0 自增
array.addAndGet(0, 50);      // 下标 0 加 50
array.compareAndSet(0, 150, 200);  // 下标 0 的 CAS

// 场景：多个线程统计不同分区的数据
public class Stats {
    private final AtomicIntegerArray counts = new AtomicIntegerArray(16);

    public void increment(int partition) {
        counts.incrementAndGet(partition);  // 每个分区独立计数
    }
}
```
和普通数组+synchronized对比
```
// ❌ 普通数组 + synchronized
int[] array = new int[10];
synchronized (array) {
    array[0]++;
}
// 锁粒度是整个数组

// ✅ AtomicIntegerArray
AtomicIntegerArray array2 = new AtomicIntegerArray(10);
array2.incrementAndGet(0);
// 每个元素独立 CAS，无锁，粒度更细
```
### FieldUpdater字段更新器
为什么需要FieldUpdater?
```
// 需求：不修改现有类的代码，只对某个字段做原子操作
// 场景：第三方类、POJO 不想改造成 Atomic 类型

// 前提条件：
// ① 字段必须是 volatile
// ② 字段不能是 private（或使用反射获取）
// ③ 字段类型要匹配
```
使用示例
```
// 现有类（不想改造成 AtomicInteger）
public class Order {
    // 注意：字段必须 volatile！
    public volatile int status;

    public void updateStatus() {
        // ...
    }
}

// 使用 FieldUpdater 原子更新 status
public class OrderService {
    // 创建字段更新器
    private static final AtomicIntegerFieldUpdater<Order> STATUS_UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(Order.class, "status");

    // 原子更新状态（只允许从 0 变为 1）
    public boolean tryApprove(Order order) {
        return STATUS_UPDATER.compareAndSet(order, 0, 1);
    }
}
```
对比AtomciReferenceFieldUpdater
```
// AtomicReferenceFieldUpdater —— 引用类型的字段更新器
private static final AtomicReferenceFieldUpdater<Config, String> NAME_UPDATER =
    AtomicReferenceFieldUpdater.newUpdater(Config.class, String.class, "name");

NAME_UPDATER.compareAndSet(config, "old", "new");
```
什么时候用FieldUpdater?
```
// ✅ 适用：
// ① 大量对象共享一个原子字段（节省内存）
//    AtomicInteger 每个对象都要一个对象头
//    FieldUpdater 是静态的，共享一份

// 举例：100 万个对象每个都要一个计数器
// AtomicInteger：每个对象多 16 字节 → 100 万 * 16 = 16MB
// FieldUpdater：只占一个 int 字段 → 100 万 * 4 = 4MB

// ❌ 不适用：
// 一般场景直接用 AtomicInteger 更简单
```
### 工具方法
```
// ① updateAndGet —— 函数式更新（可重试）
AtomicInteger count = new AtomicInteger(5);
count.updateAndGet(x -> x * 3);  // 15

// ② accumulateAndGet —— 累加器
count.accumulateAndGet(10, (x, y) -> x + y);  // 25

// ③ getAndUpdate / getAndAccumulate —— 返回旧值版本
int old = count.getAndUpdate(x -> x - 5);  // 返回 25，变为 20
```
更新失败自动重试
```
// updateAndGet 内部也是 CAS 自旋
public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);  // 每次用最新值计算
    } while (!compareAndSet(prev, next));
    return next;
}
```
### 面试高频题
#### 题目1：AtomicInteger和Synchronized哪个性能好？
```
// 低竞争（冲突少）：AtomicInteger 快（无锁，CAS 一次成功）
// 高竞争（冲突多）：synchronized 可能更好（自旋浪费 CPU）

// AtomicInteger 吞吐量高，但延迟可能不稳定（自旋）
// synchronized 低竞争时反而慢（锁开销），高竞争时稳定

// 结论：低竞争用 AtomicInteger，高竞争用 synchronized 或 LongAdder
```
#### 题目2：AtomicInteger和volatile int的区别？
```
// volatile int：
// ① 保证可见性、有序性
// ② 不保证原子性（i++ 不安全）

// AtomicInteger：
// ① 保证可见性、有序性
// ② 保证原子性（CAS）

// 场景：单个"读写"操作 → volatile 够用
// 场景："读-改-写"复合操作 → AtomicInteger
```
#### 题目3：AtomicReference怎么用
```
// 原子更新对象引用，常用于：
// ① 无锁的配置热更新
// ② 无锁的链表栈（CAS 更新头节点）
// ③ 无锁的共享对象交换
```
题目4：AtomicStampedReference解决什么问题？
```
// ABA 问题：
// A 线程读到值为 X
// B 线程 X → Y → X（改回）
// A 线程 CAS 发现还是 X → 成功（但中间被改过）

// AtomicStampedReference 用版本号：
// CAS 同时比较值和版本号，版本号每次 +1
// A 线程版本号是 0，B 修改后版本号变 2，A 的 CAS 失败
```

