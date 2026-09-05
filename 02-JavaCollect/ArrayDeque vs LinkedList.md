## ArrayDeque vs LinkedList
Deque概念->底层结构-核心操作对比->源码分析->作用为栈/队列使用->差异化结论->选型场景结论
### Deque概念
什么是Deque
Double-Ended Queue 双端队列：两端可以插入和删除的队列
```java
// 接口定义
public interface Deque<E> extends Queue<E> {
    // 两端插入
    void addFirst(E e);   // 头部插入（失败抛异常）
    void addLast(E e);    // 尾部插入（失败抛异常）
    boolean offerFirst(E e);  // 头部插入（失败返回 false）
    boolean offerLast(E e);   // 尾部插入（失败返回 false）

    // 两端删除
    E removeFirst();      // 头部删除（空抛异常）
    E removeLast();       // 尾部删除（空抛异常）
    E pollFirst();        // 头部删除（空返回 null）
    E pollLast();         // 尾部删除（空返回 null）

    // 两端查看
    E getFirst();         // 查看头部（空抛异常）
    E getLast();          // 查看尾部（空抛异常）
    E peekFirst();        // 查看头部（空返回 null）
    E peekLast();         // 查看尾部（空返回 null）

    // 栈操作
    void push(E e);       // 入栈（头部）
    E pop();              // 出栈（头部）
}

// 两个主要实现：
Deque<String> deque1 = new ArrayDeque<>();   // 数组实现
Deque<String> deque2 = new LinkedList<>();   // 链表实现
```
为什么需要Deque?
```
// Java 早期：
// Stack（栈）—— 继承 Vector，性能差，不推荐
// LinkedList（队列）—— 可以当队列，但性能一般

// Java 6 引入 Deque 接口，推荐用 ArrayDeque 替代
// Stack → ArrayDeque（栈）
// LinkedList（队列）→ ArrayDeque（队列）
```
### 底层结构
ArrayDeque：循环数组
```java
public class ArrayDeque<E> extends AbstractCollection<E> implements Deque<E> {
    // 底层：循环数组
    transient Object[] elements;

    // 头指针（指向头部元素）
    transient int head;

    // 尾指针（指向尾部元素的下一个位置）
    transient int tail;

    // 默认容量 16
    // 容量始终是 2 的幂次方！
}

// 结构图（循环数组）：
// 数组：[_, _, A, B, C, _, _]
//             ↑         ↑
//           head      tail
//
// 当 tail 到达数组末尾时，循环到开头：
// 数组：[D, _, A, B, C, _, _]
//             ↑  ↑
//           tail head（循环回绕）
```
LinkedList：双向链表
```
public class LinkedList<E> extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, Serializable {

    // 头节点
    transient Node<E> first;
    // 尾节点
    transient Node<E> last;

    // 节点：数据 + 前后指针
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
    }
}
```
核心区别

| 对比  | ArrayDeque | LinkedList  |
| --- | ---------- | ----------- |
| 底层  | 循环数组       | 双向链表        |
| 容量  | 固定 可扩容     | 无限          |
| 内存  | 连续 紧凑      | 分散，每个节点2个指针 |
| 扩容  | 数组扩容 翻倍    | 不需要         |
### 核心操作对比
时间复杂度

