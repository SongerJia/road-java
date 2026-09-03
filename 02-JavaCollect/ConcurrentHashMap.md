## ConcurrentHashMap
CHM设计目标->1.7分段加锁->1.8CAS+Synchronized->put流程->get流程->size统计->并发扩容->面试高频题
### CHM设计目标
为什么需要ConcurrentHashMap?
```java
// ❌ Hashtable —— 全表锁，并发极差
public synchronized V put(K key, V value) { ... }
public synchronized V get(Object key) { ... }
// 所有方法都加 synchronized，读和写互斥，并发量低

// ❌ HashMap —— 线程不安全，多线程下可能死循环/数据丢失
Map<String, String> map = new HashMap<>();

// ✅ ConcurrentHashMap —— 线程安全 + 高并发
Map<String, String> map = new ConcurrentHashMap<>();
//设计目标
1.线程安全，多线程读写正常，不丢数据
2.读操作无锁，写操作只锁部分区域
3.一致性
```
### 1.7分段锁（Segment锁）
数据结构
```java
// JDK 1.7 的 ConcurrentHashMap
public class ConcurrentHashMap<K, V> {
    // 段数组（默认 16 个 Segment）
    final Segment<K,V>[] segments;

    // 每个 Segment 继承 ReentrantLock，相当于一个小 HashMap
    static final class Segment<K,V> extends ReentrantLock {
        transient volatile HashEntry<K,V>[] table;  // 每个 Segment 内部数组  代表里面又是一小段Map数组
        transient int count;        // 元素个数
        transient int modCount;     // 修改次数
        transient int threshold;    // 扩容阈值
        final float loadFactor;     // 负载因子
    }
}

// 结构图：
ConcurrentHashMap
  ┌───────────────┐
  │  Segment[0]   │ → HashEntry[] → HashEntry → HashEntry（链表）
  ├───────────────┤
  │  Segment[1]   │ → HashEntry[] → HashEntry（链表）
  ├───────────────┤
  │  Segment[2]   │ → HashEntry[] →（空）
  ├───────────────┤
  │  ...          │
  ├───────────────┤
  │  Segment[15]  │ → HashEntry[] → HashEntry → HashEntry（链表）
  └───────────────┘
```
分段锁的原理
```java
// 1.7 的 put 流程
public V put(K key, V value) {
    int hash = hash(key);
    // 通过 hash 定位到某个 Segment
    Segment<K,V> segment = segmentForHash(hash);

    // 只锁这一个 Segment，其他 Segment 不受影响
    return segment.put(key, hash, value, false);
}

// Segment 的 put —— 继承 ReentrantLock，自己加锁
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 尝试获取锁（非阻塞 tryLock）
    if (!tryLock()) {
        // 获取不到则自旋等待
        scanAndLockForPut(key, hash, value);
    }

    try {
        // 往 Segment 内部的 HashEntry[] 中 put
        // 和 HashMap 1.7 的逻辑一样（数组 + 链表，头插法）
        // ...
    } finally {
        unlock();  // 释放锁
    }
}
```
1.7的并发度
```
// 默认并发度 = 16（Segment 数组长度）
// 意味着 16 个线程可以同时写不同的 Segment

// 并发度可以通过构造器指定
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>(16, 0.75f, 32);
// 初始容量 16，负载因子 0.75，并发度 32（32 个 Segment）
```
1.7的缺点
```
// ① 锁粒度还是不够细
// 一个 Segment 保护的是一整段数组，内部还是链表的 O(n) 查找

// ② 扩容时，是 Segment 内部扩容，不是整个 CHM 扩容
// 每个 Segment 独立扩容，可能导致数据分布不均

// ③ 查询要先定位 Segment，再定位 HashEntry，两次 hash
```
### 1.8CAS+Synchronized
1.8的变化
```java
// JDK 1.8 完全重写了 ConcurrentHashMap
// 去掉了 Segment，直接用 Node 数组 + CAS + synchronized

// 数据结构 = 和 HashMap 1.8 一样：数组 + 链表 + 红黑树
public class ConcurrentHashMap<K, V> {
    transient volatile Node<K,V>[] table;  // volatile 保证可见性

    // 和 HashMap 一样的 Node
    static class Node<K,V> {
        final int hash;
        final K key;
        volatile V val;       // volatile 保证可见性
        volatile Node<K,V> next;  // volatile 保证可见性
    }

    // 红黑树节点
    static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;
        boolean red;
    }
}
```
1.8的锁机制预览

