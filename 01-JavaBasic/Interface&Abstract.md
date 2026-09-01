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
	//接口只有一个抽象方法可以用lambda表达式
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

### Java8接口的变化
Java8之前接口只能有抽象方法
```java
public interface Flyable {
    void fly();  // 只能声明，不能实现  抽象方法（默认 public abstract）
}
```
java8：default方法和static方法
```java
public interface Flyable {
    void fly();

    // default 方法：提供默认实现，实现类可以选择不重写
    default void glide() {
        System.out.println("Default gliding...");
    }

    // static 方法：工具方法，接口名直接调用
    static boolean isBird(Flyable f) {
        return f instanceof Bird;
    }
}
```
default方法解决了什么问题？
解决接口演进问题，给现有的接口增加新方法，不影响原有实现
```java
// Java 7 的 Collection 接口
public interface Collection<E> {
    int size();
    boolean isEmpty();
    // ...
}

// Java 8 给 Collection 加了 stream() 方法
public interface Collection<E> {
    // 新增 default 方法，所有实现类自动获得，无需修改
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }
}
```
java9：private方法
```java
public interface Flyable {
    default void glide() {
        log("gliding...");   // 调用 private 方法
    }

    default void dive() {
        log("diving...");    // 调用 private 方法
    }

    // private 方法：多个 default 方法共享代码，不对外暴露
    private void log(String msg) {
        System.out.println("[Flyable] " + msg);
    }
}
```
### 面试高频追问

> **Q:** "default 方法有冲突怎么办？一个类实现的两个接口有相同签名的 default 方法？" 
> **A:** "必须重写解决歧义——在实现类中重写该方法，并可以用 `InterfaceName.super.method()` 指定调用哪个接口的默认实现。"
```java
public interface A {
    default void hello() {
        System.out.println("A");
    }
}

public interface B {
    default void hello() {
        System.out.println("B");
    }
}

public class MyClass implements A, B {
    @Override
    public void hello() {
        A.super.hello();  // 指定调用 A 的 default 方法
        B.super.hello();  // 也可以调 B 的
        System.out.println("MyClass");
    }
}
```
>**Q:** "接口能有 main 方法吗？" 
>**A:** "能，Java 8+ 接口可以有 static 方法，所以 `main` 方法可以直接写在接口里运行。"

### 设计模式中的运用
模版方法模式---抽象类使用
```java
// 判题流程模板
public abstract class JudgeTemplate {

    // 模板方法：定义判题流程骨架，子类不能重写
    public final JudgeResult judge(Submission submission) {
        validate(submission);           // 步骤 1：校验
        Object output = run(submission); // 步骤 2：运行（子类实现）
        JudgeResult result = check(submission, output); // 步骤 3：判分（子类实现）
        save(result);                   // 步骤 4：保存结果
        return result;
    }

    protected void validate(Submission s) {
        // 通用校验逻辑
    }

    protected abstract Object run(Submission s);      // 子类实现：不同语言运行方式不同
    protected abstract JudgeResult check(Submission s, Object output);  // 子类实现

    private void save(JudgeResult r) {
        // 通用保存逻辑
    }
}

// 具体子类
public class JavaJudge extends JudgeTemplate {
    @Override
    protected Object run(Submission s) {
        // 编译并运行 Java 代码
    }

    @Override
    protected JudgeResult check(Submission s, Object output) {
        // 对比预期输出
    }
}
```

策略模式---接口使用
```java
// 接口：定义判题策略的契约
public interface JudgeStrategy {
    JudgeResult execute(Submission submission);
}

// 多种实现：互不相关，但都遵循同一个契约
public class JavaJudgeStrategy implements JudgeStrategy { ... }
public class PythonJudgeStrategy implements JudgeStrategy { ... }
public class JavaScriptJudgeStrategy implements JudgeStrategy { ... }

// 使用方：只依赖接口，不依赖具体实现
public class JudgeService {
    private Map<String, JudgeStrategy> strategyMap;

    public JudgeResult judge(Submission submission) {
        JudgeStrategy strategy = strategyMap.get(submission.getLanguage());
        return strategy.execute(submission);  // 多态
    }
}
```

什么时候使用接口或抽象类
```
有共享代码（字段/方法）需要复用？
  ├── 是 → 需要构造器？ → 抽象类
  │         不需要构造器？ → 抽象类 或 接口+default 方法
  │
  └── 否 → 定义行为契约？ → 接口
           不相关类共有行为？ → 接口
           需要多继承？ → 接口
```

### 面试高频抉择题目
#### 场景1：Java的List和AbstractList
接口定义契约，抽象类提供功能骨架实现，提供通用实现，减少子类工作量
```java
// 为什么有个 List 接口，还要有个 AbstractList 抽象类？
public interface List<E> {
    int size();
    boolean isEmpty();
    E get(int index);
    // ...
}

public abstract class AbstractList<E> implements List<E> {
    // 提供通用实现，减少子类的工作量
    public boolean isEmpty() {
        return size() == 0;
    }
}

// 具体类只需要实现最核心的方法
public class ArrayList<E> extends AbstractList<E> {
    // 只需要实现 size() 和 get()，isEmpty() 由 AbstractList 提供
}
```
#### 场景2：Spring的Aware接口
```java
// 多个标记接口，Spring 容器会检测并注入
public interface BeanNameAware { void setBeanName(String name); }
public interface ApplicationContextAware { 
void setApplicationContext(ApplicationContext ctx); }
public interface BeanFactoryAware { void setBeanFactory(BeanFactory factory); }

// 一个类可以实现多个 Aware 接口
public class MyService implements BeanNameAware, ApplicationContextAware {
    // 不需要任何共享代码，只是标记能力
}
```
为什么使用接口：实现类可能已经继承了其他类，用接口可以实现多绑定
#### 场景3：JDBC设计
```java
// 接口：定义数据库操作的契约
public interface Connection {
    Statement createStatement();
    PreparedStatement prepareStatement(String sql);
    // ...
}

// 不同数据库厂商各自实现
public class MysqlConnection implements Connection { ... }
public class OracleConnection implements Connection { ... }
public class PostgresConnection implements Connection { ... }
```
为什么用接口：不同驱动之前完全不相关，没有公用代码，只需要定义契约