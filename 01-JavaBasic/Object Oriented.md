## 面向对象

### Encapsulation 
封装：private/protected → 防御性拷贝 → record → JIT 内联优化
#### private/protected

|           | 本类      | 同包      | 子类      | 任意      |
| --------- | ------- | ------- | ------- | ------- |
| private   | &#10004 |         |         |         |
| default   | &#10004 | &#10004 |         |         |
| protected | &#10004 | &#10004 | &#10004 |         |
| public    | &#10004 | &#10004 | &#10004 | &#10004 |
**Q:** "protected 修饰的成员，不同包的子类能访问吗？" 
**A:** "能，但只能通过继承关系访问，不能通过父类实例访问。"

```java
// 父类 com.example.Parent
protected void doSomething() {}

// 子类 com.other.Child extends Parent
// ✅ 正确：通过继承访问
this.doSomething();
// ❌ 错误：不能通过父类实例访问
new Parent().doSomething();  // 编译报错
//为什么编译会报错?
//不同包，不能通过new父类对象去访问被protected成员，子类访问的是自己继承的protected方法
```

#### Defensive copy 防御性拷贝
问题场景
```java
public class Person {
    private Date birthDate;  // Date 是可变的！

    public Person(Date birthDate) {
        this.birthDate = birthDate;  // ❌ 直接赋值，外部还能改
    }

    public Date getBirthDate() {
        return birthDate;  // ❌ 直接返回，调用方能改内部状态
    }
}
```

代码封装被彻底破坏了，外部能拿到Date对象实例随意修改
```java
Person p = new Person(new Date(2024, 0, 1));
p.getBirthDate().setYear(2030);  // 直接改了内部状态！封装失效
```
正确做法：防御性拷贝
```java
public class Person {
    private final Date birthDate;  // final 保证引用不换，但 Date 内容可变

    public Person(Date birthDate) {
        // ① 构造时拷贝：防止外部改传入的对象
        this.birthDate = new Date(birthDate.getTime());
    }

    public Date getBirthDate() {
        // ② 返回时拷贝：防止调用方改内部状态
        return new Date(this.birthDate.getTime());
    }
}
```
不可变对象满足条件
1.字段是final类型（引用不可变），字段被修饰后对外不提供setter方法 ?
2.字段类型本身是不可以变的（String,LocalDate,8种基本类型）
3.如果有可变对象，做防御性拷贝
4.不提供修改方法
**面试官追问：**

> **Q:** "防御性拷贝什么时候可以不做？" 
> **A:** "当字段是**不可变类型**时——比如 `String`、`Integer`、`LocalDate`，它们本身不可变，赋值和返回都安全，不需要拷贝。"
                               字段final + 无修改方法 + 新对象返回
> **Q:** "防御性拷贝的性能问题怎么解决？" 
> **A:** "如果拷贝非常频繁，可以用 `LocalDate` 替代 `Date`（不可变），或者考虑 `record` 自动处理。"

#### record（Java14正式版/Java16正式版）
```java
// 一行代码 = 自动封装 + 不可变 + 自动生成 getter / equals / hashCode / toString
public record Person(LocalDate birthDate, String name) {}
```
record自动帮忙做了防御性拷贝和不可变封装

| 特性              | 传统类          | record             |
| --------------- | ------------ | ------------------ |
| 构造器             | 手动创建         | 自动生成               |
| getter          | 手动创建         | 字段() 不是get字段()     |
| 可变性             | 默认可变需要加final | 所有字段 private final |
| 防御性拷贝           | 手动实现         | 自动处理               |
| equals/hashCode | 手动实现/lombok  | 自动生成               |
> **Q:** "record 能继承其他类吗？" 
> **A:** "不能，record 隐式继承 `java.lang.Record`，是 final 类，不能再继承其他类。但可以实现接口。"

> **Q:** "record 什么时候用，什么时候不用？" 
> **A:** "适合做**数据传输对象（DTO）**、**值对象**（Value Object）；不适合做 JPA 实体（需要无参构造器 + 可变字段）。"