| 场景      | 锁机制              |
| ------- | ---------------- |
| 数组位置为空  | CAS无锁            |
| 数组位置不为空 | synchronized锁头节点 |
| 读操作     | 无锁，volatile保证可见性 |
### put流程
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    // CHM 不允许 null key 和 null value
    if (key == null || value == null) throw new NullPointerException();

    int hash = spread(key.hashCode());  // 扰动函数

    Node<K,V>[] tab; Node<K,V> f; int n, i, fh;

    // ① 数组为空 → 初始化（懒加载，CAS 保证线程安全）
    while ((tab = table) == null || (tab.length = 0) == 0)
        tab = initTable();

    // ② 该位置为空 → CAS 无锁插入
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
        // CAS 原子操作：如果该位置为 null，则插入新节点
        if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
            break;  // 插入成功，跳出循环
        // 如果 CAS 失败，说明其他线程已经插入了，重试
    }

    // ③ 正在扩容 → 帮忙扩容
    else if ((fh = f.hash) == MOVED)  // MOVED = -1
        tab = helpTransfer(tab, f);

    // ④ 该位置不为空 → synchronized 锁头节点
    else {
        V oldVal = null;
        // synchronized 锁住桶的头节点（细粒度锁）
        synchronized (f) {
            // 再次检查头节点是否被其他线程修改了
            if (tabAt(tab, i) == f) {

                // ④-1 链表（hash >= 0）
                if (fh >= 0) {
                    int binCount = 0;
                    for (Node<K,V> e = f;; ++binCount) {
                        K ek;
                        // 找到相同 key → 覆盖
                        if (e.hash == hash &&
                            ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                            oldVal = e.val;
                            if (!onlyIfAbsent)
                                e.val = value;
                            break;
                        }
                        // 到达尾部 → 尾插法插入
                        Node<K,V> pred = e;
                        if ((e = e.next) == null) {
                            pred.next = new Node<K,V>(hash, key, value, null);
                            break;
                        }
                    }
                    // 链表长度 ≥ 8 → 树化
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                }

                // ④-2 红黑树
                else if (f instanceof TreeBin) {
                    // 树插入
                    // ...
                }
            }
        }
        // 返回旧值
        if (oldVal != null)
            return oldVal;
    }

    addCount(1L, binCount);  // 计数 + 检查是否需要扩容
    return null;
}
```
put流程图
```
put(key, value)
    │
    ├── key/value 为 null？ → 抛 NPE
    │
    ├── table 为空？ → initTable() CAS 初始化
    │
    ├── 该位置为空？ → CAS 插入 ✅ 无锁
    │
    ├── 正在扩容？ → helpTransfer() 帮忙扩容
    │
    ├── 该位置不为空？ → synchronized(f) 锁头节点
    │   ├── 链表 → 遍历，覆盖或尾插
    │   ├── 红黑树 → 树插入
    │   └── 长度 ≥ 8 → treeifyBin()
    │
    └── addCount() 计数，检查扩容
```
1.8锁粒度优化
```
// 1.7：锁 Segment（一个 Segment 包含多个桶）
// 1.8：锁桶的头节点（只锁一个桶）

// 假设数组长度 16，16 个线程写不同位置 → 可并行写
// 比 1.7 的 16 个 Segment 更细粒度
```
### get流程
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;

    int h = spread(key.hashCode());

    // ① 数组不为空，且该位置有元素
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {

        // ② 头节点匹配 → 直接返回
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }

        // ③ 正在扩容或红黑树 → 特殊查找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;

        // ④ 链表 → 遍历查找
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
get流程图
```
getValue(key)
│
├──数组为空或者位置没有元素 返回null
│
├──头节点匹配，直接返回value
│
├──正在扩容或者是树化结构，特殊查找
│
└──链表遍历查找返回
```
get为什么无锁
```
// 关键：Node 的 val 和 next 都是 volatile 的
static class Node<K,V> {
    volatile V val;       // volatile：保证写操作对其他线程立即可见
    volatile Node<K,V> next;  // volatile：保证链表结构变化可见
}

// volatile 保证：
// ① 写 volatile 变量时，会把修改强制刷新到主内存
// ② 读 volatile 变量时，会从主内存重新读取
// ③ 禁止指令重排序

// 所以 get 不需要加锁，直接 volatile 读就能拿到最新值
```
弱一致性
```
// ConcurrentHashMap 的 get 是弱一致性的
// 不是"实时一致性"——可能读到刚写入但还没刷新的数据

// 场景：
// 线程 A：map.put("key", "value")
// 线程 B：map.get("key")  → 可能返回 null（还没读到）

