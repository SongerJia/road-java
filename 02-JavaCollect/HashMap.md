## HashMap
底层数据结构->哈希算法和扰动函数->put流程->get流程->扩容机制->树化与红黑树->1.7vs1.8->线程安全->面试高频题
### 底层数据结构
JDK1.7：数组+链表
```java
// 1.7 的结构
Entry[] table;  // 数组
Entry {         // 链表节点
    int hash;
    K key;
    V value;
    Entry next;  // 单向链表
}
```
JDK1.8：数组+链表+红黑树
```java
// 1.8 的结构
Node[] table;    // 数组
Node {           // 链表节点 static   implements Map.Entry<K,V>
    int hash;
    K key;
    V value;
    Node next;
}
TreeNode {       // 红黑树节点（链表长度 > 8 时转换） static  final   extends LinkedHashMap.Entry<K,V>
    int hash;
    K key;
    V value;
    TreeNode parent;
    TreeNode left;
    TreeNode right;
    TreeNode prev;  // 用于链表化时恢复链表
    boolean red;
}
```
结构图
```
table 数组
  ┌────┬────┬────┬────┬────┬────┬────┐
  │ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │
  └────┴────┴────┴────┴────┴────┴────┘
    ↓
  Node A → Node B → Node C  ← 链表（长度 ≤ 8）
  
  或
  
    ↓
  TreeNode(红黑树) ← 链表长度 > 8，数组长度 ≥ 64
```
默认参数
```java
// 默认初始容量
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;  // 16

// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 树化阈值
static final int TREEIFY_THRESHOLD = 8;

// 非树化阈值
static final int UNTREEIFY_THRESHOLD = 6;

// 最小树化容量（数组长度达到 64 才允许树化）
static final int MIN_TREEIFY_CAPACITY = 64;
```
### 哈希算法和扰动函数
计算key的hash值
```java
// 1.7 的扰动函数 4次位运算 + 5次异或（外层有一次）目的是分2轮将高位折叠进低位，比1.8更彻底但更费运算
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
// 1.8 的扰动函数 —— 让高位也参与运算，减少碰撞  hashcode  32位的int
static final int hash(Object key) {
    int h;
    //key的hash方法里面是key的hashCode的低16位和高16位做异或，得到的结果高16位不变，低16位是异或的结果
    //计算下标时 n-1通常很小，直接使用高16位根本参与不上，
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 计算数组下标  由于数组长度是2的n次幂 %和&的效果一样，&是按位与，直接操作二进制，更快
// 1.7：indexFor(hash, table.length) —— 用 & 运算代替 %  抽出一个方法里面就是 将&代替%
// 1.8：直接 (n - 1) & hash   没必要抽出方法，直接内联
int index = (table.length - 1) & hash;
```
为什么这个设计扰动函数
```
// 假设 hashCode 是：     1111 1111 1111 1111 0000 0000 0000 0101
// 如果不扰动，低位是：    0101
// 数组长度 16，下标 = 0101 & 1111 = 0101 = 5    //& 都是1才是1 其他为0   | 都是0才是0，其他为1  ~ 取反
// 高位完全没参与，不同对象的高位差异被浪费了

// 扰动后：
// hashCode >>> 16：     0000 0000 0000 0000 1111 1111 1111 1111
// hashCode ^ (h>>>16)： 1111 1111 1111 1111 1111 1111 1111 1010   //^  相同为0  不同为1
// 低位变成了：          1010
// 高位的影响被混入低位了！
```
> **Q:** "为什么用 `(n-1) & hash` 而不是 `hash % n`？" 
> **A:** "位运算比取模快得多。前提是 n 必须是 2 的幂次方，这样 `(n-1) & hash` 等价于 `hash % n`。这就是为什么 HashMap 容量总是 2 的幂次方。"

