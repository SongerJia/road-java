## 反射
反射基础->核心API->反射的应用场景->反射的性能问题->优化方案->安全问题
### 反射基础
#### 什么是反射
反射：程序在运行期可以动态获取类的完整信息，操作类的构造器、字段、方法等成员
```java
// 正常编码 —— 编译期就确定了
User user = new User();
user.setName("张三");

// 反射 —— 运行期才知道
Class<?> clazz = Class.forName("com.example.User");
Object obj = clazz.getDeclaredConstructor().newInstance();
Method method = clazz.getMethod("setName", String.class);
method.invoke(obj, "张三");
```
获取Class的三种方式
```java
// 方式一：Class.forName("全限定名") —— 最常用，类名可以来自配置文件
Class<?> clazz1 = Class.forName("com.example.User");

// 方式二：类名.class —— 直接获取，不会触发静态初始化
Class<User> clazz2 = User.class;

// 方式三：对象.getClass() —— 已有实例时
User user = new User();
Class<? extends User> clazz3 = user.getClass();

// 三种方式获取的是同一个 Class 对象（JVM 中每个类只有一个 Class 实例）
System.out.println(clazz1 == clazz2);  // true
System.out.println(clazz2 == clazz3);  // true
```
Class.forName和类名.class的区别
```java
public class MyClass {
    static {
        System.out.println("静态代码块执行了");
    }
}

// ① Class.forName() —— 会触发静态初始化
Class<?> c1 = Class.forName("com.example.MyClass");
// 输出："静态代码块执行了"

// ② 类名.class —— 不会触发静态初始化
Class<MyClass> c2 = MyClass.class;
// 没有输出，静态代码块没执行

// ③ Class.forName() 可以控制是否初始化
Class<?> c3 = Class.forName("com.example.MyClass", false, classLoader);
// false 表示不执行静态初始化
```
**面试追问：**

