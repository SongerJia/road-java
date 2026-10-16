> **模块边界**：接口默认方法 → 基础模块；`Optional` 在集合/反射中的落地 → 各自模块；虚拟线程的调度实现、ZGC 分代 → JVM / 并发模块（本模块只讲 API 与语义）。 **版本地图**：8 = 函数式与流 · 11 = 实用化 · 17 = 数据导向起步 · 21 = 并发革命。

**① Lambda 与函数式编程**

1. Lambda 表达式语法
2. 函数式接口与 @FunctionalInterface
3. 四大核心函数式接口（Function / Consumer / Supplier / Predicate）
4. 函数式接口的复合方法（and / or / negate / andThen / compose）
5. Lambda 的类型推断与变量捕获（effectively final）
6. 方法引用的四种形式
7. Lambda vs 匿名内部类（this 指向、invokedynamic 编译产物）
8. Lambda 的性能（invokedynamic 引导开销与预热）

**② Stream API** 9. Stream 的创建方式 10. 中间操作（filter / map / distinct / sorted / peek / limit / skip） 11. flatMap vs map 12. 终端操作（collect / forEach / reduce / count / find / anyMatch） 13. 惰性求值与流的一次性 14. Collectors 常用收集器与坑（toMap key 冲突、groupingBy 下游） 15. Collectors 进阶（teeing、下游收集器嵌套） 16. 并行流：适用场景、分割性差异、有状态操作、commonPool 17. parallelStream 与 ForkJoinPool.commonPool 的关系（交界并发）

**③ Optional** 18. Optional 的创建与使用 19. Optional 的禁忌与 orElse vs orElseGet

**④ 日期时间 API** 20. 旧日期 API 的问题 21. LocalDate / LocalTime / LocalDateTime 22. Instant / Duration / Period 23. DateTimeFormatter / ZonedDateTime 与时区处理 24. 与旧 API 的互转

**⑤ 异步与并发增强** 25. CompletableFuture 基础 26. CompletableFuture 编排（thenApply / thenCompose / thenCombine / allOf） 27. CompletableFuture 异常处理与默认线程池 28. **HttpClient 的 sendAsync 与 CompletableFuture 的衔接**

**⑥ Java 9 模块化** 29. JPMS：module-info.java 30. 模块的 requires / exports / opens、模块化对反射的强封装影响 31. 接口私有方法

**⑦ 主流版本特性（Java 8-11）** 32. var 局部变量类型推断（Java 10） 33. 集合工厂方法 `List.of` / `Map.of`（Java 9） 34. String 新方法（Java 11） 35. HTTP Client（Java 11） 36. Java 8 的重复注解与类型注解

**⑧ 现代版本特性（Java 12-21）** 37. switch 表达式（Java 12/13 预览 → Java 14 正式） 38. 文本块（Java 13 预览 → Java 15 正式） 39. record：能力与限制（Java 14 预览 → Java 16 正式） 40. instanceof 模式匹配（Java 14 预览 → Java 16 正式） 41. sealed 类（Java 15 预览 → Java 17 正式） 42. sealed + record 组合（代数数据类型 ADT 思路） 43. 虚拟线程：原理、适用场景、Thread.ofVirtual、synchronized pinning（Java 19 预览 → Java 21 正式） 44. 结构化并发（预览） 45. 模式匹配 switch 与记录模式、`when` 守卫（Java 21 正式） 46. 序列集合 SequencedCollection（Java 21） 47. Java 18 UTF-8 默认字符集

**⑨ 版本治理与升级** 48. 预览特性机制（--enable-preview） 49. JDK 升级策略与 LTS 选择 50. Java 17 相对 11 的移除 / 弃用项（Nashorn、SecurityManager、AOT） 51. 四个 LTS 的版本地图与演进主题