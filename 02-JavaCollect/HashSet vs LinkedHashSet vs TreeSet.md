## HashSet vs LinkedHashSet vs TreeSet
底层实现->顺序特性->null支持->性能对比->各自特有功能->差异化结论->选型场景
### 底层实现
三者的关系
```java
// HashSet 的底层是 HashMap
public class HashSet<E> extends AbstractSet<E> implements Set<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();  // 固定 dummy value

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // 元素存到 HashMap 的 key
    }
}

// LinkedHashSet 的底层是 LinkedHashMap，继承了HashSet但构造器调的是LinkedHashMap，  accessOrder 默认false
public class LinkedHashSet<E> extends HashSet<E> implements Set<E> {
    // 关键：调用父类HashSet的构造器，调用了不带 accessOrder 参数的 LinkedHashMap 构造器，排序方式是插入顺序
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        // 底层创建的是 LinkedHashMap！
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
}

// TreeSet 的底层是 TreeMap
public class TreeSet<E> extends AbstractSet<E> implements NavigableSet<E> {
    private transient NavigableMap<E, Object> m;
    private static final Object PRESENT = new Object();

    public TreeSet() {
        this(new TreeMap<>());  // 底层是 TreeMap
    }
}
```
结构图
```
HashSet：
HashMap（key 存元素，value 是固定对象）
┌────┬────┬────┬────┬────┬────┬────┐
│  A │    │  C │    │  B │    │    │  ← 无序，按哈希值分布
└────┴────┴────┴────┴────┴────┴────┘

LinkedHashSet：
LinkedHashMap（哈希表 + 双向链表）
链表：A ←→ B ←→ C ←→ D（插入顺序）
哈希表：按哈希值分布，但遍历走链表

TreeSet：
TreeMap（红黑树）
            C
          /   \
        A       E
              /   \
            D       F  ← 按排序顺序
```
### 顺序特性
演示三种顺序
```java
// 添加顺序：C, A, E, B, D
String[] input = {"C", "A", "E", "B", "D"};

// HashSet —— 无序
Set<String> hashSet = new HashSet<>();
for (String s : input) hashSet.add(s);
System.out.println(hashSet);  // [A, B, C, D, E]（或任意顺序，取决于哈希值）

// LinkedHashSet —— 插入顺序
Set<String> linkedHashSet = new LinkedHashSet<>();
for (String s : input) linkedHashSet.add(s);
System.out.println(linkedHashSet);  // [C, A, E, B, D] —— 保持插入顺序

// TreeSet —— 排序（自然顺序） /指定排序规则
Set<String> treeSet = new TreeSet<>();
for (String s : input) treeSet.add(s);
System.out.println(treeSet);  // [A, B, C, D, E] —— 按字典序
```
### null支持
```java
// HashSet —— 允许一个 null
Set<String> hashSet = new HashSet<>();
hashSet.add("A");
hashSet.add(null);  // ✅ 允许（HashMap 允许一个 null key）
hashSet.add(null);  // 不会添加，还是只有一个 null

// LinkedHashSet —— 允许一个 null
Set<String> linkedHashSet = new LinkedHashSet<>();
linkedHashSet.add(null);  // ✅ 允许

// TreeSet —— 不允许 null
Set<String> treeSet = new TreeSet<>();
treeSet.add(null);  // ❌ NullPointerException！比较时抛异常

// 即使是自定义 Comparator 也要自己处理 null
Set<String> treeSet2 = new TreeSet<>((a, b) -> {
    if (a == null) return -1;
    if (b == null) return 1;
    return a.compareTo(b);
});
treeSet2.add(null);  // ✅ 自定义 Comparator 处理了 null
```
### 性能对比
|操作|HashSet|LinkedHashSet|TreeSet|
|---|---|---|---|
|**add**|O(1)|O(1)|O(log n)|
|**contains**|O(1)|O(1)|O(log n)|
|**remove**|O(1)|O(1)|O(log n)|
|**遍历**|O(n) 无序|O(n) 插入顺序|O(n) 排序|
实测对比
```
// 10 万个元素 
// HashSet：add/contains 约 0.05μs
// LinkedHashSet：add/contains 约 0.06μs（略慢，维护链表）
// TreeSet：add/contains 约 0.8μs（慢 15 倍左右）

//内存占用对比
// HashSet：HashMap 结构，无额外开销
// LinkedHashSet：每个元素多 2 个指针（双向链表）
// TreeSet：红黑树节点（parent/left/right/color），内存最多
```
### 各自特有功能
HashSet特点
```
// ① 性能最高
// ② 去重（利用 equals + hashCode）
// ③ 无序

// 典型应用：去重
List<String> list = Arrays.asList("A", "B", "A", "C", "B");
Set<String> unique = new HashSet<>(list);
System.out.println(unique);  // [A, B, C] —— 去重
```
LinkedHashSet特点
```
// ① 保持插入顺序
// ② 性能接近 HashSet
// ③ 去重

// 典型应用：去重 + 保持顺序
List<String> list = Arrays.asList("A", "B", "A", "C", "B");
Set<String> unique = new LinkedHashSet<>(list);
System.out.println(unique);  // [A, B, C] —— 去重且保持首次出现顺序
// 这个特性常用于：URL 去重、消息去重后保持原始顺序
```
TreeSet特点
```
// ① 排序
// ② 范围查询
// ③ 去重

// 典型应用：排序去重
TreeSet<Integer> scores = new TreeSet<>();
scores.add(85);
scores.add(92);
scores.add(78);
scores.add(92);  // 重复，不会添加

System.out.println(scores);     // [78, 85, 92] —— 排序且去重
scores.first();                 // 78 —— 最低分
scores.last();                  // 92 —— 最高分
scores.floor(88);               // 85 —— 小于等于 88 的最大值
scores.ceiling(88);             // 92 —— 大于等于 88 的最小值
scores.subSet(80, 90);          // [85] —— 区间 [80, 90)
```
### 差异化结论
| 维度       | HashSet         | LinkedHashSet   | TreeSet               |
| -------- | --------------- | --------------- | --------------------- |
| **底层**   | HashMap         | LinkedHashMap   | TreeMap               |
| **顺序**   | 无序              | 插入顺序            | 排序                    |
| **性能**   | O(1) ✅          | O(1)            | O(log n)              |
| **null** | ✅ 一个            | ✅ 一个            | ❌                     |
| **去重依据** | equals/hashCode | equals/hashCode | Comparable/Comparator |
| **范围查询** | ❌               | ❌               | ✅                     |
| **内存**   | 少 ✅             | 中               | 多                     |
### 选型场景结论
```
面试官："HashSet、LinkedHashSet、TreeSet 的区别？什么时候用哪个？"

"三个都是 Set 接口的实现，都不能有重复元素，区别在顺序和性能：

① HashSet：无序，O(1) 性能最高，日常默认选择
   —— 比如用户 ID 去重，不关心顺序

② LinkedHashSet：保持插入顺序，性能接近 HashSet
   —— 比如 URL 去重后按首次出现顺序展示

③ TreeSet：自动排序，O(log n)，支持范围查询
   —— 比如排行榜、分数排序去重

底层实现上：
HashSet 用 HashMap，LinkedHashSet 用 LinkedHashMap，
TreeSet 用 TreeMap，都是 key 存元素、value 存固定对象。

我的选择原则：默认 HashSet，需要顺序用 LinkedHashSet，
需要排序或范围查询才用 TreeSet（性能代价 O(log n)）。"
```
