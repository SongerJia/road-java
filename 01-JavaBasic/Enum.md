## 枚举
枚举基础->枚举的本质->枚举的字段和方法->枚举的抽象方法->EnumMap/EnumSet->枚举单例
### 枚举基础
什么是枚举？
```java
// 定义枚举 —— 一组命名的常量
public enum Color {
    RED, GREEN, BLUE
}

// 使用
Color c = Color.RED;
System.out.println(c);          // "RED"
System.out.println(c.name());   // "RED"
System.out.println(c.ordinal());// 0 —— 枚举常量的顺序号
```
枚举的基本特性
```java
// ① 枚举是类型安全的 —— 不能赋不存在的枚举里的值
Color c = Color.RED;  // ✅
Color c2 = 0;         // ❌ 编译报错，不能赋 int

// ② 枚举可以用在 switch 中
Color c = Color.RED;
switch (c) {
    case RED:
        break;
    case GREEN:
        break;
    case BLUE:
        break;
}

// ③ 枚举提供了默认方法
Color[] values = Color.values();  // 所有枚举常量数组
Color c = Color.valueOf("RED");   // 根据字符串查找枚举常量
```
枚举vs静态常量
```java
// ❌ 旧方式：用静态常量
public class Color {
    public static final int RED = 0;
    public static final int GREEN = 1;
    public static final int BLUE = 2;
}
// 问题：不是类型安全，可以传任意 int
// void setColor(int c) 可以传 100

// ✅ 枚举方式
public enum Color {
    RED, GREEN, BLUE
}
// 类型安全：void setColor(Color c) 只能传 Color 类型
```
### 枚举的本质
枚举本质是 final class
```java
// 源码
public enum Color {
    RED, GREEN, BLUE
}

// 反编译后等价于
public final class Color extends java.lang.Enum<Color> {
    public static final Color RED = new Color("RED", 0);
    public static final Color GREEN = new Color("GREEN", 1);
    public static final Color BLUE = new Color("BLUE", 2);

    private static final Color[] $VALUES = {RED, GREEN, BLUE};

    private Color(String name, int ordinal) {
        super(name, ordinal);  // 调用 Enum 的构造器
    }

    public static Color[] values() {
        return $VALUES.clone();
    }

    public static Color valueOf(String name) {
        return Enum.valueOf(Color.class, name);
    }
}
```

| 特性             | 说明                               |
| -------------- | -------------------------------- |
| 隐式final        | 枚举类不能被继承，也不能继承其他类(已继承Enum类)      |
| 隐式继承Enum       | 所有枚举自动继承 java.lang.Enum          |
| 构造器private     | 枚举构造器默认private,外部不能创建            |
| 枚举常量是静态final实例 | 每个枚举常量是类的一个静态实例                  |
| 线程安全           | 枚举实例的创建由jvm保证，线程安全               |
| 序列化安全          | jvm保证枚举序列化/反序列化只返回同一个实例(不调用构造方法) |
### 面试追问

> **Q:** "枚举可以用 == 比较吗？" 
> **A:** "可以。因为枚举常量是单例的，每个枚举常量在 JVM 中只有唯一实例，`==` 和 `equals()` 结果一样，而且 `==` 更快。"
```java
Color c1 = Color.RED;
Color c2 = Color.RED;
System.out.println(c1 == c2);        // true —— 同一个实例
System.out.println(c1.equals(c2));   // true —— 效果一样

// 推荐用 ==，因为 Enum 的 equals 内部也是用 ==
// Enum.equals() 源码：
public final boolean equals(Object other) {
    return this == other;
}
```

