## 字符串
String的不可变行->字符串常量池->String常用方法->StringBuilder vs StringBuffer->字符串拼接优化->面试高频题
### String的不可变性
什么是不可变？
```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    // 存储字符的数组 —— final 且 private，没有提供修改方法
    private final char value[]; //fianl修饰的如果没有直接赋值的，必须在所有的构造器都要赋值，否则会编译报错
	//怎么做到不可变的？
    //1. String 类本身是 final 的，不能被继承
    //2. value 是 final 的，引用不能变,执行的数组不能换
    //3. 不暴露修改方法，所有"修改"操作都返回新对象
}
//jdk9+ 将char value[] 变为了byte value[]
是为了省内存，char是2个字节，英文字母是1个字节，如果存储的都是英文的话使用byte比char省一半内存，jdk9+新增了一个编码标识字段 byte coder来标识是否都是latin-1字符
```
不可变的好处？
```java
// ① 常量池复用 —— 安全
String s1 = "hello";
String s2 = "hello";  // 指向同一个对象
// 如果 String 可变，s1 变了 s2 也会变，违背直觉

// ② 线程安全 —— 不需要同步
public class StringHolder {
    private String name;  // 不可变，不需要加锁

    public String getName() {
        return name;  // 直接返回，安全
    }
}

// ③ HashMap 的 key 安全
Map<String, Integer> map = new HashMap<>();
map.put("key", 100);
// 如果 String 可变，key 被改了，再也找不到这个 entry 了

// ④ 类加载机制安全
// 类名是 String，如果 String 可变，可以修改类名加载恶意类
```
### 面试追问

> **Q:** "String 真的完全不可变吗？" 
> **A:** "通过反射可以修改内部的 `value` 数组——但这是暴力破坏封装，正常编码不会这么做。"
```java
/ 反射破坏 String 不可变性（仅演示，不要在生产用）
String s = "hello";
Field valueField = String.class.getDeclaredField("value");
valueField.setAccessible(true);
char[] value = (char[]) valueField.get(s);
value[0] = 'H';  // 修改了内部数组
System.out.println(s);  // "Hello"
```
### 字符串常量池
两种创建方式的区别
```java
// 方式一：字面量 —— 放入常量池 
String s1 = "hello";
// JVM 检查常量池，如果有直接返回引用，没有则创建再放入字符串常量池

// 方式二：new —— 在堆中创建新对象
String s2 = new String("hello");
// 创建了两个对象：常量池中的 "hello" + 堆中的 String 对象

// 对比
System.out.println(s1 == s2);  // false —— 一个在常量池，一个在堆
System.out.println(s1.equals(s2));  // true —— 内容相同
```
常量池工作原理
```java
String s1 = "hello";
String s2 = "hello";
String s3 = "he" + "llo";  // 编译期常量折叠 —— 编译时就是 "hello"
String s4 = new String("hello");
String s5 = s4.intern();     // 从常量池中获取

System.out.println(s1 == s2);  // true  —— 同一个常量池对象
System.out.println(s1 == s3);  // true  —— 编译期优化，s3 也是常量池中的 "hello"
System.out.println(s1 == s4);  // false —— s4 是堆中的新对象
System.out.println(s1 == s5);  // true  —— intern() 返回常量池中的对象
//使用常量池的目的/不使用会怎么样？
目的是为了字符串复用，如果不使用，会导致相同字符串内容会反复创建，内存和GC压力变大。常量池让相同内容的字符串只存一份。
```
intern()方法
```java
// intern() —— 从常量池中获取字符串
// 如果常量池中有，直接返回；如果没有，把当前字符串放入常量池再返回

String s1 = new String("hello");
String s2 = s1.intern();  // 常量池中已有 "hello"（因为字面量加载时已经放入）
String s3 = "hello";

System.out.println(s1 == s2);  // false —— s1 是堆对象，s2 是常量池对象
System.out.println(s2 == s3);  // true  —— 都是常量池中的同一个对象

// intern() 的应用 —— 大量重复字符串时节省内存
String[] data = loadLargeData();  // 大量重复字符串
for (int i = 0; i < data.length; i++) {
    data[i] = data[i].intern();  // 只保留一份，大幅减少内存
}
```

