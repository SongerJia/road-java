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
| SychronousQueue       | 无实际存储                   | 无容量           | CAS+自旋+LockSupport （有ReentrantLock 是为了兜底）      | 1.生产者必须等消费者来取，反之亦然 2.公平模式用队列，非公平用栈                    |
| LinkTransferQueue     | 链表                      | 无界            | CAS+自旋+LockSupport                             | 1.元素实现TransferQueue接口  2.transfer(e)必须有消费者等才放         |
| LinkBlockingDeque     | 双向链表                    | 可选有界，默认int最大值 | 1把ReentrantLock + 2个Condition                  | 1.双端操作    2.只有一把锁，双端操作互斥                              |
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
// 父节点数组下标 i: 左子 2i+1, 右子 2i+2
// 子节点数组下标 i: 父 (i-1)/2
private static int parent(int n) { return (n - 1) >>> 1; }
数组[1, 3, 2, 5, 9] 逻辑上是：
		1
       / \
      3   2
     / \
    5   9
```
#### DelayQueue
延迟队列
```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
        implements BlockingQueue<E> {

    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<>();   // 注意：不是 PriorityBlockingQueue 外层已经有一把锁了
    private Thread leader = null;                               // Leader-Follower 模式核心
    //假设堆定还有5秒到期，如果没有leader，10个消费线程同时take后awatitNano(5s),5秒后一起醒，抢锁，只有一个拿到元素，其他消费线程发现堆顶变了又重新计算等待时间（惊群+重复计时）。有了leader,只有一个消费线程去等待过期时间，其他的无限等待，当这个线程获取到元素后，再唤醒其他一个等待消费线程，让他成为leader重新计算等待时长
    private final Condition available = lock.newCondition();
}
//元素必须实现 Delayed接口：按照剩余延迟排序，堆顶是最早到期的
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);   // 剩余延迟，<=0 表示已到期
}

// 示例：延迟任务
class DelayedTask implements Delayed {
    private String name;
    private long executeTime;  // 执行时间（毫秒）

    public DelayedTask(String name, long delay) {
        this.name = name;
        this.executeTime = System.currentTimeMillis() + delay;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(executeTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.executeTime, ((DelayedTask) o).executeTime);
    }
}

// 使用
DelayQueue<DelayedTask> queue = new DelayQueue<>();
queue.put(new DelayedTask("任务1", 5000));  // 5秒后执行
queue.put(new DelayedTask("任务2", 3000));  // 3秒后执行

// take 会阻塞，直到最早的到期元素可用
DelayedTask task = queue.take();  // 3秒后拿到"任务2"
task = queue.take();              // 5秒后拿到"任务1"

//put 入队
public void put(E e) { offer(e); }   // 无界，put 直接委托 offer 入队
//offer 入队
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        q.offer(e);                 // 塞进内部小顶堆，按 compareTo 上浮
        if (q.peek() == e) {       // 新元素成了堆顶（更早到期）：只有新元素比原堆顶元素更早到期，才需要打断leader的awaitNode(delay),否则不需要打断
            leader = null;          // 作废旧的 leader 计时
            available.signal();    // 唤醒一个等着的消费者重算等待时间
        }
        return true;
    } finally {
        lock.unlock();
    }
}

//take 出队
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null) {
                available.await();          // 1. 队列空，等 offer 来 signal
            } else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0) {
                    return q.poll();        // 2. 已到期，直接出队
                }
                first = null;               // 3. 避免持有堆顶引用导致 GC 受阻
                if (leader != null) {
                    available.await();      // 4. 已有 leader 在计时，当前线程当 follower 干等
                } else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;    // 5. 当前线程当 leader
                    try {
                        available.awaitNanos(delay);  // 6. 只等"剩余延迟"这么久
                    } finally {
                        if (leader == thisThread)
                            leader = null;  // 7. 醒来或被唤醒后让出 leader 身份
                    }
                }
            }
        }
    } finally {
        // 8. 离开方法前：如果没 leader 且还有元素，signal 一个 follower 接棒
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
//poll 出队
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E first = q.peek();
        // 堆空，或堆顶还没到期 → 直接 null
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return q.poll();        // 到期了才真正出堆
    } finally {
        lock.unlock();
    }
}
//DelayQueue并发模型
					┌─────────────────────┐
                    │  消费者线程池         │
                    │  (比如 4 个线程)      │
                    └─────────┬───────────┘
                              │ take()
                              ▼