#### JIT内联优化
问题：封装等于Getter，那么Getter有性能损耗吗
```java
public class Person {
    private String name;

    public String getName() {
        return this.name;  // 每次调用都是一次方法调用
    }
}
```
如果每次访问都有一次方法调用，那么是不是性能开销
答案：JIT编译器会把它优化掉
JIT在运行时发现getName方法：
1.足够小，只有一行代码
2.被频繁调用
就会做方法内联：把方法调用替换为字段访问
```java
// 源码调用
String n = person.getName();

// JIT 内联优化后的机器码 ≈
String n = person.name;  // 直接读字段，没有方法调用开销
```
> **Q:** "什么情况下 JIT 不会内联？" 
> **A:** "方法体太大（超过 -XX:MaxInlineSize 默认 325 字节）、多态调用（虚方法表无法确定具体实现）、递归方法等。"

> **Q:** "内联对封装有什么意义？" 
> **A:** "封装不要担心性能——JIT 内联让 getter 的成本接近于零。所以放心用 getter，不用为了性能就直接暴露 `public` 字段。"

---
## Extends
继承：extends → 构造器链 → 重写规则 → 继承 vs 组合 → 继承破坏封装
### extends 关键字
```java
public class Animal {
    protected String name;

    public void eat() {
        System.out.println(name + " is eating");
    }
}

public class Dog extends Animal {
    public void bark() {
        System.out.println("Woof!");
    }
}
```

| 关键点         | 说明                                 |
| ----------- | ---------------------------------- |
| 单继承         | 一个子类只能extends一个父类                  |
| 多层继承        | A extends B extends C 是允许的，继承链可以很长 |
| 所有类继承Object | 任何类没有写extends,默认继承Object           |
| 子类继承什么      | 继承父类除了private的成员变量和方法              |
| 子类不能继承什么    | private成员、构造器、静态方法                 |
### Constructor Chaining
构造器链
```java
class Parent {
    public Parent() {
        System.out.println("Parent 构造器");
    }
}

class Child extends Parent {
    public Child() {
        // 这里隐式调用 super() —— 编译器自动加的
        System.out.println("Child 构造器");
    }
}

new Child();
// 输出：
// Parent 构造器
// Child 构造器
```
核心规则：
1.子类第一行必须是super()或this()
2.如果子类没有写，编译器自动加super()
3.构造器链从顶向下执行，Object->Parent->Child
面试高频陷阱
```java
class Parent {
    private String name;

    // ❌ 父类没有无参构造器
    public Parent(String name) {
        this.name = name;
    }
}

class Child extends Parent {
    // ❌ 编译报错！因为父类没有无参构造器
    // 编译器试图加 super() 但找不到
}
```
解决方法
```java
// 方案一：子类显式调用 super(name)
class Child extends Parent {
    public Child(String name) {
        super(name);  // ✅ 必须显式调用
    }
}

// 方案二：父类提供一个无参构造器
class Parent {
    public Parent() {}  // 或者不写任何构造器（编译器自动生成）
}
```
**Q:** "如果父类构造器调用了可被重写的方法，会发生什么？" 
**A:** "会调用子类的重写方法，但此时子类还没初始化完——这是**危险的**，不要在构造器里调用可重写的方法。"
```java
class Parent {
    public Parent() {
        init();  // ❌ 危险！调用了可重写的方法
    }
    protected void init() {
        System.out.println("Parent init");
    }
}

class Child extends Parent {
    private String value = "hello";

    @Override
    protected void init() {
        System.out.println(value.length());  // ❌ NullPointerException！
        // 因为此时子类的 value 还没初始化（null）
    }
}
```
这个场景是发生在 new Child()时。构建顺序是
```
1. new Child() 被调用
2. Child 构造器隐式调用 super()
3. Parent 构造器开始执行
4. Parent 构造器里调用 doSomething()
5. 因为实际对象是 Child，所以调的是 Child.doSomething()
6. Child.doSomething() 里访问 value
7. 但此时 Child 的字段还没初始化！value = null（默认值）
8. Parent 构造器执行完毕
9. Child 构造器继续执行
10. value 才被赋值为 hello
```
为什么实际调用对象是Child呢
```
new Child() 那一刻：

堆里分配了一块内存，类型是 Child
┌─────────────────────────────┐
│  Child 对象                  │
│  ┌─────────────────────────┐ │
│  │ Parent 部分              │ │
│  │  (Parent 构造器正在跑)    │ │
│  ├─────────────────────────┤ │
│  │ Child 部分               │ │
│  │  value = null  ← 还没赋值！ │ │
│  └─────────────────────────┘ │
└─────────────────────────────┘

this 指向这个对象，类型是 Child
所以 this.doSomething() → Child.doSomething()
Java的方法调用是动态绑定的，doSomething()不是final、static和private的
在构造父类时，this已经是子类对象，但是子类还没有初始化完成，多态机制会调用到子类方法
```