|操作|ArrayDeque|LinkedList|
|---|---|---|
|**头部插入 addFirst**|O(1) ✅|O(1) ✅|
|**头部删除 pollFirst**|O(1) ✅|O(1) ✅|
|**尾部插入 addLast**|O(1) ✅|O(1) ✅|
|**尾部删除 pollLast**|O(1) ✅|O(1) ✅|
|**中间访问 get(i)**|O(n)|O(n)|
|**查找 contains**|O(n)|O(n)|
看起来都是O(1) 但实际性能差别很大（缓存局限性）
```
// ArrayDeque 的元素在内存中连续存储
// CPU 缓存命中率高，实际操作非常快

// LinkedList 的元素分散存储
// 每次操作都要通过指针跳转，CPU 缓存命中率低
// 实际速度慢 3~5 倍
```
实测对比
```
// 头部插入 100 万次
// ArrayDeque：约 20ms
// LinkedList：约 80ms（慢 4 倍，指针跳转 + 缓存不命中）

// 尾部插入 100 万次
// ArrayDeque：约 15ms
// LinkedList：约 70ms

// 结论：同是 O(1)，ArrayDeque 的常数小得多
```
### 源码对比
ArrayDeque的扩容
```
// ArrayDeque 扩容：容量翻倍，始终是 2 的幂
// 为什么是 2 的幂？为了用位运算计算下标

private void doubleCapacity() {
    // 当前容量
    int n = elements.length;
    // 头部元素个数
    int r = n - head;

    // 新容量 = 2 倍
    int newCapacity = n << 1;

    // 检查溢出
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");

    // 复制到新数组
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, head, a, 0, r);        // 头部及之后
    System.arraycopy(elements, 0, a, r, head);        // 0 到 head
    elements = a;
    head = 0;
    tail = n;
}
```
位运算符下标
```
// ArrayDeque 用位运算（容量是 2 的幂）
// 头部添加：head = (head - 1) & (elements.length - 1)
// 尾部添加：tail = (tail + 1) & (elements.length - 1)

// 例如：容量 16
// (15 - 1) & 15 = 14  → head 回绕到 14
// (15 + 1) & 15 = 0   → tail 回绕到 0

// 位运算 & 代替 % 取模，性能极高
```
ArrayDeque不允许为null
```
// ArrayDeque 不允许 null 元素    
//1.API语义冲突，获取到的值返回的null是存的null,还是真的不存在
//2.循环数组底层拿null当空槽标记
ArrayDeque<String> deque = new ArrayDeque<>();
deque.addFirst(null);  // ❌ NullPointerException

// LinkedList 允许 null
//是用节点对象Node 包装了item值，item为null,节点本身存在，前后指针完整。
LinkedList<String> list = new LinkedList<>();
list.addFirst(null);  // ✅ 允许
```
### 作为栈/队列使用
做为栈 LIFO
```java
// ✅ 推荐：ArrayDeque 作为栈
Deque<String> stack = new ArrayDeque<>();
stack.push("A");  // 入栈
stack.push("B");
stack.push("C");
stack.pop();      // "C" 出栈

// ❌ 不推荐：Stack 类（继承 Vector，性能差）
Stack<String> oldStack = new Stack<>();
oldStack.push("A");
oldStack.pop();
// Stack 所有方法都加 synchronized，单线程下性能浪费

// 对比：Stack vs ArrayDeque
// Stack：继承 Vector，全部加锁，性能差
// ArrayDeque：无锁，位运算，性能高 10 倍+
```
作为队列 FIFO
```java
// ✅ 推荐：ArrayDeque 作为队列
Queue<String> queue = new ArrayDeque<>();
queue.offer("A");
queue.offer("B");
queue.poll();  // "A"

// LinkedList 也可以当队列，但性能差一些
Queue<String> queue2 = new LinkedList<>();

// 对比：ArrayDeque vs LinkedList 作为队列
// ArrayDeque：循环数组 + 位运算，更快
// LinkedList：链表 + 指针跳转，更慢
```
双向操作
```java
Deque<String> deque = new ArrayDeque<>();

// 两端插入
deque.addFirst("A");   // [A]
deque.addLast("B");    // [A, B]
deque.addFirst("C");   // [C, A, B]

// 两端删除
deque.pollFirst();     // "C" → [A, B]
deque.pollLast();      // "B" → [A]

// 两端查看
deque.peekFirst();     // "A"
deque.peekLast();      // "A"
```
### 差异化结论
|维度|ArrayDeque|LinkedList|
|---|---|---|
|**底层**|循环数组|双向链表|
|**性能**|高（位运算+缓存命中）|低（指针跳转+缓存不命中）|
|**null 支持**|❌|✅|
|**索引访问**|❌（只能遍历）|✅（get(i) 但 O(n)）|
|**内存**|连续紧凑|分散，每个节点多 2 指针|
|**实现接口**|Deque|List + Deque|
|**扩容**|翻倍扩容|不需要|
|**推荐程度**|✅ 首选|⚠️ 特殊场景|
### 选型场景结论
```
面试官："ArrayDeque 和 LinkedList 的区别？怎么选？"

"两者都实现了 Deque 接口，都能当栈和队列用，
但底层实现不同：

① ArrayDeque：循环数组
   - 连续内存，CPU 缓存命中率高
   - 位运算计算下标，性能极高
   - 不允许 null
   - 推荐首选：作为栈和队列都比 LinkedList 快

② LinkedList：双向链表
   - 元素分散，指针跳转慢
   - 允许 null
   - 额外实现了 List 接口，可以索引访问
   - 需要同时用 List 功能时才用它

③ 和 Stack 的对比：
   官方也推荐用 ArrayDeque 替代 Stack
   Stack 继承 Vector 全部加锁，单线程下性能浪费

所以我的选择：栈和队列都用 ArrayDeque，
只有需要 List 接口时才用 LinkedList。"
```

