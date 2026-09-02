## 注解
注解基础->元注解->自定义注解->注解处理->框架中的注解->注解vs其他方案
### 注解基础
#### 什么是注解？
注解是一种metadata元数据形式，给代码打标签。本身不改变代码行为，但可以被编译器或运行期读取处理。
```java
// 注解本质上是一个接口 —— 隐式继承 java.lang.annotation.Annotation
public @interface Override {
    // 没有属性 —— 标记注解
}

// 使用
@Override
public String toString() {
    return "...";
}
```
#### Java内置注解
```java
// ① @Override —— 标记重写，编译器检查
@Override
public boolean equals(Object obj) {
    // 编译器检查：如果方法签名不对，编译报错
}

// ② @Deprecated —— 标记废弃，编译器警告
@Deprecated
public void oldMethod() {
    // 使用时会编译警告
}

// ③ @SuppressWarnings —— 抑制编译器警告
@SuppressWarnings("unchecked")
public void process(List list) {
    List<String> strings = list;  // 不会警告
}

// ④ @FunctionalInterface（Java 8+）—— 标记函数式接口
@FunctionalInterface
public interface Runnable {
    void run();
    // 如果有两个抽象方法，编译报错
}
```
### Meta-Annotation
元注解：元注解是注解的注解，定义注解本身的特性
@Target：注解可以用在什么地方
```java
@Target({
    ElementType.TYPE,               // 类、接口、枚举
    ElementType.FIELD,              // 字段（包括枚举常量）
    ElementType.METHOD,             // 方法
    ElementType.PARAMETER,          // 方法参数
    ElementType.CONSTRUCTOR,        // 构造器
    ElementType.LOCAL_VARIABLE,     // 局部变量
    ElementType.ANNOTATION_TYPE,    // 注解类型
    ElementType.PACKAGE,            // 包
    ElementType.TYPE_PARAMETER,     // 类型参数（Java 8+）
    ElementType.TYPE_USE            // 类型使用（Java 8+）
})
public @interface MyAnnotation {}
```
@Relention：注解保留到什么时候
```java
@Retention(RetentionPolicy.SOURCE)   // 源码期，编译后丢弃
@Retention(RetentionPolicy.CLASS)    // 保留到 class 文件，但运行期不可反射（默认值）
@Retention(RetentionPolicy.RUNTIME)  // 保留到运行期，可通过反射读取
```
应用场景对比：

| Retention | 典型注解                         | 作用             |
| --------- | ---------------------------- | -------------- |
| SOURCE    | @Override  @SuppressWarnings | 编译期检查，运行期不需要   |
| CLASS     | Lombok的 @Getter              | 编译期生成代码，运行期不需要 |
| RUNTIME   | @Autowried  @Transactional   | 框架运行期通过反射获取    |
@Documented：是否包含在Javadoc中
```java
@Documented
public @interface MyAnnotation {}
// 使用 @MyAnnotation 的类，生成 Javadoc 时会包含注解信息
```
@Inherited：子类是否继承父类的注解
```Java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {}

@MyAnnotation
public class Parent {}

// Child 自动继承了 @MyAnnotation
public class Child extends Parent {}

// 测试
Child.class.getAnnotation(MyAnnotation.class);  // ✅ 不为 null
//@Inherited 只对类上的注解生效，对方法或字段的注解不生效
```
@Repeatable：重复注解（JDK8+）
```java
// Java 8 前：不能重复使用同一个注解
// @Schedule("10:00")
// @Schedule("14:00")  // ❌ 编译报错

// Java 8+：通过 @Repeatable 支持重复使用   需要先写这个
@Repeatable(Schedules.class)  // 指定容器注解
@Retention(RetentionPolicy.RUNTIME)
public @interface Schedule {
    String value();
}

// 容器注解 —— 用来存储多个 @Schedule    再定义这个
@Retention(RetentionPolicy.RUNTIME)
public @interface Schedules {
    Schedule[] value();
}

// 使用 —— 可以重复写       前面2步完成才能使用
@Schedule("10:00")
@Schedule("14:00")
@Schedule("18:00")
public void dailyTask() {}

// 读取
Schedule[] schedules = method.getAnnotationsByType(Schedule.class);
```
### 自定义注解
定义注解
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
    // 注解的属性（看起来像方法，实际上叫属性）
    String value() default "";              // 有默认值，使用时可省略
    LogLevel level() default LogLevel.INFO; // 枚举类型
    boolean recordParams() default true;    // 基本类型
    Class<?>[] groups() default {};         // Class 类型
}

public enum LogLevel {
    DEBUG, INFO, WARN, ERROR
}
```
注解属性的类型限制：

| 支持的类型   | 举例                      |
| ------- | ----------------------- |
| 基本类型    | int、boolean、double      |
| String  | String value()          |
| 枚举      | LogLevel level()        |
| Class   | Class< ? > class()      |
| 注解      | AnotherAnnotation ann() |
| 以上类型的数组 | String[] tag()          |
使用注解
```java
public class UserService {

