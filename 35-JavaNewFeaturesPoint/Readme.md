> 边界：接口默认方法 → 基础模块；Optional 在集合/反射中的落地 → 各自模块；虚拟线程的调度实现、ZGC 分代 → JVM/并发模块（本模块只讲 API 与语义）。**版本地图**：8=函数式与流 · 11=实用化 · 17=数据导向起步 · 21=并发革命。

---

# ① Lambda 与函数式编程

### 【#1】Lambda 表达式语法

**题型**：辨析语法形式｜原理｜场景｜判断　**别名**：同义｜Lambda 怎么写　反向｜参数类型能省略吗　场景｜单表达式 vs 语句块　对比·陷阱｜`return` 在单表达式中

code

起点：Lambda 的语法形式有哪些？
├─ 若答得含糊（降维）：先问"`(a, b) -> a + b` 里还写没写参数类型？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ `(参数) -> 表达式` / `(参数) -> { 语句块 }`？
│    ├─ 参数类型可省略（靠推断）、单参数可省括号、单表达式可省 return？
│    └─ 陷阱分支：单表达式形式里的值是"返回值"，不能写 `return`，能说清吗？
└─ 跨点长链：它能写成 Lambda，前提是实现的是哪种接口？（引 #2 函数式接口）

### 【#2】函数式接口与 @FunctionalInterface

**题型**：原理定义｜辨析注解作用｜场景｜判断　**别名**：同义｜什么是函数式接口　反向｜@FunctionalInterface 是必须的吗　场景｜自定义一个函数式接口　对比·陷阱｜只允许一个抽象方法

code

起点：什么是函数式接口？@FunctionalInterface 有什么用？
├─ 若答得含糊（降维）：先问"一个接口有几个抽象方法才能用 Lambda？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 只有一个抽象方法（SAM）？
│    ├─ @FunctionalInterface 只是编译期检查，非必须？
│    └─ 陷阱分支：接口有默认方法 / 静态方法，还能是函数式接口吗？（能，不算抽象方法）
└─ 跨点长链：它和基础模块的"接口默认方法"什么关系？（引 基础 #43）

### 【#3】四大核心函数式接口

**题型**：辨析四接口｜原理方法签名｜场景选型｜判断　**别名**：同义｜Function / Consumer / Supplier / Predicate 区别　反向｜有入参无返回用哪个　场景｜怎么记住它们　对比·陷阱｜Predicate 返回什么

code

起点：四大核心函数式接口分别是什么？
├─ 若答得含糊（降维）：先问"要接受参数、返回结果，用哪个？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ Function<T,R>（apply）、Consumer<T>（accept，无返回）、Supplier<T>（get，无参有返回）、Predicate<T>（test，返回 boolean）？
│    └─ 陷阱分支：💻 看到一个 `XXXFunction`，怎么快速判断它是不是核心四接口的变体？（前缀 + 前缀/后缀）
└─ 跨点长链：Stream 里的 map / filter 分别用哪个？（引 #10）

### 【#4】函数式接口的复合方法

**题型**：原理复合｜辨析各方法｜场景｜判断　**别名**：同义｜andThen 和 compose 区别　反向｜Predicate 怎么组合　场景｜链式判断 / 转换　对比·陷阱｜andThen vs compose 的执行顺序

code

起点：函数式接口的复合方法有哪些？andThen 和 compose 差在哪？
├─ 若答得含糊（降维）：先问"两个 Predicate 怎么'与'起来？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ Predicate：and / or / negate；Function：andThen（先自己后参数）/ compose（先参数后自己）？
│    └─ 陷阱分支：💻 andThen 和 compose 执行顺序相反，能举一反三说清吗？
└─ 跨点长链：这些复合在 Stream 的链式调用里怎么体现？（引 #10）

### 【#5】Lambda 的类型推断与变量捕获

**题型**：原理捕获｜辨析 effectively final｜场景｜判断　**别名**：同义｜Lambda 能改外部变量吗　反向｜为什么必须 effectively final　场景｜闭包怎么用　对比·陷阱｜捕获的是变量还是值

code

起点：Lambda 捕获外部变量有什么限制？
├─ 若答得含糊（降维）：先问"Lambda 里能改外部的局部变量吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 只能捕获 effectively final（不再被重新赋值）的变量？
│    └─ 陷阱分支：它捕获的是变量本身还是值拷贝？（值拷贝，生命周期不一致）
└─ 跨点长链：这和基础模块"内部类捕获"是同一规则吗？（引 基础 #46）

### 【#6】方法引用的四种形式

**题型**：辨析四形式｜原理｜场景｜判断　**别名**：同义｜方法引用有哪几种　反向｜`Class::method` 和 `instance::method` 区别　场景｜怎么把方法引用写对　对比·陷阱｜构造器引用

code

