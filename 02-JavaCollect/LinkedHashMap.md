## LinkedHashMap
底层结构->两种顺序模式->LRU缓存实现->核心方法源码->与HashMap对比->选型场景
### 底层结构
LinkedHashMap = 双向链表 + HashMap
```java
// LinkedHashMap 继承 HashMap，在 HashMap 基础上加了双向链表
public class LinkedHashMap<K, V> extends HashMap<K, V> implements Map<K, V> {

    // 双向链表的头尾节点
    transient LinkedHashMap.Entry<K, V> head;
    transient LinkedHashMap.Entry<K, V> tail;

    // 排序模式：true = 访问顺序，false = 插入顺序（默认）
    final boolean accessOrder;
}
```
节点结构
```java
// LinkedHashMap 的节点继承了 HashMap 的 Node
// 多了 before 和 after 两个指针维护双向链表

static class Entry<K, V> extends HashMap.Node<K, V> {
    Entry<K, V> before;  // 前驱指针（链表顺序）
    Entry<K, V> after;   // 后驱指针（链表顺序）
}

// HashMap 的 Node
static class Node<K, V> {
    int hash;
    K key;
    V value;
    Node<K, V> next;  // 这是桶内链表指针（解决哈希冲突）
}
```
结构图
```
┌──────────────────────────────────────────────────────────────────┐
│                    双向链表： 维护元素顺序                          │
│  head → A   ←→ B   ←→ C     ←→ D    ←→ E     ←→ F   ←→ G → tail  │
└──────────────────────────────────────────────────────────────────┘
           │     │       │       │       │        │      │
           ▼     ▼       ▼       ▼       ▼        ▼      ▼
      ┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐
      │ A    ││ B    ││ C    ││ D    ││ E    ││ F    ││ G    │
      └──────┘└──────┘└──────┘└──────┘└──────┘└──────┘└──────┘
         ↓
      ┌──────┐
      │ next │ → 桶内链表（解决哈希冲突） 只是说元素里原有next字段的目的
      └──────┘
// LinkedHashMap 维护了两个结构：
// ① 哈希表（继承自 HashMap）：用于快速查找 —— O(1)
// ② 双向链表（自己维护的）：用于记录顺序 —— 遍历时按链表走

// 所以 LinkedHashMap 的：
// - get/put 还是 O(1)（继承 HashMap 的哈希表）
// - 遍历时按双向链表顺序，不是按数组顺序

//单独在Entr继承HashMap的Node,再维护了before after，单独创建一个双向链表来记录顺序
```
### 两种顺序模式
插入顺序：默认 
```
// accessOrder = false（默认）
// 按元素插入的顺序维护链表

LinkedHashMap<String, String> map = new LinkedHashMap<>();
map.put("A", "1");
map.put("B", "2");
map.put("C", "3");
map.put("D", "4");

// 遍历顺序 = 插入顺序
map.forEach((k, v) -> System.out.println(k + "=" + v));
// 输出：
// A=1
// B=2
// C=3
// D=4
```
访问顺序
```
// accessOrder = true
// 每次访问（get/put）后，被访问的元素移到链表尾部
// 最近访问的 → 在尾部
// 最早访问的 → 在头部

LinkedHashMap<String, String> map = new LinkedHashMap<>(16, 0.75f, true);
map.put("A", "1");
map.put("B", "2");
map.put("C", "3");
map.put("D", "4");

// 初始顺序：A → B → C → D

map.get("B");  // 访问 B，B 移到尾部
map.get("A");  // 访问 A，A 移到尾部

// 现在顺序：C → D → B → A
// 头部（最久未访问）→ 尾部（最近访问）
map.forEach((k, v) -> System.out.println(k + "=" + v));
// 输出：
// C=3
// D=4
// B=2
// A=1
```
两种模式对比