> **Q:** "为什么 key 为 null 时 hash 为 0？" 
> **A:** "HashMap 允许 key 为 null，null 的 hash 值固定为 0，所以存在 table[0] 的位置。"
### put流程
1.8的put流程
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;

    // ① 数组为空 → 初始化（懒加载）
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // ② 计算下标，该位置为空 → 直接插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;

        // ③ 头节点和要插入的 key 相同 → 覆盖
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;

        // ④ 是红黑树节点 → 走树插入
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

        // ⑤ 是链表 → 遍历链表
        else {
            for (int binCount = 0; ; ++binCount) {
                // ⑤-1 到达链表尾部 → 尾插法插入
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表长度 ≥ 8 → 树化
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // ⑤-2 找到相同 key → 覆盖
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }

        // ⑥ 覆盖旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            return oldValue;
        }
    }

    ++modCount;
    // ⑦ 超过阈值 → 扩容
    if (++size > threshold)
        resize();
    return null;
}
```
流程图：
```
put(key, value)
    │
    ├── table 为空？ → resize() 初始化
    │
    ├── 计算下标 i = (n-1) & hash
    │
    ├── table[i] == null？ → 直接插入
    │
    ├── table[i].hash == key.hash && equals？
    │   └── 是 → 覆盖
    │
    ├── table[i] 是 TreeNode？ → 红黑树插入
    │
    ├── 遍历链表
    │   ├── 找到相同 key → 覆盖
    │   └── 到达尾部 → 尾插法插入
    │       └── 链表长度 ≥ 8 → treeifyBin()
    │
    └── size > threshold？ → resize()
