## 高并发计数器
AtomicLong的痛点->LongAdder的原理->热点分离->核心源码->LongAdder vs AtomicLong->面试高频题
### AtomicLong的痛点
高并发下的问题
```
// AtomicLong 用的是 CAS 自旋
// 问题：高并发时，多个线程同时 CAS 同一个变量 → 大量失败 → 疯狂自旋

// 场景：100 个线程同时 incrementAndGet()
// 每个线程都 CAS 同一个值
// 只有一个成功，99 个失败重试 → CPU 空转！

// 问题本质：所有线程竞争"同一个热点"（单个 value 变量）
AtomicLong count = new AtomicLong(0);

// 高并发下：
// 线程 1：CAS(0→1) 成功
// 线程 2：CAS(0→1) 失败 → 重试 CAS(1→2)
// 线程 3：CAS(0→1) 失败 → 重试 CAS(1→2) 失败 → 重试 CAS(2→3)
// ... 竞争越激烈，重试越多，性能越差

// 大量线程自旋 → CPU 打满 → 性能反而下降
```
### LongAdder原理
核心思想：热点分散
```
AtomicLong：
┌──────────────┐
│   value      │  ← 所有线程都竞争这一个热点
│  (一个热点)   │
└──────────────┘
    ↑ ↑ ↑ ↑
    线程们疯狂 CAS

LongAdder：
┌──────────────┐
│  base        │  ← 低竞争时用
└──────────────┘
┌───────┬───────┬───────┬───────┐
│ Cell0 │ Cell1 │ Cell2 │ Cell3 │  ← 高竞争时分散到多个 Cell
└───────┴───────┴───────┴───────┘
  ↑       ↑       ↑       ↑
线程1  线程2   线程3   线程4
(各自更新自己的 Cell，互不竞争)
```
工作流程
```
// ① 低竞争时：直接 CAS 更新 base（类似 AtomicLong）
// ② 竞争激烈时：创建 Cell 数组，线程分散到不同的 Cell 更新
// ③ 获取总和：base + 所有 Cell 的值

// 每个线程映射到哪个 Cell？
// 通过 ThreadLocalRandom.getProbe() 的哈希值
// 线程的 probe 值 % cells.length → 下标
// 不同线程哈希不同 → 分散到不同 Cell → 减少竞争
```
### 核心源码
LongAdder的继承结构
```
// LongAdder 继承 Striped64
public class LongAdder extends Striped64 implements Serializable {

    // Striped64 中的核心字段：
    // transient volatile long base;       // 基础值（低竞争时用）
    // transient volatile Cell[] cells;    // Cell 数组（高竞争时用）
}

// Cell 内部类 —— 每个 Cell 是一个独立的计数单元
// @sun.misc.Contended 注解：防止伪共享   让变量占一个缓存行，CPU是以缓存行为单位
@sun.misc.Contended
static final class Cell {
    volatile long value;

    Cell(long x) {
        value = x;
    }

    final boolean cas(long cmp, long val) {
        return U.compareAndSwapLong(this, VALUE, cmp, val);
    }
}
```
add方法
```
public void add(long x) {
    Cell[] as;
    long b, v;
    int m;
    Cell a;

    // ① cells 不为空（已有竞争），或者 base CAS 失败（有竞争）
    if ((as = cells) != null || !casBase(b = base, b + x)) {

        boolean uncontended = true;

        // ② cells 为空 → 初始化
        if (as == null || (m = as.length - 1) < 0 ||
            // ③ 当前线程对应的 Cell 为空 → 扩容
            (a = as[getProbe() & m]) == null ||
            // ④ CAS 当前 Cell 失败（竞争）→ 扩容
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```
longAccumulate方法
```
final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    int h;
    // 初始化 probe
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current();
        h = getProbe();
        wasUncontended = true;
    }

    boolean collide = false;
    for (;;) {
        Cell[] as; Cell a; int n; long v;

        // 情况一：cells 不为空
        if ((as = cells) != null && (n = as.length) > 0) {
            // 当前线程的 Cell 为空 → 创建
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {
                    Cell r = new Cell(x);
                    // CAS 获取扩容锁
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;  // 释放扩容锁
                        }
                        if (created) break;
                        continue;
                    }
                }
                collide = false;
            }
            // CAS 当前 Cell 成功 → 完成
            else if (!wasUncontended)
                wasUncontended = true;
            // 正常 CAS 更新
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                         fn.applyAsLong(v, x))))
                break;
            // 需要扩容
            else if (n >= NCPU || cells != as)
                collide = false;
            else if (!collide)
                collide = true;
            // 真正扩容：翻倍
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    if (cells == as) {
                        Cell[] rs = new Cell[n << 1];  // 2 倍扩容
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;
            }
            // 换个 Cell 重试
            h = advanceProbe(h);
        }

        // 情况二：cells 为空 → 初始化
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {
                if (cells == as) {
                    Cell[] rs = new Cell[2];  // 初始 2 个 Cell
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init) break;
        }

        // 情况三：都失败 → 回退到 base
        else if (casBase(v = base,
                         ((fn == null) ? v + x : fn.applyAsLong(v, x))))
            break;
    }
}
```
sum方法
```
// 获取总和：base + 所有 Cell
public long sum() {
    Cell[] as = cells;
    Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}

// 注意：sum() 不是精确的！
// 求和过程中可能有线程在更新 Cell
// 只能得到"近似值"
// 适合统计类场景，不适合精确计数
```
### 伪共享问题
什么是伪共享
```
// CPU 缓存行（Cache Line）通常是 64 字节
// 相邻的内存地址共享同一个缓存行

// 问题：两个 Cell 在同一个缓存行里
// 线程 1 修改 Cell0 → 整个缓存行失效
// 线程 2 的 Cell1 也被迫失效 → 重新从内存加载
// 虽然两个线程不竞争同一个变量，但竞争同一个缓存行！
```
解决方案：@Contended注解
```
// Striped64 的 Cell 类：
@sun.misc.Contended  // 告诉 JVM：这个类的实例要填充缓存行
static final class Cell {
    volatile long value;
}

// 效果：每个 Cell 独占 64 字节缓存行
// 修改 Cell0 不会影响 Cell1 的缓存行

// 使用：
// 需要 JVM 参数：-XX:-RestrictContended（JDK 8）
// JDK 9+ 默认开启
```
### 面试高频