| JDK版本 | 常量池位置        | 说明           |
| ----- | ------------ | ------------ |
| JDK6  | 永久代          | 大小固定容易OOM    |
| JDK7  | 堆中           | 移到了堆中，可以GC回收 |
| JDK8+ | 堆中(元空间代替永久代) | 常量池仍在堆中      |
### String 常用方法
```java
public class StringMethodExample {

    public static void main(String[] args) {
        String s = "  Hello, Java World!  ";

        // ① 长度与判断
        s.length();             // 21
        s.isEmpty();            // false
        s.isBlank();            // false（Java 11+，判断空白字符）

        // ② 比较
        s.equals("hello");                  // false
        s.equalsIgnoreCase("  hello, java world!  ");  // true
        s.compareTo("Hello");               //int 字典序比较,从第一个字符开始逐个比较unicode，遇到不同返回unicode差值，如果前面一致，返回长度差值

        // ③ 查找
        s.indexOf('o');         // 8
        s.lastIndexOf('o');     // 13
        s.contains("Java");     // true
        s.startsWith("  He");   // true
        s.endsWith("!  ");      // true

        // ④ 截取
        s.substring(2, 7);      // "Hello"（含头不含尾）
        s.charAt(2);            // 'H'

        // ⑤ 转换
        s.toUpperCase();        // "  HELLO, JAVA WORLD!  "
        s.toLowerCase();        // "  hello, java world!  "
        s.trim();               // "Hello, Java World!" —— 去除首尾空格
        s.strip();              // "Hello, Java World!" —— Java 11+，支持全角空格
        s.replace('o', '0');    // "  Hell0, Java W0rld!  "
        s.replaceAll("\\s+", "");  // "Hello,JavaWorld!" —— 正则

        // ⑥ 拆分与合并
        String[] parts = s.split(", ");  // ["  Hello", "Java World!  "]
        String joined = String.join("-", "2024", "01", "01");  // "2024-01-01"

        // ⑦ 判断
        s.matches(".*Java.*");  // true —— 正则匹配
        "".isEmpty();           // true
        "".isBlank();           // true（Java 11+）

        // ⑧ 转换
        String.valueOf(123);    // "123"
        Integer.parseInt("123"); // 123
        String.format("Hello, %s!", "World");  // "Hello, World!"
    }
}
```
### StringBuilder vs StringBuffer
为什么需要可变字符串
```java
// ❌ String 拼接 —— 每次创建新对象，性能差
String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;  // 每次创建新的 String 对象，O(n²) 复杂度
}

// ✅ StringBuilder —— 可变，不创建新对象
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);  // 在内部数组上追加，O(n) 复杂度
}
String result = sb.toString();
```
对比
```java
// StringBuilder —— 线程不安全，性能好
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");
sb.insert(5, ",");
sb.replace(6, 11, "Java");
sb.delete(5, 6);
String result = sb.toString();  // "HelloJava"

// StringBuffer —— 线程安全（方法加 synchronized），性能差
StringBuffer sbf = new StringBuffer("Hello");
sbf.append(" World");
String result2 = sbf.toString();
```

| 对比   | StringBuilder | StringBuffer     |
| ---- | ------------- | ---------------- |
| 线程安全 | 线程不安全         | 安全（synchronized） |
| 性能   | 快             | 慢                |
| 引入版本 | JDK5          | JDK1.0           |
| 推荐   | 单线程           | 基本不用             |
源码分析
```java
// StringBuilder 和 StringBuffer 都继承 AbstractStringBuilder
abstract class AbstractStringBuilder {
    char[] value;    // 存储字符 —— 不是 final，可以扩容
    int count;       // 已使用的字符数

    // 默认容量 16
    AbstractStringBuilder() {
        value = new char[16];
    }

    // 扩容机制：新容量 = 需要的容量
    public void ensureCapacity(int minimumCapacity) {
        if (minimumCapacity > value.length) {
            expandCapacity(minimumCapacity);
        }
    }
	//扩容机制：新容量 = 原容量 * 2 + 2
    private void expandCapacity(int minimumCapacity) {
        int newCapacity = (value.length + 1) * 2;  // 翻倍
        if (newCapacity < minimumCapacity) {
            newCapacity = minimumCapacity;
        }
        value = Arrays.copyOf(value, newCapacity);  // 扩容 + 拷贝
    }
}
```
**面试追问：**