// 原因：put 操作先写 Node，再赋值到数组
// 如果 get 在赋值之前读取，就看不到

// 但这在大多数业务场景下是可以接受的
// 如果要强一致性，用 synchronizedMap 或锁
```
### Size统计
1.7的size
```java
// 1.7：累加所有 Segment 的 count
// 先不加锁累加一次，如果两次累加结果一致则返回
// 不一致则加锁再累加

public int size() {
    long sum = 0;
    int modCount = 0;
    // 尝试两次不加锁统计
    for (int tries = 0; tries < 2; tries++) {
        sum = 0;
        for (Segment<K,V> seg : segments) {
            sum += seg.count;
        }
        if (modCount == 0) {
            // 两次统计结果一致 → 返回
            break;
        }
    }
    // 不一致 → 加锁统计
    for (Segment<K,V> seg : segments) {
        seg.lock();
        sum += seg.count;
    }
    for (Segment<K,V> seg : segments) {
        seg.unlock();
    }
    return (int) sum;
}
```
1.8的size
```java
// 1.8 引入了 CounterCell 机制，用 CAS 无锁计数

// 计数时，先尝试 CAS 更新 baseCount
// 如果 CAS 竞争激烈，就分散到 CounterCell 数组中
// 每个线程绑定一个 CounterCell，减少竞争

// 关键字段
private transient volatile long baseCount;          // 基础计数
private transient volatile CounterCell[] counterCells;  // 辅助计数数组

// CounterCell —— 内部类
static final class CounterCell {
    volatile long value;  // 每个 Cell 的计数
}