    // 只写 value，其他用默认值
    @Loggable("执行用户查询")
    public User findUser(Long id) {
        return userDao.findById(id);
    }

    // 指定多个属性
    @Loggable(
        value = "删除用户",
        level = LogLevel.WARN,
        recordParams = true
    )
    public void deleteUser(Long id) {
        userDao.deleteById(id);
    }

    // 属性的快捷方式 —— 如果只有一个属性叫 value，使用时可省略属性名
    @Loggable("查询全部")  // 等价于 @Loggable(value = "查询全部")
    public List<User> findAll() {
        return userDao.findAll();
    }
    
    // 属性的value是特殊的，如果给其他属性赋值，必须显式写value
    @Loggable(level = LogLevel.WARN)  //❌报错 还需要给value赋值
    public List<User> findAll() {
        return userDao.findName(String name);
    }
}
```
### 注解处理
#### 方式1：运行期反射处理
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
    String value() default "";
    LogLevel level() default LogLevel.INFO;
}

// ① 通过反射读取注解   这个是手动调用的效果，如果是字段注解，可以使用
public class AnnotationReader {
    public static void printAnnotations(Object obj) {
        Class<?> clazz = obj.getClass();

        for (Method method : clazz.getDeclaredMethods()) {
            // 判断是否有 @Loggable 注解
            if (method.isAnnotationPresent(Loggable.class)) {
                // 获取注解实例
                Loggable loggable = method.getAnnotation(Loggable.class);
                System.out.println("方法: " + method.getName());
                System.out.println("  描述: " + loggable.value());
                System.out.println("  级别: " + loggable.level());
            }
        }
    }
}

// ② 结合 AOP 实现切面（Spring AOP 的原理）  只能拦截外部对方法调用
@Aspect
@Component
public class LoggableAspect {

    @Around("@annotation(loggable)")  // 匹配有 @Loggable 的方法
    public Object logExecution(ProceedingJoinPoint joinPoint, Loggable loggable) throws Throwable {
        long start = System.currentTimeMillis();
        String methodName = joinPoint.getSignature().getName();

        // 前置通知
        log.info("[{}] 开始执行: {}", loggable.level(), methodName);

        try {
            Object result = joinPoint.proceed();  // 执行原方法
            long duration = System.currentTimeMillis() - start;
            // 后置通知
            log.info("[{}] 执行完成: {}, 耗时 {}ms", loggable.level(), methodName, duration);
            return result;
        } catch (Exception e) {
            log.error("[{}] 执行失败: {}, 异常: {}", loggable.level(), methodName, e.getMessage());
            throw e;
        }
    }
}
```
#### 编译期APT处理
Annotation Processing Tool：编译时生成代码
```java
// ① 定义注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)  // 编译期使用，运行期丢弃
public @interface Builder {
    // 自动生成 Builder 类
}

// ② 注解处理器
@SupportedAnnotationTypes("com.example.Builder")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
	    //javax.lang.model.element.Element.getAnnotation(Class<T>)
        for (Element element : roundEnv.getElementsAnnotatedWith(Builder.class)) {
            if (element.getKind() == ElementKind.CLASS) {
                TypeElement typeElement = (TypeElement) element;
                // 读取类信息，生成 Builder 源码
                generateBuilder(typeElement);
            }
        }
        return true;
    }

    private void generateBuilder(TypeElement typeElement) {
        // 生成代码的 String
        String code = generateCode(typeElement);
        // 写入文件
        try {
            JavaFileObject file = processingEnv.getFiler()
                .createSourceFile(typeElement.getSimpleName() + "Builder");
            try (Writer writer = file.openWriter()) {
                writer.write(code);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
//执行流程
javac 编译
  ↓
扫描源码中的注解
  ↓
找到 @Builder 标注的类
  ↓
调用 BuilderProcessor.process()
  ↓
生成 XxxBuilder.java 源码文件
  ↓
编译器继续编译这个新生成的文件
  ↓
最终产物：.class 文件
```
Lombok的做法：
```java
// Lombok 的 @Getter —— 编译期自动生成 getter 方法
@Getter
@Setter
@Builder
public class User {
    private String name;
    private int age;
}

// 编译后自动生成：
// getName()、setName()、UserBuilder 类
```
两种方式对比：