> **Q:** "`Class.forName()` 和 `类名.class` 有什么区别？" 
> **A:** "
> 1. `Class.forName()` 默认会执行类的静态初始化（静态代码块），`类名.class` 不会
> 2. `Class.forName()` 需要处理 `ClassNotFoundException`（受检异常），`类名.class` 编译期就能确定
### 核心API
#### 获取类的基本信息
```java
Class<?> clazz = User.class;

// 类名
clazz.getName();              // "com.example.User"   —— 全限定名
clazz.getSimpleName();        // "User"               —— 简单类名
clazz.getCanonicalName();     // "com.example.User"   —— 规范名（用于 import）

// 修饰符
int mod = clazz.getModifiers();
Modifier.isPublic(mod);       // true
Modifier.isFinal(mod);        // false
Modifier.isAbstract(mod);     // false

// 包
Package pkg = clazz.getPackage();  // package com.example

// 父类
Class<?> superClass = clazz.getSuperclass();  // class java.lang.Object

// 接口
Class<?>[] interfaces = clazz.getInterfaces();  // 实现的接口数组

// 类加载器
ClassLoader loader = clazz.getClassLoader();

// 注解
Annotation[] annotations = clazz.getAnnotations();
```
#### 操作构造器
```java
Class<User> clazz = User.class;

// 获取所有 public 构造器
Constructor<?>[] constructors = clazz.getConstructors();

// 获取指定参数类型的构造器
Constructor<User> constructor = clazz.getConstructor(String.class, int.class);

// 创建实例 —— 反射调用构造器
User user = constructor.newInstance("张三", 25);

// 获取所有构造器（包括 private）
Constructor<?>[] declaredConstructors = clazz.getDeclaredConstructors();

// 调用 private 构造器
Constructor<User> privateConstructor = clazz.getDeclaredConstructor();  // 无参
privateConstructor.setAccessible(true);  // 绕过访问权限检查
User user2 = privateConstructor.newInstance();

//getConstructors()和getDeclaredConstructors()区别
//getConstructor()----只返回的public构造器，包含父类的
//getDeclaredConstructor()----返回本类所有的构造器，包括私有的
```
#### 操作方法
```java
Class<?> clazz = User.class;
Object obj = clazz.getDeclaredConstructor().newInstance();

// 获取所有 public 方法（包括父类的）
Method[] methods = clazz.getMethods();

// 获取本类所有方法（包括 private，不包括父类）
Method[] declaredMethods = clazz.getDeclaredMethods();

// 获取指定方法 —— 参数：方法名，参数类型
Method setNameMethod = clazz.getMethod("setName", String.class);
Method getNameMethod = clazz.getMethod("getName");
Method setAgeMethod = clazz.getMethod("setAge", int.class);

// 调用方法 —— invoke(对象实例, 参数...)
setNameMethod.invoke(obj, "张三");                // obj.setName("张三")
String name = (String) getNameMethod.invoke(obj); // obj.getName()
setAgeMethod.invoke(obj, 25);                     // obj.setAge(25)

// 调用 private 方法
Method privateMethod = clazz.getDeclaredMethod("privateMethod");
privateMethod.setAccessible(true);  // 绕过访问检查
privateMethod.invoke(obj);

// 调用静态方法
Method staticMethod = clazz.getMethod("staticMethod");
staticMethod.invoke(null);  // 静态方法不需要实例，传 null
```
#### 操作字段
```java
Class<?> clazz = User.class;
Object obj = clazz.getDeclaredConstructor().newInstance();

// 获取所有 public 字段（包括父类的）
Field[] fields = clazz.getFields();

// 获取本类所有字段（包括 private，不包括父类）
Field[] declaredFields = clazz.getDeclaredFields();

// 获取指定字段
Field nameField = clazz.getDeclaredField("name");
nameField.setAccessible(true);  // 绕过 private 检查

// 读取和设置字段值
nameField.set(obj, "李四");                    // obj.name = "李四"
String name = (String) nameField.get(obj);     // obj.name

// 操作静态字段
Field staticField = clazz.getDeclaredField("STATIC_FIELD");
staticField.set(null, "new value");  // 静态字段实例传 null
```
#### 操作数组
```java
// 反射创建数组
int[] array = (int[]) Array.newInstance(int.class, 10);
Array.set(array, 0, 100);
Array.set(array, 1, 200);
int value = Array.getInt(array, 0);  // 100
```
### 反射的应用场景
#### 场景1：框架的核心---Spring的IOC
```java
// Spring 通过反射创建 Bean 并注入依赖
public class SpringContainer {
    private Map<String, Object> beans = new HashMap<>();

    public void registerBean(Class<?> clazz) throws Exception {
        // ① 反射创建实例
        Object instance = clazz.getDeclaredConstructor().newInstance();

        // ② 反射处理 @Autowired 注入
        for (Field field : clazz.getDeclaredFields()) {
            if (field.isAnnotationPresent(Autowired.class)) {
                field.setAccessible(true);
                Object dependency = beans.get(field.getType().getName());
                field.set(instance, dependency);  // 反射注入
            }
        }

        // ③ 注册到容器
        beans.put(clazz.getName(), instance);
    }
}
```
#### 场景2：JDBC驱动加载
```java
// Class.forName() 触发 Driver 的静态代码块
Class.forName("com.mysql.cj.jdbc.Driver");
// Driver 的静态代码块中：
// DriverManager.registerDriver(new Driver());
```
#### 场景3：动态代理
```java
// JDK 动态代理基于反射          InvocationHandler 调用处理器
public class LogProxy implements InvocationHandler {
	//被代理的对象
    private Object target;

    public LogProxy(Object target) {
        this.target = target;
    }
	//所有通过代理对象调用的方法最终都会走到这个方法
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("方法执行前: " + method.getName());
        Object result = method.invoke(target, args);  // 反射调用
        System.out.println("方法执行后: " + method.getName());
        return result;
    }

    public static <T> T createProxy(T target) {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(), //类加载器
            target.getClass().getInterfaces(), //接口  JDK动态代理限制必须基于接口
            new LogProxy(target) // InvocationHandler 调用处理器
        );
    }
}
//运行时JVM会动态生成一个类
class $Proxy0 implements UserService {
    InvocationHandler h;

    public void save() {
        h.invoke(this, methodSave, args);
    }
}
//使用方式
UserService service = new UserServiceImpl();
UserService proxy = LogProxy.createProxy(service);

proxy.save();  // 会打印日志，再执行真实方法
//执行流程
proxy.save()
   ↓
$Proxy0.save()
   ↓
InvocationHandler.invoke()
   ↓
method.invoke(target, args)   // 反射调用真实对象
   ↓
UserServiceImpl.save()
```
#### 场景4：配置文件驱动
```java
// 配置文件中配置类名，运行时动态加载
// config.properties:
// user.service=com.example.impl.UserServiceImpl
// user.dao=com.example.impl.UserDaoImpl

public class BeanFactory {
    public static <T> T getBean(String className) {
        try {
            Class<?> clazz = Class.forName(className);
            return (T) clazz.getDeclaredConstructor().newInstance();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}

// 使用
UserService userService = BeanFactory.getBean(
    PropertiesUtil.getProperty("user.service")
);
```
### 反射的性能问题
#### 反射为什么慢？
```java
// 正常调用 —— 约 1~2 纳秒
user.setName("张三");

// 反射调用 —— 约 1000~2000 纳秒（慢 1000 倍）
Method method = User.class.getMethod("setName", String.class);
method.invoke(user, "张三");
```

