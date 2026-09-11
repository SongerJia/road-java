## 类加载机制
类加载过程五步->类加载层级->双亲委派模型->双亲委派破坏->自定义类加载器->面试高频题
### 类加载过程五步
```
// 一个类的完整生命周期：
// 加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
//                 （前五步是"类加载"过程）

// 其中：验证、准备、解析合称"链接"（Linking）
```
Loading（加载）
```
// 做什么：
// ① 通过类的全限定名获取二进制字节流（.class 文件）
// ② 把字节流转化为方法区/元空间的运行时数据结构
// ③ 在堆中生成 Class 对象（作为访问入口）

// 字节流来源：
// ① 本地 .class 文件
// ② JAR 包
// ③ 网络（Applet）
// ④ 动态生成（代理类）
// ⑤ 其他文件（JSP → Servlet）
```
Verification（验证）
```
// 做什么：确保字节流符合 JVM 规范，保证安全
// 四步验证：
// ① 文件格式验证：魔数（CAFEBABE）、版本号
// ② 元数据验证：类是否有父类、是否实现了抽象方法
// ③ 字节码验证：操作数栈类型、跳转指令安全性
// ④ 符号引用验证：引用的类/方法/字段是否存在

// 作用：防止恶意字节码攻击 JVM
// -Xverify:none 可跳过（不推荐，有安全风险）
```
Preparation（准备）
```
// 做什么：为类的静态变量分配内存并设置默认值
// 注意：这里只是"默认零值"，不是赋初始值！

public class Demo {
    private static int count = 100;  // 准备阶段：count = 0（不是 100）
    private static final int MAX = 100;  // final 常量：准备阶段就赋 100

    // 准备阶段（JDK 8+）：
    // count = 0（默认值）
    // MAX = 100（final 常量直接赋值，因为是编译期常量）
}

// 真正赋值 100 是在"初始化"阶段
```
Resolution（解析）
```
// 做什么：把常量池中的"符号引用"替换为"直接引用"

// 符号引用：类名字符串（"com.example.User"）
// 直接引用：内存中的实际地址/句柄

// 解析的内容：
// 类、接口、字段、方法、方法类型、方法句柄等

// 类加载阶段？还是使用阶段？
// JVM 规范没有规定时机，HotSpot 是"延迟解析"——用到才解析
```
Initialization（初始化）
```
// 做什么：执行类的静态代码块和静态变量赋值
// 本质：调用 <clinit>() 方法

public class Demo {
    static int count = 100;          // 这里赋值
    static {
        System.out.println("静态块执行");  // 这里执行
    }
}

// <clinit>() 方法的生成规则：
// ① 编译器自动收集静态变量赋值 + 静态代码块
// ② 按代码顺序合并
// ③ 父类的 <clinit>() 先于子类执行
// ④ 接口不要求 <clinit>() 先执行

// 初始化时机（6 种主动使用触发）：
// ① new 实例
// ② 访问静态字段（非 final）
// ③ 调用静态方法
// ④ 反射调用
// ⑤ 初始化子类（先初始化父类）
// ⑥ 作为 JVM 启动入口（main 类）
```
### 面试高频

> **Q:** "类加载过程有哪几步？" 
> **A:** "加载 → 验证 → 准备 → 解析 → 初始化。加载：读字节流生成 Class 对象；验证：检查字节码安全；准备：静态变量分配内存设默认值；解析：符号引用转直接引用；初始化：执行静态块和静态变量赋值。"

> **Q:** "准备阶段和初始化阶段的区别？" 
> **A:** "准备阶段给静态变量分配内存并设**默认零值**（count=0）；初始化阶段才执行静态变量赋值和静态代码块（count=100）。注意 final 常量在准备阶段就赋值了。"
### 类加载器层级
三种内置类加载器
```
// ① Bootstrap ClassLoader（启动类加载器）
// 加载：JAVA_HOME/lib 下的核心库
// 例如：rt.jar（JDK8）、java.lang、java.util 等
// 实现：C++ 实现（不是 Java 类），没有父加载器
// 获取：ClassLoader.getSystemClassLoader() 无法获取（null）

// ② Extension ClassLoader（扩展类加载器，JDK8）
// 加载：JAVA_HOME/lib/ext 下的扩展库
// JDK 9+ 改为 Platform ClassLoader（平台类加载器）
// 实现：Java 类（sun.misc.Launcher$ExtClassLoader）

// ③ Application ClassLoader（应用/系统类加载器）
// 加载：classpath 下的所有类（我们的代码）
// 实现：sun.misc.Launcher$AppClassLoader
// 获取：ClassLoader.getSystemClassLoader() 默认返回它
```
验证加载器层级
```
// 查看各加载器
public class ClassLoaderDemo {
    public static void main(String[] args) {
        ClassLoader app = ClassLoaderDemo.class.getClassLoader();
        System.out.println(app);                  // AppClassLoader

        ClassLoader ext = app.getParent();
        System.out.println(ext);                  // ExtClassLoader / PlatformClassLoader

        ClassLoader boot = ext.getParent();
        System.out.println(boot);                 // null（Bootstrap 是 C++ 实现）

        // 核心类由 Bootstrap 加载
        System.out.println(String.class.getClassLoader());  // null
    }
}
```
JDK8与JDK9+加载器变化

