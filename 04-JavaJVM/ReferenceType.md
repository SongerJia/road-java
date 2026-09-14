## 引用类型与GC判定
GC判定方法->可达性分析->GC Roots->四种引用类型->软/弱引用的应用->面试高频题
### GC判定方法
对象什么时候应该被回收？
```
// 判断对象是否"已死"有两种方法：
// ① 引用计数法（Reference Counting）
// ② 可达性分析（Reachability Analysis）
```
#### 引用计数法（已淘汰）
```
// 原理：每个对象维护一个引用计数器
// 被引用 +1，引用失效 -1
// 计数为 0 → 可回收

// 缺点：无法解决循环引用！
public class Node {
    public Node next;
}

Node a = new Node();
Node b = new Node();
a.next = b;
b.next = a;  // 互相引用，计数都是 1
a = null;
b = null;
// 但 a、b 互相引用，计数不为 0 → 永远无法回收 → 内存泄漏

// 所以 JVM 不用引用计数法
// Python 用引用计数 + 循环检测
```
#### 可达性分析
```
// 原理：从"GC Roots"出发，沿着引用链遍历
// 能被访问到的对象 → 存活
// 访问不到的 → 可回收

// 类似"从根出发的树遍历"
// 像看哪座岛和陆地相连，相连的活着，孤岛沉没
```
可达性分析图
```
GC Roots（根）
    │
    ├──→ A → B → C（可达，存活）
    │
    ├──→ D → E（可达，存活）
    │
    └──→ F ──→ G（可达）
                │
                └──→ H
                     │
                     └──→ G（循环引用，但可达，存活）

    X → Y → Z（从根不可达 → 全部回收）
    │    ↑
    └────┘（X 和 Y 循环引用也没用，还是回收）
```
### GC Roots有哪些
GC Roots 包括
```
// ① 虚拟机栈（栈帧局部变量表）中引用的对象
public void method() {
    User user = new User();  // user 是 GC Root（栈中引用）
    // 方法结束后，user 不再是 GC Root
}

// ② 静态变量引用的对象
public class Config {
    public static User currentUser;  // 静态引用是 GC Root
    // 只要类不被卸载，currentUser 一直是 GC Root
}

// ③ 常量引用的对象
public class Constant {
    public static final User DEFAULT = new User();  // 常量引用是 GC Root
}

// ④ native 方法引用的对象（JNI）

// ⑤ 被 synchronized 持有的对象（锁对象）
synchronized (lockObject) {
    // lockObject 是 GC Root（被锁持有）
}

// ⑥ 活跃线程对象本身
```
常见的GC Roots汇总

| GC Root | 说明              |
| ------- | --------------- |
| 栈帧局部变量  | 方法中引用的对象        |
| 静态变量    | 类级引用的对象         |
| 常量      | final引用的对象      |
| JNI引用   | native方法引用      |
| 锁对象     | synchronized持有的 |
| 活跃线程    | 正在运行的线程         |
### 面试高频