起点：方法引用有哪四种形式？
├─ 若答得含糊（降维）：先问"`String::length` 属于哪一种？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 静态方法引用（Class::staticM）、实例方法引用（instance::m）、类实例方法引用（Class::m，第一个参数作接收者）、构造器引用（Class::new）？
│    └─ 陷阱分支：`String::valueOf` 和 `System.out::println` 各是哪一种？（前者静态、后者实例）
└─ 跨点长链：方法引用和等价的 Lambda，编译产物一样吗？（引 #7）

### 【#7】Lambda vs 匿名内部类

**题型**：辨析两点差异｜原理 this 指向｜场景｜判断　**别名**：同义｜Lambda 和匿名内部类有什么区别　反向｜Lambda 里的 this 指向谁　场景｜什么时候只能用匿名内部类　对比·陷阱｜invokedynamic 产物

code

起点：Lambda 和匿名内部类有什么本质区别？
├─ 若答得含糊（降维）：先问"Lambda 里的 this 指向谁？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ Lambda 的 this 指向外层对象；匿名内部类的 this 指向它自己？
│    ├─ 编译产物：Lambda 用 invokedynamic，匿名内部类生成 class 文件？
│    └─ 陷阱分支：什么场景只能用匿名内部类、不能 Lambda？（需要状态/多方法时）
└─ 跨点长链：invokedynamic 和 JVM 的方法调用指令什么关系？（引 JVM #48）

### 【#8】Lambda 的性能

**题型**：原理引导开销｜辨析预热｜场景｜判断　**别名**：同义｜Lambda 性能怎么样　反向｜它比匿名内部类慢吗　场景｜什么时候会有性能问题　对比·陷阱｜首次引导 vs 后续

code

起点：Lambda 有性能开销吗？在哪？
├─ 若答得含糊（降维）：先问"Lambda 第一次调用会不会慢？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：首次 invokedynamic 引导有开销（生成调用点/类），预热后趋于零；和匿名内部类差别不大？
└─ 跨点长链：这和 JIT 预热什么关系？（引 JVM #47）

---

# ② Stream API

### 【#9】Stream 的创建方式

**题型**：辨析多种创建｜原理｜场景｜判断　**别名**：同义｜怎么创建 Stream　反向｜数组能转 Stream 吗　场景｜无限流怎么造　对比·陷阱｜iterate vs generate、of vs ofNullable

code

起点：创建 Stream 有哪几种方式？
├─ 若答得含糊（降维）：先问"从集合怎么创建？从数组呢？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ collection.stream()、Stream.of(...)、Arrays.stream(...)、Stream.iterate / generate（无限）、IntStream.range？
│    └─ 陷阱分支：无限流必须配合什么才安全？（limit / 短路终止操作）
└─ 跨点长链：并行流怎么创建？（引 #16）

### 【#10】中间操作

**题型**：辨析各操作｜原理惰性｜场景｜判断　**别名**：同义｜filter / map / distinct / sorted 干什么　反向｜peek 有什么用　场景｜链式操作怎么写　对比·陷阱｜中间操作 vs 终端操作

code

起点：Stream 的中间操作有哪些？它们执行吗？
├─ 若答得含糊（降维）：先问"只写 filter 不写终端操作，会执行吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ filter / map / distinct / sorted / peek / limit / skip 都是中间操作？
│    ├─ 中间操作用来"组装流水线"、是惰性的？
│    └─ 陷阱分支：peek 主要用于调试，别用它做业务副作用，为什么？
└─ 跨点长链：什么时候才真正执行？（引 #13 惰性求值）

### 【#11】flatMap vs map

**题型**：辨析两者｜原理扁平化｜场景｜判断　**别名**：同义｜flatMap 和 map 区别　反向｜什么时候用 flatMap　场景｜把多个集合合并成一个流　对比·陷阱｜map 产生"流中流"

code

起点：flatMap 和 map 有什么区别？
├─ 若答得含糊（降维）：先问"map 得到一个 Stream<Stream<X>>，想去掉一层用谁？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：map 是 1→1 映射；flatMap 是 1→N 再扁平化，能各举一个例子吗？
└─ 跨点长链：处理嵌套集合 / 拆分单词时怎么用？（引 场景）

### 【#12】终端操作

**题型**：辨析各终端操作｜原理触发执行｜场景｜判断　**别名**：同义｜有哪些终端操作　反向｜collect / reduce / forEach 区别　场景｜聚合计算结果　对比·陷阱｜findFirst vs findAny

code

起点：Stream 的终端操作有哪些？
├─ 若答得含糊（降维）：先问"想让流真正跑起来，要加什么？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ collect / forEach / reduce / count / findFirst / findAny / anyMatch / allMatch / min / max？
│    └─ 陷阱分支：findFirst 和 findAny 在并行流里行为有何不同？
└─ 跨点长链：reduce 的三参数版本是什么？（引 并行/#16）

### 【#13】惰性求值与流的一次性

**题型**：原理惰性｜辨析一次性消费｜场景｜判断　**别名**：同义｜Stream 为什么是惰性的　反向｜Stream 能重复用吗　场景｜复用流会怎样　对比·陷阱｜无终端操作不执行

