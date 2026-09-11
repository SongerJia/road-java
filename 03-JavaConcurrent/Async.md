## 异步
Futrue的痛点->CompletableFuture核心->异步编排方法->异步处理->ForkJoinPool->对比总结
### Future的痛点
传统Future的问题
```
// Java 5 引入 Future，但很难用：

// 问题 1：get() 阻塞
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<Integer> future = executor.submit(() -> compute());

Integer result = future.get();  // ❌ 阻塞！等到任务完成
// 期间什么都做不了 —— 和同步调用没区别

// 问题 2：无法异步编排
// 需要"任务 A 完成后，异步执行任务 B"很难实现
Future<Integer> futureA = executor.submit(taskA);
// ❌ 没有"完成回调"机制，只能轮询 future.isDone()

// 问题 3：多个任务组合很难
// 等 A、B 都完成再执行 C
// 等 A 或 B 任一完成再执行 C
// 用 Future 实现非常麻烦

// 结论：Future 是"伪异步"——最终还是要阻塞等待
```
### CompletableFuture核心
CompletableFuture：增强版Future，支持真正的异步编程，完成回调，异步编排，多任务组合，异常处理。
创建方式
```
// ① runAsync：无返回值
CompletableFuture<Void> f1 = CompletableFuture.runAsync(() -> {
    System.out.println("执行任务（无返回值）");
});

// ② supplyAsync：有返回值
CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> {
    return 42;  // 返回计算结果
});

// ③ 指定线程池（重要！不指定用 ForkJoinPool.commonPool）
ExecutorService pool = Executors.newFixedThreadPool(4);
CompletableFuture<Integer> f3 = CompletableFuture.supplyAsync(() -> {
    return 42;
}, pool);  // ✅ 指定线程池

// ④ 获取结果
Integer result = f2.get();       // 阻塞获取（和 Future 一样）
Integer result2 = f2.join();     // 阻塞获取（不抛受检异常）
```
完成回调（核心特性）
```java
// 任务完成后自动执行回调，不阻塞！
CompletableFuture.supplyAsync(() -> compute())
    .thenAccept(result -> {       // 完成后执行（无返回值）
        System.out.println("结果: " + result);
    })
    .thenRun(() -> {             // 完成后执行（不关心结果）
        System.out.println("全部完成");
    });

// 回调线程：默认在完成任务的线程上执行
// 也可以指定线程池：
.thenAcceptAsync(result -> { }, pool);
```
### 异步编排方法
#### 串形编排（thenApply/thenCompose）
```
// thenApply：转换结果（同步）
CompletableFuture<Integer> future =
    CompletableFuture.supplyAsync(() -> 42)
        .thenApply(x -> x * 2)       // 42 → 84
        .thenApply(x -> x + 1);      // 84 → 85
// 结果：85

// thenApplyAsync：转换结果（异步，指定线程池）
CompletableFuture.supplyAsync(() -> 42)
    .thenApplyAsync(x -> x * 2, pool);

// thenCompose：扁平化 —— 前一个返回 CompletableFuture
CompletableFuture.supplyAsync(() -> 42)
    .thenCompose(x -> CompletableFuture.supplyAsync(() -> x * 2));
// 区别：thenApply 返回普通值，thenCompose 返回 CompletableFuture
```
#### 并形组合（thenCombine/allOf/anyOf）
```
// thenCombine：两个任务都完成后，合并结果
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + " " + b);
// 结果："Hello World"

// allOf：等待所有任务完成（返回 Void）
CompletableFuture<Integer> t1 = CompletableFuture.supplyAsync(() -> task1());
CompletableFuture<Integer> t2 = CompletableFuture.supplyAsync(() -> task2());
CompletableFuture<Integer> t3 = CompletableFuture.supplyAsync(() -> task3());

CompletableFuture<Void> all = CompletableFuture.allOf(t1, t2, t3);
all.join();  // 等所有完成
// 然后分别拿结果
int result = t1.join() + t2.join() + t3.join();

// anyOf：任一任务完成就返回
CompletableFuture<Object> any = CompletableFuture.anyOf(t1, t2, t3);
Object firstResult = any.join();  // 最快的那个任务的结果
```
#### 场景示例：并行查多个接口
```
// 判题平台：并行查询题目、用户信息、判题结果
public UserJudgeInfo getUserJudgeInfo(Long userId, Long questionId) {
    // 并行执行三个查询
    CompletableFuture<User> userFuture =
        CompletableFuture.supplyAsync(() -> userService.findById(userId), pool);

    CompletableFuture<Question> questionFuture =
        CompletableFuture.supplyAsync(() -> questionService.findById(questionId), pool);

    CompletableFuture<List<JudgeRecord>> recordsFuture =
        CompletableFuture.supplyAsync(() -> judgeService.findByUserAndQuestion(userId, questionId), pool);

    // 等三个都完成，组装结果
    return CompletableFuture.allOf(userFuture, questionFuture, recordsFuture)
        .thenApply(v -> new UserJudgeInfo(
            userFuture.join(),
            questionFuture.join(),
            recordsFuture.join()
        ))
        .join();
}
// 原来串行 300ms → 并行后 100ms（提升 3 倍）
```
### 异常处理
三种处理方式
```
// ① exceptionally：异常时提供默认值（类似 catch）
CompletableFuture<Integer> future =
    CompletableFuture.supplyAsync(() -> {
        if (Math.random() > 0.5) throw new RuntimeException("计算失败");
        return 42;
    })
    .exceptionally(ex -> {
        System.out.println("异常: " + ex.getMessage());
        return 0;  // 兜底值
    });
// 结果：42 或 0

// ② handle：无论成功失败都执行（类似 finally + 参数）
CompletableFuture<Integer> future =
    CompletableFuture.supplyAsync(() -> 42)
        .handle((result, ex) -> {
            if (ex != null) {
                System.out.println("失败: " + ex.getMessage());
                return 0;  // 失败兜底
            }
            return result * 2;  // 成功转换
        });

// ③ whenComplete：无论成功失败都执行（不改变结果）
CompletableFuture.supplyAsync(() -> 42)
    .whenComplete((result, ex) -> {
        if (ex == null) {
            System.out.println("成功: " + result);
        } else {
            System.out.println("失败: " + ex.getMessage());
        }
    });
```
对比