> **Q:** "StringBuilder 的初始容量和扩容机制？" 
> **A:** "默认初始容量 16，扩容时新容量 = (原容量 + 1) * 2。如果还不够，就用需要的容量。所以如果知道大概长度，建议指定初始容量：`new StringBuilder(1000)`，避免多次扩容。"
### 字符串拼接优化
编译期常量折叠
```java
// 编译期确定 —— 编译器直接优化为 "hello world"
String s1 = "hello" + " " + "world";  // 编译后：String s1 = "hello world";

// 运行时确定 —— 不能优化
String s2 = "hello";
String s3 = s2 + " world";  // 运行时拼接，走 StringBuilder
```
反编译看本质
```java
// 源码
String s1 = "hello";
String s2 = "world";
String s3 = s1 + s2;

// 反编译后等价于（Java 5+）
String s3 = new StringBuilder()
    .append(s1)
    .append(s2)
    .toString();
```
循环中的陷阱
```java
// ❌ 错误：每次循环都创建新的 StringBuilder
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // 等价于：result = new StringBuilder().append(result).append(i).toString()
    // 每次循环：创建 StringBuilder → append → toString → 丢弃 StringBuilder
    // 1000 次循环，创建了 1000 个 StringBuilder 对象！
}

// ✅ 正确：自己创建 StringBuilder，循环外用
StringBuilder sb = new StringBuilder(1000);
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
String result = sb.toString();  // 只创建了一个 StringBuilder
//性能对比
// 拼接 10000 次
// + 拼接：约 500ms —— 创建了 10000 个 StringBuilder 对象
// StringBuilder：约 0.5ms —— 只创建了 1 个对象
// 差距：1000 倍！
```
### 高频面试题
1.经典equals题目
```java
String s1 = "hello";
String s2 = new String("hello");
String s3 = "he" + "llo";

System.out.println(s1 == s2);          // false —— 常量池 vs 堆
System.out.println(s1.equals(s2));     // true
System.out.println(s1 == s3);          // true —— 编译期常量折叠
System.out.println(s1 == s2.intern()); // true —— intern 返回常量池对象
```
2.字符串对象创建数量
```java
String s = new String("hello");
// 创建了几个对象？
// 答案：1 个或 2 个
// - 如果常量池中已有 "hello"：只创建了 1 个堆对象
// - 如果常量池中没有 "hello"：创建了 2 个（常量池 1 个 + 堆 1 个）
```
3.StringBuilder线程安全
```java
// StringBuilder 在多线程下不安全
StringBuilder sb = new StringBuilder("start");
// 线程 A：sb.append("A");
// 线程 B：sb.append("B");
// 可能结果："stABrt" —— 内部数组操作被中断，数据错乱

// 解决方案：用 StringBuffer 或自己加锁
```
4.String的split的陷阱
```java
String s = "a,b,c,";
String[] parts = s.split(",");
System.out.println(parts.length);  // 3 —— 尾部的空字符串被丢弃了！

// 如果想去掉尾部空字符串的丢弃行为，用 limit 参数
parts = s.split(",", -1);  // -1 表示保留尾部空字符串
System.out.println(parts.length);  // 4 —— ["a", "b", "c", ""]
```
5.String在switch的使用
```java
// Java 7+ String 支持 switch
String s = "hello";
switch (s) {
    case "hello":
        break;
    case "world":
        break;
}

// 反编译后：实际上用的是 hashCode() + equals()
// switch (s.hashCode()) {
//     case 99162322:  // "hello" 的 hashCode
//         if (s.equals("hello")) { ... }
//         break;
//     case 113318802: // "world" 的 hashCode
//         if (s.equals("world")) { ... }
//         break;
// }
```