┌─────────────────────────────────────────────────┐
│  DelayQueue                                     │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  堆（PriorityQueue）                     │    │
│  │  [任务A(2s) 任务B(5s) 任务C(10s) ...]     │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  lock: 同一时刻只有一个线程能 poll                 │
│  leader: 同一时刻只有一个线程在精确计时             │
│  available: 其余线程在这等                        │
└─────────────────────────────────────────────────┘
```
#### SynchronousQueue
同步队列
```java
public class SynchronousQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, Serializable {

    private transient volatile Transferer<E> transferer; //是TransferQueue 和TransferStack的父类

    public SynchronousQueue() { this(false); }      // 默认非公平

    public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<>() : new TransferStack<>();
    }
}
//所有 `put / offer / take / poll` 全部委托给 `transferer.transfer(...)`：
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, false, 0) == null) {
        Thread.interrupted();
        throw new InterruptedException();
    }
}

public E take() throws InterruptedException {
    E e = transferer.transfer(null, false, 0);   // 传 null = 请求数据
    if (e != null) return e;
    Thread.interrupted();
    throw new InterruptedException();
}
//transfer 方法判断是读还是写，使用e == null 是take ，e!=null 是put
abstract static class Transferer<E> {
    abstract E transfer(E e, boolean timed, long nanos);
}
//默认2个实现 
//TransferStack 默认 非公平 LIFO 后进先出
static final class TransferStack<E> extend Transferer<E> {
	static final int REQUEST    = 0;   // take
	static final int DATA       = 1;   // put
	static final int FULFILLING = 2;   // 正在配对

