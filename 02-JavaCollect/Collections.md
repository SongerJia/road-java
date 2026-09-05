## Collections
Collections总览->排序与查找->不可变集合->同步包装->空集合与单元素结合->其他实用方法->差异化结论
### Collections总览
什么是Collections?
Collections是操作集合的静态工具类。
```java
// 注意区分：
// Collection —— 接口，集合体系的顶层接口
// Collections —— 工具类，提供操作集合的静态方法

// 所有方法都是 static，直接类名调用
Collections.sort(list);
Collections.unmodifiableList(list);
```
方法分类总览

|分类|方法|说明|
|---|---|---|
|**排序**|`sort()`、`reverse()`、`shuffle()`|排序、反转、打乱|
|**查找**|`binarySearch()`、`max()`、`min()`|二分查找、最大最小|
|**填充**|`fill()`、`copy()`、`replaceAll()`|填充、复制、替换|
|**不可变**|`unmodifiableXxx()`|返回只读视图|
|**同步**|`synchronizedXxx()`|线程安全包装|
|**空/单元素**|`emptyXxx()`、`singletonXxx()`|空集合、单元素集合|
|**其他**|`frequency()`、`disjoint()`、`rotate()`|频次、无交集、旋转|
### 排序与查找
排序
```java
// ① 自然排序
List<Integer> list = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5));
Collections.sort(list);          // [1, 1, 3, 4, 5] —— 升序

// ② 自定义排序
Collections.sort(list, Comparator.reverseOrder());  // 降序
Collections.sort(list, (a, b) -> b - a);            // 降序（Lambda）

// ③ 反转
Collections.reverse(list);  // 反转顺序

// ④ 打乱
Collections.shuffle(list);  // 随机打乱
```
查找
```java
// ① 二分查找（前提：列表已排序）
List<Integer> list = Arrays.asList(1, 3, 5, 7, 9);
Collections.sort(list);  // 必须先排序！
int index = Collections.binarySearch(list, 5);  // 2

// 没找到时返回负数：-(插入点) - 1
int notFound = Collections.binarySearch(list, 6);  // -4（应插在索引3，-(3)-1=-4）

// ② 最大最小值
List<Integer> nums = Arrays.asList(3, 1, 4, 1, 5);
Collections.max(nums);  // 5
Collections.min(nums);  // 1
Collections.max(nums, Comparator.reverseOrder());  // 1（按降序的最大=最小）

// ③ 频次统计
Collections.frequency(nums, 1);  // 2（1 出现 2 次）
```
### 不可变集合
为什么需要不可变集合？
```java
// ① 安全 —— 防止外部代码修改内部数据
public class Config {
    // 配置项不应该被外部修改
    public static final List<String> PERMISSIONS =
        Collections.unmodifiableList(
            Arrays.asList("READ", "WRITE", "DELETE"));
}

// ② 线程安全 —— 不可变对象天然线程安全
// ③ 可作为常量共享

// 如果不做保护：
public static final List<String> PERMISSIONS = new ArrayList<>();
// 外部可以直接：Config.PERMISSIONS.add("HACK") —— 危险！
```
不可变包装
```java
// 三种不可变包装
List<String> list = new ArrayList<>(Arrays.asList("A", "B"));
Map<String, String> map = new HashMap<>();
Set<String> set = new HashSet<>();

// 返回不可变视图
List<String> unmodifiableList = Collections.unmodifiableList(list);
Map<String, String> unmodifiableMap = Collections.unmodifiableMap(map);
Set<String> unmodifiableSet = Collections.unmodifiableSet(set);

// 尝试修改会抛异常
unmodifiableList.add("C");  // ❌ UnsupportedOperationException
unmodifiableList.remove(0); // ❌ UnsupportedOperationException
unmodifiableList.set(0, "X");  // ❌ UnsupportedOperationException
```
不可变视图的陷阱
```java
// 注意：unmodifiableXxx 只是"包装"，不是"拷贝"
List<String> original = new ArrayList<>(Arrays.asList("A", "B"));
List<String> unmodifiable = Collections.unmodifiableList(original);

// 修改原列表 —— 视图也会变化！
original.add("C");
System.out.println(unmodifiable);  // [A, B, C] —— 视图跟着变了！

// 要真正不可变，需要拷贝：
List<String> trulyUnmodifiable =
    Collections.unmodifiableList(new ArrayList<>(original));
// 修改 original 不影响 trulyUnmodifiable
```
java9+的不可变集合
```java
// Java 9 提供了更简洁的不可变集合（List.of 等）
// 注意：这个是完全不可变的，不是视图
List<String> immutable = List.of("A", "B", "C");   // ✅ 真正不可变
Set<String> immutableSet = Set.of("A", "B");       // ✅
Map<String, String> immutableMap = Map.of("A", "1", "B", "2");  // ✅

immutable.add("D");  // ❌ UnsupportedOperationException

// 对比：
// Collections.unmodifiableList(原列表)：包装视图，原列表变它也变
// List.of(...)：真正的不可变集合，独立副本

// 限制：
// List.of() 不接受 null 元素
List.of("A", null);  // ❌ NullPointerException
// List.of() 没有 null、无序
```
### 同步包装
```java
// 把非线程安全的集合包装成线程安全的
// 原理：所有方法加 synchronized 锁整个集合

List<String> list = new ArrayList<>();
List<String> syncList = Collections.synchronizedList(list);

Map<String, String> map = new HashMap<>();
Map<String, String> syncMap = Collections.synchronizedMap(map);

Set<String> set = new HashSet<>();
Set<String> syncSet = Collections.synchronizedSet(set);
```
注意事项
```java
// ① 同步包装的方法本身线程安全
// 但复合操作不安全！
Collections.synchronizedList(list);

// 多个操作组合时不安全：
if (!list.isEmpty()) {  // 检查
    list.get(0);        // 使用 —— 中间可能被其他线程修改！
}
// 复合操作需要手动加锁
synchronized (list) {
    if (!list.isEmpty()) {
        list.get(0);
    }
}

// ② 遍历时需要手动加锁
synchronized (syncList) {
    for (String s : syncList) {
        // 遍历期间其他线程不能修改
    }
}
```
对比 ConcurrentHashMap

