## 版本节奏&Java8
版本节奏->Java8核心特性->Lambda->Stream->Optional->新日期API->接口变化->面试高频题
### 版本节奏
为什么Java版本迭代这么快？
```
// 2017 年前：大版本慢（Java 6→7 用了 5 年，7→8 用了 3 年）
// 2017 年后：每 6 个月一个版本（快速迭代）

// 原因：
// ① Oracle 改变发布策略 —— 固定节奏，定期交付
// ② 小步快跑 —— 新特性不用等大版本
// ③ 社区压力 —— 被 Kotlin、Go 等语言追赶
```
LTS与非LTS

|版本|类型|说明|
|---|---|---|
|**Java 8**|LTS|最经典，企业标配（2014）|
|Java 9~10|非 LTS|模块化、var|
|**Java 11**|LTS|第一个长期支持（2018）|
|Java 12~16|非 LTS|Records、switch 表达式|
|**Java 17**|LTS|目前主流（2021）|
|Java 18~20|非 LTS|—|
|**Java 21**|LTS|虚拟线程（2023）|
```
// 面试回答模板：
// "Java 现在每 6 个月发布一个版本，LTS 每两年一个。
// 生产环境只用 LTS（8、11、17、21），
// 目前主流是 17，新项目开始用 21。"
```
各版本默认GC
```
// JDK 8：Parallel
// JDK 9~16：G1
// JDK 17+：G1（ZGC 可选）
// CMS：JDK 9 废弃，JDK 14 移除
```
### Java8特性总览
```
// Java 8 是革命性版本，四大核心：
// ① Lambda 表达式 —— 函数式编程
// ② Stream API —— 集合的流式处理
// ③ Optional —— 防空指针
// ④ 新日期 API —— 解决 Date 的痛点

// 另外：
// ⑤ 接口的 default/static 方法
// ⑥ CompletableFuture
// ⑦ 方法引用
// ⑧ 重复注解
```
### Lambda表达式
基本语法
```
// 语法：参数 -> 方法体

// 传统匿名类：
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
};

// Lambda：
Runnable r2 = () -> System.out.println("Hello");

// 带参数：
Comparator<String> c1 = (a, b) -> a.length() - b.length();

// 多行体：
BiFunction<Integer, Integer, Integer> add = (a, b) -> {
    int result = a + b;
    return result;
};
```
函数式接口
```
// Lambda 只能用于函数式接口（只有一个抽象方法的接口）

// 内置四大函数式接口（必须记住）：
// ① Predicate<T>：判断 —— boolean test(T t)
Predicate<String> notEmpty = s -> !s.isEmpty();

// ② Consumer<T>：消费 —— void accept(T t)
Consumer<String> print = s -> System.out.println(s);

// ③ Function<T, R>：转换 —— R apply(T t)
Function<String, Integer> length = s -> s.length();

// ④ Supplier<T>：生产 —— T get()
Supplier<Double> random = () -> Math.random();

// @FunctionalInterface 注解：检查是否函数式接口
@FunctionalInterface
interface MyInterface {
    void doSomething();
    // 两个抽象方法 → 编译报错
}
```
一张图总结
```
		 ┌─────────────────────────────────────────────┐
         │          四大函数式接口                       │
         ├──────────┬──────────┬──────────┬────────────┤
         │Predicate │ Consumer │ Function │  Supplier  │
         ├──────────┼──────────┼──────────┼────────────┤
    输入  │    T     │    T     │    T     │   (无)     │
         ├──────────┼──────────┼──────────┼────────────┤
    输出  │ boolean  │  void    │    R     │     T     │
         ├──────────┼──────────┼──────────┼────────────┤
    典型  │ filter() │forEach() │  map()   │orElseGet() │
    场景  │ 筛选判断  │ 遍历消费  │  类型转换 │  延迟提供    │
         └──────────┴──────────┴──────────┴────────────┘
```
方法引用
```
// 方法引用：Lambda 的简写
// 四种形式：

// ① 静态方法引用：类名::静态方法    参数是给方法的
Function<String, Integer> f1 = Integer::parseInt;  // 等价 s -> Integer.parseInt(s)

// ② 实例方法引用：实例::方法
String str = "hello";
Supplier<Integer> s1 = str::length;  // 等价 () -> str.length()

// ③ 任意对象方法：类名::实例方法      第一个参数是方法调用方
Function<String, Integer> f2 = String::length;  // 等价 s -> s.length()

// ④ 构造器引用：类名::new
Supplier<List<String>> s2 = ArrayList::new;  // 等价 () -> new ArrayList<>()
```
### Stream API
流是什么？
```
// Stream：对集合的声明式处理（函数式）
// 类似 SQL：过滤、排序、聚合一步到位

// 三个步骤：
// ① 创建流（数据源）
// ② 中间操作（过滤、映射等，懒执行）
// ③ 终止操作（收集结果，触发执行）
```
常规操作
```
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// ① 过滤 filter（Predicate）
numbers.stream()
    .filter(n -> n % 2 == 0)        // 偶数：2,4,6,8,10
    .forEach(System.out::println);

// ② 映射 map（Function）
numbers.stream()
    .map(n -> n * n)                // 平方
    .collect(Collectors.toList());  // [1,4,9,...]

// ③ 去重 distinct
Arrays.asList(1, 2, 2, 3, 3, 3).stream()
    .distinct()                     // [1,2,3]

// ④ 排序 sorted
numbers.stream()
    .sorted((a, b) -> b - a)        // 降序

// ⑤ 聚合 reduce
int sum = numbers.stream()
    .reduce(0, (a, b) -> a + b);    // 55

// ⑥ 统计
long count = numbers.stream().count();
Optional<Integer> max = numbers.stream().max(Integer::compareTo);
Optional<Integer> min = numbers.stream().min(Integer::compareTo);

// ⑦ 匹配
boolean anyEven = numbers.stream().anyMatch(n -> n % 2 == 0);
boolean allPositive = numbers.stream().allMatch(n -> n > 0);
boolean noneNegative = numbers.stream().noneMatch(n -> n < 0);

// ⑧ 收集
List<Integer> list = numbers.stream().collect(Collectors.toList());
Set<Integer> set = numbers.stream().collect(Collectors.toSet());
Map<Integer, List<Integer>> groupBy = numbers.stream()
    .collect(Collectors.groupingBy(n -> n % 2));  // 按奇偶分组
```
流 vs 集合