code

起点：什么是惰性求值？Stream 能重复使用吗？
├─ 若答得含糊（降维）：先问"一个 Stream 被 forEach 之后还能再用吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 中间操作只是记录、终端操作才触发遍历（惰性）？
│    ├─ Stream 一次性，消费后复用抛 IllegalStateException？
│    └─ 陷阱分支：惰性 + 短路，让 limit 能提前终止，能说清示例吗？
└─ 跨点长链：这跟迭代器的"一次性遍历"什么关系？（引 集合 #2）

### 【#14】Collectors 常用收集器与坑

**题型**：原理常用收集器｜辨析坑｜场景｜判断　**别名**：同义｜Collectors 有哪些常见用法　反向｜toMap 键冲突会怎样　场景｜分组统计　对比·陷阱｜toList / toSet / joining / groupingBy

code

起点：Collectors 有哪些常用收集器？有什么坑？
├─ 若答得含糊（降维）：先问"怎么把流收集成 List？分组怎么弄？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ toList / toSet / toMap / joining / groupingBy / partitioningBy？
│    ├─ groupingBy 可带"下游收集器"？
│    └─ 陷阱分支：💻 toMap 遇到重复 key 会怎样？（抛 IllegalStateException，要传 merge 函数）
├─ 陷阱分支：toMap 的 value 为 null 也会抛异常？（会，NPE）
└─ 跨点长链：分组后继续统计怎么做？（引 #15）

### 【#15】Collectors 进阶

**题型**：原理进阶收集器｜辨析 teeing / 下游嵌套｜场景｜判断　**别名**：同义｜teeing 是干什么的　反向｜下游收集器怎么嵌套　场景｜一次遍历得出多种结果　对比·陷阱｜groupingBy + counting / mapping

code

起点：Collectors 有哪些进阶用法？
├─ 若答得含糊（降维）：先问"分组之后想再统计数量，怎么写？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 下游收集器嵌套：groupingBy(classifier, counting()) / mapping(...)？
│    └─ 陷阱分支：teeing 能一次得到两个结果，什么场景用？（平均值同时要总和与个数）
└─ 跨点长链：这些和普通"多次遍历"比，优势在哪？（一次遍历完成）

### 【#16】并行流

**题型**：原理适用场景｜辨析分割性｜场景｜判断　**别名**：同义｜并行流什么时候用　反向｜什么情况下并行流更慢　场景｜有状态操作有什么问题　对比·陷阱｜ArrayList vs LinkedList 的分割性

code

起点：并行流适合什么场景？有什么坑？
├─ 若答得含糊（降维）：先问"是不是所有流都该加 parallel()？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 适合：数据量大、可分割、计算密集、无共享状态？
│    ├─ 分割性：ArrayList / 数组好，LinkedList / Stream.iterate 差？
│    ├─ 有状态操作（sorted / distinct / limit）会破坏并行收益？
│    └─ 陷阱分支：结果依赖顺序的操作用并行流会怎样？（findFirst vs findAny / forEachOrdered）
└─ 跨点长链：它用哪个线程池？（引 #17）

### 【#17】parallelStream 与 ForkJoinPool.commonPool

**题型**：原理共用池｜辨析为何要自定义池｜场景｜判断　**别名**：同义｜并行流用哪个线程池　反向｜commonPool 大小是多少　场景｜并行流互相干扰怎么办　对比·陷阱｜commonPool 与 IO 阻塞

code

起点：parallelStream 用的线程池是什么？有什么隐患？
├─ 若答得含糊（降维）：先问"它默认用哪个池？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 默认用 ForkJoinPool.commonPool（大小 = 核数 - 1）？
│    └─ 陷阱分支：并行流里做 IO 阻塞会拖垮公共池、影响其他并行流，怎么办？（自定义池 + 提交任务）
└─ 跨点长链：ForkJoinPool 的工作窃取原理？（引 并发 #73）

---

# ③ Optional

### 【#18】Optional 的创建与使用

**题型**：原理｜辨析创建方式｜场景｜判断　**别名**：同义｜Optional 怎么用　反向｜of 和 ofNullable 区别　场景｜怎么优雅取默认值　对比·陷阱｜map / filter / orElseThrow / ifPresent

code

起点：Optional 怎么创建和使用？
├─ 若答得含糊（降维）：先问"of 和 ofNullable 有什么区别？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 创建：of（非 null）/ ofNullable / empty？
│    ├─ 使用：map / filter / flatMap / orElse / orElseGet / orElseThrow / ifPresent / ifPresentOrElse？
│    └─ 陷阱分支：Optional 的 map 遇到空会怎样？（直接返回 empty，不 NPE）
└─ 跨点长链：它想解决的核心问题是什么？（消灭嵌套判空）

### 【#19】Optional 的禁忌与 orElse vs orElseGet

