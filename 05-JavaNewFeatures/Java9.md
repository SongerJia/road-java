模块化JPMS->JShell->集合工厂方法->其他特性->面试高频题
### JPMS 模块化
为什么需要模块化
```
// 痛点：JRE 太大
// 一个 Hello World 也要跑整个 JDK（200+ MB）
// 嵌入式设备、微服务需要精简

// 痛点：类路径冲突
// classpath 上的类没有边界
// 两个 jar 同名类 → 冲突无法检测
// 无法控制依赖可见性

// 解决：模块系统（Java Platform Module System）
// 模块 = 打包的代码 + 明确的依赖 + 明确的导出
```
模块是什么？
```
// 一个模块包含：
// ① 代码（包）
// ② module-info.java（模块描述符）
// ③ 依赖声明 + 导出声明

// module-info.java 示例：
module com.example.orderservice {
    requires java.sql;               // 依赖 java.sql 模块
    requires com.example.common;     // 依赖自己的模块

    exports com.example.orderservice.api;   // 导出 API 包（别人可以用）
    exports com.example.orderservice.dto;

    // 不导出的包 → 模块内部私有（封装性更强）
}
```
模块的三个关键词
```
// ① module：声明模块
module com.example.orderservice { ... }

// ② requires：声明依赖
requires java.sql;

// ③ exports：导出包（对外可见）
exports com.example.orderservice.api;

// 补充：
// opens：开放包（反射用，如 Hibernate）
// provides/uses：服务提供（SPI）
```
模块化的好处
```
// ① 强封装：不导出的包外部访问不了（比 public 更严）
// ② 显式依赖：编译期检查依赖（缺依赖直接报错）
// ③ 精简运行：可以用 jlink 只打包需要的模块
// ④ 安全：减少攻击面
```
模块化的影响
```
// ① 反射访问模块内部类受限
// 模块内的类，反射需要 opens 才能访问
// 影响：Hibernate、Spring 访问模块私有类需要 opens

// ② rt.jar 拆分成模块
// JDK 9 把核心库拆成 ~95 个模块（java.base、java.sql 等）

// ③ jlink：只打包需要的模块
jlink --module-path $JAVA_HOME/jmods --add-modules java.base \
      --output /path/to/custom-jre

// ④ classpath 兼容：没写 module-info 的代码当"未命名模块"处理
// 所以老项目基本无感（兼容性）
```
### 面试高频

> **Q:** "什么是模块化？有什么好处？" 
> **A:** "模块化把代码按模块组织，每个模块声明依赖（requires）和导出（exports）。好处：① 强封装（不导出的包外部不可见）；② 显式依赖（编译期检测）；③ 可以 jlink 精简 JRE；④ 解决类路径冲突。影响：反射访问模块内部需要 opens。"
### JShell
```
// JShell：Java 的 REPL（交互式编程环境）
// 不用写完整类，直接输入表达式执行

// 启动：jshell

jshell> int a = 10;
a ==> 10

jshell> int b = 20;
b ==> 20

jshell> a + b
$3 ==> 30

jshell> List.of(1, 2, 3).stream().map(x -> x * 2).forEach(System.out::println);
2
4
6
```
有什么用？
```
// ① 快速验证 API（不用写 main 方法）
// ② 学习新特性（直接试）
// ③ 写原型（快速迭代）
// ④ 调试表达式

// 注意：生产代码不用，只是开发辅助工具
// 面试一般不会深问，知道是什么就行
```
### 集合工厂方法
创建不可变集合
```
// ❌ 繁琐的旧写法
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("C");
list = Collections.unmodifiableList(list);

// 或者 Arrays.asList（但返回的列表不支持 add/remove）
List<String> list2 = Arrays.asList("A", "B", "C");
```
Java9工厂方法
```
// ✅ 一行创建不可变集合
List<String> list = List.of("A", "B", "C");
Set<String> set = Set.of("A", "B", "C");
Map<String, Integer> map = Map.of("A", 1, "B", 2);

// 超过 10 个元素的 Map：
Map.ofEntries(
    Map.entry("A", 1),
    Map.entry("B", 2)
);

// 特点：
// ① 不可变：add/remove/set 都抛 UnsupportedOperationException
// ② 不允许 null：List.of(null) 抛 NPE
// ③ 底层优化：小集合用专用实现（省内存）
```
和Arrays.asList的区别

|对比|Arrays.asList|List.of|
|---|---|---|
|**可变性**|可 set（不可增删）|完全不可变|
|**null**|允许|不允许|
|**结构**|固定大小数组视图|专用不可变实现|
|**性能**|一般|优化（小集合）|
### 其他Java9特性
接口私有方法
```
public interface MyInterface {
    default void greet() {
        log("hello");
    }

    default void farewell() {
        log("bye");
    }

    // 私有方法：多个 default 方法共享代码（Java 9+）
    private void log(String msg) {
        System.out.println(msg);
    }
}
```
try-with-resources增强
```
// Java 9 之前：try 括号里才能声明资源
// Java 9+：可以引用外部 final/effectively final 变量

BufferedReader br = new BufferedReader(...);
// Java 9+ 这样写：
try (br) {
    // 使用 br，自动关闭
}
```
diamond运算符增强
```
// Java 9+：匿名内部类也能用 diamond 菱形运算符
// 之前：
new Foo<String>() { ... }
// Java 9：
new Foo<>() { ... }
```
Optional增强
```
// Java 9+ 新方法：
Optional.empty().ifPresentOrElse(
    x -> System.out.println(x),
    () -> System.out.println("empty")
);
Optional.empty().or(() -> Optional.of("default"));  // 或操作
Optional.empty().stream();  // 转 Stream
```
### 面试高频题
题目1：Java9新特性
```
// ① 模块化 JPMS（最重要）
// ② JShell（REPL）
// ③ 集合工厂方法（List.of 等）
// ④ 接口私有方法
// ⑤ try-with-resources 增强
```
#### 题目2：模块化的影响
```
// ① 强封装（exports/opens）
// ② 显式依赖（requires）
// ③ jlink 精简 JRE
// ④ 反射受限（需要 opens）
// ⑤ 老代码无感（未命名模块兼容）
```
#### 题目3：List.of和Arrays.asList的区别
```
// List.of：完全不可变、不允许 null、专用实现
// Arrays.asList：可 set、允许 null、数组视图  返回的是 java.util.Arrays.ArrayList
```
#### 题目4：JShell是什么
```
// REPL 交互式工具
// 快速验证代码、学习新特性
// 生产不用
```