// addCount 方法
private final void addCount(long x, int check) {
    CounterCell[] cs; long b, s;

    // ① 先尝试 CAS 更新 baseCount
    if ((cs = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {

        // ② CAS 失败 → 用 CounterCell 分散计数
        CounterCell c; long v;
        boolean uncontended = true;
		//getProbe会得到一个随机值，得到数组下标后，如果这个下标位置有值，就在fullAddCount方法里再重新找一个位置
        if (cs == null ||  // 数组为空
            (c = cs[ThreadLocalRandom.getProbe() & (cs.length - 1)]) == null ||  // 位置为空
            !(uncontended = U.compareAndSwapLong(c, CELLVALUE, v = c.value, v + x))) {
            // ③ 竞争激烈 → fullAddCount 方法处理
            fullAddCount(x, uncontended);
            return;
        }
        // ...
    }
}

// size 方法 —— 累加 baseCount + 所有 CounterCell
public int size() {
    long sum = baseCount;
    CounterCell[] cs = counterCells;
    if (cs != null) {
        for (CounterCell c : cs) {
            if (c != null)
                sum += c.value;
        }
    }
    return (int) sum;
}
```
**Q:** "为什么 1.8 的 size 不用加锁了？" 
**A:** "因为用了 CAS 无锁计数。baseCount 是 CAS 更新的，如果竞争激烈就分散到 CounterCell 数组，每个线程 CAS 自己的 Cell，最后汇总就行。整个过程不需要加锁。"
### 并发机制
1.8的扩容机制
```
// 1.8 的扩容是并发扩容 —— 多线程可以一起帮忙扩容

// 扩容时，原数组会被分成多个任务
// 每个线程认领一段范围，负责迁移自己的那一段

// 关键：ForwardingNode
// 当某个桶迁移完成后，在原来位置放一个 ForwardingNode（hash = -1 即 MOVED）
// 其他线程看到 MOVED，就知道这个桶已经迁移了

// helpTransfer —— 帮忙扩容
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    // 检查是否正在扩容
    // 如果是，参与扩容
    // 每个线程负责迁移一部分桶
}
```
扩容时的读写
```
// 扩容时，get 操作不受影响
// 看到 ForwardingNode → 去新数组查找
// 看到普通 Node → 直接读取

// 扩容时，put 操作：
// 看到 ForwardingNode → 先帮忙扩容，再 put
// 看到普通 Node → synchronized 锁头节点，然后 put
```
扩容的触发条件
```java
//扩容的触发场景
// ① put 后调用 addCount()，检查是否需要扩容
// ② 链表树化时，数组长度 < 64，触发扩容
// ③ 其他线程正在扩容时，当前线程可以帮忙

private final void addCount(long x, int check) {
    // ... CAS 计数 ...

    // 检查是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);  // 扩容戳

            // 正在扩容（sc < 0）
            if (sc < 0) {
                // 判断是否需要帮忙扩容
                // ① sc >>> RESIZE_STAMP_SHIFT != rs → 扩容戳不匹配
                // ② sc == rs + 1 → 最后一个线程（BUG，实际是 rs + 2）
                // ③ sc == rs + MAX_RESIZERS → 达到最大帮忙线程数
                // ④ (nt = nextTable) == null → 新数组还没创建
                // ⑤ transferIndex <= 0 → 所有桶已分配完毕
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;

                // 帮忙扩容：CAS 增加 sc（+1 表示多一个线程帮忙）
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 没有在扩容 → 当前线程发起扩容
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```
resizeStamp：扩容戳
```java
// 生成扩容戳：高 16 位是扩容标识 （-1），低 16 位是并行扩容线程数
static final int resizeStamp(int n) {
    // 返回 n 的前导零个数 | 1 << 15
    // 例如 n=16（10000），前导零个数=27  可以反推出当前扩容基于哪个容量
    // 27 | 32768 = 32795
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}

// sizeCtl 的含义：
// -1：正在初始化
// < -1：正在扩容，低 16 位表示参与扩容的线程数 +2（从2开始，保证负数语义不冲突）  高16位是=容量的前导0个数
// 0：默认值，还没初始化
// > 0：下一次扩容的阈值
```
transfer
```java
// 这才是真正的"数据迁移"核心方法
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;

    // ① 计算每个线程负责的步长（最少 16 个桶）
    // 单核 CPU：stride = n（一个线程做所有）
    // 多核 CPU：stride = (n >>> 3) / NCPU，最少 16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;  // 16

    // ② 初始化新数组（2 倍容量）
    if (nextTab == null) {
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;  // 从最后一个桶开始分配
    }

    int nextn = nextTab.length;

    // ③ ForwardingNode —— 标记"这个桶已经迁移完了"
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;    // 是否继续处理下一个桶
    boolean finishing = false; // 是否全部完成

    // ④ 从后往前遍历，每个线程处理 stride 个桶
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;

        // ⑤ 获取当前线程要处理的任务范围
        while (advance) {
            int nextIndex, nextBound;
            // 当前批次还没处理完 → 继续
            if (--i >= bound || finishing)
                advance = false;
            // 所有桶已分配完 → 结束
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 从 transferIndex 分配一段任务
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }

        // ⑥ 处理完成 → 收尾
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                // 全部完成：替换 table，重置 nextTable 和 sizeCtl
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);  // 新阈值 = 新容量 * 0.75
                return;
            }
            // 当前线程的任务完成，CAS 减少扩容线程数
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 检查是不是最后一个扩容线程
                // (sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT
                // 如果不是最后一个，直接返回
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // 是最后一个线程 → 再检查一遍
                finishing = advance = true;
                i = n;  // recheck
            }
        }

        // ⑦ 处理当前桶 i
        // ⑦-1 桶为空 → 放 ForwardingNode
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);

        // ⑦-2 已经处理过了 → 跳过
        else if ((fh = f.hash) == MOVED)
            advance = true;

        // ⑦-3 真正的迁移
        else {
            synchronized (f) {  // 锁住头节点
                // 再次检查防止被其他线程修改
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;

                    // 链表迁移
                    if (fh >= 0) {
                        // 和 HashMap 1.8 一样的优化
                        // 根据 (e.hash & oldCap) == 0 分为两组
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;

                        // 先找到最后一段不变的节点
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // lastRun 之后的节点 hash&n 都相同
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        } else {
                            hn = lastRun;
                            ln = null;
                        }

                        // 遍历 lastRun 之前的节点，头插法重组
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }

                        // 低位链表放到原位置
                        setTabAt(nextTab, i, ln);
                        // 高位链表放到原位置 + oldCap
                        setTabAt(nextTab, i + n, hn);
                        // 原位置放 ForwardingNode
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }

                    // 红黑树迁移
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;

                        // 遍历红黑树节点
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            // 低位链表
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            // 高位链表
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }

                        // 如果链表长度 ≤ 6，红黑树退化为链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;

                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```
扩容的读写策略
```java
// 扩容时，get 操作如何工作？
// 看到 ForwardingNode → 去新数组 nextTable 查找
// 看到普通 Node → 直接读取（原数组还没被回收）