	static final class SNode {
    volatile SNode next;
    volatile SNode match;     // 配对成功后指向对方节点
    volatile Thread waiter;   // 阻塞的线程
    Object item;              // DATA 存数据，REQUEST 为 null
    int mode;
	}
}
//transfer 核心流程
E transfer(E e, boolean timed, long nanos) {
    SNode s = null;
    int mode = (e == null) ? REQUEST : DATA;
    for (;;) {
        SNode h = head; //栈顶    所有操作都作用在栈顶
        if (h == null || h.mode == mode) {
            // 1) 栈空，或栈顶同模式（都是 put 或都是 take）
            if (timed && nanos <= 0) return null;        // 超时直接退
            if (s == null) s = new SNode(e, mode);
            if (!casHead(h, s)) continue;                // 压栈   将自己这个线程等别人来配
            // 自旋一小会儿 + 阻塞等配对
            SNode m = awaitFulfill(s, timed, nanos);//先按CPU计算spins,自旋期间不断看match/item变化  todo
            if (m == s) {                                // 被取消
                clean(s);
                return null;
            }
            // 配对成功，出栈
            if ((h = head) != null && h.next == s)
                casHead(h, s.next);
            return (E) (mode == REQUEST ? m.item : e);
        }
        else if ((h.mode & FULFILLING) == 0) {
            // 2) 栈顶是互补模式（put 遇 take，或反过来）且没在配对中。不进栈，直接把栈顶那个等待线程唤醒，把数据塞给它或者拿数据
            SNode m = h;                                 // 栈顶等待者
            if (!casHead(h, new SNode(e, mode | FULFILLING)))
                continue;
            // 把数据/请求填进 m，唤醒 m.waiter
            m.match = s;
            m.item = (mode == DATA) ? e : null;
            LockSupport.unpark(m.waiter);
            // 自己也被匹配
            return (E) (mode == REQUEST ? e : m.item);
        }
        // 3) 栈顶正在 FULFILLING，帮忙推进后重试
        else { casHead(h, h.next); }
    }
}
//transferQueue fair=true 公平 先进先出 带dummy头节点
static final class TransferQueue<E> extends Transferer<E> {
    static final class QNode {
        volatile QNode next;
        volatile Object item;      // DATA 非 null，REQUEST 为 null，取消时为 this
        volatile Thread waiter;
        final boolean isData;
    }
    transient volatile QNode head;   // 永远是 dummy 或已出队节点
    transient volatile QNode tail;
    transient volatile QNode cleanMe; // 删除尾节点时的标记位
}
//构造时
TransferQueue() {
    QNode h = new QNode(null, false);  // dummy
    head = h; tail = h;
}
//transfer 核心流程
E transfer(E e, boolean timed, long nanos) {
    QNode s = null;
    boolean isData = (e != null);              // e!=null 是 put(DATA)，null 是 take(REQUEST)
    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)            // 防御：看到未初始化值
            continue;
        // ===== 分支 A：空队列，或队尾同模式 =====
        if (h == t || t.isData == isData) {
            QNode tn = t.next;
            if (t != tail)                     // 读不一致，重试
                continue;
            if (tn != null) {                  // tail 滞后，帮推 tail
                advanceTail(t, tn);
                continue;
            }
            if (timed && nanos <= 0)           // 超时且时间到，直接退
                return null;
            if (s == null)
                s = new QNode(e, isData);      // 建节点
            if (!t.casNext(null, s))           // 挂到 tail 后面
                continue;
            advanceTail(t, s);                 // 推 tail
            Object x = awaitFulfill(s, e, timed, nanos);   // 自旋+park 等配对
            if (x == s) {                      // 被取消
                clean(t, s);
                return null;
            }
            if (!s.isOffList()) {              // 还没出队（配对成功但头没推进）
                advanceHead(t, s);             // dummy 头往前挪，s 成为新 dummy
                if (x != null)
                    s.item = s;                // 释放引用
                s.waiter = null;
            }
            return (x != null) ? (E) x : e;    // 消费者拿到 x，生产者拿到 e 的回执
        }
        // ===== 分支 B：队头 next 是互补模式 =====
        else {
            QNode m = h.next;                  // 等在那里的互补节点
            if (t != tail || m == null || h != head)   // 不一致读，重试
                continue;
            Object x = m.item;
            if (isData == (x != null) ||        // m 已经被配过了
                x == m ||                      // m 被取消
                !m.casItem(x, e)) {            // CAS 改 m.item 失败（别人抢先配了）
                advanceHead(h, m);             // 把 m 出队，重试
                continue;
            }
            advanceHead(h, m);                 // 配对成功，m 出队（dummy 头前进）
            LockSupport.unpark(m.waiter);      // 唤醒等着的那个线程
            return (x != null) ? (E) x : e;    // 自己拿对方的数据（take）或回执（put）
        }
    }
}
```
#### LinkedTransferQueue
传输队列
```java
public class LinkedTransferQueue<E> extends AbstractQueue<E>
        implements TransferQueue<E>, Serializable {

    // xfer 的四种模式
    private static final int NOW   = 0;  // poll / tryTransfer：不阻塞，立即返回
    private static final int ASYNC = 1;  // offer / put / add：入队就返回，永不阻塞
    private static final int SYNC  = 2;  // transfer / take：阻塞到配对
    private static final int TIMED  = 3; // 带超时的 poll / tryTransfer

    transient volatile Node head;
    private transient volatile Node tail;
    private transient volatile int sweepVotes;   // 清理取消节点的投票计数
}
//Node 双节点模式
static final class Node {  //双队列：节点可以代表2种角色 
    final boolean isData;     // true=生产者数据节点，false=消费者请求节点
    volatile Object item;      // 数据节点:非null→未匹配,null→已匹配;请求节点:null→未匹配,非null→已匹配
    volatile Node next;
    volatile Thread waiter;    // 阻塞在这个节点上的线程

