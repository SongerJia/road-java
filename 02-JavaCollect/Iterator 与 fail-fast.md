## Iterator与fail-fast
迭代器概念->迭代器接口->for-each的本质->fail-fast机制->modCount源码->fail-safe对比->面试高频题目
### 迭代器概念
什么是迭代器？
Iterator：一种设计模式，提供统一的方式遍历集合，而不是暴露集合的内部结构。
```
// 为什么需要迭代器？
// ① 屏蔽底层差异 —— 数组/链表/树遍历方式不同，但迭代器接口统一
// ② 遍历中安全删除 —— 集合自身的 remove 方法在遍历时可能出问题
// ③ 统一接口 —— 所有 Collection 都能用同样的方式遍历
```
迭代器接口
```java
public interface Iterator<E> {
    boolean hasNext();  // 是否还有下一个元素
    E next();           // 返回下一个元素
    default void remove() { throw new UnsupportedOperationException(); }  // 删除当前元素
    default void forEachRemaining(Consumer<? super E> action) { ... }     // 遍历剩余元素
}

// 增强版（List 专用）
public interface ListIterator<E> extends Iterator<E> {
    boolean hasPrevious();  // 是否有前一个元素
    E previous();           // 返回前一个元素
    int nextIndex();        // 下一个元素的索引
    int previousIndex();    // 前一个元素的索引
    void set(E e);          // 修改当前元素
    void add(E e);          // 在当前位置插入
}
```
### 迭代器使用
```java
List<String> list = Arrays.asList("A", "B", "C");

// 传统迭代器
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    System.out.println(s);
}

// 遍历中安全删除
List<String> list2 = new ArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> iterator = list2.iterator();
while (iterator.hasNext()) {
    String s = iterator.next();
    if (s.equals("B")) {
        iterator.remove();  // ✅ 安全删除
    }
}
// list2 = [A, C]
```
ListIterator双向遍历
```java
List<String> list = Arrays.asList("A", "B", "C");

ListIterator<String> it = list.listIterator(list.size());  // 从尾部开始
while (it.hasPrevious()) {   //与尾部开始对应
    System.out.println(it.previous());  // C B A
}
```
### for-each 本质
```java
// for-each 是语法糖，本质就是迭代器
List<String> list = Arrays.asList("A", "B", "C");

// 这种写法：
for (String s : list) {
    System.out.println(s);
}

// 编译后等价于：
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    System.out.println(s);
}

// 所以 for-each 中不能直接调用 list.remove()
for (String s : list) {
    list.remove(s);  // ❌ ConcurrentModificationException！ 使用的迭代器不是一个
}
```
for-each局限性
```java
// ① 不能删除元素（会抛异常）
for (String s : list) {
    list.remove(s);  // ❌ 异常
}

// ② 不能修改元素
for (String s : list) {
    s = "X";  // ❌ 只是局部变量，不影响集合
}

// ③ 不能获取索引
for (String s : list) {
    // 不知道当前是第几个
}
```
### fail-fast机制
什么是fail-fast?
```
fail-fast 快速失败：当迭代器发生集合在遍历过程中被结构性修改时，立即抛出ConcurrentModificationException,而不是继续遍历
```
示例
```java
// 结构性修改：add、remove、clear 等改变集合大小的操作
// 注意：set（修改元素值）不算结构性修改

List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C", "D"));

Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("B")) {
        list.add("X");  // ❌ 结构性修改 → 抛出异常
    }
}
// 异常：ConcurrentModificationException
```
为什么叫快速失败
```
// 对比两种策略：
// ① fail-fast：发现问题立即抛异常（快速失败）
//    好处：尽早暴露问题，避免继续遍历产生错误结果

// ② fail-safe：不抛异常，继续遍历（安全失败）
//    问题：遍历结果可能不正确，问题被掩盖

// 快速失败的理念：宁可报错，也不给错误的结果
```
### modCount源码
ArrayList的modCount
```java
// AbstractList 中定义了 modCount
public abstract class AbstractList<E> {
    // 修改计数器：每次结构性修改 +1
    protected transient int modCount = 0;
}

// 结构性修改都会 modCount++
public class ArrayList<E> {
    public boolean add(E e) {
        modCount++;  // 结构性修改
        ...
    }

    public E remove(int index) {
        modCount++;  // 结构性修改
        ...
    }

    public void clear() {
        modCount++;  // 结构性修改
        ...
    }

    // set 不算结构性修改，不会 modCount++
    public E set(int index, E element) {
        // 没有 modCount++
        ...
    }
}
```
迭代器内部实现
```java
// 迭代器创建时记录 modCount
private class Itr implements Iterator<E> {
    // 创建迭代器时记录当前的 modCount
    int expectedModCount = modCount;

    public E next() {
        checkForComodification();  // 每次 next 前检查
        ...
    }

    public void remove() {
        ...
        // 迭代器的 remove 会同步更新 expectedModCount
        expectedModCount = modCount;  // 同步
    }

    // 检查：如果 modCount 变了，说明有其他操作修改了集合
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```
执行流程
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> it = list.iterator();
// 此时：modCount = 0，expectedModCount = 0

