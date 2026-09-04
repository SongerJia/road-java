## 阻塞队列
阻塞队列概念->核心API->7种实现预览->逐个详解->应用场景->选型结论
### 阻塞队列概念
什么是阻塞队列？
```
// BlockingQueue 是 JUC 包下的接口
// 特点：线程安全，支持阻塞的入队和出队

// 当队列满时，put() 会阻塞，直到有空间
// 当队列空时，take() 会阻塞，直到有元素
```
### 核心API
```java
public interface Queue<E> extends Collection<E> {
	boolean add(E e) //入队    抛异常是不能被添加时，如果是正常也是返回true,其他不正常还是会抛异常
	boolean offer(E e) //入队  返回特殊值布尔，其他不正常还是会抛异常
	E remove() //出队队首        如果队列是空的抛异常
	E poll()  //出队队首       如果队列为空返回 null
	E element() //查看队首      如果队列是空的抛异常
	E peek() //查看队首         如果队列为空返回 null
}
public interface BlockingQueue<E> extends Queue<E> {
    //增加的方法
    //入队
    void put(E e) throws InterruptedException;  // 满时阻塞
    boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException; // 限时阻塞

    // 出队
    E take() throws InterruptedException;  // 空时阻塞
    E poll(long timeout, TimeUnit unit) throws InterruptedException;  // 限时阻塞

    // 检查
    int remainingCapacity();  // 剩余容量
    boolean contains(Object o);
}
```
### 7种实现总览