// ForwardingNode 的 find 方法
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;

    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);  // hash = -1
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // 到新数组 nextTable 中查找
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```
扩容流程图
```
transfer()
    │
    ├── 计算步长 stride（每个线程处理 >= 16 个桶）
    │
    ├── 创建新数组（2 倍容量）
    │
    ├── 从 transferIndex 分配任务段
    │   └── CAS 更新 transferIndex（保证不冲突）
    │
    ├── 遍历分配的桶（从后往前）
    │   ├── 桶为空 → 放 ForwardingNode
    │   ├── 已迁移 → 跳过
    │   └── 有数据 → synchronized 锁头节点
    │       ├── 链表：按 hash&oldCap 分两组
    │       ├── 红黑树：拆成两个链表，≤6 则退化
    │       └── 放 ForwardingNode 标记完成
    │
    ├── 当前线程任务完成 → CAS 减少线程数
    │
    └── 最后一个线程→ 替换 table，重置 sizeCtl
```
多线程扩容示意图
```
线程 A 负责：桶 15 ~ 8         线程 B 负责：桶 7 ~ 0
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 15 │ 14 │ 13 │ 12 │ 11 │ 10 │ 9  │ 8  │
└────┴────┴────┴────┴────┴────┴────┴────┘
  FWD  FWD  FWD  FWD  FWD  FWD  FWD  FWD
  ↑                                    ↑
线程 A 完成                        线程 A 完成

┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 7  │ 6  │ 5  │ 4  │ 3  │ 2  │ 1  │ 0  │
└────┴────┴────┴────┴────┴────┴────┴────┘
  FWD  FWD  FWD  FWD  FWD  FWD  FWD  FWD
  ↑                                    ↑
线程 B 完成                        线程 B 完成

FWD = ForwardingNode，表示该桶已迁移到新数组
```
> **Q:** "扩容时一个线程负责多少桶？" 
> **A:** "多核 CPU 下，每个线程最少负责 16 个桶，具体公式是 `(n >>> 3) / NCPU`，确保单核也能高效扩容。"

> **Q:** "扩容时其他线程能 put 吗？" 
> **A:** "能。如果桶还没迁移，直接 synchronized 锁住桶后 put；如果桶已经迁移了（看到 ForwardingNode），先帮忙扩容，再去新数组 put。"

> **Q:** "扩容时 sizeCtl 用来做什么？" 
> **A:** "sizeCtl 是一个多用途的状态标记：<br>- -1：正在初始化<br>- < -1：正在扩容，低 16 位记录参与扩容的线程数<br>- > 0：下一次扩容的阈值"

> **Q:** "扩容过程中数据会丢失吗？" 
> **A:** "不会。扩容是线程安全的——每个桶的迁移都在 synchronized 块中完成，迁移完成后放 ForwardingNode 标记，其他线程看到标记就知道该去新数组找。"
### 面试高频题目
#### 题目1：CHM为什么不允许null key和null value
```
// HashMap 允许 null key（hash=0，存在 table[0]）
// ConcurrentHashMap 不允许

// 官方解释：如果允许 null，get 返回 null 时
// 无法区分是"key 不存在"还是"key 对应的 value 就是 null"
// 在并发环境下，这个歧义无法通过 containsKey 解决
// （因为 containsKey 和 get 之间可能被其他线程修改）

// 而 HashMap 是单线程的，不存在这个问题
```
#### 题目2：1.7和1.8对比
|对比|1.7|1.8|
|---|---|---|
|**底层结构**|Segment + HashEntry|Node + 链表 + 红黑树|
|**锁机制**|分段锁（ReentrantLock）|CAS + synchronized|
|**锁粒度**|一个 Segment 一个锁|一个桶一个锁（更细）|
|**并发度**|固定（Segment 数量）|动态（数组长度）|
|**读操作**|无锁（volatile）|无锁（volatile）|
|**size 统计**|先无锁试两次，失败加锁|CounterCell CAS 无锁|
|**扩容**|Segment 内独立扩容|多线程并发扩容|
|**红黑树**|❌ 无|✅ 有|
#### 题目3：为什么1.8使用synchronized不使用reentrantLock
```
// ① synchronized 在 Java 6 后做了大量优化
//    偏向锁 → 轻量级锁 → 重量级锁，性能已经接近 ReentrantLock

// ② synchronized 不用手动释放锁
//    代码更简洁，不容易出错

// ③ JVM 可以自由优化 synchronized
//    比如锁消除、锁粗化等

// 结论：synchronized 已经足够好，而且代码更简洁
```
#### 题目4：弱一致性对业务有什么影响
```
// 场景：先 put 再 get，可能 get 不到刚 put 的值
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

// 线程 A
map.put("key", "value");

// 线程 B（在 A 之后立即执行）
String v = map.get("key");  // 可能为 null！

// 解决方案：
// ① 如果业务要求强一致性，加锁
// ② 大部分场景下，弱一致性是可接受的（缓存、配置等）
// ③ 如果不行，用 synchronizedMap 或自定义锁
```