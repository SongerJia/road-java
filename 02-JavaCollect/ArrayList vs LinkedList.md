## ArrayList vs LinkedList
底层结构->时间复杂度对比->扩容机制->内存占用->RandomAccess->功能对比->差异化结论->选型场景结论
### 底层结构
```java
// ArrayList —— 动态数组
public class ArrayList<E> {
    // 底层就是一个 Object 数组
    transient Object[] elementData;
    private int size;
}

// LinkedList —— 双向链表   节点有前后，但不是环形
public class LinkedList<E> {
    // 底层是节点，每个节点有前后指针
    transient Node<E> first;
    transient Node<E> last;

    private static class Node<E> {
        E item;          // 数据
        Node<E> next;    // 后驱指针
        Node<E> prev;    // 前驱指针
    }
}
```
结构图：
```
ArrayList（数组）：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │    │    │
│ A  │ B  │ C  │ D  │ E  │ F  │    │    │
└────┴────┴────┴────┴────┴────┴────┴────┘
  ↑                        ↑              ↑
first                   last         预留空间

LinkedList（双向链表）：
┌─────────┐    ┌────────┐    ┌─────────┐    ┌─────────┐
│prev=null│→   │prev    │→   │prev     │→   │prev     │
│ item=A  │    │ item=B │    │ item=C  │    │ item=D  │
│  next   │←   │  next  │←   │  next   │←   │next=null│
└─────────┘    └────────┘    └─────────┘    └─────────┘
  ↑                                              ↑
 first                                          last
```
 
 ### 时间复杂度比较
 
| 操作               | ArrayList | LinkedList |
| ---------------- | --------- | ---------- |
| 尾部添加 add(e)      | O(1)      | O(1)       |
| 指定位置插入add(i,e)   | O(n)      | O(n)       |
| 头部插入add(0,e)     | O(n)      | O(1)       |
| 尾部删除remove(last) | O(1)      | O(1)       |
| 头部删除remove(0)    | O(n)      | O(1)       |
| 中间删除remove(i)    | O(n)      | O(n)       |
| 随机访问get(i)       | O(1)      | O(n)       |
| 查找contains(e)    | O(n)      | O(n)       |
> **Q:** "LinkedList 中间插入真的比 ArrayList 快吗？" 
> **A:** "不一定。虽然 LinkedList 的插入本身是 O(1)，但需要先 O(n) 遍历找到插入位置。如果插入位置在中间，两者都是 O(n)。只有插入位置在头部时，LinkedList 才有优势（O(1) vs O(n)）。"

> **Q:** "那什么情况下 LinkedList 比 ArrayList 快？" 
> **A:** "
> 1. 频繁在**头部**插入/删除
> 2. 用 LinkedList 作为**队列**（FIFO）或**双端队列**
> 3. 除此之外，大部分场景 ArrayList 更快 "
### 扩容机制
ArrayList扩容机制
```java
// ArrayList 默认容量 10
// 扩容：1.5 倍（old + old >> 1）
// 扩容时：创建新数组 → System.arraycopy() 拷贝

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 数组拷贝 —— O(n) 操作
    elementData = Arrays.copyOf(elementData, newCapacity);
}

// 扩容次数对性能的影响
// 16 → 24 → 36 → 54 → 81 → 121 → 181 → ... → 约 15 次扩容到 10 万
// 如果已知大小，建议指定初始容量：new ArrayList<>(10000)
```
LinkedList不需要扩容
```
// LinkedList 没有容量限制
// 每次添加只新建一个 Node 节点，链接到链表尾部
// 没有数组拷贝开销

// 但代价：每个节点多占内存（前后指针）
```
扩容对比