**题型**：原理禁忌｜辨析 orElse / orElseGet｜场景｜判断　**别名**：同义｜Optional 有什么使用禁忌　反向｜orElse 和 orElseGet 区别　场景｜别把 Optional 当字段/参数　对比·陷阱｜orElse 一定执行

code

起点：Optional 有哪些使用禁忌？orElse 和 orElseGet 有什么区别？
├─ 若答得含糊（降维）：先问"orElse 的默认值参数会不会被执行？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 不要用它做字段 / 方法参数 / 集合元素（设计初衷只是返回值）？
│    ├─ 别动不动 .get()（等于没防）？
│    └─ 陷阱分支：💻 orElse(x) 无论有没有值都会求值 x，orElseGet(() -> x) 才惰性，差别能举例吗？
└─ 跨点长链：Optional 在集合/反射里的落地归哪个模块？（边界标注）

---

# ④ 日期时间 API

### 【#20】旧日期 API 的问题

**题型**：原理缺陷｜辨析为何重写｜场景｜判断　**别名**：同义｜Date / Calendar 有什么问题　反向｜为什么月份从 0 开始　场景｜老代码怎么迁移　对比·陷阱｜可变性与线程安全

code

起点：旧的日期 API（Date / Calendar）有什么问题？
├─ 若答得含糊（降维）：先问"Date 是线程安全的吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 可变（SimpleDateFormat 非线程安全）、设计混乱（月份从 0）、时区处理难？
│    └─ 陷阱分支：SimpleDateFormat 多线程下会出什么错？（解析结果错乱，用 DateTimeFormatter 替代）
└─ 跨点长链：新 API 怎么解决这些？（引 #21–#23）

### 【#21】LocalDate / LocalTime / LocalDateTime

**题型**：辨析三类型｜原理不可变｜场景｜判断　**别名**：同义｜三个 Local 类型怎么用　反向｜它们有时区吗　场景｜只想要日期用什么　对比·陷阱｜不可变、无时区

code

起点：LocalDate、LocalTime、LocalDateTime 有什么区别？
├─ 若答得含糊（降维）：先问"它们带时区吗？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：它们是"本地日期时间、无时区"，且不可变（操作返回新对象）？
└─ 跨点长链：要带时区用什么？（引 #23 ZonedDateTime）

### 【#22】Instant / Duration / Period

**题型**：辨析三者｜原理时间戳 vs 时长｜场景｜判断　**别名**：同义｜Instant、Duration、Period 区别　反向｜Duration 和 Period 有什么不同　场景｜算两个日期差　对比·陷阱｜时间轴 vs 日历

code

起点：Instant、Duration、Period 分别是什么？
├─ 若答得含糊（降维）：先问"哪个表示时间戳？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ Instant（时间戳，UTC）、Duration（时间量，按时/分/秒）、Period（日期量，按年/月/日）？
│    └─ 陷阱分支：Duration 和 Period 都表示"差"，区别是什么？（时间轴 vs 日历维度）
└─ 跨点长链：数据库时间戳 / 跨时区存储怎么选？（引 #23）

### 【#23】DateTimeFormatter / ZonedDateTime 与时区

**题型**：原理格式化与时区｜辨析线程安全差异｜场景｜判断　**别名**：同义｜新 API 怎么格式化　反向｜它线程安全吗　场景｜时区转换怎么做　对比·陷阱｜DateTimeFormatter vs SimpleDateFormat

code

起点：DateTimeFormatter 和 ZonedDateTime 怎么用？时区怎么处理？
├─ 若答得含糊（降维）：先问"格式化日期用什么？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ DateTimeFormatter 不可变、线程安全（对比 SimpleDateFormat）？
│    ├─ ZonedDateTime / ZoneId / OffsetDateTime 处理时区？
│    └─ 陷阱分支：跨时区换算 / 夏令时，用哪个类型最稳？（ZonedDateTime）
└─ 跨点长链：与旧 API 怎么互转？（引 #24）

### 【#24】与旧 API 的互转

**题型**：原理互转方法｜辨析｜场景｜判断　**别名**：同义｜Date 和 LocalDateTime 怎么互转　反向｜Instant 和 Date 什么关系　场景｜老代码迁移　对比·陷阱｜Instant 是桥梁

code

起点：新的日期 API 和旧 API 怎么互转？
├─ 若答得含糊（降维）：先问"Date 和 Instant 能直接转吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ Instant ←→ Date；LocalDateTime ← 转换 ← Instant（配合 ZoneId）？
│    └─ 陷阱分支：互转为什么要经过 Instant/ZoneId？（时间轴是通用桥梁）
└─ 跨点长链：迁移时有哪些坑？（时区默认值、精度）

---

# ⑤ 异步与并发增强

### 【#25】CompletableFuture 基础

**题型**：原理创建｜辨析 runAsync / supplyAsync｜场景｜判断　**别名**：同义｜CompletableFuture 怎么用　反向｜它比 Future 强在哪　场景｜异步任务怎么起　对比·陷阱｜有无返回值

