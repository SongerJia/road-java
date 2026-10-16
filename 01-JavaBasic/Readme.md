> **模块定位**：语言本身 + JDK 提供的语言级通用机制。 **边界**：集合框架 / 并发与线程 / JVM 内部 / Java 8+ 新特性（Lambda、Stream、Optional、Record、sealed、模块化）——均不在此模块，仅在交界处标注。

### 分层导航

|层|含义|包含组|
|---|---|---|
|**认识层**|这个语言是什么、由什么构成|① ②|
|**语法层**|代码怎么写|③ ④ ⑤ ⑥ ⑦|
|**语义层**|代码按什么规则生效|⑧ ⑨ ⑩|
|**机制层**|JDK 提供的语言级能力|⑪ ⑫ ⑬|

---

### 认识层

**① Java 平台与运行机制**

1. JDK / JRE / JVM 的包含关系与各自职责
2. 从源码到运行的链路：`.java` → `javac` → `.class` → 类加载 → 执行
3. 字节码与跨平台原理（Write Once, Run Anywhere）
4. 解释执行与 JIT 编译并存的执行模型
5. Java 版本与 LTS 节奏（8 / 11 / 17 / 21）

**② 语言规范与工具链** 
6. 常用命令工具：`javac` / `java` / `javap` / `jar` / `jshell` / `jdb` 
7. classpath 与 `package` 机制、包名与目录对应 
8. `import` 与静态导入、同名类冲突处理 
9. 标识符规则、关键字与保留字 
10. 命名规范与编码约定 
11. 注释三类：行注释、块注释、文档注释（javadoc） 
12. 断言 `assert`（开发期契约）

---

### 语法层

**③ 基本数据类型与变量** 
13. 8 种基本类型及取值范围、字面量写法（进制、下划线、`\uXXXX`） 
14. 浮点数：IEEE 754 表示、精度问题、`BigDecimal` 精确计算 
15. 类型转换：自动提升、强制转换、精度丢失与溢出；**复合赋值的隐式转换（`s += 1` 陷阱）** 
16. 包装类与自动装箱/拆箱、拆箱引发的 NPE 
17. 包装类缓存池及范围（`Integer` 缓存 -128~127；`Double`/`Float` 无缓存） 
18. 变量分类：局部 / 成员 / 静态 / 常量；作用域与生命周期

**④ 运算符与流程控制** 
19. 算术、赋值、自增自减、字符串拼接 `+`（含 `null` 处理） 
20. 位运算：`& | ^ ~ << >> >>>`
21. 逻辑运算与短路求值；三元运算符；运算符优先级 
22. `instanceof` 运算符与类型检查 
23. `if`、`switch`（**支持类型：`byte/short/char/int` → `String` → 枚举**；switch 表达式、箭头标签、fall-through 穿透） 
24. 循环：`for` / 增强 `for` / `while` / `do-while` 
25. `break` / `continue` / 标签跳转；递归基础

**⑤ 数组** 
26. 一维数组的声明、初始化、默认值；数组内存布局 
27. 多维数组与不规则数组 
28. `Arrays` 工具类与数组拷贝（`System.arraycopy` / `clone`） 
29. 数组协变与 `ArrayStoreException`

**⑥ 面向对象基础** 
30. 类与对象、构造器（重载、`this()` 调用） 
31. 封装与四种访问修饰符 
32. 继承、方法重写、`super` 
33. 多态、动态绑定、向上/向下转型 
34. 重载 vs 重写、重载解析（最具体匹配）、方法隐藏 vs 方法覆盖、静态绑定 vs 动态绑定 
35. `this` 与 `super` 关键字 
36. `static`：变量 / 方法 / 代码块 / 静态导入；静态上下文的限制 
37. `final`：类 / 方法 / 变量 
38. `==` 与 `equals` 的区别 
39. Java 的传参机制（只有值传递，引用按值传递） 
40. 可变参数（本质是数组） 
41. 深拷贝与浅拷贝

**⑦ 面向对象进阶** 
42. 抽象类与抽象方法 
43. 接口：默认方法、静态方法、隐式 `public static final` 常量（函数式接口 → 衔接新特性的 Lambda） 
44. 接口默认方法冲突的解决（`接口名.super.method()`） 
45. 抽象类 vs 接口的取舍 
46. 内部类：成员、局部、匿名、静态嵌套；匿名内部类对外部变量的捕获（effectively final） 
47. `Object` 类通用方法：`equals` / `hashCode` / `toString` / `getClass` / `clone` / `finalize` 
48. `equals` 与 `hashCode` 的契约（语言级契约；集合中的应用归集合模块） 
49. 枚举 `enum` 基础（含枚举实现接口、枚举中的方法） 
50. 类的初始化时机（首次主动使用才初始化）与对象初始化顺序（静态块 → 实例块 → 构造器） 
51. `main` 方法签名与命令行参数

---

### 语义层

**⑧ 字符串与字符编码** 
52. `String` 的不可变性及设计原因 
53. 字符串常量池与 `intern`（常量池位置随 JDK 版本变迁，交界 JVM）；编译期常量折叠 
54. `String` / `StringBuilder` / `StringBuffer` 的差异与选择 
55. 字符串常用操作、转义字符、数字与字符串互转 
56. 字符编码：ASCII / Unicode / UTF-8 / GBK；乱码成因；`char` 与码点、代理对

**⑨ 异常处理**
57. `Throwable` 体系：`Error` / `Exception` 的分类 
58. 受检异常 vs 运行时异常；`throw` / `throws` 
59. `try-catch-finally` 的执行顺序（含 `finally` 与 `return` 陷阱） 
60. `try-with-resources` 与资源关闭顺序 
61. 多异常捕获（multi-catch）与异常抑制（suppressed） 
62. 自定义异常、异常链、异常使用最佳实践

**⑩ 泛型** 
63. 泛型类 / 接口 / 方法；菱形语法与类型推断 
64. 类型擦除及其影响；桥接方法（bridge method） 
65. 通配符 `? extends` / `? super` 与 PECS 原则 
66. 受限类型参数 `<T extends X>`；泛型的不变性；泛型不能用基本类型 
67. 泛型与数组、可变参数的边界（堆污染 heap pollution 与 `@SafeVarargs`）

---

### 机制层

**⑪ 注解** 
68. 内置注解：`@Override` / `@Deprecated` / `@SuppressWarnings` / `@FunctionalInterface` 
69. 元注解：`@Retention` / `@Target` / `@Documented` / `@Inherited` 
70. 自定义注解与通过反射读取注解

**⑫ 反射** 
71. `Class` 对象及三种获取方式 
72. 动态创建对象、调用方法、访问字段；`setAccessible` 与访问控制 
73. 获取类结构信息（方法 / 字段 / 构造器 / 修饰符）；反射的性能开销

**⑬ IO 与序列化** 
74. 流的概念与分类（字节 / 字符、输入 / 输出、节点 / 处理） 
75. 字节流：`InputStream` / `OutputStream`；字符流：`Reader` / `Writer` 
76. 缓冲流、转换流，及 IO 中的装饰器模式 
77. 标准流：`System.in` / `out` / `err`
78. `File` 与路径处理；NIO.2 的 `Path` / `Files` 
79. 序列化：`Serializable`、`serialVersionUID`、`transient` 
80. NIO 基础：`Buffer` / `Channel` / `Selector`、内存映射（与网络编程交叉，仅作了解）