| 模式   | accessOrder | 链表顺序    | 适用场景        |
| ---- | ----------- | ------- | ----------- |
| 插入模式 | false       | 按put顺序  | 日常使用，保持插入顺序 |
| 访问模式 | true        | 最近访问在尾部 | LRU缓存       |
### LRU缓存实现
removeEldestEntry方法
```java
// LinkedHashMap 提供了一个 protected 方法
// 每次 put 后会自动调用，返回 true 则删除头部元素

protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return false;  // 默认不删除
}

// 我们只需要重写这个方法，就能实现 LRU 缓存      但是要求顺序是访问顺序，如果是插入顺序不支持
```
实现LRU缓存
```java
// 实现一个容量为 3 的 LRU 缓存
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        // 初始容量、负载因子、访问顺序模式
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // 当元素个数超过容量时，删除最久未访问的元素
        return size() > capacity;
    }
}

// 使用
LRUCache<String, String> cache = new LRUCache<>(3);
cache.put("A", "1");  // 缓存: A
cache.put("B", "2");  // 缓存: A → B
cache.put("C", "3");  // 缓存: A → B → C
cache.get("A");       // 缓存: B → C → A（A 被访问，移到尾部）
cache.put("D", "4");  // 缓存: C → A → D（B 被淘汰！）
//LinkedHashMap重写了afterNodeInsertion方法，然后在put元素后，调用了这个方法，这个方法里提供的removeEldestEntry，可以在继承后重写实现LRU的逻辑

// 输出
cache.forEach((k, v) -> System.out.println(k + "=" + v));
// C=3
// A=1
// D=4
```
源码分析
```
// LinkedHashMap 的 put 方法 —— 继承自 HashMap
// 但 HashMap 的 put 中调用了 afterNodeInsertion 钩子方法
// LinkedHashMap 重写了这个钩子

// HashMap.putVal() 中的回调：
void afterNodeInsertion(boolean evict) {  // evict 是否驱逐
    // 可能删除最老的元素
}

// LinkedHashMap 的实现：  可能删除最老的元素
void afterNodeInsertion(boolean evict) {
    LinkedHashMap.Entry<K, V> first;
    // 如果允许驱逐，且头部不为空，且 removeEldestEntry 返回 true   默认是false 重写
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        // 删除头部元素
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}



// get 方法 —— LinkedHashMap 重写了
public V get(Object key) {
    Node<K, V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    // 如果是访问顺序模式，把访问的节点移到尾部
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}

// afterNodeAccess —— 把节点移到链表尾部
void afterNodeAccess(Node<K, V> e) {
    LinkedHashMap.Entry<K, V> last;
    // 如果 accessOrder 模式，且 e 不是尾节点
    if (accessOrder && (last = tail) != e) {
        // 把 e 从链表中拆出来
        LinkedHashMap.Entry<K, V> p =
            (LinkedHashMap.Entry<K, V>) e, b = p.before, a = p.after;
        p.after = null;

        // 让前后节点直接连接
        if (b == null)
            head = a;
        else
            b.after = a;

        if (a != null)
            a.before = b;
        else
            last = b;

        // 把 e 放到尾部
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
    }
}
```
> **Q:** "removeEldestEntry 什么时候被调用？" 
> **A:** "每次 put 新元素后，afterNodeInsertion 钩子会调用它。如果返回 true，就删除链表头部元素（最久未访问的）。"

> **Q:** "为什么 LinkedHashMap 适合做 LRU 缓存？" 
> **A:** "因为 LinkedHashMap 本身支持访问顺序模式，每次 get 会把访问的节点移到尾部，头部就是最久未访问的节点。再加上 removeEldestEntry 钩子，只需要重写这一个方法就能实现 LRU，不需要额外维护任何数据结构。"