code

起点：CompletableFuture 怎么创建异步任务？
├─ 若答得含糊（降维）：先问"需要返回值用什么方法？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ runAsync（无返回）/ supplyAsync（有返回）？
│    └─ 陷阱分支：相比 Future，它支持链式回调和非阻塞组合，能说清吗？
└─ 跨点长链：Future 的局限是什么？（引 并发 #79）

### 【#26】CompletableFuture 编排

**题型**：原理编排方法｜辨析 thenApply / thenCompose / thenCombine｜场景｜判断　**别名**：同义｜thenCompose 和 thenApply 区别　反向｜怎么合并两个异步结果　场景｜串联多个异步调用　对比·陷阱｜allOf / anyOf

code

起点：CompletableFuture 的编排方法有哪些？thenCompose 和 thenApply 有什么区别？
├─ 若答得含糊（降维）：先问"上一个结果要继续传给下一个异步调用，用哪个？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ thenApply（同步映射）/ thenCompose（扁平化、返回新 CF）/ thenCombine（合并两个独立 CF）？
│    ├─ allOf（等全部）/ anyOf（任一）？
│    └─ 陷阱分支：💻 thenApply 里又返回一个 CF 会得到 CF<CF<?>>，为什么要用 thenCompose？（同 flatMap）
└─ 跨点长链：和 Stream 的 map / flatMap 是不是同一套思路？（引 #11）

### 【#27】CompletableFuture 异常处理与默认线程池

**题型**：原理异常方法｜辨析默认池｜场景｜判断　**别名**：同义｜异步异常怎么处理　反向｜它默认用哪个线程池　场景｜异常吞掉了怎么排查　对比·陷阱｜exceptionally / handle / whenComplete

code

起点：CompletableFuture 怎么处理异常？默认用哪个线程池？
├─ 若答得含糊（降维）：先问"异步任务抛异常了，怎么兜底？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ exceptionally（兜底）/ handle（正常+异常统一处理）/ whenComplete（观察，不改变结果）？
│    ├─ 默认用 ForkJoinPool.commonPool（除非传了 Executor）？
│    └─ 陷阱分支：任务里抛异常但没处理，会静默丢失吗？（会，要用 join/get 或兜底）
└─ 跨点长链：生产环境为什么建议传自定义线程池？（引 #17 commonPool）

### 【#28】HttpClient 的 sendAsync 与 CompletableFuture

**题型**：原理衔接｜辨析同步/异步｜场景｜判断　**别名**：同义｜HttpClient 异步怎么用　反向｜sendAsync 返回什么　场景｜并发调多个接口　对比·陷阱｜send vs sendAsync

code

起点：HttpClient 的 sendAsync 返回什么？怎么和 CompletableFuture 衔接？
├─ 若答得含糊（降维）：先问"send 和 sendAsync 有什么区别？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：sendAsync 返回 CompletableFuture<HttpResponse>，可直接 thenApply 编排出"并发调 N 个接口再合并"？
└─ 跨点长链：怎么用 allOf 聚合多个接口结果？（引 #26）

---

# ⑥ Java 9 模块化

### 【#29】JPMS：module-info.java

**题型**：原理模块描述符｜辨析 requires/exports｜场景｜判断　**别名**：同义｜module-info.java 是什么　反向｜模块化解决了什么问题　场景｜怎么迁移到模块化　对比·陷阱｜模块 vs 包

code

起点：JPMS 是什么？module-info.java 里写什么？
├─ 若答得含糊（降维）：先问"模块和包有什么区别？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ module-info.java 声明模块名、requires（依赖）、exports（对外暴露的包）、opens？
│    └─ 陷阱分支：模块化主要解决什么？（强封装、可靠配置、依赖清晰，依赖地狱）
└─ 跨点长链：它对反射有什么影响？（引 #30）

### 【#30】requires / exports / opens 与强封装

**题型**：原理指令语义｜辨析 exports vs opens｜场景｜判断　**别名**：同义｜exports 和 opens 有什么区别　反向｜模块化后反射还能用吗　场景｜框架反射被拦怎么办　对比·陷阱｜强封装对反射的影响

code

起点：requires / exports / opens 分别是什么？模块化对反射有什么影响？
├─ 若答得含糊（降维）：先问"想让别的模块用我的包，用哪个指令？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ requires（依赖）、exports（编译期可见）、opens（运行期反射可访问）？
│    ├─ 强封装后，默认不能反射访问未 opens 的私有成员？
│    └─ 陷阱分支：老框架靠反射访问 JDK 内部类，在 JDK9+ 会怎样？（被强封装拦截，需 --add-opens）
└─ 跨点长链：这和 JVM 模块边界、反射的 setAccessible 什么关系？（引 JVM/#72）

### 【#31】接口私有方法

**题型**：原理｜辨析用途｜场景｜判断　**别名**：同义｜接口里能有私有方法吗　反向｜为什么要有它　场景｜默认方法之间复用代码　对比·陷阱｜Java 9 新增