list.add("D");  // modCount = 1

it.next();  // checkForComodification():
            // modCount(1) != expectedModCount(0) → 抛异常！🚨
```
迭代器remove为什么安全？
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("B")) {
        it.remove();  // ✅ 安全
        // 因为迭代器的 remove() 会： 使用一个迭代器
        // ① 调用 list 的 remove（modCount++）
        // ② 同步 expectedModCount = modCount
        // 所以下一次检查能通过
    }
}
```
### fail-safe对比
fail-safe 安全失败
```java
// fail-safe 的集合：遍历时修改不会抛异常
// 但也不保证读到最新数据（弱一致性）

CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("A");
list.add("B");

Iterator<String> it = list.iterator();
list.add("C");  // ✅ 不抛异常

while (it.hasNext()) {
    System.out.println(it.next());  // A B —— 快照，看不到 C
}

// 例子：ConcurrentHashMap
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put("A", "1");

Iterator<String> it2 = map.keySet().iterator();
map.put("B", "2");  // ✅ 不抛异常

// 弱一致性：迭代器创建后新增的元素，遍历时可能看不到
```
对比总结

|对比|fail-fast|fail-safe|
|---|---|---|
|**原理**|遍历时检测 modCount|遍历快照或副本|
|**修改时行为**|抛 ConcurrentModificationException|不抛异常|
|**数据一致性**|可能读到不一致数据（已改部分）|弱一致性（读旧快照）|
|**典型实现**|ArrayList、HashMap、LinkedList|CopyOnWriteArrayList、ConcurrentHashMap|
|**适用**|单线程或同步控制的场景|高并发场景|
### 面试高频

> **Q:** "fail-fast 和 fail-safe 的区别？" 
> **A:** " ① fail-fast（快速失败）：遍历时检测到结构性修改，立即抛 ConcurrentModificationException —— ArrayList、HashMap ② fail-safe（安全失败）：遍历的是快照或副本，修改不影响遍历，不抛异常 —— CopyOnWriteArrayList、ConcurrentHashMap（弱一致性）
> 注意：fail-safe 只是"不抛异常"，但读到的是旧数据，不能保证实时一致。
### 面试高频题目
#### 题目1：for-each循环中删除元素的所有正确方式
```java
// ❌ 错误：for-each 中直接删除
for (String s : list) {
    list.remove(s);  // ❌ ConcurrentModificationException
}

// ❌ 错误：for 循环索引删除（会漏元素）
for (int i = 0; i < list.size(); i++) {
    if (条件) list.remove(i);  // ❌ 删除后索引错位，漏掉元素
}

// ✅ 正确一：迭代器 remove
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (条件) it.remove();
}

// ✅ 正确二：removeIf（Java 8+）
list.removeIf(s -> 条件);

// ✅ 正确三：收集后删除
List<String> toRemove = new ArrayList<>();
for (String s : list) {
    if (条件) toRemove.add(s);
}
list.removeAll(toRemove);

// ✅ 正确四：倒序遍历删除
for (int i = list.size() - 1; i >= 0; i--) {
    if (条件) list.remove(i);  // 从后往前删，索引不会错位
}
```
#### 题目2：HashMap遍历时删除
```java
// ❌ 错误
for (String key : map.keySet()) {
    map.remove(key);  // ❌ ConcurrentModificationException
}

// ✅ 正确：迭代器
Iterator<String> it = map.keySet().iterator();
while (it.hasNext()) {
    String key = it.next();
    it.remove();  // ✅
}

// ✅ 正确：removeIf（Java 8+）
map.keySet().removeIf(key -> 条件);

// ✅ 正确：entrySet 遍历
map.entrySet().removeIf(entry -> 条件);
```
#### 题目3：为什么set元素值不抛异常
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> it = list.iterator();

list.set(0, "X");  // ✅ 不抛异常！set 不算结构性修改
// 因为 set 不改变集合大小，modCount 不变

while (it.hasNext()) {
    System.out.println(it.next());  // X B C —— 正常遍历
}
```
#### 题目4：modCount溢出问题
```
// modCount 是 int，可能溢出
// 溢出后 modCount 可能等于 expectedModCount，检查失效
// 但实际几乎不可能（需要修改 21 亿次）
```
