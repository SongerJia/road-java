## Java17
密封类->switch模式匹配->强封装JDK内部API->其他特性->面试高频题
### Sealed Classes 密封类
解决什么问题？
```
// 问题：继承不受控制
public class Shape {}  // 谁都能继承
public class Circle extends Shape {}
public class Square extends Shape {}
public class HackerClass extends Shape {}  // ❌ 随便继承

// 需求：限定子类范围
// 已知子类集合时，希望禁止其他类继承
```
密封类的使用
```
// sealed：声明密封类（允许继承的列表）
// permits：允许继承的子类

public sealed class Shape permits Circle, Square, Triangle {}

// 子类必须是三种之一：
public final class Circle extends Shape {}      // final：不再被继承
public non-sealed class Square extends Shape {}  // non-sealed：重新开放继承
public sealed class Triangle extends Shape      // sealed：继续密封
        permits RightTriangle {}                 // 下一层密封

// ❌ 错误：未在 permits 列表中的类继承
public class HackerClass extends Shape {}  // ❌ 编译错误！
```
三种子类修饰符

|修饰符|含义|示例|
|---|---|---|
|**final**|继承链终止|`public final class Circle extends Shape`|
|**non-sealed**|重新开放继承|`public non-sealed class Square extends Shape`|
|**sealed**|继续密封|`public sealed class Triangle extends Shape`|
密封类的约束
```
// ① permits 列出的子类必须在同一个模块/包中
// ② 子类必须直接继承（不能隔代）
// ③ 子类必须有 final/sealed/non-sealed 修饰
// ④ 所有 permits 的子类必须被声明
// ⑤ 子类不能是匿名类、局部类等

// 简化写法：同包内可省略 permits
// 编译器自动识别同包直接子类
public sealed class Shape {}
// 同包内的直接子类自动成为 permitted 子类
```
配合模式匹配
```
// 密封类 + switch 模式匹配 = 穷尽性检查
public String describe(Shape shape) {
    return switch (shape) {
        case Circle c -> "圆形";
        case Square s -> "方形";
        case Triangle t -> "三角形";
        // 因为 Shape 密封，编译器知道子类只有这三个
        // 可以不写 default！穷尽性检查 ✅
    };
}
```
### 面试高频

> **Q:** "密封类是什么？解决什么问题？" 
> **A:** "密封类用 sealed + permits 限定继承范围，只允许指定的子类继承。解决继承不受控制的问题。子类必须是 final（终止）、non-sealed（开放）、sealed（继续密封）之一。配合 switch 模式匹配可以实现穷尽性检查（不用写 default）。"
### switch模式匹配
演进过程
```
// Java 16：switch 模式匹配预览（类型模式）
// Java 17：预览继续
// Java 21：正式版！

// 可以在 switch 中做类型匹配
```
使用（21正式）
```
// ❌ 旧方式：instanceof 一个个判断
public String describe(Object obj) {
    if (obj instanceof String s) {
        return "字符串: " + s;
    } else if (obj instanceof Integer i) {
        return "整数: " + i;
    } else if (obj instanceof Long l) {
        return "长整数: " + l;
    } else {
        return "未知类型";
    }
}

// ✅ 新方式：switch 模式匹配
public String describe(Object obj) {
    return switch (obj) {
        case String s -> "字符串: " + s;
        case Integer i -> "整数: " + i;
        case Long l -> "长整数: " + l;
        default -> "未知类型";
    };
}
```
模式匹配细节
```
// ① null 处理
// switch 模式匹配支持 null 模式（Java 21 预览）
switch (obj) {
    case null -> "null";
    case String s -> "字符串: " + s;
    default -> "其他";
}

// ② 守卫模式（Java 21 预览）
switch (obj) {
    case String s when s.length() > 5 -> "长字符串: " + s;
    case String s -> "短字符串: " + s;
    default -> "其他";
}

// ③ 记录模式（Java 21）
record Point(int x, int y) {}

Object obj = new Point(1, 2);
switch (obj) {
    case Point(int x, int y) -> System.out.println("点: " + x + ", " + y);
    default -> System.out.println("其他");
}
```
### 强封装JDK内部API
```
// JDK 9 模块化时，未导出的内部 API 默认不开放
// 但为了兼容，用 --illegal-access=permit（默认允许警告）

// Java 17 开始：强封装
// 内部 API 默认禁止反射访问
// --illegal-access 参数被移除
```
影响
```
// ① sun.misc.Unsafe 等内部 API 不可直接访问
// ② 依赖反射访问内部 API 的库可能报错

// 常见的反射访问：
// sun.misc.Unsafe（许多框架底层）
// sun.reflect 相关
// jdk.internal.* 包

// 解决：
// ① 使用公开 API 替代
// ② 用 --add-opens 打开特定包
java --add-opens java.base/java.lang=ALL-UNNAMED -jar app.jar

// ③ 用 --add-exports 导出
```
### java17其他特性
```
// ① 伪随机数生成器增强（JEP 356）
// RandomGenerator 接口统一

// ② 上下文特定反序列化过滤器（JEP 415）
// 反序列化过滤，增强安全（防反序列化攻击）

// ③ 弃用安全管理器（Security Manager）
// 为移除做准备

// ④ ZGC 改进（堆大小自适应）

// ⑤ 打包工具（JEP 392）
// jpackage：打包原生安装包（exe/dmg）

// ⑥ 默认 GC：G1（JDK 9+ 一直默认）
```
### 面试高频题
#### 题目1：Java17核心特性
```
// ① 密封类（sealed/permits）—— 最重要
// ② switch 模式匹配（预览）
// ③ 强封装 JDK 内部 API
// ④ 伪随机数生成器
// ⑤ 反序列化过滤器
```
#### 题目2：密封类和final类的区别
```
// final：完全不能继承
// sealed：只能被指定类继承（部分限制）
```
#### 题目3：sealed子类有哪三种
```
// final：终止继承
// non-sealed：开放继承
// sealed：继续密封
```
#### 题目4：强封装影响什么
```
// 反射访问 JDK 内部 API 被限制
// 用 --add-opens 打开
```
#### 题目5：Java17默认GC
```
// G1（JDK 9+ 默认都是 G1）
// 可用 ZGC（-XX:+UseZGC）
```