> **Q:** "哪些对象可以作为 GC Roots？" 
> **A:** "栈帧局部变量、静态变量、常量、JNI 引用、锁对象、活跃线程。核心：JVM 自身存活的东西 + 被全局/栈引用但不被 GC 管理的对象。"
### 四种引用类型
引用强度对比
```
强引用 > 软引用 > 弱引用 > 虚引用

强引用：绝不回收（除非不可达）
软引用：内存不足时回收
弱引用：下次 GC 就回收
虚引用：随时可能回收，无法通过它获取对象
```
#### 强引用（Strong Reference）
```
// 最常见的引用：new 出来的
User user = new User();  // 强引用

// 只要强引用存在，GC 永远不会回收该对象
// 即使 OOM 也不回收（宁可抛 OOM）

// 内存泄漏的常见原因：
// 静态集合持有强引用不放 → 无法回收
```
弱引用（Soft Reference）
```
// 内存充足时不回收，内存不足时才回收
// 适合：缓存（内存紧张时可以腾出空间）

import java.lang.ref.SoftReference;

User user = new User("张三", 25);
SoftReference<User> softRef = new SoftReference<>(user);
user = null;  // 只有软引用了

// 内存充足 → 对象存活
// 内存不足（即将 OOM）→ 对象被回收

User user2 = softRef.get();  // 获取对象（可能为 null，已被回收）
if (user2 != null) {
    // 还在，使用
} else {
    // 被回收了，重新加载
}

// 经典应用：图片缓存
Map<String, SoftReference<Image>> imageCache = new HashMap<>();
// 内存不足时，图片自动被回收，不会 OOM
```
#### 弱引用（Weak Reference）
```
// 无论内存是否充足，下次 GC 一定回收
// 适合：需要被自动清理的对象

import java.lang.ref.WeakReference;

User user = new User();
WeakReference<User> weakRef = new WeakReference<>(user);
user = null;  // 只有弱引用了

System.gc();  // 触发 GC（演示用，生产不要调）
User user2 = weakRef.get();  // null —— 已被回收    但是只有get就会变为强引用

// 经典应用：ThreadLocal 的 key
// ThreadLocalMap 的 Entry 的 key 就是弱引用
// 所以 ThreadLocal 对象可以被回收
// （这也是为什么 value 要 remove()，防止 value 泄漏）
```
#### 虚引用（Phantom Reference）
```
// 最弱的引用：无法通过 get() 获取对象（永远返回 null）
// 唯一用途：对象被回收时收到通知（用于清理资源）

import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;

ReferenceQueue<User> queue = new ReferenceQueue<>();
User user = new User();
PhantomReference<User> phantomRef = new PhantomReference<>(user, queue);
user = null;

// 对象被回收时，虚引用会被放入 ReferenceQueue
// 通过轮询 queue，可以知道对象被回收了

// 经典应用：堆外内存回收通知
// DirectByteBuffer 的 Cleaner 用虚引用：
// 对象被回收 → Cleaner 收到通知 → 释放直接内存
```
### 引用类型对比总结
|引用类型|回收时机|get()|用途|
|---|---|---|---|
|**强引用**|绝不回收（除非不可达）|✅|常规对象|
|**软引用**|内存不足时|✅（可能 null）|缓存|
|**弱引用**|下次 GC|✅（可能 null）|ThreadLocal key|
|**虚引用**|随时|❌（永远 null）|回收通知|
为什么ThreadLocal用弱引用？
```
// ThreadLocal 的 Entry：
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;  // 值还是强引用！
}

// key 用弱引用：
// ThreadLocal 对象没有强引用时 → 可以被回收 → key 变 null
// 但 value 还有强引用 → 无法回收 → 需要 remove()

// 如果 key 用强引用：
// ThreadLocal 对象永远被 key 持有 → 无法回收 → 泄漏更严重

// 所以：弱引用 key 是"尽量减少泄漏"，但要配合 remove() 彻底解决
```
### 软/弱引用的实际应用
#### 应用1：简单缓存（软引用）
```
// 一个简单的软引用缓存
public class SoftCache<K, V> {
    private final Map<K, SoftReference<V>> cache = new HashMap<>();

    public void put(K key, V value) {
        cache.put(key, new SoftReference<>(value));
    }

    public V get(K key) {
        SoftReference<V> ref = cache.get(key);
        if (ref == null) return null;
        V value = ref.get();
        if (value == null) {
            // 被 GC 回收了，移除脏数据
            cache.remove(key);
        }
        return value;
    }
}
```
#### 应用2：WeakHashMap
```
// WeakHashMap：key 用弱引用
// key 不再被外部引用时 → 自动从 map 中移除

import java.util.WeakHashMap;

WeakHashMap<String, byte[]> cache = new WeakHashMap<>();

String key = new String("data");  // 注意：不能用字面量（常量池强引用）
cache.put(key, new byte[1024]);

key = null;  // 只有 WeakHashMap 中的弱引用了
// 下次 GC → 该 entry 自动被清除
// 不需要手动 remove()

// 场景：需要自动清理的映射
```
#### 应用3：直接内存回收（虚引用）
```
// DirectByteBuffer 的 Cleaner 机制
// 堆中对象被回收 → Cleaner（虚引用）触发 → 释放堆外内存
// 防止堆外内存泄漏
```
### 面试高频题
#### 题目1：判断对象可回收的两种方法？
```
// 引用计数法（有循环引用问题，JVM 不用）
// 可达性分析（JVM 使用，从 GC Roots 遍历）
```
#### 题目2：四种引用的区别？
```
// 强：绝不回收
// 软：内存不足回收
// 弱：下次 GC 回收
// 虚：随时回收，用于通知
```
#### 题目3：ThreadLocal为什么用弱引用？
```
// key 用弱引用，ThreadLocal 可被回收
// 但 value 强引用，需 remove() 防止泄漏
// 弱引用是"尽量减害"，remove() 是"彻底解决"
```
#### 题目4：什么情况下软引用对象被回收？
```
// JVM 内存不足（即将 OOM）时
// 软引用对象会在 OOM 之前被回收
// 回收后依然不足 → 才抛 OOM
```
#### 题目5：怎么判断一个对象是垃圾？
```
// 可达性分析：从 GC Roots 出发
// 不可达 → 第一次标记
// 没有覆盖 finalize() → 直接回收
// 覆盖了 finalize() → 进入 F-Queue 等待执行
// finalize 后仍未复活 → 回收
// （finalize 已废弃，JDK 9+ 不推荐）
```