### Override Rules
重写规则
```java
public class Parent {
    protected Number doSomething(String input) throws IOException {
        return 1;
    }
}

public class Child extends Parent {
    @Override
    // 规则①：方法名必须相同
    // 规则②：参数列表必须相同
    // 规则③：返回类型可以是原类型的子类（协变返回类型）
    // 规则④：访问权限不能更严格（protected → public ✅，protected → private ❌）
    // 规则⑤：抛出的异常不能更宽（IOException → Exception ❌，IOException → FileNotFoundException ✅）
    public Integer doSomething(String input) throws FileNotFoundException {
        return 2;
    }
}
```
> **Q:** "`@Override` 注解有什么用？不写行不行？"
> **A:** "不写也能重写（编译会检查）。`@Override` 的作用是**让编译器帮你检查**——如果你写错了方法名或参数，编译器会报错，而不是当成新方法。"

> **Q:** "静态方法能重写吗？" 
> **A:** "不能。静态方法属于类，属于**静态绑定**——编译时确定调用哪个。子类写同名的静态方法叫**隐藏（Hide）**，不是重写。调用时取决于引用类型，不是实际对象类型。"

### 继承 vs 组合
什么时候用继承？
```
继承的条件（is-a 关系）：
  ✅ Dog is-a Animal → 继承
  ✅ Car is-a Vehicle → 继承
```
什么时候优先用组合？
```java
// ❌ 继承 —— 只是为了用 ArrayList 的方法
class MyList extends ArrayList<String> {
    public void logAndAdd(String item) {
        System.out.println("Adding: " + item);
        super.add(item);
    }
}

// ✅ 组合 —— 持有 ArrayList 的引用，只暴露需要的方法
class MyList {
    private List<String> list = new ArrayList<>();  // 组合

    public void logAndAdd(String item) {
        System.out.println("Adding: " + item);
        list.add(item);
    }

    public int size() {
        return list.size();
    }
}
```

| 对比   | 继承            | 组合               |
| ---- | ------------- | ---------------- |
| 关系   | is-a          | has-a            |
| 耦合   | 强耦合(子类依赖父类实现) | 弱耦合(只依赖接口)       |
| 复用   | 白盒复用(父类实现暴露)  | 黑盒复用(只通过接口调用)    |
| 修改影响 | 父类变->子类可能会影响  | 接口内部实现类变->外部不受影响 |
| 灵活性  | 编译时确定，不能灵活改   | 运行时可替换实现         |
**面试必问：**

> **Q:** "继承有什么缺点？举个例子？" 
> **A:** "继承破坏封装——子类依赖父类的实现细节。比如 `Stack extends Vector`，Stack 本应是 LIFO，但因为继承了 Vector，用户可以直接调用 `vector.add(index, element)` 往中间插数据，破坏了栈的语义。所以 Java 官方自己都后悔了。"

> **Q:** "那实际开发中什么时候用继承？" 
> **A:** "1. 子类确实是父类的一种（is-a） 2. 子类不会改变父类的行为，只是扩展 3. 继承层次不超过 3 层。比如 Spring 的 `ResponseEntityExceptionHandler`，你继承它并重写 `handleException` 方法，这是合理的继承。"

### 继承破坏封装
父类内部实现变化，子类不知情，但行为被影响
```java
class Parent {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}

class Child extends Parent {
    private int childCount = 0;

    @Override
    public void increment() {
        super.increment();  //调用父类方法，父类count + 1
        childCount++; //子类 childCount + 1
    }
}
```
看着没问题，当父类重构新增方法后
```java
// 父类重构：增加批量操作
class Parent {
    private int count = 0;

    public void increment() {
        count++;
    }

    // 新增方法：一次加 5
    public void incrementFive() {
        increment();  // 调用了 5 次 increment()
        increment();
        increment();
        increment();
        increment();
    }
}
//此时  incrementFive方法，子类调用后，由于父类不知道子类重写了increment方法，导致父类参数也被 + 5了
```

