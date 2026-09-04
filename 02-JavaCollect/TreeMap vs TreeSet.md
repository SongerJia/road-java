## TreeMap vs  TreeSet
底层数据结构->排序规则->时间复杂度->TreeMap特有功能->TreeSet底层->对比->选型场景结论
### 底层数据结构
TreeMap的底层
```java
// TreeMap 的底层是红黑树（Red-Black Tree）
public class TreeMap<K, V> {
    // 根节点
    private transient Entry<K, V> root;

    // 比较器（决定排序规则）
    private final Comparator<? super K> comparator;

    // 节点 —— 红黑树节点
    static final class Entry<K, V> {
        K key;
        V value;
        Entry<K, V> left;   // 左子节点
        Entry<K, V> right;  // 右子节点
        Entry<K, V> parent; // 父节点
        boolean color = BLACK;  // 红色或黑色
    }
}
```
TreeSet的底层结构
```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, Serializable {

    private transient NavigableMap<E, Object> m;

    // 所有元素作为 key 存入 TreeMap，value 都用同一个占位对象
    private static final Object PRESENT = new Object();
	//基于TreeMap创建：底层也是红黑树
    public TreeSet() {
        this.m = new TreeMap<>();
    }

    public boolean add(E e) {
        return m.put(e, PRESENT) == null;
    }

    public boolean remove(Object o) {
        return m.remove(o) == PRESENT;
    }
}
```
### 排序规则
Comparable vs Comparator
```java
// 方式一：自然排序（key 实现 Comparable）
// key 必须实现 Comparable 接口
TreeMap<String, String> map1 = new TreeMap<>();
// String 实现了 Comparable，按字典序排序

// 方式二：自定义排序（传入 Comparator）
// 比较灵活，不修改 key 的源码
TreeMap<String, String> map2 = new TreeMap<>((a, b) -> b.compareTo(a));
// 按字符串倒序

//如果传了排序规则，构造器创建时就会固定排序规则，并且不要求key实现Comparable接口。即使key实现了，也使用传入的排序规则

// 自定义对象的排序
TreeMap<Integer, String> map3 = new TreeMap<>(Comparator.reverseOrder());
// 按 key 倒序
```