| 维度      | ArrayList | LinkedList |
| ------- | --------- | ---------- |
| 是否有容量限制 | 有         | 无          |
| 扩容方式    | 1.5倍数组拷贝  | 新建节点，链接    |
| 扩容时阻塞   | 需要O(n)拷贝  | 不需要        |
| 预分配     | 可指定初始容量   | 不需要        |
### 内存占用
```
// ArrayList —— 紧凑
// 只存数据本身 + 少量预留空间
// 元素 = 数据本身

// LinkedList —— 膨胀
// 每个节点 = 数据 + 前驱指针 + 后驱指针
// 元素 = 数据 + 2 个引用（16 字节额外开销）
```
内存对比
```
// 存储 100 万个 int 值
// ArrayList<Integer>：约 4MB（数组本身）+ 对象头 ≈ 4~5MB
// LinkedList<Integer>：约 20MB+（每个节点多 2 个指针）

// 结论：LinkedList 内存占用是 ArrayList 的 4~5 倍
```
空间局部性
```
// ArrayList 的数据在内存中连续存储
// 遍历时 CPU 缓存命中率高（空间局部性）

// LinkedList 的数据在内存中分散存储
// 遍历时频繁缓存未命中，实际遍历速度比 ArrayList 慢
// 虽然都是 O(n)，但 ArrayList 的常数小得多

// 实际测试：
// 遍历 100 万个元素
// ArrayList：约 5ms
// LinkedList：约 50ms（慢 10 倍，因为缓存不命中）
```
### RandomAccess标记接口
```java
// ArrayList 实现了 RandomAccess 接口
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, Serializable {
}

// LinkedList 没有实现
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, Serializable {
}

// RandomAccess 是标记接口（没有方法）
// 表示"支持高效的随机访问"
```
有什么作用？
```java
// 根据是否实现 RandomAccess 选择不同的遍历方式
public static <T> void print(List<T> list) {
    if (list instanceof RandomAccess) {
        // ArrayList：for 循环 + get(i) 更快
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    } else {
        // LinkedList：迭代器更快（get(i) 是 O(n)）
        for (T item : list) {
            System.out.println(item);
        }
    }
}

// 如果不区分，用 for 循环遍历 LinkedList
// 每次 get(i) 都要从头遍历 → O(n²)！
List<String> list = new LinkedList<>();
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));  // ❌ O(n²)！每次 get 都是 O(n)
}
```
### 功能对比
LinkedList额外实现里的Deque
```java
// LinkedList 实现了 Deque 接口，可以当队列/栈用
// ArrayList 只实现了 List

// LinkedList 作为队列（FIFO）
Queue<String> queue = new LinkedList<>();
queue.offer("A");
queue.offer("B");
String first = queue.poll();  // "A"

// LinkedList 作为栈（LIFO）
Deque<String> stack = new LinkedList<>();
stack.push("A");
stack.push("B");
String top = stack.pop();  // "B"

// 但注意：作为栈，ArrayDeque 比 LinkedList 更快
```
初始化方式
```java
// ArrayList 可以指定初始容量
List<String> list1 = new ArrayList<>(10000);  // ✅ 预分配

// LinkedList 不能指定容量
List<String> list2 = new LinkedList<>();  // 不需要容量
```
序列化
```
// ArrayList 的序列化更高效（只序列化有数据的部分）
// 因为 elementData 数组可能有空位，用 transient 修饰
// 自定义 writeObject() 只写 size 个元素

// LinkedList 的序列化就是序列化所有节点
```
### 差异化结论
```
ArrayList底层是动态数组，连续存储，实现了RandomAccess标记接口，支持随机访问，适合读多写少的场景
LinkList底层是双向链表，节点分散，实现了Dueque接口，头尾操作快，适合队列场景
```
选型场景结论
```
面试官："ArrayList 和 LinkedList 怎么选？"

"大部分场景用 ArrayList，原因有三：

① 实际项目中，90% 以上的操作是遍历和随机访问，
   ArrayList 的 O(1) 随机访问完胜 LinkedList 的 O(n)

② ArrayList 的内存更紧凑，遍历时 CPU 缓存命中率高，
   实际遍历速度比 LinkedList 快 10 倍左右

③ LinkedList 只有在频繁头部插入/删除时有优势，
   但这个场景可以用 ArrayDeque 替代，性能更好

所以我的默认选择是 ArrayList，
只有明确需要队列/栈语义时才考虑 LinkedList。"
```
