## 优先级队列
堆的概念->底层结构->入队流程->出队流程->扩容机制->应用场景->面试高频题
### 堆的概念
什么是堆？
堆：一种特殊形式的完全二叉树，满足堆序性质---父节点的值不大于（不小于）子节点的值
```
// 小顶堆（min-heap）：父节点 ≤ 子节点，堆顶是最小值
// 大顶堆（max-heap）：父节点 ≥ 子节点，堆顶是最大值

//不是排序好的树，只保证局部有序，兄弟之间无序
// PriorityQueue 默认是小顶堆，通过 Comparator 可以变成大顶堆
```
小顶堆结构
```
                1 ← 堆顶（最小值）
              /   \
             3     5
            / \   / \
           7   8 6   9
          /
         10

数组存储（层序遍历）：
[1, 3, 5, 7, 8, 6, 9, 10]
 ↑
 下标 0 = 堆顶

```
 父子下标关系
```
// 用数组存储完全二叉树
// 下标从 0 开始：
// 父节点下标 = (i - 1) / 2
// 左子节点下标 = i * 2 + 1
// 右子节点下标 = i * 2 + 2

// 例如：
// 数组：[1, 3, 5, 7, 8, 6, 9, 10]
// 下标： 0  1  2  3  4  5  6  7

// 下标 1（值 3）的：
// 父节点 = (1-1)/2 = 0（值 1）✅
// 左子 = 1*2+1 = 3（值 7）✅
// 右子 = 1*2+2 = 4（值 8）✅
```
### 底层结构
```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {

    // 底层就是 Object 数组（堆）
    transient Object[] queue;

    // 元素个数
    int size;

    // 比较器（null 表示使用元素的自然顺序）
    private final Comparator<? super E> comparator;

    // 构造器
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);  // 默认容量 11
    }

    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }
}

// 默认初始容量 11
private static final int DEFAULT_INITIAL_CAPACITY = 11;
```
使用示例
```java
// 自然顺序（小顶堆）—— 最小的先出
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(5);
pq.offer(1);
pq.offer(3);
pq.offer(2);

pq.poll();  // 1（最小）
pq.poll();  // 2
pq.poll();  // 3
pq.poll();  // 5

// 大顶堆 —— 通过 Comparator 反转
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5);
maxHeap.offer(1);
maxHeap.offer(3);

maxHeap.poll();  // 5（最大）
maxHeap.poll();  // 3
```
### 入队流程 offer/add
siftUp 上浮操作
```java
// 入队：新元素加到数组尾部，然后"上浮"到正确位置
public boolean offer(E e) {
    // 不允许 null
    if (e == null) throw new NullPointerException();
    modCount++;

    int i = size;
    // 容量不够 → 扩容
    if (i >= queue.length)
        grow(i + 1);

    size = i + 1;

    // 第一个元素，直接放堆顶
    if (i == 0)
        queue[0] = e;
    else
        // 上浮调整
        siftUp(i, e);
    return true;
}

// 上浮（小顶堆）
private void siftUp(int k, E x) {
    while (k > 0) {
        // 父节点下标
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];

        // 如果 x 不小于父节点，停止上浮
        if (key.compareTo((E) e) >= 0)
            break;

        // 父节点下沉，x 上浮
        queue[k] = e;
        k = parent;
    }
    queue[k] = x;  // 放入最终位置
}
```
入队演示
```
// 向小顶堆插入 2：[5, 8] → [2, 8, 5]
// 数组：[5, 8]（size=2）
// 新元素 2 放到下标 2

// 第 1 步：k=2，父节点=(2-1)/2=0（值 5）
// 5 > 2 → 5 下沉到下标 2，2 准备上浮到下标 0
// 数组：[5, 8, 5] → k=0

// 第 2 步：k=0，到达堆顶，停止
// 数组：[2, 8, 5] ✅

// 复杂度：O(log n) —— 最多上浮到堆顶
```
### 出队操作 poll/remove
siftDowm 下沉操作
```java
// 出队：取堆顶元素，把最后一个元素移到堆顶，然后"下沉"到正确位置
public E poll() {
    if (size == 0)
        return null;

    int s = --size;
    modCount++;
    Object[] array = queue;
    E result = (E) array[0];  // 堆顶（最小/最大）

    E x = (E) array[s];       // 最后一个元素
    array[s] = null;          // 清空

    if (s != 0)
        siftDown(0, x);       // 从堆顶开始下沉
    return result;
}

// 下沉（小顶堆）
private void siftDown(int k, E x) {
    int half = size >>> 1;  // 只有前一半节点有子节点

    while (k < half) {
        // 左子节点
        int child = (k << 1) + 1;
        Object c = queue[child];
        int right = child + 1;

        // 如果右子节点更小，选右子节点
        if (right < size && ((Comparable) c).compareTo(queue[right]) > 0)
            c = queue[child = right];

        // 如果 x 不大于最小子节点，停止下沉
        if (key.compareTo((E) c) <= 0)
            break;

        // 子节点上浮，x 下沉
        queue[k] = c;
        k = child;
    }
    queue[k] = x;
}
```
出队演示
```
// 从 [1, 3, 5, 7, 8] 中取出最小值 1
// 数组：[1, 3, 5, 7, 8]（size=5）

// 第 1 步：取堆顶 1
// 把最后一个元素 8 移到堆顶
// 数组：[8, 3, 5, 7, null]

// 第 2 步：8 从堆顶开始下沉
// 比较子节点 3 和 5，选小的 3
// 8 > 3 → 3 上浮，8 下沉到下标 1
// 数组：[3, 8, 5, 7, null]

// 第 3 步：继续下沉，子节点 7
// 8 > 7 → 7 上浮，8 下沉到下标 3
// 数组：[3, 7, 5, 8, null]

// 结果：[3, 7, 5, 8] ✅ 堆顶是新的最小值 3

// 复杂度：O(log n)
```
### 扩容机制
```java
// PriorityQueue 的扩容
private void grow(int minCapacity) {
    int oldCapacity = queue.length;

    // 扩容规则：
    // 容量 < 64：翻倍 + 2
    // 容量 >= 64：扩容 1.5 倍
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :   // 小容量：2 倍 + 2
                                     (oldCapacity >> 1));  // 大容量：1.5 倍

    // 溢出保护
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        newCapacity = hugeCapacity(minCapacity);
    }

    queue = Arrays.copyOf(queue, newCapacity);
}
```
### peiorityQueue的特性与限制
特性
```
// ① 线程不安全（需要线程安全用 PriorityBlockingQueue）
PriorityQueue<Integer> pq = new PriorityQueue<>();  // 单线程用
PriorityBlockingQueue<Integer> pbq = new PriorityBlockingQueue<>();  // 多线程用

// ② 不支持 null
pq.offer(null);  // ❌ NullPointerException   需要做比较

// ③ 元素必须可比较（实现 Comparable 或提供 Comparator）
// ④ 不是"全局有序"！只是保证堆顶是最小值
// ⑤ 遍历顺序 ≠ 排序顺序（只有 poll 才保证顺序）
```
重要坑：遍历不排序
```
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(5);
pq.offer(1);
pq.offer(3);
pq.offer(2);
pq.offer(4);

// 遍历 —— 不是有序的！
System.out.println(pq);  // [1, 2, 3, 5, 4] —— 堆结构，不是完全排序

// 只有 poll 才保证从小到大的顺序     
while (!pq.isEmpty()) {
    System.out.print(pq.poll() + " ");  // 1 2 3 4 5 ✅
}
```
自定义对象排序
```java
// 任务按优先级排序
class Task {
    String name;
    int priority;

    public Task(String name, int priority) {
        this.name = name;
        this.priority = priority;
    }
}

// 方式一：实现 Comparable
class Task implements Comparable<Task> {
    // ...

    @Override
    public int compareTo(Task o) {
        return Integer.compare(this.priority, o.priority);  // 优先级小的先出
    }
}

// 方式二：传入 Comparator
PriorityQueue<Task> queue = new PriorityQueue<>((t1, t2) ->
    Integer.compare(t1.priority, t2.priority));
```
### 应用场景
#### 场景1：Top K问题
```
// 求数组中的前 K 个最大元素
public static int[] topK(int[] nums, int k) {
    // 小顶堆（容量 K）—— 堆顶是 K 个元素中最小的
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);

    for (int num : nums) {
        if (minHeap.size() < k) {
            minHeap.offer(num);  // 堆未满，直接加入
        } else if (num > minHeap.peek()) {
            minHeap.poll();      // 新元素比堆顶大，淘汰堆顶
            minHeap.offer(num);
        }
    }

    return minHeap.stream().mapToInt(Integer::intValue).toArray();
}

// 时间复杂度：O(n log k)，空间 O(k)
// 比排序 O(n log n) 更优
```
场景2：合并K个有序链表
```java
public ListNode mergeKLists(ListNode[] lists) {
    // 小顶堆，按节点值排序
    PriorityQueue<ListNode> heap = new PriorityQueue<>((a, b) -> a.val - b.val);

    // 所有头节点入堆
    for (ListNode node : lists) {
        if (node != null) heap.offer(node);
    }

    ListNode dummy = new ListNode(0);
    ListNode cur = dummy;

    while (!heap.isEmpty()) {
        ListNode node = heap.poll();  // 取最小节点
        cur.next = node;
        cur = cur.next;
        if (node.next != null) {
            heap.offer(node.next);    // 下一个节点入堆
        }
    }
    return dummy.next;
}
```
### 差异化结论

| 对比    | LinkedList | PriorityQueue |
| ----- | ---------- | ------------- |
| 顺序    | FIFO 先进先出  | 优先级  最小/最大    |
| 底层    | 双向链表       | 二叉堆  数组小顶堆    |
| offer | O(1)       | O(logn)       |
| poll  | O(1)       | O(logn)       |
| peek  | O(1)       | O(1)  直接取堆顶   |
| null  | 可以         | 不能            |
| 线程安全  | 不安全        | 不安全           |