> **Q:** "什么是伪共享？怎么解决？" 
> **A:** "CPU 缓存行是 64 字节，相邻变量共享缓存行。线程 A 修改变量 X，导致同缓存行的变量 Y 缓存失效，线程 B 访问 Y 要重新加载，性能下降。解决：@Contended 注解或手动填充 padding（LongAdder 的 Cell 用了 @Contended）。"
### LongAdder vs AtomicLong
| 对比           | AtomicLong              | LongAdder          |
| ------------ | ----------------------- | ------------------ |
| **原理**       | 单个变量 CAS 自旋             | base + Cell 数组热点分离 |
| **低竞争性能**    | 快                       | 略慢（多判断）            |
| **高竞争性能**    | 差（疯狂自旋）                 | 好（分散竞争）            |
| **sum() 精度** | 精确                      | 近似（可能不准确）          |
| **适用场景**     | 精确计数（ID 生成）             | 统计类（次数、流量）         |
| **排序相关**     | ✅ 有 incrementAndGet 返回值 | ❌ 没有返回值            |
使用场景选择
```
// ✅ LongAdder：统计场景
// 请求次数、流量统计、热点数据统计
LongAdder requestCount = new LongAdder();
requestCount.increment();
long total = requestCount.sum();  // 近似即可

// ✅ AtomicLong：精确计数场景
// 唯一 ID 生成、需要拿到自增后的值
AtomicLong idGenerator = new AtomicLong(0);
long nextId = idGenerator.incrementAndGet();  // 需要返回值
```
性能对比数据
```
// 16 线程并发递增 1000 万次
// AtomicLong：约 3000ms（大量自旋）
// LongAdder：约 300ms（热点分离，快 10 倍）

// 低竞争（2 线程）：
// AtomicLong：略快（LongAdder 多一层判断）
```
### LongAccumulator
```
// LongAccumulator：LongAdder 的通用版
// 不限于"加法"，可以自定义操作

// 构造器：累加函数 + 初始值
LongAccumulator accumulator =
    new LongAccumulator(Long::max, Long.MIN_VALUE);  // 求最大值

// 累加
accumulator.accumulate(10);
accumulator.accumulate(50);
accumulator.accumulate(30);

accumulator.get();  // 50 —— 最大值

// 其他用法
LongAccumulator sumAcc = new LongAccumulator(Long::sum, 0);      // 求和
LongAccumulator minAcc = new LongAccumulator(Long::min, Long.MAX_VALUE);  // 求最小
LongAccumulator mulAcc = new LongAccumulator((a, b) -> a * b, 1);  // 乘积
```
### 面试高频题目
#### 题目1：LongAdder为什么比AtomicLong快？
```
// 热点分离：
// AtomicLong 所有线程竞争同一个 value
// LongAdder 把计数分散到 base + N 个 Cell
// 每个线程更新自己的 Cell，互不竞争
// 高竞争下性能提升明显（快 10 倍）
```
题目2：LongAdder的sum()准确吗？
```
// 不准确（近似值）
// sum() 求和过程中可能有线程在更新 Cell
// 适用于：统计类（次数、流量），允许少量误差
// 不适用：需要精确结果的场景
```
题目3：什么时候用LongAdder，什么时候用AtomicLong
```
// LongAdder：统计场景（请求数、流量、次数）
// AtomicLong：需要精确值和返回值的场景（ID 生成器）
```
题目4：LongAdder是怎么初始化的
```
// 懒初始化：
// 低竞争：只用 base，不创建 Cell
// 竞争发生时：创建 2 个 Cell
// 竞争加剧：扩容到 4、8、16...（不超过 CPU 核数）
```