| 性能开销    | 来源                              |
| ------- | ------------------------------- |
| 方法查找    | getMethod需要遍历类的方法列表，匹配方法名和类型列表  |
| 装箱拆箱    | 基本类型需要包装为Object（int->Integer）   |
| 安全检查    | 每次invoke调用都要权限检查                |
| 可变参数    | 每次invoke调用需要创建Object数组          |
| JIT无法内联 | 反射调用是动态的，JIT无法内联优化              |
| 内联缓存失效  | 正常调用会有内联缓存（InLine Cache），反射调用没有 |
性能对比数据
```java
Benchmark                      Mode  Cnt     Score    Error  Units
正常调用                       avgt    5     2.1 ns          ns/op
反射调用（无优化）             avgt    5  1200.5 ns          ns/op
反射调用 + setAccessible      avgt    5   800.3 ns          ns/op
MethodHandle（Java 7+）       avgt    5    50.2 ns          ns/op
LambdaMetafactory             avgt    5     3.5 ns          ns/op（接近直接调用）
```
### 优化方案
#### 方案1：缓存Method对象
```java
// ❌ 错误：每次调用都查找方法
public class BadReflection {
    public void invokeMethod(Object obj) throws Exception {
        // 每次调用都 getMethod() —— 性能极差
        Method method = obj.getClass().getMethod("setName", String.class);
        method.invoke(obj, "张三");
    }
}

// ✅ 正确：缓存 Method 对象
public class GoodReflection {
    private static final Map<String, Method> CACHE = new ConcurrentHashMap<>();

    public void invokeMethod(Object obj) throws Exception {
        String key = obj.getClass().getName() + "#setName";
        //如果缓存中key的value不存在，就通过反射得到Method,如果已经存在就返回
        Method method = CACHE.computeIfAbsent(key, k -> {
            try {
                return obj.getClass().getMethod("setName", String.class);
            } catch (NoSuchMethodException e) {
                throw new RuntimeException(e);
            }
        });
        method.invoke(obj, "张三");
    }
}
```
#### 方案2：setAccessible(true)
```java
// 关闭安全检查，提升约 30% 性能
Method method = clazz.getDeclaredMethod("privateMethod");
method.setAccessible(true);  // 关闭安全检查，后续调用不再检查
method.invoke(obj);
```
#### 方案3：MethodHandler（JDK7+）
```java
// MethodHandle 比反射快，接近直接调用 定义了固定的方法签名 todo
public class MethodHandleExample {
    private static MethodHandle handle;

    static {
        try {
            MethodHandles.Lookup lookup = MethodHandles.lookup();
            MethodType mt = MethodType.methodType(void.class, String.class);
            handle = lookup.findVirtual(User.class, "setName", mt);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void invoke(User user, String name) throws Throwable {
        handle.invoke(user, name);  // 比反射快 10~20 倍
    }
}
```
#### 方案4：LambdaMetafactory
```java
// 最极致的优化 —— 生成函数式接口实现  todo
@FunctionalInterface
interface UserNameSetter {
    void setName(User user, String name);
}

public class LambdaFactoryExample {
    private static UserNameSetter setter;

    static {
        try {
            MethodHandles.Lookup lookup = MethodHandles.lookup();
            MethodHandle target = lookup.findVirtual(
                User.class, "setName",
                MethodType.methodType(void.class, String.class)
            );

            // 通过 LambdaMetafactory 生成实现
            CallSite site = LambdaMetafactory.metafactory(
                lookup,
                "setName",
                MethodType.methodType(UserNameSetter.class),
                MethodType.methodType(void.class, User.class, String.class),
                target,
                MethodType.methodType(void.class, User.class, String.class)
            );
            setter = (UserNameSetter) site.getTarget().invokeExact();
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }

    public void invoke(User user, String name) {
        setter.setName(user, name);  // 接近直接调用的性能
    }
}
```
### 反射的安全问题
#### 问题1：破坏封装
```java
public class BankAccount {
    private double balance = 10000;

    private void deductAll() {
        balance = 0;  // 兜底方法，正常情况下无法调用
    }
}

// 反射可以调用 private 方法
BankAccount account = new BankAccount();
Method method = BankAccount.class.getDeclaredMethod("deductAll");
method.setAccessible(true);  // 暴力破解封装
method.invoke(account);      // 余额被清空！
```
#### 问题2：绕过泛型检查
```java
List<String> list = new ArrayList<>();
list.add("hello");

// 反射绕过泛型，插入 Integer
Method addMethod = List.class.getMethod("add", Object.class);
addMethod.invoke(list, 123);  // ✅ 编译不报错，运行也不报错

// 但取出时会出问题 —— 强转失败
String item = list.get(1);  // ❌ ClassCastException: Integer cannot be cast to String
```
#### 问题3：单例模式被破坏
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}

// 反射可以调用 private 构造器，破坏单例
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton s2 = constructor.newInstance();  // 创建了第二个实例！
//给INSTANCE赋值的时刻
1.访问类的静态方法  Singleton.getInstance()
2.访问类的静态字段（非final的基本类型/字符串）
3.new 这个类 
4.反射调用这个类  Class.forName("Singleton")
```
防御方案：
```java
public class SafeSingleton {
    private static final SafeSingleton INSTANCE = new SafeSingleton();

    private SafeSingleton() {
        // 防止反射创建第二个实例
        if (INSTANCE != null) {
            throw new RuntimeException("单例模式被破坏，禁止反射创建实例");
        }
    }

    public static SafeSingleton getInstance() {
        return INSTANCE;
    }
}
```