|方法|成功时|异常时|返回值|
|---|---|---|---|
|**thenApply**|执行|不执行（异常传递）|新结果|
|**exceptionally**|不执行|执行（返回兜底）|新结果|
|**handle**|执行|执行|新结果|
|**whenComplete**|执行|执行|原结果不变|
### ForkJoinPool
ForkJoinPool：Divide and Conquer 分治框架的线程池。把大任务拆成小任务（fork）,并行执行，再合并结果（join）。
核心特性：工作窃取（work-stealing）
```
// 每个线程有自己的双端队列
// 线程做完自己的任务后，从其他线程队列"尾部"偷任务

线程 1 队列：Task1 → Task2 → Task3
线程 2 队列：（空了）
    ↓ 线程 2 从线程 1 的尾部窃取
线程 2 队列：Task3 ←（窃取）

// 好处：负载均衡，线程不空闲
// 窃取尾部而不是头部：减少竞争
```
基本使用
```java
// 示例：并行求和
public class SumTask extends RecursiveTask<Long> {
    private final long[] array;
    private final int start, end;
    private static final int THRESHOLD = 10000;  // 阈值

    public SumTask(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        // 任务足够小 → 直接计算
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        }

        // 任务太大 → 拆分
        int mid = (start + end) / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);

        // Fork：异步执行子任务
        left.fork();
        // Join：获取子任务结果
        Long rightResult = right.compute();  // 当前线程继续算右边
        Long leftResult = left.join();       // 等左边结果

        return leftResult + rightResult;
    }
}

// 使用
long[] array = new long[1000000];
ForkJoinPool pool = new ForkJoinPool(4);  // 4 个线程

SumTask task = new SumTask(array, 0, array.length);
long result = pool.invoke(task);  // 提交并等待结果
```
ForkJoinPool vs 普通线程池

|对比|ThreadPoolExecutor|ForkJoinPool|
|---|---|---|
|**任务拆分**|❌ 不支持|✅ 支持分治|
|**队列**|单一共享队列|每个线程独立双端队列|
|**负载均衡**|队列竞争|工作窃取|
|**适用**|普通任务|可拆分的计算任务|
|**CompletableFuture 默认**|—|使用 commonPool|
注意事项
```
// ① ForkJoinPool.commonPool 是全局共享的
// 默认线程数 = CPU 核数 - 1
// 不要在里面做阻塞操作！

// ② 任务要足够大才有拆分价值
// 小任务拆分反而更慢（线程切换开销）

// ③ 使用场景少
// 一般用 CompletableFuture 就够了
// 只有明确的分治场景才用 ForkJoin
```
### 对比总结
Future vs CompletableFuture

|对比|Future|CompletableFuture|
|---|---|---|
|**获取结果**|get() 阻塞|join()/回调|
|**完成回调**|❌|✅ thenApply/thenAccept|
|**组合任务**|难|✅ thenCombine/allOf|
|**异常处理**|ExecutionException|✅ exceptionally/handle|
|**异步编排**|❌|✅|
使用建议
简单异步任务？         → CompletableFuture.supplyAsync
任务串联？            → thenApply / thenCompose
任务并行合并？         → thenCombine / allOf
任务取最快？          → anyOf
异常兜底？            → exceptionally / handle
大任务分治计算？       → ForkJoinPool（RecursiveTask）