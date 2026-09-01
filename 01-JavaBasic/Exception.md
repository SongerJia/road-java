## 异常体系
异常分类->受检VS非受检->try-with-resources->异常链与性能->最佳实践
### 异常分类
```code
				   Throwable
                   /        \
              Error         Exception
              (不可恢复)      (可处理)
                           /          \
                    RuntimeException   其他 Exception
                    (非受检异常)         (受检异常)
```
#### Error
```java
// 这些是 JVM 层面的严重问题，程序无法处理，不要 try-catch 
Error e = new OutOfMemoryError();     // 堆内存溢出
Error e2 = new StackOverflowError();  // 栈溢出
Error e3 = new NoClassDefFoundError();// 类找不到
Error e4 = new AssertionError();      // 断言错误
```
#### Exception

| 类别               | 包括                                              | 是否强制处理                |
| ---------------- | ----------------------------------------------- | --------------------- |
| RubtimeException | NullPointException、IllegalArgumentException     | 编译器不强制try-catch       |
| 其他Exception      | IOException、SQLException、ClassNotFoundException | 编译器强制try-catch或throws |
### 受检异常 VS 非受检异常
#### 语法区别
```java
// 受检异常 —— 必须处理
public void readFile(String path) throws IOException {  // ① 声明 throws
    // 或者
    try {
        FileInputStream fis = new FileInputStream(path);  // ② try-catch
    } catch (IOException e) {
        e.printStackTrace();
    }
}

// 非受检异常 —— 可以不处理
public void divide(int a, int b) {
    int result = a / b;  // 可能抛出 ArithmeticException，但不需要声明
}
```
#### 设计意图区别
```java
// 受检异常 —— 调用方可以合理恢复的场景
public class BankService {
    // 余额不足 —— 调用方可以提示用户充值
    public void withdraw(double amount) throws InsufficientBalanceException {
        if (balance < amount) {
            throw new InsufficientBalanceException("余额不足");
        }
    }
}

// 非受检异常 —— 编程错误或不可恢复的场景
public class UserService {
    // 传 null 是调用方的错 —— 非受检
    public User findById(Long id) {
        if (id == null) {
            throw new IllegalArgumentException("id 不能为 null");
        }
    }
}
```
### 面试高频

> **Q:** "受检异常和非受检异常怎么选？什么时候用哪个？" 
> **A:** "
> - **受检异常**：调用方可以合理恢复的，比如重试、提示用户。例如 `IOException`、`SQLException`、`InsufficientBalanceException`。
> - **非受检异常**：调用方无法恢复的，比如编程错误（传 null、越界）、配置错误、不可预料的运行时问题。
> 
> 业界趋势：**倾向用非受检异常**。Spring、Hibernate 等框架都把受检异常包装成了非受检异常（`DataAccessException`、`HibernateException`），因为受检异常导致大量 try-catch 样板代码，破坏了代码可读性。"

> **Q:** "Spring 为什么把 SQLException 转成了 DataAccessException？" 
> **A:** "因为 `SQLException` 是受检异常，每层都要 try-catch 或 throws，污染了业务代码。Spring 把它包装成 `DataAccessException`（非受检），业务层可以只在需要的地方处理。"

### try-with-resources
JDK7以前的写法
```java
// ❌ 繁琐，容易忘记关闭
FileInputStream fis = null;
try {
    fis = new FileInputStream("test.txt");
    // 读取文件
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fis != null) {
        try {
            fis.close();  // 关闭也会抛异常，又套一层 try-catch
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
JDK7+新写法
```java
// ✅ 自动关闭，简洁安全
try (FileInputStream fis = new FileInputStream("test.txt");
     BufferedReader br = new BufferedReader(new InputStreamReader(fis))) {
    // 读取文件
    String line = br.readLine();
} catch (IOException e) {
    e.printStackTrace();
}
// fis 和 br 自动关闭，无需 finally
```
原理：AutoCloseable接口
```java
public interface AutoCloseable {
    void close() throws Exception;
}

// 几乎所有资源类都实现了这个接口
public class FileInputStream extends InputStream implements Closeable {
    // Closeable 继承了 AutoCloseable
    public void close() throws IOException { ... }
}
```
### 面试追问
> **Q:** "try-with-resources 的关闭顺序是什么？" 
> **A:** "**后创建的先关闭**（栈顺序）。`br` 先关，`fis` 后关。因为 `br` 包装了 `fis`，先关 `br` 会 flush 缓冲区，再关 `fis`。"

> **Q:** "try-with-resources 中 close 抛异常了，try 块也抛异常了，怎么处理？" 
> **A:** "try 块的异常会**抑制**（suppressed）close 抛出的异常。可以通过 `e.getSuppressed()` 获取被抑制的close异常。Java 7 引入了 `Throwable.addSuppressed()` 方法来解决这个问题。"
```java
try (MyResource r = new MyResource()) {
    throw new RuntimeException("try 异常");
} catch (RuntimeException e) {
    // e 是 "try 异常"
    // e.getSuppressed() 可以拿到 close 抛出的异常
    Throwable[] suppressed = e.getSuppressed();
}
```
### 异常链与性能
异常链
```java
public class UserService {
    public User findById(Long id) {
        try {
            return userDao.findById(id);
        } catch (DataAccessException e) {
            // 包装异常，保留原始原因
            throw new UserServiceException("查询用户失败", e);  // 第二个参数是 cause
        }
    }
}

