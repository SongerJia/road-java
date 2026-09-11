## Phaser & Exchanger & ThreadLocalRandom & CopyOnWriteArraySet
### Phaser 阶段屏障
Phaser是什么？
Phaser：可复用的阶段同步器，比CyclicBarrier更灵活，支持按阶段推进，参与线程数可动态变化。
基本使用
```
// 场景：分阶段任务，每个阶段等所有线程到齐
public class PhaserDemo {
    public static void main(String[] args) {
        int parties = 3;
        Phaser phaser = new Phaser(parties);  // 3 个参与者

        for (int i = 1; i <= parties; i++) {
            new Thread(() -> {
                // 阶段 1
                System.out.println(Thread.currentThread().getName() + " 执行阶段1");
                phaser.arriveAndAwaitAdvance();  // 到达并等待其他人

                // 阶段 2（所有线程到齐后一起进入）
                System.out.println(Thread.currentThread().getName() + " 执行阶段2");
                phaser.arriveAndAwaitAdvance();

                // 阶段 3
                System.out.println(Thread.currentThread().getName() + " 执行阶段3");
                phaser.arriveAndDeregister();  // 到达并注销（退出）
            }, "线程" + i).start();
        }
    }
}
```
对比CyclicBarrier

|对比|CyclicBarrier|Phaser|
|---|---|---|
|**参与人数**|固定|可动态增减|
|**阶段数**|单阶段（循环）|多阶段推进|
|**复用**|自动重置|自动推进|
|**注册/注销**|❌|✅ register()/arriveAndDeregister()|
|**返回阶段号**|❌|✅ getPhase()|
|**灵活性**|低|高|
### Exchanger 交换器
什么是Exchanger?
Exchanger：两个线程交换数据的同步点。双方到达后，交换数据，然后继续各自执行。
基本使用
```
// 场景：两个线程交换数据
public class ExchangerDemo {
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();

        // 线程 A
        new Thread(() -> {
            try {
                String data = "线程A的数据";
                String received = exchanger.exchange(data);  // 阻塞等交换
                System.out.println("线程A收到: " + received);
            } catch (InterruptedException e) {
            }
        }).start();

        // 线程 B
        new Thread(() -> {
            try {
                String data = "线程B的数据";
                String received = exchanger.exchange(data);  // 阻塞等交换
                System.out.println("线程B收到: " + received);
            } catch (InterruptedException e) {
            }
        }).start();

        // 输出：
        // 线程A收到: 线程B的数据
        // 线程B收到: 线程A的数据
    }
}
```
应用场景
```
// ① 双人协作：生产者/消费者各一个线程，直接交换数据
// ② 遗传算法：两个线程交换基因数据
// ③ 两个线程间的数据交换（很少用，一般用 BlockingQueue 更简单）

// 面试一句话：两个线程交换数据的工具，双方都调用 exchange() 才交换，
// 匹配不到会一直阻塞。实际开发中用得少，了解即可。
```
### ThreadLocalRandom 线程本地随机数
什么是ThreadLocalRandom?
ThreadLocalRandom：线程隔离的随机数生成器。每个线程维护独立的随机种子，避免多线程竞争同一个Random对象。
为什么需要？
```
// ❌ 问题：共享 Random 对象有竞争
Random random = new Random();
// 多个线程调用 nextInt() → CAS 竞争同一个种子 → 性能差

// ✅ ThreadLocalRandom：每个线程独立的种子 → 无竞争
ThreadLocalRandom.current().nextInt(100);
```
使用
```
// 使用方式：ThreadLocalRandom.current() 获取当前线程的实例
int randomNum = ThreadLocalRandom.current().nextInt(100);  // 0~99
int randomRange = ThreadLocalRandom.current().nextInt(10, 20);  // 10~19
double randomDouble = ThreadLocalRandom.current().nextDouble();
long randomLong = ThreadLocalRandom.current().nextLong();

// 比 Math.random() 快，比共享 Random 并发更好
```
对比

|对比|Random|ThreadLocalRandom|
|---|---|---|
|**线程安全**|✅（但竞争）|✅（无竞争）|
|**种子**|共享|每线程独立|
|**并发性能**|差（竞争）|好|
|**获取方式**|new Random()|current()|
### CopyOnWriteArraySet 并发set
什么是CopyOnWriteArraySet
CopyOnWriteArraySet：基于CopyOnWriteArrayList实现的线程安全的Set。
实现原理
```
// 底层就是一个 CopyOnWriteArrayList
public class CopyOnWriteArraySet<E> extends AbstractSet<E> {
    // 内部持有 COWList
    private final CopyOnWriteArrayList<E> al;

    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }

    // add 方法：调 COWList 的 addIfAbsent（不存在才添加）
    public boolean add(E e) {
        return al.addIfAbsent(e);  // ✅ 原子去重
    }
}

// addIfAbsent：
public boolean addIfAbsent(E e) {
    // 写时复制 + 去重
    // 复制数组 → 检查是否已存在 → 不存在则添加 → 替换引用
```
特性

|特性|说明|
|---|---|
|**线程安全**|✅（写时复制）|
|**去重**|✅（addIfAbsent）|
|**迭代器**|快照式，不抛 ConcurrentModificationException|
|**顺序**|保持插入顺序|
|**性能**|读 O(1)，写 O(n)（复制）|
|**适用**|读多写少 + 需要去重|
使用场景
```
// 场景：订阅者列表（读多写少 + 去重）
CopyOnWriteArraySet<String> subscribers = new CopyOnWriteArraySet<>();

// 订阅（重复订阅不会重复添加）
subscribers.add("user1");
subscribers.add("user1");  // 不会重复

// 遍历安全（遍历时其他线程可以改）
for (String user : subscribers) {
    // 安全，快照遍历
}
```
对比

|对比|HashSet|CopyOnWriteArraySet|
|---|---|---|
|**线程安全**|❌|✅|
|**底层**|HashMap|CopyOnWriteArrayList|
|**迭代器**|fail-fast|快照|
|**顺序**|无序|插入顺序|
|**性能**|写 O(1)|写 O(n)|
|**适用**|单线程|读多写少并发|