> **Q:** "LinkedHashMap 实现的 LRU 缓存线程安全吗？" 
> **A:** "不安全。多线程环境下需要用 Collections.synchronizedMap 包装，或者用 ConcurrentLinkedHashMap（第三方库）。"
### 核心方法源码
put流程
```
// LinkedHashMap 没有重写 put 方法
// 直接使用 HashMap 的 put

// 但 HashMap 的 put 中有三个回调钩子：
// ① afterNodeAccess(e) —— 节点被访问后
// ② afterNodeInsertion(evict) —— 节点插入后
// ③ afterNodeRemoval(e) —— 节点被删除后

// LinkedHashMap 重写了这三个钩子来维护双向链表
```
afterNodeRemoval
```java
// 删除节点时，也要从双向链表中移除
void afterNodeRemoval(Node<K, V> e) {
    LinkedHashMap.Entry<K, V> p =
        (LinkedHashMap.Entry<K, V>) e, b = p.before, a = p.after;
    p.before = p.after = null;

    // 把前后节点接上
    if (b == null)
        head = a;
    else
        b.after = a;

    if (a == null)
        tail = b;
    else
        a.before = b;
}
```
containsValue优化
```java
// HashMap 的 containsValue 是遍历数组，O(n²)
// LinkedHashMap 重写了，遍历双向链表，O(n)

// HashMap 的 containsValue：
public boolean containsValue(Object value) {
    Node<K, V>[] tab;
    // 遍历数组 → 遍历每个桶的链表
    for (int i = 0; i < tab.length; i++) {
        for (Node<K, V> e = tab[i]; e != null; e = e.next) {
            if (valEquals(value, e.value))
                return true;
        }
    }
}

// LinkedHashMap 的 containsValue（重写）：
public boolean containsValue(Object value) {
    // 遍历双向链表，不需要遍历整个数组
    for (LinkedHashMap.Entry<K, V> e = head; e != null; e = e.after) {
        if (valEquals(value, e.value))
            return true;
    }
}
```
### 和HashMap对比
|对比维度|HashMap|LinkedHashMap|
|---|---|---|
|**底层**|数组+链表+红黑树|数组+链表+红黑树+双向链表|
|**顺序**|无序|插入顺序/访问顺序 ✅|
|**get 性能**|O(1)|O(1)（一样）|
|**put 性能**|O(1)|O(1)（略慢，维护链表）|
|**遍历性能**|遍历数组|遍历链表（更快）|
|**额外内存**|无|每个节点多 2 个指针|
|**线程安全**|❌|❌|
|**null key**|✅|✅|
|**特殊功能**|无|LRU 缓存 ✅|
性能差异
```
// put 性能：LinkedHashMap 略慢约 5%
// 因为每次 put 后需要额外维护双向链表

// 遍历性能：LinkedHashMap 更快
// HashMap 遍历需要遍历整个数组（包括空桶）
// LinkedHashMap 遍历只需要遍历双向链表（只含实际元素）

// 内存：LinkedHashMap 每个节点多 16 字节（before + after 指针）
```
### 差异化结论
核心差异

|维度|HashMap|LinkedHashMap|TreeMap|
|---|---|---|---|
|**顺序**|无序|插入/访问顺序|排序 ✅|
|**性能**|O(1) ✅|O(1)|O(log n)|
|**额外内存**|无|一点点（双向链表）|较多（红黑树）|
|**范围查询**|❌|❌|✅|
|**LRU 缓存**|❌|✅|❌|
|**null key**|✅|✅|❌|
### 选型场景结论
```
面试官："LinkedHashMap 有什么用？什么时候用？"

"LinkedHashMap 在 HashMap 的基础上加了双向链表来维护顺序，
主要有两个用途：

① 需要保持插入顺序
   比如数据库查询结果，希望按查询的字段顺序展示
   用 LinkedHashMap 存，遍历时就是插入顺序

② LRU 缓存
   重写 removeEldestEntry 方法，一行代码实现 LRU 淘汰策略
   超出时自动淘汰最久未访问的

和 HashMap 比，性能几乎一样，只是多了一点点内存开销。
所以如果不需要排序，但需要保持顺序，用 LinkedHashMap 就对了。"
```