```
### get流程
```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;

    // ① 数组不为空，且该位置有元素
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {

        // ② 检查头节点是否匹配
        if (first.hash == hash &&
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;

        // ③ 有下一个节点
        if ((e = first.next) != null) {
            // ④ 红黑树 → 树查找 O(log n)
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // ⑤ 链表 → 遍历查找 O(n)
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    // ⑥ 没找到 → 返回 null
    return null;
}
```
流程图
```
get(key)
    │
    ├── 计算 hash
    │
    ├── table 为空？ → 返回 null
    │
    ├── 该位置为空？ → 返回 null
    │
    ├── 头节点匹配？ → 返回
    │
    ├── 是 TreeNode？ → 红黑树查找 O(log n)
    │
    ├── 遍历链表查找 O(n)
    │   ├── 找到 → 返回
    │   └── 没找到 → 返回 null
    │
    └── 返回 null
```
### resize
扩容机制
```
// 什么时候扩容？
// ① 首次 put 时（懒加载）  首次put才创建数组分配空间
// ② size > threshold = capacity * loadFactor

// 例如：容量 16，负载因子 0.75
// threshold = 16 * 0.75 = 12
// 当第 13 个元素 put 时触发扩容
```
扩容流程
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;

    // ① 计算新容量和新阈值
    if (oldCap > 0) {
        // 容量翻倍
        newCap = oldCap << 1;  // 2 倍
        newThr = oldThr << 1;  // 阈值也翻倍
    } else {
        // 首次初始化
        newCap = DEFAULT_INITIAL_CAPACITY;  // 16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  // 12
    }

    // ② 创建新数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;

    // ③ 迁移数据
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;  // 帮助 GC

                // 只有一个节点
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;

                // 红黑树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);

                // 链表
                else {
                    // 1.8 优化：不用重新计算 hash
                    // 根据 (e.hash & oldCap) == 0 分为两组
                    Node<K,V> loHead = null, loTail = null;  // 低位组（位置不变）
                    Node<K,V> hiHead = null, hiTail = null;  // 高位组（位置 + oldCap）
                    do {
                        // 关键优化：e.hash & oldCap
                        // 等于 0 → 留在原位
                        // 不等于 0 → 移到 原位置 + oldCap
                        if ((e.hash & oldCap) == 0) {
                            // 低位链表
                            if (loTail == null) loHead = e;
                            else loTail.next = e;
                            loTail = e;
                        } else {
                            // 高位链表
                            if (hiTail == null) hiHead = e;
                            else hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = e.next) != null);

                    // 放到新数组
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;           // 位置不变
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;  // 位置 + oldCap
                    }
                }
            }
        }
    }
    return newTab;
}
```
JDK8扩容优化
```
// 1.7：重新计算 hash，效率低
// 1.8：根据 (e.hash & oldCap) == 0 判断位置

// 举例：oldCap = 16（10000），newCap = 32（100000）
// 某个元素原来的 hash = 5（00101）
// oldCap = 16（10000）
// e.hash & oldCap = 00101 & 10000 = 0 → 留在原位（下标 5）

// 另一个元素 hash = 21（10101）
// e.hash & oldCap = 10101 & 10000 = 10000 ≠ 0 → 移到 5 + 16 = 21

// 原因：扩容后 n-1 多了一位，原来 hash 的这多出的一位是 0 还是 1 //原来的n在这多出一位的值是1，其他位为0，只需要看hash与n的结果
// 决定了新位置是原位置还是原位置 + oldCap
```
### 树化与红黑树
树化条件
```java
// 两个条件必须同时满足：
// ① 链表长度 ≥ 8（TREEIFY_THRESHOLD）
// ② 数组长度 ≥ 64（MIN_TREEIFY_CAPACITY）

// 如果链表长度 ≥ 8 但数组长度 < 64 → 先扩容，不树化
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();  // 扩容
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // 转红黑树
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null) hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```
为什么树化阈值是8？
```
// 官方注释给出了解释：基于泊松分布
// 在理想随机哈希码下，链表长度达到 8 的概率约为 0.00000006
// 所以正常情况下几乎不会触发树化

// 长度概率分布：
// 0: 0.60653066
// 1: 0.30326533
// 2: 0.07581633
// 3: 0.01263606
// 4: 0.00157952
// 5: 0.00015795
// 6: 0.00001316
// 7: 0.00000094
// 8: 0.00000006  ← 千万分之六

// 那为什么还要有树化？
// 为了防止恶意哈希攻击——如果 key 的 hashCode 被故意设计成相同的值
// 所有元素都放到同一个桶里，链表会变得很长，性能从 O(1) 退化到 O(n)
// 红黑树保证 O(log n) 的最坏情况
```
为什么退化阈值是6？
```
// 树退化回链表的阈值是 6，不是 8
// 留下 2 的缓冲：如果阈值也是 8，链表长度在 8 附近来回震荡
// 会导致频繁树化 ↔ 退化，影响性能
```
红黑树的5条性质
```
// ① 每个节点是红色或黑色     //颜色是标记用来约束结构
// ② 根节点是黑色            //约定，让所有外部路径的起点一致
// ③ 叶子节点（NIL）是黑色    //不是叶子节点，是空指针位置指向的虚拟黑节点
// ④ 红色节点的子节点必须是黑色（不能有连续红色节点）   //一条路径上节点太密，树退化为链表
// ⑤ 从任意节点到其每个叶子的路径上，黑色节点数量相同   //每个叶子，就是每个虚拟黑节点。如果从根节点出发符合，那么从其他节点也一样
 
// 这 5 条性质保证了红黑树的平衡  所以红黑树的高度 ≤ 2 × log₂(n+1)
// 最长路径不超过最短路径的 2 倍
// 时间复杂度 O(log n)
```
### JDK1.7和1.8对比

| 对比维度      | 1.7      | 1.8                    |
| --------- | -------- | ---------------------- |
| 底层结构      | 数组+链表    | 数组+链表+红黑树              |
| 链表插入      | 头插法      | 尾插法                    |
| 扩容时rehash | 重新计算hash | 通过hash & oldcap 是否为0判断 |
| 扰动函数      | 4次位运算    | 1次异或运算                 |
| 初始化时机     | 构造时初始化   | 首次put时懒加载              |
| 线程安全      | 不安全      | 不安全                    |
| 性能退化      | O(n)     | O(logn)                |
头插法和尾插法
```
// 1.7 头插法：新元素插在链表头部
// 好处：无需遍历链表，效率高
// 坏处：扩容时多线程下会形成死循环链表

// 1.8 尾插法：新元素插在链表尾部
// 好处：不会形成死循环链表
// 坏处：需要遍历链表
```
### 线程安全问题
1.7死循环问题
```
// 多线程同时扩容时，头插法 + 链表反转 → 环形链表 → 死循环

// 简化场景：
// 线程 A 和线程 B 同时在扩容
// 链表 A → B → null

// 线程 A 执行到：e = A, next = B，被挂起
// 线程 B 完成扩容，链表变成 B → A → null  //此时B.next->A A.next->null  引用变了
// 线程 A 恢复执行： //此时还是旧的引用
//   e = A, next = B; 新桶=A->null;  e = next 
//   next = e.next;   //重新获取B.next ->A T2线程修改的Entry对象的.next字段
//   然后又搬了一次A  A.next->B  导致了环
```
1.8数据丢失问题
```
// 1.8 尾插法不会死循环了，但多线程下仍然不安全

// 场景：两个线程同时 put
// 线程 A 和 B 同时判断 table[i] == null
// 两个线程都走到 new Node() 然后赋值
// 后赋值的会覆盖前一个 → 数据丢失
```
安全替代方案
```
// 方案一：Hashtable（不推荐，全部加锁，性能差）
Map<String, String> map = new Hashtable<>();

// 方案二：Collections.synchronizedMap（包装类，全表锁）
Map<String, String> map = Collections.synchronizedMap(new HashMap<>());

// 方案三：ConcurrentHashMap（推荐，分段锁/CAS，性能好）
Map<String, String> map = new ConcurrentHashMap<>();
```
### 面试高频题
#### 题目1：HashMap为什么是2的幂次方
```java
// 因为 (n-1) & hash 等价于 hash % n
// 只有 n 是 2 的幂次方时，位运算才等价于取模
// 位运算比取模快得多

// 如果用户传入的容量不是 2 的幂次方怎么办？
// HashMap 会把它转成大于等于它的最小 2 的幂次方
public HashMap(int initialCapacity) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    // 返回大于等于 cap 的最小 2 的幂次方
    this.threshold = tableSizeFor(initialCapacity);
}

static final int tableSizeFor(int cap) {
    int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
// 例如：传入 17 → 返回 32
```
#### 题目2：HashMap的负载因子为什么是0.75？
```
// 时间和空间的权衡
// 负载因子越大（如 1）：空间利用率高，但哈希碰撞概率大，查询慢
// 负载因子越小（如 0.5）：哈希碰撞概率小，查询快，但空间浪费多

// 0.75 是官方经过大量测试得出的经验值
// 在时间和空间之间达到了平衡
```
#### HashMap的key一般是什么类型的
```
// 最好用不可变对象作为 key
// 因为 hashCode 一旦变了，就找不到原来的 value 了

// ✅ 推荐：String、Integer 等不可变类型
Map<String, User> userMap = new HashMap<>();
Map<Integer, String> idMap = new HashMap<>();

// ❌ 不推荐：可变对象
Map<List<String>, String> badMap = new HashMap<>();
List<String> key = new ArrayList<>();
key.add("A");
badMap.put(key, "value");
key.add("B");  // hashCode 变了！
badMap.get(key);  // null —— 找不到了！
```
#### 两个不同的key的hashCode相同会怎么样
```
// 会哈希碰撞，放到同一个桶里
// 1.8 中，先以链表形式存储
// 链表长度 ≥ 8 且数组长度 ≥ 64 时，转为红黑树
// 查找时，先比较 hash，再比较 equals
```