| 对比   | Comparable     | Comparator         |
| ---- | -------------- | ------------------ |
| 所在包  | java.lang      | java.util          |
| 方法   | compareTo(T o) | compare(T1 o,T2 o) |
| 实现方  | 被比较的类本身实现      | 单独创建一个比较器类         |
| 修改原类 | 需要修改           | 不需要                |
| 灵活性  | 单一排序规则         | 可以有多个比较器           |
| 使用场景 | 自然排序           | 自定义排序规则            |
```java
// 示例：自定义对象排序
public class Person implements Comparable<Person> {
    private String name;
    private int age;

    // Comparable：自然排序（按年龄）
    @Override
    public int compareTo(Person o) {
        return Integer.compare(this.age, o.age);
    }
}

// Comparator：自定义排序（按姓名）
Comparator<Person> byName = (p1, p2) -> p1.getName().compareTo(p2.getName());

// 使用
TreeMap<Person, String> map1 = new TreeMap<>();  // 按年龄排序
TreeMap<Person, String> map2 = new TreeMap<>(byName);  // 按姓名排序   创建对象时将排序规则传入
```
compareTo的返回规则
```
// 返回负数：当前对象 < 比较对象
// 返回 0：当前对象 = 比较对象
// 返回正数：当前对象 > 比较对象

// 必须满足：
// ① 自反性：x.compareTo(x) == 0
// ② 对称性：x.compareTo(y) == -y.compareTo(x)
// ③ 传递性：x > y 且 y > z → x > z

// 必须和 equals 保持一致（推荐）：
// x.compareTo(y) == 0 时，x.equals(y) 应该为 true
// 否则 TreeSet/TreeMap 的 contains 行为会不一致
```
### 时间复杂度
| 操作              | TreeMap  | HashMap |
| --------------- | -------- | ------- |
| **put**         | O(log n) | O(1)    |
| **get**         | O(log n) | O(1)    |
| **remove**      | O(log n) | O(1)    |
| **containsKey** | O(log n) | O(1)    |
| **遍历**          | O(n) 有序  | O(n) 无序 |
| **首尾元素**        | O(1) ✅   | O(n) ❌  |
```
// 100 万个元素
// HashMap：put/get 约 0.05μs
// TreeMap：put/get 约 0.5μs（慢 10 倍）

// 结论：HashMap 性能远高于 TreeMap
// 只有需要排序时才用 TreeMap
```
### TreeMap特有功能
范围查询
```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "A");
map.put(3, "C");
map.put(5, "E");
map.put(7, "G");
map.put(9, "I");

// ① 获取首尾元素
map.firstKey();      // 1
map.lastKey();       // 9
map.firstEntry();    // 1=A
map.lastEntry();     // 9=I

// ② 获取小于等于/大于等于的最近元素
map.floorKey(4);     // 3（≤4 的最大key）
map.ceilingKey(4);   // 5（≥4 的最小key）
map.floorEntry(4);   // 3=C
map.ceilingEntry(4);  // 5=E

// ③ 获取小于/大于的元素
map.lowerKey(5);     // 3（<5 的最大key）
map.higherKey(5);    // 7（>5 的最小key）

// ④ 获取子图
map.subMap(3, 7);      // {3=C, 5=E} —— [3, 7)
map.subMap(3, true, 7, true);  // {3=C, 5=E, 7=G} —— [3, 7]
map.headMap(5);        // {1=A, 3=C} —— <5
map.tailMap(5);        // {5=E, 7=G, 9=I} —— ≥5
```
实际应用场景
```
// 场景：获取最近的时间点
TreeMap<Long, String> timeMap = new TreeMap<>();
timeMap.put(1700000000000L, "事件A");
timeMap.put(1700000100000L, "事件B");
timeMap.put(1700000200000L, "事件C");

// 查询某个时间点最近的事件
long targetTime = 1700000150000L;
String nearest = timeMap.floorEntry(targetTime).getValue();  // "事件B"
```
### TreseSet的底层
```java
// TreeSet 的底层就是 TreeMap
// 和 HashSet 底层是 HashMap 一样

public class TreeSet<E> {
    // 底层就是 TreeMap，value 是固定 dummy（假） 对象
    private transient NavigableMap<E, Object> m;
    private static final Object PRESENT = new Object();

    public TreeSet() {
        this(new TreeMap<>());  // 默认用 TreeMap
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }

    public boolean add(E e) {
        return m.put(e, PRESENT) == null;  // 用 TreeMap 的 key 存元素
    }

    public boolean contains(Object o) {
        return m.containsKey(o);
    }

    // 继承 TreeMap 的所有范围查询功能
    public E first() { return m.firstKey(); }
    public E last() { return m.lastKey(); }
    public E floor(E e) { return m.floorKey(e); }
    public E ceiling(E e) { return m.ceilingKey(e); }
    public NavigableSet<E> subSet(E from, E to) { ... }
```
TreeSet特有方法
```java
TreeSet<Integer> set = new TreeSet<>();
set.add(1);
set.add(3);
set.add(5);
set.add(7);
set.add(9);

set.first();       // 1
set.last();        // 9
set.floor(4);      // 3
set.ceiling(4);    // 5
set.lower(5);      // 3
set.higher(5);     // 7
set.subSet(3, 7);  // [3, 5]
set.headSet(5);    // [1, 3]
set.tailSet(5);    // [5, 7, 9]
```
### 对比
| 对比维度         | HashMap         | HashSet         | TreeMap               | TreeSet               |
| ------------ | --------------- | --------------- | --------------------- | --------------------- |
| **底层**       | 数组+链表+红黑树       | HashMap         | 红黑树                   | TreeMap               |
| **顺序**       | 无序              | 无序              | 排序 ✅                  | 排序 ✅                  |
| **null key** | ✅ 允许一个          | ✅ 允许一个 null     | ❌ 不允许                 | ❌ 不允许                 |
| **时间复杂度**    | O(1)            | O(1)            | O(log n)              | O(log n)              |
| **范围查询**     | ❌               | ❌               | ✅                     | ✅                     |
| **比较方式**     | equals/hashCode | equals/hashCode | Comparable/Comparator | Comparable/Comparator |
| **内存**       | 少               | 少               | 多（红黑树节点）              | 多                     |
为什么TreeMap不允许null Key
```
// 因为 TreeMap 需要比较 key 来排序
// 如果 key 为 null，compareTo(null) 会抛 NPE

TreeMap<String, String> map = new TreeMap<>();
map.put(null, "value");  // ❌ NullPointerException

// 即使传递了 Comparator，如果 Comparator 不处理 null 也会抛异常
TreeMap<String, String> map2 = new TreeMap<>((a, b) -> {
    if (a == null) return -1;  // 需要自己处理 null
    if (b == null) return 1;
    return a.compareTo(b);
});
map2.put(null, "value");  // ✅ 自定义 Comparator 处理了 null
```
和LinkedHashMap对比

|对比|HashMap|LinkedHashMap|TreeMap|
|---|---|---|---|
|**顺序**|无序|插入/访问顺序|排序|
|**性能**|O(1)|O(1)|O(log n)|
|**底层**|数组|数组+双向链表|红黑树|
|**用途**|默认|需要保持顺序|需要排序|
### 选型场景结论
```
面试官："TreeMap 和 HashMap 怎么选？"

"大部分场景用 HashMap，只有需要排序时才用 TreeMap：

① 默认选择 HashMap：O(1) 性能，日常 90% 场景

② 需要保持插入顺序：LinkedHashMap，性能接近 HashMap

③ 需要排序或范围查询：TreeMap
   - 比如排行榜、时间线、区间查询
   - 代价是 O(log n) 的性能

④ 具体到我项目里：
   - 用户信息缓存 → HashMap
   - 最近访问记录 → LinkedHashMap（按访问顺序）
   - 排行榜 → TreeMap（按分数排序）
   - 时间线查询 → TreeMap（按时间戳范围查询）
"
```