|JDK 8|JDK 9+|
|---|---|
|Bootstrap|Bootstrap（基础模块）|
|Extension（rt.jar/ext）|Platform（平台模块）|
|Application|Application（classpath）|
### 双亲委派模型
双亲委派模型：当一个类加载器收到类加载请求时，先不自己加载，而是把请求委派给父加载器。父加载器处理不了，才自己加载。
工作流程
```
// 请求加载 com.example.User
Application ClassLoader 收到请求
    ↓ 先委派
Extension/Platform ClassLoader
    ↓ 再委派
Bootstrap ClassLoader —— 加载核心库
    ↓ 找不到 com.example.User（不在核心库）
    ↓ 返回失败，向下传递
Extension/Platform ClassLoader 尝试 —— 也找不到
    ↓ 返回失败，向下传递
Application ClassLoader 自己加载 —— 在 classpath 中找到 ✅
```
为什么双亲委派？
```
// 核心原因：保证 Java 核心类不被篡改

// 场景：如果自己写一个 java.lang.String 放到 classpath
// 没有双亲委派 → 应用加载器直接加载你的 String → 核心库被替换！
// 有双亲委派 → 请求先到 Bootstrap → Bootstrap 加载真正的 String
// → 你的"山寨 String"永远没机会加载

// 好处：
// ① 核心类安全（避免重复加载和篡改）
// ② 避免类的重复加载（同一个类只加载一次）
// ③ 保证核心类的唯一性
```
源码实现
```
// ClassLoader.loadClass() 的核心逻辑
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException {

    synchronized (getClassLoadingLock(name)) {
        // ① 先检查类是否已经加载过
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                // ② 有父加载器 → 委派给父加载器
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // ③ 没有父加载器 → Bootstrap 加载
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // 父加载器找不到
            }

            // ④ 父加载器找不到 → 自己加载
            if (c == null) {
                c = findClass(name);
            }
        }
        if (resolve) {
            resolveClass(c);  // 解析
        }
        return c;
    }
}
```
### 双亲委派的破坏
为什么被破坏？
双亲委派的顺序是自下而上的委派、自上而下的加载，但某些场景需要先自己加载。
#### 破坏场景1：JDBC（SPI机制）
```
// 问题：
// JDBC 的 DriverManager 是核心库（Bootstrap 加载）
// 但 MySQL 驱动是第三方 jar（Application 加载）
// 核心库要调用第三方类 → 双亲委派做不到！

// 解决：SPI（Service Provider Interface）+ 线程上下文类加载器
// 核心库通过 Thread.currentThread().getContextClassLoader() 
// 获取应用类加载器，用它加载第三方实现

// 场景：
Class.forName("com.mysql.cj.jdbc.Driver");
// DriverManager（Bootstrap）需要加载 MySQL 驱动（Application）
// 通过上下文类加载器打破双亲委派
```
破坏场景2：Tomcat（容器隔离）
```
// 问题：一个 Tomcat 部署多个 Web 应用
// 两个应用可能用了不同版本的同一个类库（Spring 4 / Spring 5）
// 双亲委派会让所有应用共享同一个类 → 版本冲突！

// 解决：Tomcat 自定义类加载器
// 每个 Web 应用有独立的 WebAppClassLoader
// 优先加载 Web-INF/classes 和 WEB-INF/lib 的类
// 实现"应用间类隔离"

// Tomcat 加载顺序（倒置）：
// ① 自己的类（WEB-INF/classes）
// ② 自己 lib 的类（WEB-INF/lib）
// ③ 委派父加载器

// 这就是"破坏双亲委派"：先自己加载，再向上委派
```
#### 破坏场景3：热部署
```
// 问题：JSP 修改后需要重新加载
// 类加载器加载过的类不能卸载（没有 API）
// 解决：每次 JSP 修改，创建新的类加载器
// 新类加载器加载新的 JSP 类，旧的类加载器可以被回收

// 这就是"热部署"的原理：
// 用一个新的类加载器重新加载修改过的类
```
### 面试高频