code

起点：Java 9 给接口加了什么能力？
├─ 若答得含糊（降维）：先问"默认方法之间能互相调用吗？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：接口私有方法（Java 9）用于默认方法/静态方法之间复用逻辑，不外暴露？
└─ 跨点长链：这和基础模块接口默认方法的关系？（引 基础 #43）

---

# ⑦ 主流版本特性（Java 8-11）

### 【#32】var 局部变量类型推断

**题型**：原理｜辨析使用限制｜场景｜判断　**别名**：同义｜var 是什么　反向｜var 能用在字段/参数上吗　场景｜什么时候用 var　对比·陷阱｜var 是动态类型吗

code

起点：var 是什么？有什么限制？
├─ 若答得含糊（降维）：先问"var 是动态类型吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 局部变量类型推断，仍是静态类型（编译期确定）？
│    ├─ 只能用于局部变量（含 for、try-with-resources），不能用于字段/参数/返回？
│    └─ 陷阱分支：`var x = null;` 能编译吗？（不能，推断不出类型）
└─ 跨点长链：它和 Lambda 的类型推断是一回事吗？（引 #5）

### 【#33】集合工厂方法

**题型**：原理｜辨析不可变｜场景｜判断　**别名**：同义｜List.of / Map.of 怎么用　反向｜它们返回的能改吗　场景｜快速初始化集合　对比·陷阱｜不可变、不允许 null

code

起点：Java 9 的 List.of / Map.of 有什么特点？
├─ 若答得含糊（降维）：先问"List.of 返回的能 add 吗？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：返回真正不可变集合、不允许 null；和 Arrays.asList（固定大小视图）不同？
└─ 跨点长链：这归集合模块还是新特性？（引 集合 #52）

### 【#34】String 新方法

**题型**：辨析新方法｜原理｜场景｜判断　**别名**：同义｜Java 11 给 String 加了什么　反向｜isBlank 和 isEmpty 区别　场景｜多行字符串处理　对比·陷阱｜strip vs trim

code

起点：Java 11 给 String 加了哪些方法？
├─ 若答得含糊（降维）：先问"判断空白字符串用什么？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ isBlank / lines / strip / stripLeading / stripTrailing / repeat？
│    └─ 陷阱分支：strip 和 trim 有什么区别？（strip 按 Unicode 空白，trim 只按 <= U+0020）
└─ 跨点长链：这和基础模块 String 操作什么关系？（引 基础 #55）

### 【#35】HTTP Client

**题型**：原理｜辨析 vs HttpURLConnection｜场景｜判断　**别名**：同义｜Java 11 的 HttpClient 怎么用　反向｜它支持 HTTP/2 吗　场景｜发一个 HTTP 请求　对比·陷阱｜同步 send 和异步 sendAsync

code

起点：Java 11 的 HttpClient 有什么特点？
├─ 若答得含糊（降维）：先问"它和老的 HttpURLConnection 比强在哪？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：支持 HTTP/2、WebSocket、同步 send 与异步 sendAsync（配 CompletableFuture）？
└─ 跨点长链：异步怎么和 CompletableFuture 衔接？（引 #28）

### 【#36】重复注解与类型注解

**题型**：原理｜辨析两种注解｜场景｜判断　**别名**：同义｜重复注解怎么用　反向｜类型注解和普通注解区别　场景｜能在泛型/类型上用注解吗　对比·陷阱｜@Repeatable 容器

code

起点：Java 8 的重复注解和类型注解分别是什么？
├─ 若答得含糊（降维）：先问"同一个注解能贴两次吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 重复注解靠 @Repeatable + 容器注解实现？
│    ├─ 类型注解（ElementType.TYPE_USE）能贴在泛型、类型转换等"类型"位置？
│    └─ 陷阱分支：类型注解为谁服务？（为静态分析工具/编译器检查铺路）
└─ 跨点长链：这和基础模块注解元注解什么关系？（引 基础 #69）

---

# ⑧ 现代版本特性（Java 12-21）

### 【#37】switch 表达式

**题型**：原理｜辨析语句 vs 表达式｜场景｜判断　**别名**：同义｜switch 表达式是什么　反向｜箭头标签解决了什么　场景｜怎么用它赋值　对比·陷阱｜yield

code

起点：switch 表达式是什么？解决了什么问题？
├─ 若答得含糊（降维）：先问"switch 现在能直接赋值给变量了吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 箭头标签（不穿透）、作为表达式有返回值、用 yield 返回值（多语句块）？
│    └─ 陷阱分支：💻 用它做枚举/字符串映射，相比老 switch 好在哪？
└─ 跨点长链：和基础模块 switch 支持类型什么关系？（引 基础 #23）

### 【#38】文本块

**题型**：原理三引号｜辨析缩进处理｜场景｜判断　**别名**：同义｜文本块是什么　反向｜它怎么处理缩进　场景｜写 SQL / JSON 字符串　对比·陷阱｜行结束符归一化

