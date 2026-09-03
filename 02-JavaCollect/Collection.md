## 集合体系
Collection体系->List->Set->Queue->各接口对比->面试高频题
### Collection体系
继承结构
```code
				     Iterable
                        |
                    Collection
                   /    |    \
                 List   Set   Queue
                 /    \   |     |
       ArrayList  LinkedList  HashSet  LinkedList(Deque)
         |             |       |
    Vector          TreeSet  LinkedHashSet
       |
     Stack
```
三大接口核心区别

| 接口    | 是否有序         | 是否允许重复 | 是否允许为null | 主要实现                                |
| ----- | ------------ | ------ | --------- | ----------------------------------- |
| List  | 有序（插入时顺序）    | 允许     | 允许        | ArrayList、LinkedList、Vector         |
| Set   | 无序           | 不允许    | 允许部分      | HashSet、LinkedHashSet、TressSet      |
| queue | 有序（FIFO、优先级） | 允许     | 部分不允许     | LinkedList、PriorityQueue、ArrayDeque |

```java
// List —— 有序，可重复
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("A");  // ✅ 重复
System.out.println(list.get(0));  // "A" —— 按索引访问

// Set —— 不可重复
Set<String> set = new HashSet<>();
set.add("A");
set.add("B");
set.add("A");  // ❌ 不会报错，但不会添加进去
System.out.println(set.size());  // 2

// Queue —— 队列，FIFO
Queue<String> queue = new LinkedList<>();
queue.offer("A");
queue.offer("B");
String first = queue.poll();  // "A" —— 先进先出
```
### List接口
List的核心特性
```java
// ① 有序 —— 按插入顺序保持
List<String> list = new ArrayList<>();
list.add("C");
list.add("A");
list.add("B");
System.out.println(list);  // [C, A, B] —— 按插入顺序

// ② 可重复
list.add("A");
System.out.println(list);  // [C, A, B, A]

// ③ 支持索引访问
list.get(0);    // "C"
list.set(0, "X");  // 替换
list.add(1, "Y");  // 插入到指定位置
list.remove(0);    // 按索引删除
```
List三大实现对比

| 对比          | ArrayList | LinkedList  | Vector           |
| ----------- | --------- | ----------- | ---------------- |
| 底层结构        | 动态数组      | 双向链表        | 动态数组             |
| 随机访问 get(i) | O(1)      | O(n)        | O(1)             |
| 插入/删除头部     | O(n)      | O(1)        | O(n)             |
| 插入/删除尾部     | O(1)      | O(1)        | O(1)             |
| 插入/删除中间     | O(n)      | O(n)        | O(n)             |
| 线程安全        | 不安全       | 不安全         | 安全（Synchronized） |
| 扩容机制        | 1.5倍      | 不需要         | 2倍               |
| 内存占用        | 少         | 多（每个节点前后指针） | 少                |
| 使用场景        | 读多写少      | 频繁头尾删除      | 基本不用             |
面试高频：ArrayList的扩容机制
```java
// ArrayList 默认容量 10
List<String> list = new ArrayList<>();  // 默认容量 10
// 也可以指定容量
List<String> list2 = new ArrayList<>(1000);  // 避免多次扩容

// 扩容源码
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);  // 数组拷贝
}

// 为什么是 1.5 倍不是 2 倍？
// 1.5 倍是经验值：扩容太快浪费内存，扩容太慢频繁拷贝
// 1.5 倍 ≈ 黄金分割，时间和空间的平衡
```
面试高频：RandomAccess标记接口
```java
// ArrayList 实现了 RandomAccess 接口
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, Serializable {
}

// LinkedList 没有实现
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, Serializable {
}

// RandomAccess 是标记接口，用来判断是否支持高效随机访问
public static <T> void print(List<T> list) {
    if (list instanceof RandomAccess) {
        // ArrayList：用 for 循环 + get(i) 更快
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    } else {
        // LinkedList：用迭代器更快
        for (T item : list) {
            System.out.println(item);
        }
    }
}
```
**追问：**

