## Object
Object类总览->equals()->hashCode()->toString->clone()->finalize()->其他方法
### Object类总览
Object类是所有类的父类。每个类都直接或间接继承Object类。
Object类的11个方法：
```java
public class Object {
    // ① 对象比较
    public boolean equals(Object obj);          // 判断两个对象是否相等
    public int hashCode();                      // 返回哈希码

    // ② 对象字符串表示
    public String toString();                   // 返回对象的字符串表示

    // ③ 对象克隆
    protected native Object clone() throws CloneNotSupportedException;

    // ④ 线程相关
    public final native void notify();          // 唤醒一个等待线程
    public final native void notifyAll();       // 唤醒所有等待线程
    public final native void wait(long timeout) throws InterruptedException;
    public final void wait(long timeout, int nanos) throws InterruptedException;
    public final void wait() throws InterruptedException;

    // ⑤ 垃圾回收
    protected void finalize() throws Throwable; // 已废弃（Java 9+）

    // ⑥ 获取 Class 对象
    public final native Class<?> getClass();
}
```
### equals()
比较对象等，默认实现：
```java
// Object 的 equals —— 比较的是引用地址
public boolean equals(Object obj) {
    return (this == obj);  // 就是 ==
}

User user1 = new User("张三", 25);
User user2 = new User("张三", 25);

System.out.println(user1 == user2);        // false —— 不同对象
System.out.println(user1.equals(user2));   // false —— Object 默认也是比较地址
```
重写需要遵守的规则
```java
public class User {
    private String name;
    private int age;

    @Override
    public boolean equals(Object o) {
        // ① 自反性：x.equals(x) 必须为 true
        if (this == o) return true;  // 同一个对象

        // ② 非空性：x.equals(null) 必须为 false
        if (o == null || getClass() != o.getClass()) return false;  // 类型检查

        // ③ 对称性：x.equals(y) == y.equals(x)
        // ④ 一致性：多次调用结果不变（依赖的字段不能变）
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
    }

    // ⑤ 传递性：x.equals(y) && y.equals(z) → x.equals(z)
}
```