|对比|synchronizedMap|ConcurrentHashMap|
|---|---|---|
|**锁粒度**|全表锁|桶锁/CAS|
|**并发读**|读加锁，阻塞|读无锁|
|**性能**|差|好|
|**适用**|低并发、简单需求|高并发|
```
// 结论：高并发场景优先用 JUC 的并发集合
// ConcurrentHashMap → 替代 synchronizedMap
// CopyOnWriteArrayList → 替代 synchronizedList
// ConcurrentLinkedQueue → 替代 synchronizedQueue
```
### 空集合与单元素集合
空集合
```java
// 为什么需要？避免返回 null，防止 NPE
public List<String> getNames() {
    if (empty) {
        return Collections.emptyList();  // ✅ 返回空集合而不是 null
        // return null;  // ❌ 调用方可能 NPE
    }
    return names;
}

// 三种空集合
Collections.emptyList();   // List
Collections.emptySet();    // Set
Collections.emptyMap();    // Map

// 注意：返回的是同一个共享实例（不可变）
List<String> e1 = Collections.emptyList();
List<String> e2 = Collections.emptyList();
System.out.println(e1 == e2);  // true —— 同一个实例

// Java 9+ 的写法：List.of() 也能创建空集合
List<String> empty = List.of();
```
单元素集合
```java
// 为什么需要？某些 API 需要传集合类型
Collections.singletonList("A");  // 只含一个元素的 List
Collections.singleton("A");      // 只含一个元素的 Set
Collections.singletonMap("A", "1");  // 只含一个键值对的 Map

// 特性：不可变
List<String> single = Collections.singletonList("A");
single.add("B");  // ❌ UnsupportedOperationException

// 用途：批量方法只需要一个元素时
list.removeAll(Collections.singletonList("A"));  // 删除所有 "A"
list.removeAll(Collections.singleton("A"));      // 同上
```
### 其他实用方法
fill/copy/replaceAll
```java
// ① fill —— 用相同元素填充
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
Collections.fill(list, "X");  // [X, X, X]

// ② copy —— 复制
List<String> src = Arrays.asList("A", "B");
List<String> dest = new ArrayList<>(Arrays.asList("X", "Y", "Z"));
Collections.copy(dest, src);  // dest 前两位被覆盖 → [A, B, Z]
// 注意：dest.size() >= src.size()，否则异常

// ③ replaceAll —— 替换所有匹配元素
List<String> list2 = new ArrayList<>(Arrays.asList("A", "B", "A", "C"));
Collections.replaceAll(list2, "A", "X");  // [X, B, X, C]
```
rotate/swap
```java
// ① rotate —— 旋转
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
Collections.rotate(list, 2);  // 右移 2 位 → [4, 5, 1, 2, 3]

// ② swap —— 交换
Collections.swap(list, 0, 4);  // [3, 5, 1, 2, 4]
```
disjoint
```java
// 判断两个集合是否有交集
List<String> list1 = Arrays.asList("A", "B");
List<String> list2 = Arrays.asList("C", "D");
List<String> list3 = Arrays.asList("B", "D");

Collections.disjoint(list1, list2);  // true —— 无交集
Collections.disjoint(list1, list3);  // false —— 有交集（B）
```
### 差异化结论
```
面试官："Collections 工具类你用过哪些？"

"Collections 是集合操作的静态工具类，我常用的有：

① 排序查找：sort、binarySearch、max、min

② 不可变集合：unmodifiableList/unmodifiableMap
   防止外部修改内部数据，比如配置项、权限列表

③ 空集合：emptyList/emptySet/emptyMap
   方法返回时避免返回 null，防止调用方 NPE

④ 单元素集合：singletonList/singleton
   批量方法只需要一个元素时用

⑤ 同步包装：synchronizedList
   但高并发场景我会优先用 JUC 的
   ConcurrentHashMap 替代 synchronizedMap

注意点：
unmodifiableXxx 只是视图，不是拷贝
复合操作和遍历时需要手动加锁"
```