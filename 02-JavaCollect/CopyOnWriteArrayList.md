## CopyOnWriteArrayList
写时复制原理->核心源码->读操作->写操作->迭代器特性->差异化结论->选型场景
### 写时复制原理 COW
什么是写时复制？
```
Copy-On-Write,写操作时，先复制一份新数组，在新数组上修改，再替换旧数组。读操作不加锁，永远读旧数组。
结构图
写操作流程：
                    ┌─────────────────────┐
                    │  旧数组（读操作在读它）│
                    │  [A, B, C, D]        │
                    └─────────────────────┘
                              ↓ 复制一份
                    ┌─────────────────────┐
                    │  新数组（写操作修改它）│
                    │  [A, B, C, D, E]    │  ← 在副本上添加 E
                    └─────────────────────┘
                              ↓ 替换引用（原子操作）
                    ┌─────────────────────┐
                    │  新数组成为当前数组    │
                    │  [A, B, C, D, E]    │
                    └─────────────────────┘
```
### 核心源码
```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, Serializable {

    // 唯一的存储：volatile 数组
    private transient volatile Object[] array;

    // 写锁：ReentrantLock
    final transient ReentrantLock lock = new ReentrantLock();

    // 获取数组
    final Object[] getArray() {
        return array;
    }

    // 设置数组
    final void setArray(Object[] a) {
        array = a;
    }
}

// 关键点：
// ① array 是 volatile —— 保证数组引用的可见性
// ② 读操作直接读 array，不加锁
// ③ 写操作在副本上修改，然后替换 array
```
### 读操作 无锁
```java
// 读操作 —— 完全无锁，性能极高
public E get(int index) {
    return get(getArray(), index);  // 直接读数组
}

private E get(Object[] a, int index) {
    return (E) a[index];  // 数组访问，无锁
}

// 其他读操作
public int size() {
    return getArray().length;  // 直接读数组长度
}

public boolean contains(Object o) {
    // 遍历数组
    Object[] elements = getArray();
    return indexOf(o, elements, 0, elements.length) >= 0;
}

public boolean isEmpty() {
    return size() == 0;
}
```
为什么读操作无锁？
```
// volatile 保证：
// ① 写线程替换 array 引用后，读线程立即可见
// ② 读线程读到的数组永远是"某个时刻的完整快照"

// 读操作只读 array 引用指向的数组
// 即使写操作正在复制新数组，读操作读的还是旧数组
// 永远不会读到"半修改"的数据
```
### 写操作 加锁+复制
add(e) 添加元素
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();  // 写操作必须加锁
    try {
        Object[] elements = getArray();
        int len = elements.length;

        // ① 复制一份新数组（长度 + 1）
        Object[] newElements = Arrays.copyOf(elements, len + 1);

        // ② 在新数组上修改
        newElements[len] = e;

        // ③ 替换旧数组（volatile 写，原子操作）
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
add(index, e) 指定位置添加
```java
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;

        // 边界检查
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + len);

        //新建了一个数组  复制并插入  拷贝2次
        Object[] newElements = new Object[len + 1];
        System.arraycopy(elements, 0, newElements, 0, index);
        newElements[index] = element;
        System.arraycopy(elements, index, newElements, index + 1, len - index);

        setArray(newElements);
    } finally {
        lock.unlock();
    }
}
```
remove 删除元素
```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);

        int numMoved = len - index - 1;
        Object[] newElements;

        if (numMoved == 0) {
            // 删除最后一个，直接截断
            newElements = Arrays.copyOf(elements, len - 1);
        } else {
            // 复制并删除
            newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index, numMoved);
        }

        setArray(newElements);
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```
set  修改元素
```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        // 如果新值和旧值不同，才复制
        if (oldValue != element) {
            Object[] newElements = Arrays.copyOf(elements, elements.length);
            newElements[index] = element;
            setArray(newElements);
        }
        // 如果相同，不复制（小优化）
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```
### 迭代器特性
快照迭代器
```java
// COW 的迭代器是"快照"式的 —— 迭代的是创建时的数组
// 迭代过程中，即使其他线程修改了列表，迭代器也不会受影响

CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("A");
list.add("B");
list.add("C");

// 创建迭代器
Iterator<String> iterator = list.iterator();

// 此时其他线程修改列表
list.add("D");
list.remove("A");

// 迭代器仍然遍历旧快照 [A, B, C]
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
// 输出：A B C —— 不包含 D，也不受 A 被删除影响
```
迭代器源码
```java
// 迭代器持有创建时的数组快照
private class COWIterator<E> implements ListIterator<E> {
    private final Object[] snapshot;  // 快照数组
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;  // 持有数组引用，后续修改不影响
    }
}

// 创建迭代器时，复制数组引用
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);  // 注意：不是复制数组，是引用
}
```
### 差异化结论

| 对比维度 | ArrayList | CopyOnWriteArrayList |
| ---- | --------- | -------------------- |
| 线程安全 | 不安全       | 安全                   |
| 读操作  | 无锁        | 无锁                   |
| 写操作  | 无锁        | 有锁                   |
| 读性能  | O(1)      | O(1)                 |
| 写性能  | O(1)      | O(n)                 |
| 内存   | 少         | 多                    |
| 迭代器  | fail-fast | 快照式                  |
| 适用场景 | 通用        | 读多写少                 |

### 选型场景
```
面试官："CopyOnWriteArrayList 有什么用？什么时候用？"

"CopyOnWriteArrayList 是读多写少场景的线程安全 List，
核心原理是写时复制：

① 读操作：完全无锁，直接读 volatile 数组
② 写操作：加锁，复制新数组，在副本上修改，然后原子替换引用

优点：
- 读性能极高（无锁），并发读没有竞争
- 迭代器是快照式的，遍历时不会抛 ConcurrentModificationException

缺点：
- 写性能差（每次复制整个数组，O(n)）
- 内存占用高（写时有两份数组）
- 数据弱一致性（读可能读到旧数据）

适用场景：读多写少的场景，
比如黑名单、配置信息、监听器列表。
```