|比|集合|流|
|---|---|---|
|**存储**|存数据|不存储（只是处理流程）|
|**数据源**|本身是数据|从集合/数组/IO 来|
|**复用**|可以多次遍历|只能用一次|
|**懒加载**|立即执行|中间操作懒执行|
|**并行**|手动|parallelStream() 自动|
串行vs并行
```
// 串行流
numbers.stream().filter(...).count();

// 并行流（底层 ForkJoinPool）
numbers.parallelStream().filter(...).count();

// 注意：
// ① 并行流用共享的 commonPool（线程数 = CPU 核数 - 1）
// ② 数据量小时并行反而慢（线程切换开销）
// ③ 有状态操作（sorted、distinct）并行有额外开销
// ④ 元素少 / 顺序敏感的场景不要用并行
```
### 面试高频

> **Q:** "Stream 的中间操作和终止操作区别？" 
> **A:** "中间操作返回新的 Stream，懒执行（不触发计算），如 filter/map/sorted；终止操作触发计算并产生结果，如 collect/forEach/count。没有终止操作，中间操作不执行。"

> **Q:** "stream 能重复使用吗？" 
> **A:** "不能！Stream 是一次性的，用完后关闭。再次使用要重新创建。集合可以重复遍历，Stream 不行。"
### Optional
为什么需要？
```
// 解决空指针（NPE）
// 用 Optional 显式表达"可能为空"

// ❌ 传统：
User user = findUser(1L);
if (user != null) {
    String name = user.getName();
    if (name != null) {
        System.out.println(name.length());
    }
}

// ✅ Optional：
findUserOptional(1L)
    .map(User::getName)
    .ifPresent(name -> System.out.println(name.length()));
```
常用方法
```
// ① 创建
Optional.empty();                       // 空 Optional
Optional.of(user);                      // 非空（null 会 NPE）
Optional.ofNullable(user);              // 可空（推荐）

// ② 判断
optional.isPresent();   // 是否有值
optional.isEmpty();     // Java 11+，是否为空

// ③ 取值
optional.get();                     // 取值（空则 NoSuchElementException）
optional.orElse(defaultUser);       // 空则返回默认值
optional.orElseGet(() -> new User());  // 空则通过 Supplier 获取   延迟加载，为空返回new User对象
optional.orElseThrow();             // 空则抛异常
optional.orElseThrow(IllegalArgumentException::new);

// ④ 转换
optional.map(User::getName);        // 值转换（空则返回空 Optional）
optional.flatMap(u -> findAddress(u));  // 返回 Optional 的转换

// ⑤ 消费
optional.ifPresent(u -> System.out.println(u));
optional.ifPresentOrElse(   // Java 9+
    u -> System.out.println("有值: " + u),
    () -> System.out.println("空")
);
```
Optional使用规范
```
// ✅ 正确用法：作为返回值（表示可能没有结果）
public Optional<User> findUser(Long id) {
    User user = userDao.findById(id);
    return Optional.ofNullable(user);
}

// ❌ 不要用：作为字段、方法参数
public class User {
    // ❌ 不要这样用
    private Optional<String> name;
}

// ❌ 不要用：调用链中间用 orElseGet 复杂逻辑
// ❌ 不要用：Optional.of(x).get()（多此一举）

// 核心规范：
// ① 返回值用 Optional（Java 8 风格）
// ② 字段/参数不要用 Optional
// ③ 不要对 Optional 做 == 判断
```
### 新日期API（Java.time）
为什么替代Date
```
// 旧 API 的问题：
// ① 可变（setYear 等）—— 线程不安全
// ② 月份从 0 开始（1 月 = 0）—— 反人类
// ③ 偏移量混乱（1900 年基准）
// ④ 设计混乱（Date 和 Calendar 两套）
// ⑤ SimpleDateFormat 线程不安全

// 新 API：
// ① 不可变（线程安全）
// ② 月份从 1 开始
// ③ 清晰的类设计
```
核心类
```
// ① LocalDate —— 日期（无时间）
LocalDate today = LocalDate.now();
LocalDate date = LocalDate.of(2024, 1, 1);
date.getYear();    // 2024
date.getMonth();   // JANUARY
date.getDayOfWeek();  // MONDAY

// ② LocalTime —— 时间（无日期）
LocalTime now = LocalTime.now();
LocalTime time = LocalTime.of(14, 30, 0);

// ③ LocalDateTime —— 日期 + 时间
LocalDateTime dt = LocalDateTime.now();
LocalDateTime.parse("2024-01-01T14:30");

// ④ Instant —— 时间戳
Instant.now();  // 面向机器的时间

// ⑤ Duration / Period —— 时间间隔
Duration d = Duration.between(start, end);   // 秒级
Period p = Period.between(date1, date2);     // 天级

// ⑥ DateTimeFormatter —— 格式化（线程安全！）
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String str = LocalDateTime.now().format(formatter);
LocalDateTime parsed = LocalDateTime.parse("2024-01-01 14:30:00", formatter);
```
新旧对比

