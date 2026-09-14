## 逃逸分析
逃逸分析是什么->逃逸的三种程度->三大优化详解->实际效果->面试高频题
### 逃逸分析是什么
EscapeAnalysis：JIT编译器分析对象的作用域，判断对象是否逃逸出方法或线程，据此决定能否做栈上分配、标量替换、锁消除优化
分析什么？
```
// JIT 编译器在运行时分析：
// ① new 出来的对象，会被谁引用？
// ② 对象能存活多久？
// ③ 其他线程能不能访问到？

// 根据分析结果决定优化策略
```
谁触发？
```
// 不是程序员控制的，是 JIT 自动做的
// 参数：-XX:+DoEscapeAnalysis（JDK 6u23+ 默认开启）
// -XX:+PrintEscapeAnalysis 可以打印分析结果
```
### 逃逸的三种程度
No Escape：不逃逸
```
// 对象只在方法内部使用，不传递给外部
public long sum() {
    Point p = new Point(1, 2);  // 不逃逸
    return p.x + p.y;
}
// ✅ 可以做全部优化
```
Method Escape：方法逃逸
```
// 对象作为方法返回值、参数、或赋给外部变量
public Point createPoint() {
    return new Point(1, 2);  // 逃逸了（被返回）
}

public void process(Point p) {  // 参数引用外部对象
    System.out.println(p.x);
}
// ❌ 不能栈上分配、不能标量替换
```
Thread Escape：线程逃逸
```
// 对象被其他线程访问（赋给静态变量、放入共享集合）
public class Demo {
    private static Point shared;  // 静态共享

    public void save() {
        shared = new Point(1, 2);  // 线程逃逸！
    }
}
// ❌ 所有优化都不能做
```

|逃逸程度|栈上分配|标量替换|锁消除|
|---|---|---|---|
|**不逃逸**|✅|✅|✅|
|**方法逃逸**|❌|❌|⚠️ 部分|
|**线程逃逸**|❌|❌|❌|
### 三大优化详解
优化一：栈上分配（Stack Allocation）
```
// 原理：不逃逸的对象直接在栈上分配
// 方法结束 → 栈帧弹出 → 对象自动销毁
// 不需要 GC！

public long sum() {
    Point p = new Point(1, 2);  // 原本在堆上
    return p.x + p.y;
}

// 优化后：p 在栈帧中分配
// 方法返回 → p 直接销毁，不经过 GC

// 好处：
// ① 减少堆内存占用
// ② 减少 GC 压力
// ③ 分配快（栈指针移动即可）
```
优化二：标量替换（Scalar Replacement）
```
// 原理：把对象"拆散"成基本类型局部变量
// 对象的字段 → 独立的标量（int、double 等）

// 原始代码：
public long sum() {
    Point p = new Point(1, 2);  // 一个对象
    return p.x + p.y;
}

// 优化后（标量替换）：
public long sum() {
    int x = 1;   // 拆成局部变量
    int y = 2;
    return x + y;  // 没有 Point 对象了！
}

// 好处：连对象都不创建了！
// 比栈上分配更进一步

// 参数：-XX:+EliminateAllocations（默认开启）
```
优化三：锁消除（Lock Elimination）
```
// 原理：如果对象不逃逸，其他线程访问不到
// 那么加锁没有意义 → 消除锁

// 原始代码：
public String concat(String a, String b) {
    StringBuffer sb = new StringBuffer();  // 局部对象，不逃逸
    sb.append(a);   // StringBuffer 的方法都是 synchronized
    sb.append(b);
    return sb.toString();
}

// 优化后：synchronized 被消除（没有竞争者）
public String concat(String a, String b) {
    StringBuilder sb = new StringBuilder();  // 相当于无锁版本
    sb.append(a);
    sb.append(b);
    return sb.toString();
}

// 参数：-XX:+EliminateLocks（默认开启）
// 这就是"为什么 StringBuffer 在局部使用时性能也不差"
```
### 实际效果
```
// 场景：循环 1000 万次创建不逃逸对象
public class EscapeTest {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();

        long total = 0;
        for (int i = 0; i < 10_000_000; i++) {
            total += sum();  // 每次创建 Point
        }

        System.out.println("耗时: " + (System.currentTimeMillis() - start) + "ms");
    }

    public static long sum() {
        Point p = new Point(1, 2);  // 不逃逸
        return p.x + p.y;
    }
}

// 开启逃逸分析：约 20ms（标量替换，几乎不创建对象）
// 关闭逃逸分析：约 800ms（每次都在堆创建对象，还要 GC）
// 相差 40 倍！


# 开启（默认）：
java -XX:+DoEscapeAnalysis -XX:+EliminateAllocations -XX:+EliminateLocks EscapeTest

# 关闭：
java -XX:-DoEscapeAnalysis EscapeTest
# 观察：GC 明显变多，耗时明显上升
```
和JVM知识的串联
```
逃逸分析
  ├── 栈上分配 → 呼应  对象创建（对象不一定在堆）
  ├── 标量替换 → 呼应  TLAB（分配优化）
  ├── 锁消除 → 呼应 JUC synchronized 优化
  └── 配合 TLAB → 分配路径：栈 → TLAB → Eden → 老年代
```
完整分配路径
```
new 对象
  ├── 逃逸分析
  │   ├── 不逃逸 → 栈上分配/标量替换（免 GC）✅
  │   └── 逃逸 → 进入堆
  │       ├── TLAB 分配（无竞争）
  │       ├── TLAB 不够 → Eden 直接分配
  │       ├── 大对象 → 老年代
  │       └── Minor GC 存活 → S0/S1 → 老年代
```
### 面试高频题
#### 题目1：逃逸分析是什么？有哪些优化
```
// JIT 分析对象是否逃逸出方法
// 三大优化：
// ① 栈上分配：对象在栈上，方法结束销毁，免 GC
// ② 标量替换：对象拆成基本类型变量（最好，连对象都没有）
// ③ 锁消除：没有竞争者就删掉锁
```
#### 题目2：哪些对象不能做栈上分配
```
// ① 方法返回的对象（方法逃逸）
// ② 赋给静态变量的对象（线程逃逸）
// ③ 放入共享集合的对象（线程逃逸）
// ④ 大对象（栈空间有限）
```
#### 题目3：对象一定在堆上吗
```
// 不一定！
// 不逃逸的对象经过逃逸分析后：
// ① 栈上分配
// ② 或标量替换（拆成变量，连对象都不是）
// 所以"所有对象都在堆上"说法不严谨
```
#### 题目4：逃逸分析和TLAB的关系
```
// 逃逸分析：判断对象能否在栈上分配（不进堆）
// TLAB：堆内的分配优化（Eden 中线程私有缓冲）
// 顺序：先逃逸分析（可能栈上）→ 不行再进堆（TLAB 分配）
```
#### 题目5：锁消除原理
```
// 对象不逃逸 → 其他线程访问不到 → 锁没有意义 → 删除
// 典型：方法内局部 StringBuffer（其实不需要线程安全）
// 这是 JIT 的自动优化，程序员不用管
```