| 规则    | 说明                            | 违背后果                                                   |
| --- | ----------------------------- | ------------------------------------------------------ |
| 自反性   | x.equals(x) 必须为true           | 集合用contains判断是否存在会有问题                                  | | 非空性   | x.equals(null) 必须为false       | 否则会 NPE                                                |
| 对称性   | x.eqyals(y)  和y.equals(x)结果一致 | 两个对象互相比较结果相同                                           |
| 一致性   | 字段不变时，多次调用的结果不变               | 使用了可变字段例如 Date ，修改了内容                                  |
| 传递性   | x=y ,y=z =>x=z                | 在继承中容易违背，p.equals(c) 是true,但子类扩展了父类，c.equals(p) 是false |
典型错误：
```java
// ❌ 错误：用 instanceof 做类型检查
public boolean equals(Object o) {
    if (o instanceof User) {  // 问题：子类 instanceof 父类也是 true
        // ...
    }
}

// ✅ 正确：用 getClass() 做类型检查
public boolean equals(Object o) {
    if (o == null || getClass() != o.getClass()) return false;
    // ...
}
//为什么，使用instanceof会导致对称性被破坏
public class Employee extends User {
    private String department;
}

User user = new User("张三", 25);
Employee emp = new Employee("张三", 25, "技术部");

user.equals(emp);  // true —— User 用 instanceof，emp 是 User 的子类
emp.equals(user);  // false —— Employee 检查了 department，user 没有 department
// 对称性被破坏了！
```
用Object.equals()简化
```java
// Java 7+ 提供了 Objects.equals() —— 自带 null 安全
public class User {
    private String name;
    private int age;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
        // Objects.equals(a, b) 等价于 a == null ? b == null : a.equals(b)
    }
}
```
### hashCode()
哈希码默认实现
```java
// Object 的 hashCode —— 默认是内存地址的整数表示（不是绝对地址）
@HotSpotIntrinsicCandidate 
public native int hashCode();
//是native方法，从对象头的MarkWord中获取，获取不到，基于内存地址生成，再存到对象头
// 不同对象的 hashCode 通常不同，但 JVM 不保证唯一
```
equals和hashCode的约定
1.如果两个对象equals相同，那么hashCode一定也相同
2.如果两个对象hashCode相同，equals不一定相同
3.重写equals方法比须重写hashCode方法
```java
// ❌ 错误：只重写 equals，不重写 hashCode
public class User {
    private String name;

    @Override
    public boolean equals(Object o) {
        // 只重写了 equals
    }
    // 没重写 hashCode —— 后果：HashSet 认为两个相等的对象是不同对象
    // 复用了Object的hashCode方法，那么就会出现下面的现象
    // 使用
    Set<User> userSet = new HashSet<>();
    userSet.add(new User("张三"));
    userSet.add(new User("张三"));
    System.out.println(userSet.size());  // 2 —— 明明相等，但 hashCode 不同，被当作两个对象
}
// ✅ 正确：同时重写 equals 和 hashCode
public class User {
    private String name;
    private int age;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
    }

    @Override
    public int hashCode() {
        // 用相同的字段计算 hashCode   equals用到的字段，hashCode也要用到
        return Objects.hash(name, age);
        // 等价于：name != null ? name.hashCode() : 0 + age
    }
}
```
Object.hash()
```java
// Objects.hash() 内部用的是 31
public static int hash(Object... values) {
    return Arrays.hashCode(values);
}

// Arrays.hashCode 内部：       String的HashCode也是31
public static int hashCode(Object a[]) {
    int result = 1;
    for (Object element : a)
        result = 31 * result + (element == null ? 0 : element.hashCode());
    return result;
}
//为什么是31？
1.质数，乘法结果不容易哈希碰撞
2.性能，31*i可以被优化为i<<5-i,位运算比乘法快
```
哈希碰撞
```java
// 不同对象 hashCode 可能相等 —— 这叫哈希碰撞
String s1 = "Aa";
String s2 = "BB";
System.out.println(s1.hashCode());  // 2112
System.out.println(s2.hashCode());  // 2112 —— 撞了！

// 这就是 HashMap 内部用链表/红黑树的原因
```
### toString()
字符串表示，默认实现
```java
// Object 的 toString
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}

// 输出类似：com.example.User@1a2b3c4d
```
重写建议
```java
// ✅ 推荐：重写 toString 方便调试
public class User {
    private String name;
    private int age;

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

// 输出：User{name='张三', age=25}

// 使用 Lombok 可以一键生成
@ToString
public class User {
    private String name;
    private int age;
}
```
**面试高频：**

> **Q:** "为什么建议重写 toString()？" 
> **A:** "方便调试和日志输出。如果不重写，打印对象看到的是 `User@1a2b3c4d`，完全不知道里面有什么数据。重写后能直接看到字段值，排查问题方便得多。"
### clone()
对象克隆，默认实现
```java
// Object 的 clone —— 浅拷贝
protected native Object clone() throws CloneNotSupportedException;

// 使用条件：必须实现 Cloneable 接口（标记接口）
public class User implements Cloneable {
    private String name;
    private int age;

    // 重写 clone，扩大访问权限为 public
    @Override
    public User clone() {
        try {
            return (User) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();  // 不会发生
        }
    }
}
```
浅拷贝 vs 深拷贝
```java
public class Address {
    private String city;
    private String street;
    // getter/setter...
}

public class User implements Cloneable {
    private String name;
    private Address address;  // 引用类型

    @Override
    public User clone() {
        try {
            return (User) super.clone();  // 浅拷贝
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}

// 浅拷贝的问题
User user1 = new User("张三", new Address("北京", "长安街"));
User user2 = user1.clone();

user2.getAddress().setCity("上海");  // 修改 user2 的地址
System.out.println(user1.getAddress().getCity());  // "上海" —— user1 也被改了！
// 因为浅拷贝只复制了引用，两个 User 共享同一个 Address 对象

//深拷贝实现
// 方式一：重写 clone 进行深拷贝
public class User implements Cloneable {
    private String name;
    private Address address;

    @Override
    public User clone() {
        try {
            User cloned = (User) super.clone();
            cloned.address = this.address.clone();  // 手动深拷贝引用类型
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}

// 方式二：用构造器  new Person(p1)
public User(User other) {
    this.name = other.name;
    this.address = new Address(other.address.getCity(), other.address.getStreet());
}
//静态工厂方法 Person.copy(p1)
public static Person copy(Person other) {
        return new Person(
            other.name,
            other.age,
            new Address(other.address)
        );
    }

// 方式三：序列化（最彻底，但性能差）
public User deepClone() {
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    try (ObjectOutputStream oos = new ObjectOutputStream(bos)) {
        oos.writeObject(this);//把对象转为字节流，JVM递归遍历对象，所有字段被序列化
        try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()))) {
            return (User) ois.readObject(); //从字节流读出来
        }
    }
}
```
### 面试高频