> **Q:** "ArrayList 和 LinkedList 什么时候用哪个？" 
> **A:** "
> - 大部分场景用 **ArrayList**：随机访问多，尾部追加多
> - 只有频繁在**头部或中间插入/删除**时才考虑 LinkedList
> - 实际项目中 ArrayList 占了 90% 以上的场景 "
### Set接口
Set核心特性
```java
// ① 不允许重复 —— 通过 equals() 和 hashCode() 判断
// ② 最多一个 null 元素
// ③ 没有索引，不能 get(i)

Set<String> set = new HashSet<>();
set.add("A");
set.add("B");
set.add("A");  // 不会添加，set.size() 还是 2
set.add(null); // HashSet 允许一个 null
```
三大Set实现对比

| 对比        | HashSet   | LinkedHashSet | TreeSet         |
| --------- | --------- | ------------- | --------------- |
| 底层实现      | HashMap   | LinkedHashMap | TreeMap         |
| 顺序        | 无序        | 插入顺序          | 自然顺序或Comparator |
| 是否允许为null | 允许一个为null | 允许一个为null     | 不允许为null        |
| 时间复杂度     | O(1)      | O(1)          | O(logn)         |
| 是否排序      | 否         | 否             | 排序              |
| 使用场景      | 默认选择      | 需要保持插入顺序      | 需要排序            |
```java
// HashSet —— 无序，最快
Set<String> hashSet = new HashSet<>();
hashSet.add("C");
hashSet.add("A");
hashSet.add("B");
System.out.println(hashSet);  // [A, B, C] —— 无序（按哈希值排列）

// LinkedHashSet —— 保持插入顺序
Set<String> linkedHashSet = new LinkedHashSet<>();
linkedHashSet.add("C");
linkedHashSet.add("A");
linkedHashSet.add("B");
System.out.println(linkedHashSet);  // [C, A, B] —— 按插入顺序

// TreeSet —— 自然排序
Set<String> treeSet = new TreeSet<>();
treeSet.add("C");
treeSet.add("A");
treeSet.add("B");
System.out.println(treeSet);  // [A, B, C] —— 按字典序
```
HashSet底层实现
```java
// HashSet 的底层就是 HashMap
// 存到 HashMap 的 key 中，value 是一个固定的 dummy 对象

public class HashSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();  // dummy value

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // 用 HashMap 的 key 存元素
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public int size() {
        return map.size();
    }
}
```
面试高频：为什么重新equals必须重新hashCode?
```java
// 因为 HashSet 依赖 hashCode() 来判断元素位置
// 两个对象 equals 相等，但 hashCode 不同 → 会被存到不同位置 → 都添加成功 → 违反 Set 不重复的约定

public class User {
    private String name;

    @Override
    public boolean equals(Object o) {
        // 只重写了 equals
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(name, user.name);
    }
    // 没重写 hashCode——默认用内存地址生成的hashcode

    Set<User> set = new HashSet<>();
    set.add(new User("张三"));
    set.add(new User("张三"));
    System.out.println(set.size());  // 2 —— 明明相等，但被当作两个对象！
}
```
### Queue接口
Queue的核心特性
```java
// 队列 —— FIFO（先进先出）
Queue<String> queue = new LinkedList<>();

// 添加元素
queue.offer("A");  // ✅ 推荐：添加失败返回 false
queue.add("B");    // 添加失败抛异常

// 查看头部元素（不移除）
queue.peek();  // "A" —— 队列为空返回 null
queue.element(); // 队列为空抛 NoSuchElementException

// 移除头部元素
queue.poll();  // "A" —— 队列为空返回 null
queue.remove(); // 队列为空抛 NoSuchElementException
```
queue的两套API