| 实现                    | 底层                      | 是否有界          | 锁机制                                            | 特点                                                    |
| --------------------- | ----------------------- | ------------- | ---------------------------------------------- | ----------------------------------------------------- |
| ArrayBlockingQueue    | 数组                      | 有界            | 1把ReentrantLock+2个Condition(NotEmpty/notFull)  | 1.公平锁非公平锁可选 2.生产者消费者共用一把锁 3.内存紧凑，GC友好                 |
| LinkedBlockingQueue   | 单向链表                    | 可选有界，默认int最大值 | 2把ReentrantLock（takeLock/putLock）+各自的Condition | 1.生产者和消费者锁分开，吞吐量高 2.默认无界 3.size()要遍历 O(n)             |
| PriorityBlockingQueue | 数组实现的小顶堆                | 无界            | 1把ReentrantLock + 1个Condition（notEmpty）        | 1.元素必须实现Comparable或者提供Comparator 2.只保证出队优先级，同优先级顺序不保证 |
| DelayQueue            | 优先级堆(内部持有PriorityQueue) | 无界            | 1把ReentrantLock + 1个Condition（available）       | 1.元素实现Delayed接口 2.take时只有过期元素才能出队                     |
| SychronousQueue       | 无实际存储                   | 无容量           | CAS+自旋+LockSupport                             | 1.生产者必须等消费者来取，反之亦然 2.公平模式用队列，非公平用栈                    |
| LinkTransferQueue     | 链表                      | 无界            | CAS+自旋+LockSupport                             | 1.元素实现TransferQueue接口  2.transfer(e)必须有消费者等才放         |
| LinkBlockingDueque    | 双向链表                    | 可选有界，默认int最大值 | 1把ReentrantLock + 2个Condition                  | 1.双端操作    2.只有一把锁，双端操作互斥                              |
### 逐个详解
#### ArrayBlockingQueue
```java
// 底层：数组（循环队列）  
// 有界：必须指定容量
// 锁：一把锁（put 和 take 共用同一把锁）
// 公平：构造时可指定是否公平

ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(10);     // 非公平（默认）
ArrayBlockingQueue<String> queue2 = new ArrayBlockingQueue<>(10, true);  // 公平

// 源码关键
public class ArrayBlockingQueue<E> {
    final Object[] items;      // 数组   固定数组循环使用
    int takeIndex;             // 出队下标    达到数组长度重置为0
    int putIndex;              // 入队下标    达到数组长度重置为0
    int count;                 // 元素个数

    final ReentrantLock lock;  // 共用一把锁
    private final Condition notEmpty;  // 出队条件
    private final Condition notFull;   // 入队条件
}

// put 流程
public void put(E e) throws InterruptedException {
    lock.lockInterruptibly();     //加锁，可中断，如果线程在等锁时被interrupt(),直接抛异常退出=》阻塞队列的线程优雅退出
    try {
        while (count == items.length) //为什么while？被notFull().signal唤醒后，可能其他线程已经抢先put元素了，必须重新检查
            notFull.await();  // 满时阻塞  释放锁-》线程挂起-》等待被notFull().signal唤醒-》重新抢锁
        enqueue(e);           // 入队
    } finally {
        lock.unlock();
    }
}

private void enqueue(E e) {
	final Object[] items = this.items;
	items[putUndex] = e;  //放到当前位置
	if (++putIndex == items.length) putIndex = 0; //到末尾了，重置为0
	count ++;
	notEmpty.signal();  //唤醒等待notEmpty上的线程
}

// take 流程
public E take() throws InterruptedException {
    lock.lockInterruptibly();//加锁，可中断，如果线程在等锁时被interrupt(),直接抛异常退出=》阻塞队列的线程优雅退出
    try {
        while (count == 0)
            notEmpty.await();  // 空时阻塞
        return dequeue();      // 出队
    } finally {
        lock.unlock();
    }
}

private E dequeue(){
	final Object[] items = this.items;
	E e = (E) items[takeIndex]; //取出 e
	items[takeIndex]= null;  //置 null  帮助GC
	if (++ takeIndex == items.length) takeIndex = 0  //到末尾了，重置为0
	count --;
	if (itrs != null) {
	itrs.elementDequeued(); //迭代器支持，如果迭代器在遍历，通知它们元素被删除了
	}
	notFull.signal();  //唤醒等待notFull上的线程
	return e;
```
两个Condition的协作关系
```
put 线程                     take 线程
─────────                   ─────────
lock.lock()                 lock.lock()
count == capacity?           count == 0?
  notFull.await()              notEmpty.await()
      ↑                            ↑
      └──── notFull.signal() ───────┘  (dequeue 后腾出空位)
      └──── notEmpty.signal() ────────┘  (enqueue 后有数据)
```
问题1：为什么不用notify()和wait()
Synchronized+notify()/wait() //只能有一个等待队列，take和put混合在同一个等待队列，不能准确唤醒想要的队列
lock+Condition //可以有多个等待队列，把等空位和等数据分成两个队列，精准唤醒
问题2：为什一把锁就够用？
因为put()和take()都需要修改count，而且putIndex和takeIndex逻辑上依赖count和数组长度判断，共享状态必须互斥。
#### LinkedBlockingQueue
```java
// 底层：链表
// 有界：默认 Integer.MAX_VALUE（≈无界），可指定容量
// 锁：两把锁（put 锁和 take 锁分离）

LinkedBlockingQueue<String> queue = new LinkedBlockingQueue<>();        // 无界
LinkedBlockingQueue<String> queue2 = new LinkedBlockingQueue<>(1000);   // 有界

// 源码关键
public class LinkedBlockingQueue<E> {
	static class Node<E> {
	    E item;
	    Node<E> next;
	    Node(E x) { item = x; }
	}
    private final int capacity;  // 容量  默认Integer.MAX_VALUE
    private final AtomicInteger count = new AtomicInteger();  // 原子元素计数，put/take分别拿锁
    transient Node<E> head;                      // 哨兵头（head.item == null）
	private transient Node<E> last;              // 尾节点
	//没有putIndex/takeIndex,因为是链表，靠last尾插、head.next取

    // 两把锁分离 —— 提高并发
    private final ReentrantLock putLock = new ReentrantLock();    // 入队锁
    private final Condition notFull = putLock.newCondition();

    private final ReentrantLock takeLock = new ReentrantLock();   // 出队锁
    private final Condition notEmpty = takeLock.newCondition();
}

//put流程  生产者流程
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;

    putLock.lockInterruptibly();             // ① 只拿 putLock
    try {
        while (count.get() == capacity)      // ② 满则等 notFull
            notFull.await();                 // await 会释放 putLock

        enqueue(node);                       // ③ 尾插
        c = count.getAndIncrement();         // ④ 原子 +1，c 是旧值

        if (c + 1 < capacity)                // ⑤ 还没满，唤醒其它 put 线程
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)                              // ⑥ 原来队列是空的
        signalNotEmpty();                    //   跨锁唤醒 take 线程 只有原本是空是才唤醒
}

private void enqueue(Node<E> node) {
    last = last.next = node;     // 单链表尾插，O(1)
}

private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();             // 跨锁：拿 takeLock 才能用 notEmpty
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}

//take流程  消费者流程
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;

    takeLock.lockInterruptibly();            // ① 只拿 takeLock
    try {
        while (count.get() == 0)             // ② 空则等 notEmpty
            notEmpty.await();

        x = dequeue();                       // ③ 头取（head 是哨兵）
        c = count.getAndDecrement();         // ④ 原子 -1，c 是旧值

        if (c > 1)                           // ⑤ 取完还有 >=1 个元素
            notEmpty.signal();               //   唤醒其它 take 线程
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)                       // ⑥ 原来队列是满的
        signalNotFull();                     //   跨锁唤醒 put 线程  只有 原本是满时才唤醒
    return x;
}

private E dequeue() {
    Node<E> h = head;        // 哨兵
    Node<E> first = h.next;  // 真正第一个元素
    h.next = h;              // 自指，help GC
    head = first;            // 新哨兵
    E x = first.item;
    first.item = null;       // 断引用
    return x;
}

private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();          // 跨锁：拿 putLock 才能用 notFull
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```
#### PriorityBlockingQueue
优先级无界
```java
// 底层：堆（二叉堆）
// 无界：容量可自动扩容
// 排序：按优先级出队（自然顺序或 Comparator）

PriorityBlockingQueue<Task> queue = new PriorityBlockingQueue<>();

// 元素必须实现 Comparable 或传入 Comparator
class Task implements Comparable<Task> {
    private int priority;

    @Override
    public int compareTo(Task o) {
        return Integer.compare(this.priority, o.priority);  // 优先级高的先出
    }
}

// put 永远不阻塞（因为无界）
queue.put(new Task(1));  // 永远不会阻塞

// take 在队列为空时阻塞
Task task = queue.take();  // 空时阻塞

// 源码关键
public class PriorityBlockingQueue<E> {
    private transient Object[] queue;  // 数组实现的小顶堆，逻辑上时一棵完全二叉树（父子靠下标计算，不用指针）
    private transient int size;        //当前实际元素个数
	private transient Comparator<? super E> comparator; // 比较器（可空） 空使用元素的比较规则
    private final ReentrantLock lock;       // 一把锁  全局独占锁：所有读写操作都靠它互斥
    private final Condition notEmpty;       // 出队条件   因为无界，所以不会满不用notFull

    // 扩容时用 CAS 控制，避免锁竞争 扩容时的自旋锁标记，用 UNSAFE CAS 抢，保证同一时刻只有一个线程在分配新数组
    private transient volatile int allocationSpinLock;
}
//offer入队
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();   // 不允许 null，堆里 null 会和空槽混淆
    final ReentrantLock lock = this.lock;
    lock.lock();                           // ① 拿全局锁
    int n, cap;
    Object[] array;
    // 元素数 >= 数组长，说明满了，先扩容（内部会临时释放主锁）
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftUpComparable(n, e, array);     // ② 放数组末尾 n 位置，向上冒
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;                         // ③ 元素数 +1
        notEmpty.signal();                    // ④ 唤醒可能等在 take() 的消费者
    } finally {
        lock.unlock();                        // ⑤ 解锁
    }
    return true;                              // 无界，永远成功
}

//siftUpComparable 插入到数组末尾，不断上浮（将e插到末尾，然后不断和父比较，比父小就换）
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;          // 父节点下标，无符号右移等价于 (k-1)/2
        Object e = array[parent];
        if (key.compareTo((T) e) >= 0)       // 比父大或相等，堆性质已满足
            break;
        array[k] = e;                        // 父节点下沉到 k
        k = parent;                          // 继续往上比
    }
    array[k] = key;                          // 找到最终位置
}

// put 直接委托 offer，注释原文：never need to block
public void put(E e) { offer(e); }

//take 出队 空则阻塞
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();                // 可中断拿锁
    E result;
    try {
        // 堆空时 dequeue() 返回 null，就挂起等 notEmpty
        while ((result = dequeue()) == null)
            notEmpty.await();                // 释放锁 + 挂起，被 signal 后重新抢锁
    } finally {
        lock.unlock();
    }
    return result;                           // 永远是堆顶最小元素
}

//dequeue  取堆顶
private E dequeue() {
    int n = size - 1;
    if (n < 0)
        return null;                         // 空队列
    else {
        Object[] array = queue;
        E result = (E) array[0];             // 堆顶 = 优先级最高
        E x = (E) array[n];                  // 数组最后一个元素
        array[n] = null;                     // 置 null 帮 GC
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftDownComparable(0, x, array, n);  // 用末尾元素填堆顶，向下沉
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;                            // 元素数 -1
        return result;
    }
}
//siftDownComparable   把x放到堆顶，和左右子节点比较，比子大就换
private static <T> void siftDownComparable(int k, T x, Object[] array, int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>) x;
        int half = n >>> 1;                   // 最后一个非叶子节点下标，之后全是叶子不用沉
        while (k < half) {
            int child = (k << 1) + 1;        // 左子 2k+1
            Object c = array[child];
            int right = child + 1;           // 右子 2k+2
            // 如果右子存在且比左子小，取右子作为"较小子节点"
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            if (key.compareTo((T) c) <= 0)   // 比小子都小，位置对了
                break;
            array[k] = c;                     // 较小子节点上浮
            k = child;                        // 继续往下比
        }
        array[k] = key;
    }
}

//tryGrow 扩容  先放主锁，用CAS自旋锁分配
private void tryGrow(Object[] array, int oldCap) {
    lock.unlock(); // 必须先放主锁，否则扩容时别的线程完全进不来，并发度塌掉
    Object[] newArray = null;
    // CAS 抢 allocationSpinLock：同一时刻只有一个线程真正分配新数组
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset, 0, 1)) {
        try {
            // <64：新容量 = oldCap*2 + 2；>=64：oldCap*1.5
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) :
                                   (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) { // 防溢出
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;          // 释放自旋锁
        }
    }
    if (newArray == null) // 没抢到 CAS 的线程让出 CPU，等别人扩完
        Thread.yield();
    lock.lock();                             // 重新拿主锁，准备把老数据拷贝过去
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```
堆的父子关系
```
//完全二叉树：节点紧凑无空洞，最后一层可不满，但从左到右连续排列，支持数组存储
// 父节点 i: 左子 2i+1, 右子 2i+2
// 子节点 i: 父 (i-1)/2
private static int parent(int n) { return (n - 1) >>> 1; }
数组[1, 3, 2, 5, 9] 逻辑上是：
		1
       / \
      3   2
     / \
    5   9
```