> **Q:** "clone() 是浅拷贝还是深拷贝？" 
> **A:** "Object 的 clone() 默认是**浅拷贝**——基本类型复制值，引用类型复制引用。要实现深拷贝需要手动重写。"

> **Q:** "为什么 Cloneable 是空接口（标记接口）？" 
> **A:** "Cloneable 告诉 Object 的 clone() 方法：这个类允许克隆。如果不实现 Cloneable 就调用 clone()，会抛 `CloneNotSupportedException`。"

> **Q:** "推荐用 clone() 吗？" 
> **A:** "不推荐。clone() 机制有缺陷——不调用构造器、浅拷贝问题、`Cloneable` 是标记接口不够灵活。推荐用**拷贝构造器**或**静态工厂方法**替代。"
### finalize()
```java
// Object 的 finalize —— 对象被 GC 回收前调用
// Java 9+ 已标记为 @Deprecated(forRemoval = true)
@Deprecated(since = "9")
protected void finalize() throws Throwable {
    // 默认什么都不做
}
//为什么不推荐？
1.执行时机不确定
2.性能问题，实现了finalize的对象，GC需要多轮处理
3.异常问题，finalize抛异常会被忽略，不会影响GC
```
替代方案
```java
// ✅ 正确的资源释放方式
// 方式一：AutoCloseable + try-with-resources
public class DatabaseConnection implements AutoCloseable {
    private Connection conn;

	//这个是实现AutoCloseable的重写关闭方法，如果是自定义连接需要实现这个接口
    @Override
    public void close() {
        if (conn != null) {
            conn.close();  // 确定性的释放
        }
    }
}
//如果是jdk的连接已经实现了AutoCloseable，不需要再实现可直接使用，如果()内的资源没有实现，会编译报错。必须要实现AutoCloseable/Closeable接口
try (DatabaseConnection db = new DatabaseConnection()) {
    // 使用 db
}  // 自动调用 close()

// 方式二：Cleaner（Java 9+）  将要关闭的资源注册到CLEANER
public class CleanableResource {
    private static final Cleaner CLEANER = Cleaner.create();
    private final Cleaner.Cleanable cleanable;

    public CleanableResource() {
        this.cleanable = CLEANER.register(this, () -> {
            // 清理逻辑
            System.out.println("资源被清理了");
        });
    }
}
```
### 其他方法
getClass()：获取类Class对象
```java
// final 方法，不能被重写
public final native Class<?> getClass();

User user = new User();
Class<?> clazz = user.getClass();
System.out.println(clazz.getName());  // "com.example.User"
```
线程相关方法
```java
// 这三个方法必须在 synchronized 块中调用
public final native void notify();          // 随机唤醒一个等待线程
public final native void notifyAll();       // 唤醒所有等待线程
public final native void wait(long timeout) throws InterruptedException;  // 等待

// 它们属于 Object 而不是 Thread —— 因为锁是对象级别的
// 面试题：为什么 wait/notify 定义在 Object 里？
// 答：因为 Java 的锁是对象级别的（synchronized 锁对象），
//    每个对象都可以作为锁，所以 wait/notify 应该属于所有对象，
//    而不是 Thread 类。
```