| 对比        | 运行期（反射）   | 编译期（APT）         |
| --------- | --------- | ---------------- |
| 处理时机      | 运行期       | 编译期              |
| Retention | 需要RUNTIME | SOURCE、CLASS     |
| 性能        | 有反射开销     | 无运行时开销           |
| 示例        | Spring注解  | Lombok、MapStruct |
| 优点        | 灵活、动态     | 性能好、不依赖反射        |
### 框架中的注解
#### Spring的注解体系
```
// ① 组件注册
@Component         // 通用组件
@Service           // 业务层
@Repository        // 数据访问层
@Controller        // 控制层
@RestController    // RESTful 控制层（@Controller + @ResponseBody）

// ② 依赖注入
@Autowired         // 按类型注入
@Qualifier         // 按名称注入（配合 @Autowired）
@Resource          // 按名称注入（JDK 注解）
@Value             // 注入配置文件的值

// ③ 配置
@Configuration     // 配置类
@Bean              // 声明 Bean
@ComponentScan     // 组件扫描
@PropertySource    // 引入配置文件

// ④ 事务与AOP
@Transactional     // 声明式事务
@Aspect            // 切面
@Around            // 环绕通知
@Before            // 前置通知
@After             // 后置通知

// ⑤ MVC
@RequestMapping    // 请求映射
@GetMapping        // GET 请求
@PostMapping       // POST 请求
@PathVariable      // 路径参数
@RequestParam      // 请求参数
@RequestBody       // 请求体
```
Spring注解的读取机制
```java
// Spring 启动时，通过反射扫描所有注解
public class AnnotationScanner {

    // 扫描指定包下的所有类，查找有 @Component 注解的类
    public Set<Class<?>> scanComponents(String basePackage) {
        Set<Class<?>> components = new HashSet<>();

        // 扫描包下的所有类文件
        Set<Class<?>> classes = scanPackage(basePackage);

        for (Class<?> clazz : classes) {
            // 检查是否有 @Component 注解（包括 @Service、@Controller 等组合注解）
            if (clazz.isAnnotationPresent(Component.class)) {
                components.add(clazz);
            }
        }

        return components;
    }

    // 支持注解的组合（@Service 上包含了 @Component）
    public boolean hasComponentAnnotation(Class<?> clazz) {
        for (Annotation annotation : clazz.getAnnotations()) {
            // 判断注解本身是否有 @Component
            if (annotation.annotationType() == Component.class) {
                return true;
            }
            // 递归检查注解的注解（@Service → @Component）
            if (hasComponentAnnotation(annotation.annotationType())) {
                return true;
            }
        }
        return false;
    }
}
```
@Transactional是怎么工作的
```java
// ① 定义注解
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Transactional {
    Propagation propagation() default Propagation.REQUIRED;
    Isolation isolation() default Isolation.DEFAULT;
    Class<? extends Throwable>[] rollbackFor() default {};
}

// ② Spring AOP 拦截有 @Transactional 的方法
@Aspect
@Component
public class TransactionAspect {

    @Around("@annotation(transactional)")  // 匹配有 @Transactional 的方法
    public Object manageTransaction(ProceedingJoinPoint joinPoint, Transactional transactional) throws Throwable {
        // 根据注解属性配置事务
        TransactionDefinition def = new TransactionDefinition();
        def.setPropagationBehavior(transactional.propagation().value());
        def.setIsolationLevel(transactional.isolation().value());

        // 开启事务
        TransactionStatus status = transactionManager.getTransaction(def);
        try {
            Object result = joinPoint.proceed();  // 执行业务方法
            transactionManager.commit(status);     // 提交事务
            return result;
        } catch (Exception e) {
            // 判断是否需要回滚
            if (shouldRollback(e, transactional)) {
                transactionManager.rollback(status);  // 回滚事务
            }
            throw e;
        }
    }

    private boolean shouldRollback(Exception e, Transactional transactional) {
        for (Class<?> rollbackClass : transactional.rollbackFor()) {
            if (rollbackClass.isInstance(e)) {
                return true;  // 命中回滚异常
            }
        }
        // 默认只回滚 RuntimeException 和 Error
        return e instanceof RuntimeException || e instanceof Error;
    }
}
```
### 注解vs其他方案
#### 注解vsXML配置
```java
// 注解方式 —— 简洁，与代码在一起
@Transactional
public void transferMoney(Long from, Long to, double amount) {
    // ...
}

// XML 方式 —— 与代码分离，集中管理
// beans.xml
// <bean id="userService" class="com.example.UserService">
//     <property name="userDao" ref="userDao"/>
// </bean>
// <aop:config>
//     <aop:pointcut id="txPointcut" expression="execution(* com.example.*.*(..))"/>
//     <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointcut"/>
// </aop:config>
```

| 对比   | 注解        | XML        |
| ---- | --------- | ---------- |
| 耦合性  | 与代码耦合     | 与代码分离      |
| 简洁性  | 简洁，一行搞定   | 繁琐，需要大量配置  |
| 类型安全 | 编译期检查     | 运行期才能发现错误  |
| 适用场景 | 固定配置，不常改动 | 可能需要动态切换配置 |
#### 注解 vs 命名约定
```java
// 命名约定 —— Spring 早期的方式
// 以 "Impl" 结尾的类自动识别为实现类
public class UserServiceImpl implements UserService {}

// 注解 —— 更清晰，更灵活
@Service
public class UserService{}

// 另一个例子：JUnit
// 命名约定：方法名以 test 开头
public void testFindUser() {}

// 注解：更清晰
@Test
public void findUser() {}
```
#### 注解 vs 接口
```java
// 接口 —— 定义行为契约
public interface Flyable {
    void fly();
}

// 注解 —— 提供元数据
public @interface Flyable {
    int maxSpeed() default 100;
    boolean canLand() default true;
}

// 接口：类能做什么（行为）
// 注解：类有什么特性（元数据）
```