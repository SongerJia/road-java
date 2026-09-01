## 泛型
泛型基础语法->类型擦除->桥接方法->通配符与PECS->泛型限制与实战
### 泛型基础语法
为什么需要泛型？
```java
// ❌ 没有泛型之前 —— 需要强转，不安全
List list = new ArrayList();
list.add("hello");
list.add(123);  // 可以放任意类型
String s = (String) list.get(1);  // 运行时报 ClassCastException！

// ✅ 泛型 —— 编译期检查，安全
List<String> list = new ArrayList<>();
list.add("hello");
// list.add(123);  // ❌ 编译报错，只能放 String
String s = list.get(0);  // ✅ 不需要强转
```
三种使用方式：
```java
// ① 泛型类
public class Box<T> {
    private T item;

    public void set(T item) {
        this.item = item;
    }

    public T get() {
        return item;
    }
}

Box<String> box = new Box<>();  // Java 7+ 菱形语法

// ② 泛型接口
public interface Comparable<T> {
    int compareTo(T o);
}

public class Person implements Comparable<Person> {
    @Override
    public int compareTo(Person o) { ... }
}

// ③ 泛型方法
public class Utils {
    // 泛型方法 —— <T> 在返回值前面
    public static <T> T getMiddle(T... array) {
        return array[array.length / 2];
    }
}

String mid = Utils.<String>getMiddle("a", "b", "c");  // 显式指定类型
String mid2 = Utils.getMiddle("a", "b", "c");          // 类型推断
```
泛型参数命名约定

| 字母    | 含义            |
| ----- | ------------- |
| E     | Element(集合元素) |
| K     | Key(键)        |
| V     | Value(值)      |
| T     | Type(任意类型)    |
| N     | Number(数字)    |
| S,U,V | 第二、第三、第四类个型   |
### Type Erasure
类型擦除
泛型是编译期概念，运行期不存在
```java
List<String> stringList = new ArrayList<>();
List<Integer> intList = new ArrayList<>();

System.out.println(stringList.getClass() == intList.getClass());  // true！
// 两个的运行时类型都是 ArrayList.class，泛型信息被擦除了
// 为什么已经定义了参数类型，还是被擦除。因为它基于泛型类定义，类或者接口有类型参数E
```
擦除规则
```java
// 源码
public class Box<T> {
    private T item;

    public T get() { return item; }
    public void set(T item) { this.item = item; }
}

// 编译后（类型擦除）—— T 被替换为 Object
public class Box {
    private Object item;

    public Object get() { return item; }
    public void set(Object item) { this.item = item; }
}
```
```java
// 有边界时的擦除
public class Box<T extends Comparable<T>> {
    private T item;

    public T get() { return item; }
}

// 编译后 —— T 被替换为 Comparable
public class Box {
    private Comparable item;

    public Comparable get() { return item; }
}
```
### 面试必问