code

起点：文本块（Text Blocks）是什么？注意什么？
├─ 若答得含糊（降维）：先问"多行字符串怎么写更清爽？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：`"""` 包围、自动处理缩进、行结束符归一化；不需要再写 \n 和 + 拼接？
└─ 跨点长链：和老的字符串拼接比，可读性提升在哪？（引 #34）

### 【#39】record

**题型**：原理能力与限制｜辨析 vs 普通类｜场景｜判断　**别名**：同义｜record 是什么　反向｜record 能继承吗　场景｜当 DTO / 值对象　对比·陷阱｜自动生成的方法

code

起点：record 有什么能力和限制？
├─ 若答得含糊（降维）：先问"record 帮你自动生成了什么？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 自动生成构造器、accessor、equals / hashCode / toString？
│    ├─ 隐式 final（不能被继承），字段隐式 final（不可变）？
│    └─ 陷阱分支：record 适合做可变状态吗？（不适合，它是值对象）
└─ 跨点长链：它和 sealed 怎么组合出 ADT？（引 #42）

### 【#40】instanceof 模式匹配

**题型**：原理｜辨析 vs 手动强转｜场景｜判断　**别名**：同义｜instanceof 模式匹配是什么　反向｜它能省掉强转吗　场景｜怎么写更简洁　对比·陷阱｜绑定变量作用域

code

起点：instanceof 模式匹配是什么？
├─ 若答得含糊（降维）：先问"`if (o instanceof String s)` 里的 s 是哪来的？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：匹配成功就把值绑定到 s，省掉强转；s 的作用域受限于"确定匹配"的分支？
└─ 跨点长链：和基础模块的 instanceof 什么关系？（引 基础 #22）

### 【#41】sealed 类

**题型**：原理｜辨析 permits｜场景｜判断　**别名**：同义｜sealed 类是什么　反向｜它限制什么　场景｜限定子类范围　对比·陷阱｜sealed / non-sealed / final

code

起点：sealed 类是什么？限制什么？
├─ 若答得含糊（降维）：先问"怎么限定一个类只能被哪几个类继承？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ sealed 类用 permits 显式列出允许的子类？
│    ├─ 子类必须是 final / sealed / non-sealed 之一？
│    └─ 陷阱分支：sealed 的价值是什么？（建模封闭的类层次，配合模式匹配穷举）
└─ 跨点长链：它和 record 组合能做什么？（引 #42）

### 【#42】sealed + record 组合（ADT）

**题型**：原理 ADT｜辨析代数数据类型｜场景｜判断　**别名**：同义｜sealed + record 能做什么　反向｜什么是代数数据类型　场景｜建模"要么 A 要么 B"　对比·陷阱｜和枚举的区别

code

起点：sealed + record 组合能表达什么？
├─ 若答得含糊（降维）：先问"怎么表示'一个结果要么成功要么失败'？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：sealed 定义封闭层次、record 定义各变体 → 类似 ADT（代数数据类型）；配合模式匹配可穷举？
└─ 跨点长链：这和函数式语言里的 ADT / 模式匹配思路一致吗？（引 #45）

### 【#43】虚拟线程

**题型**：原理适用场景｜辨析 pinning｜场景｜判断　**别名**：同义｜虚拟线程是什么　反向｜它和平台线程区别　场景｜什么场景用虚拟线程　对比·陷阱｜synchronized pinning

code

起点：虚拟线程是什么？适合什么场景？
├─ 若答得含糊（降维）：先问"它和普通线程（平台线程）有什么本质不同？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 由 JVM 调度（映射到少量载体线程），创建成本极低，适合高并发 IO 密集？
│    ├─ Thread.ofVirtual() / Executors.newVirtualThreadPerTaskExecutor()？
│    └─ 陷阱分支：虚拟线程遇 synchronized 会"钉住"载体线程（pinning），为什么？该怎么办？（用 ReentrantLock）
└─ 跨点长链：调度实现细节归哪个模块？（边界 → JVM/并发）

### 【#44】结构化并发

**题型**：原理｜辨析场景｜场景｜判断　**别名**：同义｜结构化并发是什么（预览）　反向｜它解决什么问题　场景｜一组子任务统一管理　对比·陷阱｜生命周期绑定

code

起点：结构化并发是什么？（预览特性）
├─ 若答得含糊（降维）：先问"一堆并发子任务，怎么保证它们一起开始、一起结束、异常一起传播？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：把子任务的生命周期绑定在一个作用域（Scope）里，避免任务泄漏、异常丢失？
└─ 跨点长链：它和 CompletableFuture 的编排思路差别？（引 #26）

### 【#45】模式匹配 switch、记录模式、when 守卫

**题型**：原理｜辨析各能力｜场景｜判断　**别名**：同义｜Java 21 的模式匹配强在哪　反向｜when 守卫是什么　场景｜对 sealed 类型做分派　对比·陷阱｜穷举与 null 处理

code

起点：Java 21 的模式匹配 switch 带来了什么？
├─ 若答得含糊（降维）：先问"switch 能直接对类型做匹配了吗？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 类型模式 + when 守卫 + 记录模式（解构 record）？
│    ├─ 对 sealed 类型可穷举、无需 default？
│    └─ 陷阱分支：💻 用它对 `sealed Result` 做分派，比 if-else 强在哪？
└─ 跨点长链：这才是 sealed + record 的真正用武之地？（引 #42）

### 【#46】SequencedCollection

**题型**：原理｜辨析新增接口｜场景｜判断　**别名**：同义｜SequencedCollection 是什么　反向｜它统一了什么　场景｜拿首尾元素　对比·陷阱｜getFirst / getLast

code

起点：Java 21 的 SequencedCollection 是什么？
├─ 若答得含糊（降维）：先问"以前拿 List 的头尾元素怎么写？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：给集合加了统一的"有序"语义与 getFirst/getLast/reversed/ addFirst 等？
└─ 跨点长链：它统一了 List 和 Deque 的顺序操作吗？（引 集合 #7 / #9）

### 【#47】Java 18 UTF-8 默认字符集

**题型**：原理｜辨析影响｜场景｜判断　**别名**：同义｜Java 18 的字符集变化　反向｜默认编码变了有什么影响　场景｜老程序迁移的坑　对比·陷阱｜file.encoding

code

起点：Java 18 在字符编码上做了什么改变？
├─ 若答得含糊（降维）：先问"默认字符集以前是什么？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：UTF-8 成为默认字符集（JEP 400），以前依赖平台默认编码的老程序可能出乱码，要显式指定？
└─ 跨点长链：和基础模块的字符编码什么关系？（引 基础 #56）

---

# ⑨ 版本治理与升级

### 【#48】预览特性机制

**题型**：原理 --enable-preview｜辨析使用限制｜场景｜判断　**别名**：同义｜预览特性怎么开启　反向｜预览特性能在生产用吗　场景｜怎么用新语法　对比·陷阱｜版本绑定

code

起点：什么是预览特性？怎么开启？
├─ 若答得含糊（降维）：先问"编译和运行都要加什么参数？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：需 --enable-preview（编译和运行都要），且和具体 JDK 版本绑定、可能变更甚至移除，不宜上生产？
└─ 跨点长链：这解释了为什么新语法要等"正式"才能放心用？（引 #49）

### 【#49】JDK 升级策略与 LTS 选择

**题型**：原理策略｜辨析 LTS 选择｜场景｜判断　**别名**：同义｜JDK 怎么升级　反向｜为什么选 LTS　场景｜你们升级踩过什么坑　对比·陷阱｜非 LTS 的价值

code

起点：JDK 升级策略怎么定？
├─ 若答得含糊（降维）：先问"生产为什么优先选 LTS？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ LTS 有长期支持与补丁；非 LTS 尝鲜但不适合长期生产？
│    └─ 陷阱分支：升级要评估什么？（GC 默认变化、移除的 API、依赖兼容、字节码版本）
└─ 跨点长链：升级后 GC/JVM 行为变化影响？（引 JVM #34）

### 【#50】Java 17 相对 11 的移除 / 弃用项

**题型**：原理移除清单｜辨析影响｜场景｜判断　**别名**：同义｜17 相对 11 删了/弃了哪些　反向｜Nashorn 去哪了　场景｜迁移前要检查什么　对比·陷阱｜SecurityManager / AOT

code

起点：Java 17 相对 11 移除了或弃用了哪些东西？
├─ 若答得含糊（降维）：先问"11 到 17，有没有用了很久的组件被拿掉？" → 爬回正题
├─ 若答得标准（升维）：
│    └─ 陷阱分支：Nashorn（JS 引擎）移除、SecurityManager 弃用、实验性 AOT/AppCDS 变化，迁移要排查？
└─ 跨点长链：这些移除和"升级策略"怎么配合？（引 #49）

### 【#51】四个 LTS 的版本地图与演进主题

**题型**：原理版本地图｜辨析演进主题｜场景｜判断　**别名**：同义｜8/11/17/21 各是什么主题　反向｜哪几个是 LTS　场景｜选哪个版本开局　对比·陷阱｜每代的主题

code

起点：四个 LTS 的版本地图和演进主题是什么？
├─ 若答得含糊（降维）：先问"哪几个是 LTS？" → 爬回正题
├─ 若答得标准（升维）：
│    ├─ 8=函数式与流、11=实用化（var/HttpClient/String 增强）、17=数据导向起步（record/sealed/模式匹配）、21=并发革命（虚拟线程/模式匹配正式/SequencedCollection）？
│    └─ 陷阱分支：新项目开局选哪个 LTS？（当前主流 17 或 21，权衡生态与特性）
└─ 跨点长链：这跟"升级策略"一起构成版本治理全貌？（引 #49）