| 操作  | 抛异常       | 返回特殊值    |
| --- | --------- | -------- |
| 添加  | add(e)    | offer(e) |
| 移除  | remove()  | poll()   |
| 查看  | element() | peek()   |
Deque：双端队列
```java
// Deque —— 两端都可以插入和删除
Deque<String> deque = new ArrayDeque<>();

// 作为队列（FIFO）
deque.offerLast("A");  // 尾部添加
deque.offerLast("B");
String first = deque.pollFirst();  // 头部移除 —— "A"

// 作为栈（LIFO）—— 比 Stack 快
deque.push("A");     // 头部添加
deque.push("B");
String top = deque.pop();  // 头部移除 —— "B"

// 两端操作
deque.addFirst("X");
deque.addLast("Y");
deque.removeFirst();
deque.removeLast();
```
### Collection工具方法
```java
// Collection 接口提供的通用方法
Collection<String> c = new ArrayList<>();

c.add("A");           // 添加
c.addAll(otherList);  // 批量添加
c.remove("A");        // 移除
c.removeAll(otherList); // 批量移除
c.retainAll(otherList); // 只保留交集
c.clear();            // 清空
c.size();             // 大小
c.isEmpty();          // 是否为空
c.contains("A");      // 是否包含
c.containsAll(list);  // 是否包含全部
c.toArray();          // 转数组
c.iterator();         // 获取迭代器
```
### 面试高频题
#### 题目1：ArrayList的subList陷阱
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C", "D"));
List<String> subList = list.subList(1, 3);  // [B, C]

// subList 返回的是视图，不是独立副本
subList.set(0, "X");
System.out.println(list);  // [A, X, C, D] —— 原列表也被改了！

// 修改原列表会导致 subList 失效
list.add("E");
System.out.println(subList);  // ❌ ConcurrentModificationException！
```
#### 题目2：ArrayList的asList陷阱
```java
// Arrays.asList 返回的是固定大小的列表
List<String> list = Arrays.asList("A", "B", "C");//固定大小的视图，不提供add/remove等修改方法，继承的抽象类默认是抛异常
list.set(0, "X");  // ✅ 可以修改
list.add("D");      // ❌ UnsupportedOperationException！不能添加/删除  修改会影响原list
// 因为返回的是 Arrays 的内部类 ArrayList，不是 java.util.ArrayList

// 正确做法
List<String> list2 = new ArrayList<>(Arrays.asList("A", "B", "C"));
list2.add("D");  // ✅ 真正的 ArrayList
```
#### 题目3：集合转数组的推荐方式
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));

// ❌ 不推荐
String[] arr1 = (String[]) list.toArray();  // 有类型转换警告

// ✅ 推荐
String[] arr2 = list.toArray(new String[0]);  // 传入空数组
// 或者
String[] arr3 = list.toArray(new String[list.size()]);  // 传入指定大小
```
#### 题目4：集合的快速失败（fail-fast）
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));

// 遍历时修改会抛 ConcurrentModificationException
for (String s : list) {
    if (s.equals("B")) {
        list.remove(s);  // ❌ ConcurrentModificationException！
    }
}

// 正确方式一：使用迭代器的 remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("B")) {
        it.remove();  // ✅ 安全删除
    }
}

// 正确方式二：使用 removeIf（Java 8+）
list.removeIf(s -> s.equals("B"));

// 正确方式三：收集后再删除
List<String> toRemove = new ArrayList<>();
for (String s : list) {
    if (s.equals("B")) {
        toRemove.add(s);
    }
}
list.removeAll(toRemove);
//Iterator 是什么
Collection extends Iterator 接口
是java提供的一种统一遍历集合的工具接口，集合需要实现自己的遍历逻辑，虽然foreach是语法糖，编译器会生成遍历，
当Iterator被创建时会生成集合快照expectedModCount,每次的next()会校验modCount是否一样，当使用it.remove()时，在删除后会将
modCount重新赋值给expectedModCount.
// 编译器想生成
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    it.remove();   // 编译器想帮你这样写
    list.remove(s); //这个是事实上的内容，使用的不是编译器的it
}
```