> **Q:** "Java 泛型是编译期还是运行期的？和 C++ 模板有什么区别？" 
> **A:** "Java 泛型是**编译期**的，通过**类型擦除**实现——编译后泛型信息被擦除，运行期不存在泛型。C++ 模板是**运行期**的，每个类型参数都会生成一份独立代码（模板实例化）。
> **后果：** Java 泛型在运行期不知道自己的类型参数，比如 `T.class` 是编译报错的。C++ 模板每个类型都有独立的代码，类型信息保留。"
### Bridge Method
桥接方法：语言语义和JVM规则之间的适配器
#### 问题场景
```java
public class Parent {
    public Object get() {
        return "parent";
    }
}

public class Child extends Parent {
    @Override
    public String get() {  // 协变返回类型 返回类型是父类方法返回的子类型
        return "child";
    }
}
```
编译后得到
```java
// 编译后的 Child 类
public class Child extends Parent {
    // ① 自己写的
    public String get() {
        return "child";
    }

    // ② 编译器自动生成的桥接方法 —— 保持多态
    public Object get() {           // 桥接方法，和父类签名一致
        return this.get();          // 调用 String 版本的 get()
    }
    //为什么要有这个桥接方法，或者说为什么JVM无法识别子类重写的方法
    //JVM的invokevirtual是静态签名（方法名+参数类型+返回类型）匹配，在编译期/链接期决定了调用哪个方法，父类和子类的这2个是完全不同的方法。
    //JVM的重写规则和语言不同的原因是/如果JVM支持了协变返回类型
    //1.子类的虚方法表中会使用协变返回类型的方法替换父类方法，但使用Parent p = new Child()  p.get()时，去子类的虚方法表找不到对应的静态签名。
    //2.二进制兼容性，如果父类的类型发生变化，子类可能会出现不兼容的场景
}
```
为什么需要桥接方法？
因为多态依赖方法签名一致，（Java的重写和JVM重写规则不同，JVM要求返回类型也一致）。父类是Object get()，子类是String get()，因为返回类型不同，在JVM看来是不同。桥接方法保证：调用父类签名的get()时，能正确转发到子类的 String get()。
泛型中的桥接方法：
```java
public class Parent<T> {
    public void set(T item) { ... }
}

public class Child extends Parent<String> {
    @Override
    public void set(String item) { ... }
}

// 编译后：
// ① 类型擦除后，Parent.set 变成 set(Object)
// ② 子类 set(String) 不构成重写（签名不同）
// ③ 编译器自动生成桥接方法：
//    public void set(Object item) {
//        this.set((String) item);  // 强转后调用子类的方法
//    }
```
### 通配符与PECS原则
为什么需要通配符？
```java
// 问题：泛型是不变的（invariant）  类型参数的关系跟容器之间无关
List<String> strings = new ArrayList<>();
List<Object> objects = strings;  // ❌ 编译报错！List<String> 不是 List<Object> 的子类型

// 为什么？如果有这样的赋值，就可以：
objects.add(123);  // 把一个 Integer 放进本来是 String 的列表里 —— 类型不安全
//数组是协变（convariant）的  类型参数是父子关系，容器也是父子关系
```
三种通配符
```java
public class Box<T> {
    private T item;
    public T get() { return item; }
    public void set(T item) { this.item = item; }
}

// ① 无限定通配符 ? —— 表示"任意类型"
public static void print(Box<?> box) {
    Object item = box.get();           // ✅ 可以读，但只能读到 Object
    // box.set("hello");               // ❌ 不能写（除了 null）
}

// ② 上界通配符 ? extends T —— 表示"T 或 T 的子类"
public static void read(Box<? extends Number> box) {
    Number num = box.get();            // ✅ 可以读，保证至少是 Number
    // box.set(123);                   // ❌ 不能写！因为不知道具体是哪个子类
}

// ③ 下界通配符 ? super T —— 表示"T 或 T 的父类"
public static void write(Box<? super Integer> box) {
    box.set(123);                      // ✅ 可以写 Integer
    Object obj = box.get();            // ✅ 可以读，但只能读到 Object
    // Integer i = box.get();          // ❌ 不能确定是 Integer，可能是 Number
}
```
Producer Extends, Consumer Super：PECS原则
```java
// ✅ 正确用法
public class Collections {
    // Producer：只读数据（生产数据）→ 用 ? extends
    public static <T> void copy(List<? extends T> src,  // 生产者，只读
                                 List<? super T> dest) { // 消费者，只写
        for (T item : src) {
            dest.add(item);
        }
    }

    // 只读场景：从集合中取数据 → ? extends
    public static double sum(List<? extends Number> list) {
        double total = 0;
        for (Number n : list) {
            total += n.doubleValue();
        }
        return total;
    }

    // 只写场景：往集合中放数据 → ? super
    public static void addIntegers(List<? super Integer> list) {
        for (int i = 0; i < 10; i++) {
            list.add(i);  // ✅ 可以放 Integer
        }
    }
}
```
PECS口诀
```
Producer Extends  —— 生产者（只读）用 extends
Consumer Super    —— 消费者（只写）用 super
如果既读又写 —— 不要用通配符，直接用具体类型
```
#### 经典面试题
```java
// 这段代码能编译吗？
List<? extends Number> list = new ArrayList<Integer>();
list.add(123);  // ❌ 编译报错！

// 这段呢？
List<? super Integer> list = new ArrayList<Number>();
list.add(123);  // ✅ 编译通过
Integer i = list.get(0);  // ❌ 编译报错！只能读到 Object
```
### 泛型的限制和实战
#### 泛型的六大限制
```java
// ① 不能用基本类型
// List<int> list = new ArrayList<>();// ❌ 泛型擦除为Object,基本类型不是Object子类
List<Integer> list = new ArrayList<>();  // ✅ 要用包装类

// ② 不能创建泛型实例
public class Box<T> {
    // private T item = new T();  // ❌ 编译报错，类型擦除后不知道 T 是什么
    private T item;

    public Box(Class<T> clazz) throws Exception {
        this.item = clazz.newInstance();  // ✅ 通过 Class 对象反射创建
    }
}

// ③ 不能创建泛型数组
// T[] array = new T[10];  // ❌
T[] array = (T[]) new Object[10];  // ✅ 强转（但会有警告）

// ④ 不能用在静态上下文中
public class Box<T> {
    // public static T item;      // ❌ 静态变量不能引用类型参数
    // public static void test(T t) { }  // ❌ 静态方法不能引用类的类型参数
    public static <U> void test(U u) { }  // ✅ 静态方法可以有自己的类型参数
}

// ⑤ 不能 instanceof
// if (list instanceof List<String>) { }  // ❌ 泛型信息已擦除
if (list instanceof List<?>) { }  // ✅ 通配符可以

// ⑥ 不能有重载冲突
public class Example {
    // public void process(List<String> list) { }  // ❌ 擦除后和下面的方法签名相同
    // public void process(List<Integer> list) { }// ❌ 编译报错
}
```
### 泛型在项目中应用
```java
// ① 通用响应封装
public class Result<T> {
    private int code;
    private String message;
    private T data;

    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.code = 200;
        result.data = data;
        return result;
    }

    public static <T> Result<T> fail(String message) {
        Result<T> result = new Result<>();
        result.code = 500;
        result.message = message;
        return result;
    }
}

// 使用
Result<User> userResult = Result.success(user);
Result<List<Question>> questionsResult = Result.success(questionList);

// ② 泛型 Builder
public class GenericBuilder<T> {
    private T instance;

    public GenericBuilder(Class<T> clazz) throws Exception {
        this.instance = clazz.getDeclaredConstructor().newInstance();
    }

    public GenericBuilder<T> with(Consumer<T> setter) {
        setter.accept(instance);
        return this;
    }

    public T build() {
        return instance;
    }
}

// 使用
User user = new GenericBuilder<>(User.class)
    .with(u -> u.setName("张三"))
    .with(u -> u.setAge(25))
    .build();
```