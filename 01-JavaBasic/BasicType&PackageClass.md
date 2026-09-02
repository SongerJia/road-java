## 基本类型和包装类
8种基本类->包装类->自动拆箱/装箱->Integr缓存池->性能对比->面试题陷阱
### 8种基本类型
|类型|大小|默认值|范围|包装类|
|---|---|---|---|---|
|`byte`|8 位|`0`|-128 ~ 127|`Byte`|
|`short`|16 位|`0`|-32,768 ~ 32,767|`Short`|
|`int`|32 位|`0`|-2^31 ~ 2^31-1|`Integer`|
|`long`|64 位|`0L`|-2^63 ~ 2^63-1|`Long`|
|`float`|32 位|`0.0f`|约 ±3.4E-38 ~ ±3.4E+38|`Float`|
|`double`|64 位|`0.0d`|约 ±1.7E-308 ~ ±1.7E+308|`Double`|
|`char`|16 位|`'\u0000'`|0 ~ 65,535（Unicode）|`Character`|
|`boolean`|未严格定义|`false`|true / false|`Boolean`|
几个细节
```java
// ① long 的字面量必须加 L
long l1 = 123456789;        // ✅ 虽然编译通过，但右侧是 int 范围
long l2 = 123456789L;       // ✅ 推荐写法，明确是 long

// ② float 的字面量必须加 f
float f1 = 3.14;            // ❌ 编译报错，3.14 是 double 不能隐式转 float
float f2 = 3.14f;           // ✅

// ③ boolean 的大小
// JVM 规范没说 boolean 占多大
// 独立使用时：编译为 int（4 字节）
// 在 boolean 数组中：每个占 1 字节（byte）

// ④ char 可以当整数用
char c = 'A';
int i = c;                  // 65，自动类型提升
char c2 = (char) (c + 1);   // 'B'
```
### 面试追问

> **Q:** "Java 为什么保留基本类型，不像有些语言全是对象？" 
> **A:** "性能。基本类型在栈上分配，没有对象头、GC 等开销。如果全是 `Integer`，一个 `int` 要多占 4~5 倍内存，GC 压力也大。"
### 包装类
为什么要有包装类？
```java
// ① 泛型只能用引用类型，不能用基本类型
List<int> list = new ArrayList<>();   // ❌ 编译报错
List<Integer> list = new ArrayList<>();  // ✅

// ② 集合框架只能存对象
Map<String, Integer> map = new HashMap<>();
map.put("age", 25);  // int 自动装箱为 Integer

// ③ 需要 null 值表示"无"
public class User {
    private Integer score;  // null 表示"未录入"，0 表示"考了 0 分"
    // 如果用 int，默认值是 0，无法区分"未录入"和"考了 0 分"
}

// ④ 工具方法
Integer.parseInt("123");
Integer.valueOf(123);
String.valueOf(123);
```
包装类的不可变性
```java
// 包装类也是不可变的（和 String 类似）
Integer i = 100;
// i 的值不能改，i += 1 会创建新的 Integer 对象

i = i + 1;  // 等价于：Integer.valueOf(i.intValue() + 1)
// 创建了新的 Integer 对象，i 指向新的对象
```
### Auto-boxing/Unboxing
语法糖
```java
// 自动装箱 —— int → Integer
Integer i = 100;  // 编译器自动生成：Integer.valueOf(100)

// 自动拆箱 —— Integer → int
int n = i;        // 编译器自动生成：i.intValue()

// 在表达式中的自动拆箱
Integer a = 10;
Integer b = 20;
Integer c = a + b;  // a 拆箱为 int → 相加 → 结果装箱为 Integer
// 等价于：Integer.valueOf(a.intValue() + b.intValue())
```
三目运算符的陷阱
```java
// ❌ 陷阱：三目运算符会触发类型提升
Map<String, Object> map = new HashMap<>();
map.put("count", 100);

// 下面这行会 NPE！
Integer result = map != null ? map.get("count") : 0;
// 为什么？因为 map.get("count") 返回 Object
// 三目运算符要求两个分支类型一致
// 0 是 int，所以 map.get("count") 被拆箱为 int
// 如果 map.get("count") 是 null，拆箱 → NullPointerException！

// ✅ 正确写法
Integer result = map != null ? (Integer) map.get("count") : 0;
// 或者
Integer result = map != null ? map.get("count") : Integer.valueOf(0);
```
在循环中的性能问题
```java
// ❌ 错误：大量自动装箱，性能差
Integer sum = 0;
for (int i = 0; i < 1000000; i++) {
    sum += i;  // 每次：拆箱 → 相加 → 装箱 → 创建新 Integer 对象
    // 等价于：sum = Integer.valueOf(sum.intValue() + i);
    // 创建了 100 万个 Integer 对象！
}

// ✅ 正确：用基本类型
int sum = 0;
for (int i = 0; i < 1000000; i++) {
    sum += i;  // 没有装箱，没有对象创建
}
```
### Integer缓存池
缓存机制
```java
// Integer 默认缓存了 -128 ~ 127 范围内的对象
// 在这个范围内，valueOf() 返回缓存对象，不会创建新对象

Integer i1 = 100;
Integer i2 = 100;
System.out.println(i1 == i2);  // true —— 同一个缓存对象

Integer i3 = 200;
Integer i4 = 200;
System.out.println(i3 == i4);  // false —— 超出缓存范围，创建了新对象

// 等价于
Integer i1 = Integer.valueOf(100);  // 返回缓存对象
Integer i2 = Integer.valueOf(100);  // 返回同一个缓存对象
Integer i3 = Integer.valueOf(200);  // new Integer(200)
Integer i4 = Integer.valueOf(200);  // new Integer(200)
```
源码分析
```java
// Integer.valueOf() 的源码
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)  // 默认 -128 ~ 127
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}

// IntegerCache 内部类
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high 可以通过 JVM 参数配置
        // -XX:AutoBoxCacheMax=1000
        int h = 127;
        String integerCacheHighPropValue = VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            int i = parseInt(integerCacheHighPropValue);
            i = Math.max(i, 127);
            h = Math.min(i, Integer.MAX_VALUE - (-low) - 1);
        }
        high = h;
        cache = new Integer[(high - low) + 1];
        int j = low;
        for (int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);
    }
}
```
各包装类的缓存范围

