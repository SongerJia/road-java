## 接口 VS 抽象类
语法对比 → 设计意图 → Java 8 接口变化 → 设计模式中的运用 → 面试高频抉择
### 语法对比
```java
// 抽象类
public abstract class Animal {
    protected String name;              // 可以有成员变量

    public Animal(String name) {        // 可以有构造器
        this.name = name;
    }

    public abstract void sound();       // 抽象方法

    public void eat() {                 // 可以有普通方法
        System.out.println(name + " is eating");
    }
}

// 接口
public interface Flyable {
    // 变量只能是 public static final（常量）
    int MAX_SPEED = 100;                // 等价于 public static final int

    void fly();                         // 抽象方法（默认 public abstract）

    // Java 8+ 可以有 default 方法
    default void glide() {
        System.out.println("gliding...");
    }

    // Java 8+ 可以有 static 方法
    static void info() {
        System.out.println("Flyable interface");
    }
}
```

| 对比维度      | 接口                       | 抽象类            |
| --------- | ------------------------ | -------------- |
| 关键字       | interface                | abstract class |
| 多继承       | 多实现                      | 单继承            |
| 成员变量      | 只能是 public static final  | 任意类型           |
| 构造器       | 不能有                      | 可以有            |
| 普通方法      | 不能有（JDK8之前）              | 可以有            |
| default方法 | 可以有（JDK8+）               | 不能有            |
| static方法  | 可以有（JDK8+必须自己实现）         | 可以有            |
| private方法 | 可以有（JDK9+辅助default方法）    | 可以有            |
| 访问修饰符     | 默认public（JDK9+才有private） | 任意             |
### 设计意图
抽象类的设计意图：模版+共性
抽象类用来描述is-a的关系，提前子类的共有特征
适用场景：
1.相关类之间有大量相同代码
2.需要控制访问权限
3.需要构造器来初始化公共状态
```java
// 抽象类：提取所有动物的共同行为
public abstract class Animal {
    protected String name;
    protected int age;

    // 子类共有的具体行为
    public void sleep() {
        System.out.println(name + " is sleeping");
    }

    // 子类各自不同的行为——交给子类实现
    public abstract void sound();
}

// 具体子类
public class Dog extends Animal {
    @Override
    public void sound() {
        System.out.println("Woof!");
    }
}
```
接口的设计意图：契约+能力
接口用来描述能做什么has-a/can-do关系，定义行为契约
适用场景：
1.不相关的类需要有相同的行为
2.只定义行为契约，不关注具体实现
3.需要多继承的效果
4.面向接口编程-->实现方和调用方解耦
```java
// 接口：定义"可被序列化"的能力
public interface Serializable {
    // 只是契约，没有任何实现
}

// 接口：定义"可飞行"的能力
public interface Flyable {
    void fly();
}

// 一个类可以实现多个不相关的能力
public class Bird extends Animal implements Flyable, Serializable {
    @Override
    public void sound() { ... }

    @Override
    public void fly() { ... }
}
```
优先用接口，当有共用代码或者构造器时使用抽象类