// 获取原始异常
try {
    userService.findById(1L);
} catch (UserServiceException e) {
    Throwable cause = e.getCause();  // 拿到原始的 DataAccessException
    cause.printStackTrace();
}
```
为什么需要异常链：不丢失原始异常的根本原因，方便排查
异常的性能开销
```java
// ❌ 不要用异常控制流程
try {
    // 用异常判断类型 —— 性能极差
} catch (ClassCastException e) {
    // 正确的做法是用 instanceof
}

// ❌ 不要用异常做循环终止
int i = 0;
try {
    while (true) {
        array[i++];  // 越界时抛异常 —— 比正常判断慢 100 倍
    }
} catch (ArrayIndexOutOfBoundsException e) {
    // 正确的做法是用 for 循环或判断 length
}
```
异常为什么慢？一次异常的创建和抛出，比正常代码路径要慢100-1000倍

| 开销来源    | 说明                                                                 |
| ------- | ------------------------------------------------------------------ |
| 填充栈轨迹   | fillInStackTrace()遍历当前线程的完整调用栈，为每一个栈帧创建StackTraceElement，填充到异常对象中。 |
| 对象创建    | 异常是对象，有构造、GC开销  throw new  XXXexception(); //每次都是对象分配              |
| JIT优化失败 | try-catch会阻止JIT某些优化，打断控制流图、阻止某些方法内联、增加异常表维护成本                      |
优化建议
```java
// 方案一：预创建异常对象（不填充栈轨迹）
public class BusinessException extends RuntimeException {
    // 重写 fillInStackTrace 跳过栈轨迹扫描填充
    @Override
    public synchronized Throwable fillInStackTrace() {
        return this;  // 不填充栈轨迹，性能提升 10~100 倍 代价是没有堆栈信息
    }
}

// 方案二：用返回码替代异常（高频路径）
public class Result<T> {
    private boolean success;
    private T data;
    private String errorCode;

    public static <T> Result<T> ok(T data) { ... }
    public static <T> Result<T> fail(String errorCode) { ... }
}

// 使用 —— 零异常开销
Result<User> result = userService.findById(1L);
if (!result.isSuccess()) {
    // 处理失败
}
```
### 最佳实践
#### 原则1：异常不被吞掉
```java
// ❌ 错误：吞掉异常，无法排查
try {
    // ...
} catch (Exception e) {
    // 什么都不做 —— 异常被吞了！
}

// ✅ 正确：至少记录日志
try {
    // ...
} catch (Exception e) {
    log.error("操作失败", e);  // 记录日志
    throw new BusinessException("操作失败", e);  // 或者重新包装抛出
}
```
#### 原则2：不使用异常控制流程
```java
// ❌ 错误：用异常判断
try {
    return userService.findById(id);
} catch (UserNotFoundException e) {
    return null;
}

// ✅ 正确：用返回值判断
User user = userService.findById(id);
if (user == null) {
    return null;
}
// 或者用 Optional
return userService.findByIdOptional(id).orElse(null);
```
#### 原则3：抛异常要具体，不使用Exception
```java
// ❌ 错误：太笼统
throw new Exception("出错了");

// ✅ 正确：具体异常类
throw new IllegalArgumentException("参数不能为 null");
throw new UserNotFoundException("用户不存在，id=" + id);
throw new InsufficientBalanceException("余额不足，需要 " + amount + "，当前 " + balance);
```
#### 原则4：在合适的层级处理异常
```java
// Controller 层：处理异常，返回用户友好的信息
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException e) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse(e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnknown(Exception e) {
        log.error("未知异常", e);  // 记录日志
        return ResponseEntity.status(500)
            .body(new ErrorResponse("系统繁忙，请稍后重试"));
    }
}

// Service 层：抛出业务异常，不处理
public class UserService {
    public User findById(Long id) {
        User user = userDao.findById(id);
        if (user == null) {
            throw new UserNotFoundException("用户不存在");
        }
        return user;
    }
}

// DAO 层：抛出数据访问异常（DataAccessException）
```
#### 原则5：自定义异常规范
```java
// 项目中的统一异常体系
public abstract class BaseException extends RuntimeException {
    private final String errorCode;

    public BaseException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public BaseException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}

// 具体业务异常
public class UserNotFoundException extends BaseException {
    public UserNotFoundException(String message) {
        super("USER_NOT_FOUND", message);
    }
}

public class InsufficientBalanceException extends BaseException {
    public InsufficientBalanceException(String message) {
        super("BALANCE_INSUFFICIENT", message);
    }
}
```