|包装类|缓存范围|是否可配置|
|---|---|---|
|`Integer`|-128 ~ 127|✅ `-XX:AutoBoxCacheMax=size`|
|`Long`|-128 ~ 127|❌ 固定|
|`Short`|-128 ~ 127|❌ 固定|
|`Byte`|全部（-128 ~ 127）|❌ 范围本来就小|
|`Character`|0 ~ 127|❌ 固定|
|`Boolean`|true / false|❌ 只有两个值|
|`Float`|❌ 无缓存|浮点数不缓存|
|`Double`|❌ 无缓存|浮点数不缓存|
经典面试题
```java
Integer i1 = 127;
Integer i2 = 127;
Integer i3 = 128;
Integer i4 = 128;

System.out.println(i1 == i2);  // true  —— 缓存
System.out.println(i3 == i4);  // false —— 超出缓存，不同对象

// 但是：
Integer i5 = new Integer(127);
Integer i6 = new Integer(127);
System.out.println(i5 == i6);  // false —— new 一定创建新对象，不走缓存
System.out.println(i5.equals(i6));  // true —— 比较值

// 最佳实践：永远用 equals() 比较包装类
Integer a = 1000;
Integer b = 1000;
if (a == b) {  // ❌ 可能 true 也可能 false，取决于是否在缓存内
}
if (a.equals(b)) {  // ✅ 永远正确
}
```
### 性能对比
内存占用
```java
// 基本类型在栈上(纯数值，放到栈上省事)，包装类在堆上
int i = 100;      // 4 字节，栈上
Integer i2 = 100; // 16~24 字节（对象头 + 字段 + 对齐），堆上

// Integer 对象的内存布局（32 位 JVM）：
// 对象头：8 字节（mark word 4 + class pointer 4）
// 实例数据：4 字节（int value）
// 对齐填充：4 字节
// 总计：16 字节 —— 是 int 的 4 倍！
```
性能对比数据
```java
// 求和 1000 万次
int sum = 0;  // 约 15ms

Integer sum = 0;  // 约 800ms（慢 50 倍，因为大量装箱拆箱和对象创建）
```
什么时候用基本类型，什么时候用包装类
```
// 用基本类型：
// - 局部变量、方法参数、返回值（不需要 null）
// - 数学计算频繁的场景
// - 大量数据（数组、集合元素）

// 用包装类：
// - 泛型（List<Integer>）
// - 需要 null 表示"无值"
// - 数据库字段映射（可能为 null）
// - 需要调用工具方法（parseInt、compareTo 等）
```
### 面试高频陷阱
陷阱1：== 比较
```java
Integer a = 100;
Integer b = 100;
Integer c = 200;
Integer d = 200;

System.out.println(a == b);  // true
System.out.println(c == d);  // false —— 超出缓存

// 结论：包装类用 == 比较结果不确定，永远用 equals()
```
陷阱2：自动拆箱NPE
```java
public class User {
    private Integer age;  // 可能为 null

    public boolean isAdult() {
        // ❌ NPE！age 为 null 时自动拆箱抛异常
        return age >= 18;
        // 等价于：age.intValue() >= 18
    }

    // ✅ 正确
    public boolean isAdult() {
        return age != null && age >= 18;
    }
}
```
陷阱3：三目运算符NPE
```java
// 前面已经讲过，再强调一次
Integer value = null;
// ❌ NPE
Integer result = true ? value : 0;
// 等价于：Integer.valueOf(true ? value.intValue() : 0)
```
陷阱4：混合类型比较
```java
Integer a = 1000;
Long b = 1000L;

System.out.println(a == b);          // ❌ 编译报错，不同类型不能 ==
System.out.println(a.equals(b));     // false —— 类型不同
System.out.println(a.intValue() == b.longValue());  // true —— 都用基本类型比较
```
陷阱5：数值溢出
```java
// 自动装箱的溢出陷阱
Integer a = 200;
Integer b = 200;
// 这里 a + b 自动拆箱计算，结果是 400（int）
// 再装箱为 Integer，没问题

// 但如果是 long 溢出？
int max = Integer.MAX_VALUE;  // 2147483647
int result = max + 1;         // -2147483648 —— 溢出
// 包装类也一样
Integer max2 = Integer.MAX_VALUE;
Integer result2 = max2 + 1;   // -2147483648 —— 还是溢出
// 包装类不解决溢出问题！需要 BigInteger
```