|旧|新|说明|
|---|---|---|
|Date|LocalDate/LocalDateTime|不可变，线程安全|
|SimpleDateFormat|DateTimeFormatter|线程安全|
|Calendar|各类专用类|更清晰|
|Timestamp|Instant|时间戳|
### 接口变化
Java8接口的新方法
```
// ① default 方法
public interface Flyable {
    void fly();

    default void glide() {
        System.out.println("滑翔");
    }
}

// ② static 方法
public interface Flyable {
    static boolean canFly(Object obj) {
        return obj instanceof Flyable;
    }
}

// 作用：给接口加方法不破坏实现类
// Collection.stream() 就是这么加的
```
### 面试高频题
#### 题目1：Java8有哪些新特性
```
// Lambda、Stream、Optional、新日期 API
// 接口 default/static、CompletableFuture、方法引用
```
#### 题目2：Lambda 和匿名内部类的区别
```
// ① Lambda 不生成额外的 class 文件（invokedynamic）
// ② Lambda 不持有 this 引用（匿名类持有）
// ③ Lambda 捕获的变量必须 effectively final （事实上不可变）
// ④ 性能：Lambda 更快（不用创建内部类实例）
```
题目3：Stream和for循环的性能
```
// 简单场景：for 循环略快（Stream 有抽象层开销）
// 复杂流水线：Stream 可能更快（JIT 优化）
// 并行场景：parallelStream 明显快
// 结论：代码简洁优先，Stream 可读性更好
```
#### 题目4：Optional怎么避免NPE
```
// 用 Optional.ofNullable 包装可能为 null 的返回值
// 用 map/ifPresent/orElse 链式处理
// 不要直接 get()（会抛异常）
```
#### 题目5：新日期API和旧的区别
```
// 不可变、线程安全、月份从 1 开始、类更清晰
```