> **Q:** "什么是双亲委派？为什么要破坏它？" 
> **A:** " 双亲委派：类加载请求先委派给父加载器，父加载器找不到才自己加载。 好处：保证核心类安全、避免重复加载。
> 
> 破坏场景： ① JDBC/SPI：核心库需要加载第三方实现，用线程上下文类加载器 ② Tomcat：多应用版本隔离，每个应用独立类加载器，先自己加载 ③ 热部署：新类加载器重新加载修改的类
### 自定义类加载器
```
// 自定义类加载器：继承 ClassLoader，重写 findClass()
public class FileClassLoader extends ClassLoader {

    private String classPath;  // 类文件目录

    public FileClassLoader(String classPath) {
        this.classPath = classPath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // ① 把类名转为文件路径
        String fileName = classPath + name.replace('.', '/') + ".class";

        try {
            // ② 读取字节流
            FileInputStream fis = new FileInputStream(fileName);
            byte[] bytes = new byte[fis.available()];
            fis.read(bytes);
            fis.close();

            // ③ 把字节流转换为 Class
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}

// 使用
FileClassLoader loader = new FileClassLoader("/tmp/classes/");
Class<?> clazz = loader.loadClass("com.example.MyClass");
Object obj = clazz.getDeclaredConstructor().newInstance();
```
关键方法

|方法|作用|
|---|---|
|`loadClass()`|加载入口（双亲委派逻辑，一般不改）|
|`findClass()`|真正加载类（自定义加载逻辑写这里）|
|`defineClass()`|字节数组 → Class 对象（核心）|
|`findLoadedClass()`|检查是否已加载|
热部署示例
```
// 热部署的核心：新类加载器加载新版本
public class HotDeployDemo {
    public static void main(String[] args) throws Exception {
        // 模拟热部署：每次用新的类加载器加载

        // 第一次加载
        FileClassLoader loader1 = new FileClassLoader("/tmp/classes/");
        Class<?> clazz1 = loader1.loadClass("com.example.Service");
        Service service1 = (Service) clazz1.getDeclaredConstructor().newInstance();
        service1.execute();

        // 修改 Service.java 重新编译后……
        // 第二次加载（新类加载器）
        FileClassLoader loader2 = new FileClassLoader("/tmp/classes/");
        Class<?> clazz2 = loader2.loadClass("com.example.Service");
        Service service2 = (Service) clazz2.getDeclaredConstructor().newInstance();
        service2.execute();  // 执行的是新版本逻辑！

        // 注意：必须用新类加载器
        // 同一个加载器加载同一个类 → 返回同一个 Class（不会重新加载）
        Class<?> clazz1Again = loader1.loadClass("com.example.Service");
        System.out.println(clazz1 == clazz1Again);  // true —— 同一个
        System.out.println(clazz1 == clazz2);       // false —— 不同加载器加载
    }
}
```
### 面试高频题
#### 题目1：能不能自己写一个java.lang.String?
```
// 能写，但永远加载不了！
// 双亲委派：请求先到 Bootstrap，Bootstrap 加载真正的 String
// 你的 String 永远不会被加载

// 除非破坏双亲委派（自定义加载器不委派直接加载）
// 但 JVM 启动时 java.lang.String 已经被 Bootstrap 加载了
// 同一 JVM 中核心类只能由 Bootstrap 加载
```
#### 题目2：两个类相等需要什么条件？
```
// 类的"唯一性"由：类名 + 类加载器共同决定
// 同一个类，不同加载器加载 → 两个不同的 Class 对象

Class<?> c1 = loader1.loadClass("com.example.User");
Class<?> c2 = loader2.loadClass("com.example.User");
System.out.println(c1 == c2);  // false！

// 所以：两个加载器加载的同类实例，instanceof 判断会失败
```
#### 题目3：什么场景需要自定义类加载器？
```
// ① 热部署（JSP、动态更新）
// ② 加密/解密字节码（防反编译）
// ③ 从非标准位置加载（网络、数据库）
// ④ 应用隔离（Tomcat）
```
#### 题目4：JDBC为什么能加载MySQL驱动？
```
// DriverManager 是 Bootstrap 加载的核心类
// 它要加载第三方 MySQL 驱动（Application 域的类）
// 双亲委派下做不到 → 用线程上下文类加载器
// DriverManager 通过 contextClassLoader 加载驱动
```