---
## Polymorphism
多态：重载 vs 重写 → 向上转型 → 虚方法表 → 静态绑定 vs 动态绑定 → 多态在框架中的应用
### Overload VS Override
重载：编译时多态
```java
public class Calculator {
    // 方法名相同，参数列表不同
    public int add(int a, int b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }

    public double add(double a, double b) {
        return a + b;
    }
}
```
重写：运行时多态
```java
public class Animal {
    public void sound() {
        System.out.println("Animal makes sound");
    }
}

public class Dog extends Animal {
    @Override
    public void sound() {
        System.out.println("Woof!");
    }
}
```

| 要点        | 重载                   | 重写                   |
| --------- | -------------------- | -------------------- |
| 发生位置      | 同一个类中                | 父子类之间                |
| 条件        | 方法名相同，参数个数/顺序/类型不同   | 方法名相同，参数列表完成相同       |
| 返回类型      | 可以不同                 | 相同或子类                |
| 绑定时机      | 编译期(编译器根据参数类型决定使用哪个) | 运行期(JVM根据实际类型决定使用哪个) |
| 关键字       | 不需要                  | @Override 非必须，建议加    |
| 目的        | 提供多个同名方法处理不同参数       | 子类改变父类行为             |
| private方法 | 可以重载                 | 不能重写 私有方法不可见         |
| final方法   | 可以重载                 | 不能重写                 |
| static方法  | 可以重载                 | 不能重写 属于类             |
### Upcasting
向上转型
```java
Animal animal = new Dog();  // ✅ 向上转型
animal.sound();             // 输出 "Woof!" —— 多态
```
规则：编译看左边，运行看右边
编译时，编译器检查animal类型是Animal,所以只能使用父类的方法
运行时，JVM实际执行的对象是Dog,所以调用的是子类重写的方法
坑：向上转型后调用什么方法？
```java
Animal animal = new Dog();

animal.sound();      // ✅ Animal 有 sound()，运行调 Dog 的
animal.eat();        // ✅ Animal 有 eat()，运行调 Dog 的（如果 Dog 重写了）

// animal.bark();   // ❌ 编译报错！Animal 类没有 bark() 方法
// 虽然实际对象是 Dog，但编译器只看左边类型 Animal
```
Downcasting 向下转型，不安全的做法
```java
Animal animal = new Dog();

if (animal instanceof Dog) {       // 先判断类型
    Dog dog = (Dog) animal;        // 向下转型
    dog.bark();                    // ✅ 现在可以调 Dog 特有的方法了
}
```
**面试高频：**

> **Q:** "向下转型什么时候会报错？" 
> **A:** "当实际对象不是目标类型时——比如 `Animal animal = new Cat(); (Dog) animal` 会抛出 `ClassCastException`。所以转型前一定要用 `instanceof` 判断。"

> **Q:** "Java 16 的 `Pattern Matching for instanceof` 知道吗？" 
> **A:** "知道，可以简化转型代码：`if (animal instanceof Dog dog) { dog.bark(); }`，不需要显式强转了。"

### Virtual Method Table
虚方法表：多态的底层原理
问题：JVM运行期怎么知道调哪个方法
```java
Animal animal = getRandomAnimal();  // 可能是 Dog 或 Cat
animal.sound();                     // JVM 怎么知道调哪个？
```
答案：虚方法表
每个类在方法区都有一张虚方法表
```
Animal 类的虚方法表：
┌──────────────────┬────────────────────────┐
│ 方法名            │ 实际入口地址            │
├──────────────────┼────────────────────────┤
│ toString()       │ → Object.toString()    │
│ sound()          │ → Animal.sound()       │
│ eat()            │ → Animal.eat()         │
└──────────────────┴────────────────────────┘

Dog 类的虚方法表：
┌──────────────────┬────────────────────────┐
│ 方法名            │ 实际入口地址            │
├──────────────────┼────────────────────────┤
│ toString()       │ → Object.toString()    │
│ sound()          │ → Dog.sound()  ← 覆盖了 │
│ eat()            │ → Animal.eat()         │
│ bark()           │ → Dog.bark()    ← 新增  │
└──────────────────┴────────────────────────┘
```
执行流程：
```
1.从animal引用找到实际对象Dog实例
2.从Dog实例的对象头找到类信息 方法区中的Dog类
3.从Dog类中的虚方法表中找到sound()的入口地址
4.跳转到Dog.Sound()方法执行
```
这个流程是Dynamic Dispatch-->三步定位：对象->类->方法表->方法入口
### 面试追问