> **Q:** "枚举的构造器为什么是 private 的？" 
> **A:** "JVM 强制枚举构造器为 private，外部不能通过 `new` 创建枚举实例。枚举实例只在类加载时由 JVM 创建，保证了单例性。"
### 枚举的字段和方法
枚举可以有构造器、字段和方法
```java
public enum Status {
    // 枚举常量 —— 可以带参数调用构造器
    PENDING(0, "待处理"),
    PROCESSING(1, "处理中"),
    COMPLETED(2, "已完成"),
    CANCELLED(3, "已取消");

    // 字段
    private final int code;
    private final String description;

    // 构造器 —— 必须是 private（可以省略不写,默认private）
    private Status(int code, String description) {
        this.code = code;
        this.description = description;
    }

    // 方法
    public int getCode() {
        return code;
    }

    public String getDescription() {
        return description;
    }

    // 根据 code 查找枚举（工具方法）
    public static Status fromCode(int code) {
        for (Status status : Status.values()) {
            if (status.code == code) {
                return status;
            }
        }
        throw new IllegalArgumentException("未知状态码: " + code);
    }
}

// 使用
Status s = Status.PENDING;
System.out.println(s.getCode());          // 0
System.out.println(s.getDescription());   // "待处理"
Status s2 = Status.fromCode(1);           // PROCESSING
```
枚举实现接口
```java
public interface Describable {
    String getDescription();
}

public enum Status implements Describable {
    PENDING(0, "待处理"),
    PROCESSING(1, "处理中"),
    COMPLETED(2, "已完成");

    private final int code;
    private final String description;

    private Status(int code, String description) {
        this.code = code;
        this.description = description;
    }

    @Override
    public String getDescription() {
        return description;
    }
}
//目的：多个枚举有相同的行为（需要统一序列化/反序列化）
```
### 枚举的抽象方法
常量特定方法
```java
public enum Operation {
    // 每个枚举常量实现自己的 apply 方法
    ADD {
        @Override
        public int apply(int x, int y) {
            return x + y;
        }
    },
    SUBTRACT {
        @Override
        public int apply(int x, int y) {
            return x - y;
        }
    },
    MULTIPLY {
        @Override
        public int apply(int x, int y) {
            return x * y;
        }
    },
    DIVIDE {
        @Override
        public int apply(int x, int y) {
            return x / y;
        }
    };

    // 抽象方法 —— 每个枚举常量必须实现
    public abstract int apply(int x, int y);
}

// 使用
Operation op = Operation.ADD;
int result = op.apply(10, 5);  // 15

Operation op2 = Operation.DIVIDE;
int result2 = op2.apply(10, 5);  // 2
```
反编译视角
```java
// 反编译后，每个枚举常量都是一个匿名内部类,枚举类的实例
public abstract class Operation extends Enum<Operation> {
    public static final Operation ADD = new Operation("ADD", 0) {
        @Override
        public int apply(int x, int y) {
            return x + y;
        }
    };
    // ... 其他常量类似

    public abstract int apply(int x, int y);
}
```
应用场景：策略模式
```java
// 枚举实现策略模式 —— 比传统策略模式更简洁
public enum PayStrategy {
    ALIPAY {
        @Override
        public void pay(double amount) {
            System.out.println("支付宝支付: " + amount);
        }
    },
    WECHAT {
        @Override
        public void pay(double amount) {
            System.out.println("微信支付: " + amount);
        }
    },
    CREDIT_CARD {
        @Override
        public void pay(double amount) {
            System.out.println("信用卡支付: " + amount);
        }
    };

    public abstract void pay(double amount);
}

// 使用
PayStrategy strategy = PayStrategy.ALIPAY;
strategy.pay(100.0);  // 动态分派，执行对应策略
```
### EnumMap和EnumSet
EnumMap专为枚举优化的Map
```java
// 普通 HashMap 存枚举 key
Map<Status, String> map1 = new HashMap<>();
// 问题：HashMap 洐生类，有哈希计算开销

// EnumMap —— 内部用数组，性能极高
Map<Status, String> map2 = new EnumMap<>(Status.class);
map2.put(Status.PENDING, "等待中");
map2.put(Status.PROCESSING, "执行中");

// EnumMap 内部实现：
// 本质是一个数组，长度 = 枚举常量个数
// key 的 ordinal() 就是数组下标  //枚举定义的顺序
// 所以 put 和 get 都是 O(1) 数组操作，没有哈希计算
```
EnumSet专为枚举优化的Set
```java
// 普通 HashSet
Set<Status> set1 = new HashSet<>();

// EnumSet —— 内部用位向量，极致性能
Set<Status> set2 = EnumSet.of(Status.PENDING, Status.PROCESSING);
Set<Status> set3 = EnumSet.allOf(Status.class);       // 所有枚举常量
Set<Status> set4 = EnumSet.noneOf(Status.class);      // 空集合
Set<Status> set5 = EnumSet.range(Status.PENDING, Status.COMPLETED);  // 范围

// EnumSet 内部实现：
// 如果枚举常量 <= 64 个，用 long 的位（1 位代表一个枚举常量）
// 如果 > 64 个，用 long[] 位数组
// 所以 add/remove/contains 都是 O(1) 位运算！
```
性能对比
```java
// HashMap<Enum, V> vs EnumMap<Enum, V>
// HashMap：需要计算 hashCode、处理哈希冲突
// EnumMap：直接数组下标访问，快 2~3 倍

// HashSet<Enum> vs EnumSet<Enum>
// HashSet：需要计算 hashCode
// EnumSet：位运算，快 5~10 倍，内存占用极小
```
### 枚举单例
为什么枚举是实现单例的最好方式
```java
// 方式一：懒汉式 —— 需要同步，有性能问题
public class Singleton1 {
    private static Singleton1 instance;
    private Singleton1() {}
    public static synchronized Singleton1 getInstance() {
        if (instance == null) instance = new Singleton1();
        return instance;
    }
}

// 方式二：双重校验锁 —— 需要 volatile，代码复杂
public class Singleton2 {
    private static volatile Singleton2 instance;
    private Singleton2() {}
    public static Singleton2 getInstance() {
        if (instance == null) {
            synchronized (Singleton2.class) {
                if (instance == null) instance = new Singleton2();
            }
        }
        return instance;
    }
}

// 方式三：枚举单例 —— 最简单，最安全
public enum Singleton {
    INSTANCE;  // 唯一的实例

    // 可以有自己的方法
    public void doSomething() {
        System.out.println("单例方法执行");
    }
}

// 使用
Singleton.INSTANCE.doSomething();
//优点
1.JVM保证线程安全。编译后，类的初始化过程被同步，同一时刻只有一个线程能执行< clinit >方法
2.JVM保证反序列化不会创建对象，而是返回已有的
3.JVM禁止通过反射创建枚举实例，在调用构造器.newInstance方法会判断类型是不是枚举，是就会 throw new IllegalArgumentException("Cannot reflectively create enum objects");
```
### 面试高频题
#### 题目1：枚举的序列化安全
```java
// 枚举序列化后反序列化，得到的还是同一个实例
public enum Singleton {
    INSTANCE;
}

// 序列化
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("singleton.ser"));
oos.writeObject(Singleton.INSTANCE);

// 反序列化
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("singleton.ser"));
Singleton s2 = (Singleton) ois.readObject();

System.out.println(Singleton.INSTANCE == s2);  // true —— 同一个对象！
// 因为 JVM 对枚举反序列化做了特殊处理，只读取类名和枚举名，然后调用Enum.valueOf()按枚举名从已有的静态字段返回已有的枚举实例,平常的类是使用无参构造得到一个新对象，然后通过反射赋值。
```
#### 枚举可以有 main 方法吗？
```java
public enum MyEnum {
    A, B, C;

    public static void main(String[] args) {
        System.out.println("枚举也可以有 main 方法！");
        for (MyEnum e : values()) {
            System.out.println(e);
        }
    }
}
// 可以！枚举本质是类，当然可以有 main 方法
```
#### values() 从哪里来的
```java
// values() 和 valueOf() 不是从 Enum 类继承的
// 而是编译器自动生成的静态方法

// 查看 Enum 源码，会发现 Enum 类没有 values() 方法
// 这是编译器注入的语法糖
```
#### 枚举和switch的配合
```java
// 枚举在 switch 中的使用很高效
// 编译器会优化为 lookup switch 或 tableswitch
// 比 if-else 链效率高得多

public String getStatusDescription(Status s) {
    switch (s) {
        case PENDING:    return "待处理";
        case PROCESSING: return "处理中";
        case COMPLETED:  return "已完成";
        default:         return "未知";
    }
}
```