    // 匹配判断：item==this 表示取消；((item==null)==isData) 表示已匹配
    final boolean isMatched() {
        Object x = item;
        return (x == this) || ((x == null) == isData);
    }
    // 能否挂到当前节点后面：前驱是异模式且未匹配 → 不能，说明碰到了对面等待者
    final boolean cannotPrecede(boolean haveData) {
        boolean d = isData;
        Object x;
        return d != haveData && (x = item) != this && (x != null) == d;
    }
}
//全部方法委托给xfer
public void put(E e)              { xfer(e, true,  ASYNC, 0); }
public boolean offer(E e)         { xfer(e, true,  ASYNC, 0); return true; }
public boolean add(E e)           { xfer(e, true,  ASYNC, 0); return true; }
public E take() throws InterruptedException {
    E e = xfer(null, false, SYNC, 0);   //e = null 代表take
    if (e != null) return e;
    Thread.interrupted();
    throw new InterruptedException();
}
public E poll()                   { return xfer(null, false, NOW, 0); }
public void transfer(E e) throws InterruptedException {
    if (xfer(e, true, SYNC, 0) != null) { Thread.interrupted(); throw new InterruptedException(); }
}
public boolean tryTransfer(E e)   { return xfer(e, true, NOW, 0) == null; }

//xfer
private E xfer(E e, boolean haveData, int how, long nanos) {
    if (haveData && (e == null)) throw new NullPointerException();
    Node s = null;
    retry:
    for (;;) {
        // ===== 阶段1：从 head 向后扫，找未匹配异模式节点配对 =====
        for (Node h = head, p = h; p != null; ) {
            boolean isData = p.isData;
            Object item = p.item;
            // item!=p 且 (item!=null)==isData → 节点未匹配且模式有效
            if (item != p && (item != null) == isData) {
                if (isData == haveData) break;   // 同模式，不能配，跳出扫尾阶段去追加
                if (p.casItem(item, e)) {        // 异模式，CAS 改 item 完成配对
                    // 推进 head（松弛度<=2，跳过已匹配前缀）  不是每次匹配都casHead
                    for (Node q = p; q != h; ) {
                        Node n = q.next;
                        if (head == h && casHead(h, n == null ? q : n)) {
                            h.forgetNext();
                            break;
                        }
                        if ((h = head) == null || (q = h.next) == null || !q.isMatched())
                            break;
                    }
                    LockSupport.unpark(p.waiter);   // 唤醒被配对的那个线程
                    return cast(item);              // 生产者返回 e 的等价物，消费者返回数据
                }
            }
            Node n = p.next;
            p = (p != n) ? n : (h = head);   // 遇到自环跳到新 head
        }

        // ===== 阶段2：没配上，且不是 NOW 模式 → 追加新节点 =====
        if (how != NOW) {
            if (s == null) s = new Node(e, haveData);
            Node pred = tryAppend(s, haveData);
            if (pred == null) continue retry;   // 碰见异模式节点，重头扫
            // ===== 阶段3：ASYNC 直接返回；SYNC/TIMED 阻塞等配对 =====
            if (how != ASYNC)
                return awaitMatch(s, pred, e, (how == TIMED), nanos);
        }
        return e;   // NOW 模式且没配上，或 ASYNC 入队成功
    }
}
//tryAppend
private Node tryAppend(Node s, boolean haveData) {
    for (Node t = tail, p = t; ; ) {
        Node n, u;
        if (p == null && (p = head) == null) {
            if (casHead(null, s)) return s;          // 空队列，建立首节点
        } else if (p.cannotPrecede(haveData)) {
            return null;                             // 前驱是异模式未匹配 → 放弃，让 xfer 重启
        } else if ((n = p.next) != null) {
            p = (p != t && t != (u = tail)) ? (t = u) : (p != n ? n : null);  // 跳到真尾
        } else if (!p.casNext(null, s)) {
            p = p.next;                              // CAS 失败重试
        } else {
            if (p != t) {                           // 松弛更新 tail
                while ((tail != t || !casTail(t, s)) && (t = tail) != null
                       && (s = t.next) != null && (s = s.next) != null && s != t) ;
            }
            return p;                                // 返回前驱，交给 awaitMatch
        }
    }
}
```
#### LinkedBlokingDeque
双端阻塞队列
```java
public class LinkedBlockingDeque<E> extends AbstractQueue<E>
        implements BlockingDeque<E>, Serializable {

    static final class Node<E> {
        E item;
        Node<E> prev;   // 前驱（LBQ 没有这个）
        Node<E> next;   // 后继
        Node(E x) { item = x; }
    }

    transient Node<E> first;      // 队头
    transient Node<E> last;       // 队尾
    private transient int count;  // 元素数（不是 AtomicInteger，因为单锁保护）
    private final int capacity;   // 默认 Integer.MAX_VALUE

    final ReentrantLock lock = new ReentrantLock();      // 唯一一把锁  锁不能按头尾拆分
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull  = lock.newCondition();
}
//所有putX/offerX/takeX/pollX 都是调 linkFirst/linkLast/unlinkFirst/unlinkLast
//linkFirst  头插
private boolean linkFirst(Node<E> node) {
    if (count >= capacity) return false;
    Node<E> f = first;
    node.next = f;
    first = node;
    if (last == null) last = node;   // 空队列，头尾同节点
    else f.prev = node;
    ++count;
    notEmpty.signal();               // 唤醒等数据的消费者
    return true;
}
//linkLast  尾插
private boolean linkLast(Node<E> node) {
    if (count >= >= capacity) return false;
    Node<E> l = last;
    node.prev = l;
    last = node;
    if (first == null) first = node;
    else l.next = node;
    ++count;
    notEmpty.signal();
    return true;
}
//unlinkFirst  获取头
private E unlinkFirst() {
    Node<E> f = first;
    if (f == null) return null;
    Node<E> n = f.next;
    E item = f.item;
    f.item = null;
    f.next = f;          // 自指，help GC
    first = n;
    if (n == null) last = null;
    else n.prev = null;
    --count;
    notFull.signal();    // 唤醒等空位的生产者
    return item;
}
//unlinkLast   获取尾
private E unlinkLast() {
    Node<E> l = last;
    if (l == null) return null;
    Node<E> p = l.prev;
    E item = l.item;
    l.item = null;
    l.prev = l;          // 自指，help GC
    last = p;
    if (p == null) first = null;
    else p.next = null;
    --count;
    notFull.signal();
    return item;
}
```
### 应用场景
#### 线程池的应用
```
// 各种线程池用的阻塞队列
// ① FixedThreadPool → LinkedBlockingQueue（无界队列）
ExecutorService fixed = Executors.newFixedThreadPool(10);
// 内部：new LinkedBlockingQueue<Runnable>()  —— 无界

// ② CachedThreadPool → SynchronousQueue（直接交付）
ExecutorService cached = Executors.newCachedThreadPool();
// 内部：new SynchronousQueue<Runnable>()  —— 容量0

// ③ SingleThreadPool → LinkedBlockingQueue（无界队列）
ExecutorService single = Executors.newSingleThreadExecutor();
// 内部：new LinkedBlockingQueue<Runnable>()  —— 无界

// ④ ScheduledThreadPool → DelayedWorkQueue（延迟队列）
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);
// 内部：DelayedWorkQueue（类似 DelayQueue）
```
### 选型结论
```
ArrayBlockingQueue：   数组有界，一把锁，简单稳定
LinkedBlockingQueue：  链表有界/无界，两把锁，吞吐量高
PriorityBlockingQueue：无界堆，优先级出队
DelayQueue：           延迟到期才出队
SynchronousQueue：     容量0，直接传递
LinkedTransferQueue：  无界链表，支持等待消费
LinkedBlockingDeque：  双向链表，双端操作
```