> **Q:** "虚方法表对性能有影响吗？" 
> **A:** "有，但极小。一次间接寻址的开销，大约 1-2 纳秒。JIT 还会做**内联缓存**（Inline Cache）优化——如果某个调用点 99% 都指向同一个类型，JIT 直接内联那个方法，连虚方法表都不查了。"

> **Q:** "静态方法有虚方法表吗？" 
> **A:** "没有。静态方法属于类，**编译期就确定了**，用的是**静态绑定**，直接调用，不需要虚方法表。"

> **Q:** "private 方法有虚方法表吗？" 
> **A:** "也没有。private 方法不能被重写，也是静态绑定，编译期就确定了。"

### 静态绑定 VS 动态绑定

| 绑定类型 | 时机  | 依据        | 适用方法                       |
| ---- | --- | --------- | -------------------------- |
| 静态绑定 | 编译期 | 引用变量的声明类型 | private方法、final方法、静态方法和构造器 |
| 动态绑定 | 运行期 | 实际对象的类型   | 非private的实例方法              |
```java
public class Test {
    public static void main(String[] args) {
        Animal animal = new Dog();

        // 动态绑定 —— 看实际对象类型
        animal.sound();              // Dog.sound() 被执行

        // 静态绑定 —— 只看声明类型
        Animal.staticMethod();       // 不管实际类型，调 Animal 的
        animal.staticMethod();       // ❌ 不推荐这么写，但也是调 Animal 的
    }
}
```
**经典面试题：**
```java
public class Parent {
    public String name = "Parent";

    public void print() {
        System.out.println(name);
    }
}

public class Child extends Parent {
    public String name = "Child";   // 字段没有多态！

    @Override
    public void print() {
        System.out.println(name);
    }
}

// 测试
Parent p = new Child();
System.out.println(p.name);  // 输出 "Parent" —— 字段没有多态！静态绑定
p.print();                   // 输出 "Child"  —— 方法有多态！动态绑定
```
结论：
实例方法：动态绑定
字段(成员变量)：静态绑定
静态方法：静态绑定

### 多态在框架中使用
#### 策略模式 —— 多态的经典应用
```java
// 判题策略
public interface JudgeStrategy {
    JudgeResult judge(Submission submission);
}

@Component
public class JavaJudgeStrategy implements JudgeStrategy { ... }

@Component
public class PythonJudgeStrategy implements JudgeStrategy { ... }

@Component
public class JudgeService {
    private Map<String, JudgeStrategy> strategyMap;

    public JudgeResult execute(Submission submission) {
        JudgeStrategy strategy = strategyMap.get(submission.getLanguage());
        return strategy.judge(submission);  // 多态！
    }
}
```
#### 模板方法模式 —— 继承 + 多态
```java
public abstract class BaseController {
    // 模板方法 —— 定义了骨架
    public ResponseEntity<?> handleRequest(Request request) {
        validate(request);           // 子类实现
        Object data = process(request);  // 子类实现
        return success(data);
    }

    protected abstract void validate(Request request);     // 子类重写
    protected abstract Object process(Request request);    // 子类重写

    private ResponseEntity<?> success(Object data) {       // 固定逻辑
        return ResponseEntity.ok(data);
    }
}
```

工厂方法模式----多态
```java
public Animal getAnimalByType(String type) {
	if ("Dog".equlas(type)) {
		return new Dog();
	} else if ("Cat".equlas(type)) {
		return new Cat();
	}
}
```
