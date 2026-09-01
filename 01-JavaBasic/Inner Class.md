## 内部类
四种内部类语法->字节码视角->为什么需要内部类->内存泄漏问题->实际应用场景
### 四种内部类
#### Member Inner Class
成员内部类
```java
public class Outer {
    private String name = "Outer";

    // 成员内部类 —— 就像 Outer 的一个成员变量
    public class Inner {
        public void print() {
            System.out.println(name);  // ✅ 可以直接访问外部类的 private 成员
        }
    }
}

// 使用
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();  // 必须通过外部类实例创建
```

| 要点      | 说明                              |
| ------- | ------------------------------- |
| 持有外部类引用 | 隐式持有 Outer.this                 |
| 访问权限    | 可以访问外部类所有成员包括private            |
| 创建方式    | outer.new Inner()               |
| 编译后     | Outer.class + Outer$Inner.class |
#### Static Nested Class
静态内部类
```java
public class Outer {
    private static String staticName = "Static";
    private String name = "Instance";

    // 静态内部类 —— 不持有外部类引用
    public static class StaticInner {
        public void print() {
            System.out.println(staticName);  // ✅ 只能访问外部类的静态成员
            // System.out.println(name);     // ❌ 不能访问实例成员
        }
    }
}

// 使用 —— 不需要外部类实例
Outer.StaticInner inner = new Outer.StaticInner();
```

| 要点      | 说明                              |
| ------- | ------------------------------- |
| 持有外部类引用 | 不持有                             |
| 访问权限    | 只能访问外部类的static成员                |
| 创建方式    | new Outer.StaticInner()         |
| 使用场景    | 和外部逻辑相关，但不需要访问外部实例，HashMap.Node |
#### Local Inner Class
局部内部类
```java
public class Outer {
    public void method() {
        String localVar = "local";

        // 局部内部类 —— 定义在方法内
        class LocalInner {
            public void print() {
                System.out.println(localVar);  // 只能访问 final 或 effectively final 的局部变量
            }
        }

        LocalInner inner = new LocalInner();
        inner.print();
    }
}
```

| 要点     | 说明                           |
| ------ | ---------------------------- |
| 作用域    | 只在定义它的方法可见                   |
| 访问局部变量 | 变量必须是final或effectively final |
| 使用场景   | 很少使用，一版                      |
#### Anonymous inner class
匿名内部类
```java
public class Outer {
    public void method() {
        // 匿名内部类 —— 没有名字，同时定义和实例化
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("Running...");
            }
        };
        new Thread(task).start();
    }
}
```

| 要点     | 说明             |
| ------ | -------------- |
| 语法     | new 接口/类() 和实现 |
| 适用场景   | 只需要一次使用的接口/类实现 |
| 限制     | 不能有构造器、不能有静态成员 |
| java8后 | 很多被Lambda替代    |
### 字节码视角
内部类编译后会生成独立的字节码文件，命名规则：
```
Outer.java 编译后：
  Outer.class          ← 外部类
  Outer$Inner.class        ← 成员内部类
  Outer$StaticInner.class  ← 静态内部类
  Outer$1LocalInner.class  ← 局部内部类（数字编号）
  Outer$1.class            ← 匿名内部类（数字编号）
```
成员内部类隐式持有外部类的引用
```java
// 源码
public class Outer {
    private String name;

    public class Inner {
        public void print() {
            System.out.println(name);  // 实际上是 Outer.this.name
        }
    }
}
```
反编译后等价于
```java
public class Outer$Inner {
    // 编译器自动添加的字段 —— 持有外部类引用
    final Outer this$0;

    // 编译器自动添加的构造器
    public Outer$Inner(Outer outer) {
        this.this$0 = outer;
    }

    public void print() {
        // 通过持有的外部类引用访问
        System.out.println(this$0.name);
    }
}
```
**面试追问：**

> **Q:** "成员内部类为什么能访问外部类的 private 成员？" 
> **A:** "编译器在外部类中生成了 `static` 的**访问器方法**（accessor methods），内部类通过调用这些方法来访问 private 成员。本质上是语法糖。accessor只是用于绕过JVM访问限制，不是所有的private成员都生成，只有被访问的才会"

> **Q:** "静态内部类和成员内部类的根本区别是什么？" 
> **A:** "静态内部类**不持有**外部类实例的引用，所以不会导致外部类实例无法被 GC 回收。成员内部类持有 `Outer.this`，可能导致内存泄漏。"

### 为什么需要内部类
原因1：实现多继承的变通方案
```java
// 需求：一个类同时拥有两种行为
public class MusicPlayer {

    // 内部类1：继承 Thread，实现播放控制
    private class PlayThread extends Thread {
        @Override
        public void run() {
            // 播放音乐
        }
    }

    // 内部类2：继承 ActionListener，实现按钮事件
    private class ButtonListener implements ActionListener {
        @Override
        public void actionPerformed(ActionEvent e) {
            // 处理按钮点击
        }
    }
}
```
原因2：逻辑上属于同一模块，但需要独立存在
```java
public class HashMap<K, V> {
    // Node 是 HashMap 的逻辑组成部分，但需要独立存在
    static class Node<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;
    }

    // KeySet 也是 HashMap 的一部分  被final修饰不能被继承
    final class KeySet extends AbstractSet<K> {
        // ...
    }
}
```
原因3：强封装性，隐藏细节
```java
public class DatabaseConnection {
	//这是定义了一个具体处理逻辑的接口，外部调用可以使用lambda
    public interface RowCallback {
        void processRow(Map<String, Object> row);
    }

    private Connection conn; // 真实连接
	//这个私有的局部内部类，是封装了遍历逻辑
    private class ResultSetIterator implements Iterator<Map<String, Object>> {

        private final ResultSet rs;
        private final ResultSetMetaData meta;
        private boolean hasNext;

        ResultSetIterator(String sql) throws SQLException {
            Statement stmt = conn.createStatement();
            rs = stmt.executeQuery(sql);
            meta = rs.getMetaData();
            hasNext = rs.next();
        }

        @Override
        public boolean hasNext() {
            return hasNext;
        }

        @Override
        public Map<String, Object> next() {
            try {
                Map<String, Object> row = new HashMap<>();
                for (int i = 1; i <= meta.getColumnCount(); i++) {
                    row.put(meta.getColumnName(i), rs.getObject(i));
                }
                hasNext = rs.next();
                if (!hasNext) rs.close();
                return row;
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }
    }
	//这个方法是提供给外部使用的，逻辑的入口，将遍历封装，只关注实际的逻辑处理
    public void query(String sql, RowCallback callback) {
        try {
            ResultSetIterator it = new ResultSetIterator(sql);
            while (it.hasNext()) {
                callback.processRow(it.next());
            }
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
}
```
### 内存泄漏问题
成员内部类和匿名内部类的陷阱
问题场景
```java
//成员内部类
public class Outer {
    private List<String> cache = new ArrayList<>();

    public class Inner {
        public void doSomething() {
            // Inner 持有 Outer.this，Outer 不会被 GC
        }
    }

    public Inner getInner() {
        return new Inner();
    }
}

// 使用方
Outer outer = new Outer();
Outer.Inner inner = outer.getInner();
outer = null;  // ❌ Outer 不会被 GC！因为 Inner 还持有它的引用

//匿名内部类
public class Activity {
    private TextView textView;

    public void start() {
        // 匿名内部类持有 Activity 的引用
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(10000);  // 长时间操作
                } catch (Exception e) {}
                textView.setText("Done");  // 访问外部类成员
            }
        }).start();
    }
}
```
解决方案
```java
// 方案一：用静态内部类代替成员内部类
public class Outer {
    private String name;

    public static class SafeInner {
        // 不持有 Outer.this，不会导致泄漏
        public void doSomething() {
            // 但只能访问外部类的静态成员
        }
    }
}

// 方案二：匿名内部类使用弱引用
public class Activity {
    private TextView textView;

    public void start() {
        final WeakReference<Activity> ref = new WeakReference<>(this);

        new Thread(new Runnable() {
            @Override
            public void run() {
                Activity activity = ref.get();
                if (activity != null) {
                    // 安全操作
                }
            }
        }).start();
    }
}
```
**面试高频：**

> **Q:** "Lambda 表达式会导致内存泄漏吗？" 
> **A:** "Lambda 不会持有外部类的 `this` 引用（如果在实例方法中、且访问了外部类的实例成员，也会隐式持有外部类的 `this` 引用）——Lambda 本质上是 `invokedynamic` 指令，它捕获的是变量，不是 `this`。所以 Lambda 比匿名内部类更安全。"

### 实际应用场景
#### 场景1：HashMap中的Node
```java
public class HashMap<K, V> {
    // 静态内部类 —— 不需要访问外部类实例
    static class Node<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;
    }

    // 成员内部类 —— 需要访问外部类的 table 等成员
    final class KeySet extends AbstractSet<K> {
        public final int size() {
            return size;  // 访问 HashMap 的 size
        }
    }
}
```
#### 场景2：构造器模式
```java
public class User {
    private String name;
    private int age;
    private String email;

    // 静态内部类 Builder
    public static class Builder {
        private String name;
        private int age;
        private String email;

        public Builder setName(String name) {
            this.name = name;
            return this;
        }

        public Builder setAge(int age) {
            this.age = age;
            return this;
        }

        public User build() {
            User user = new User();
            user.name = this.name;   // 访问外部类的 private 字段
            user.age = this.age;
            user.email = this.email;
            return user;
        }
    }

    private User() {}  // 构造器私有，只能用 Builder 创建
}

// 使用
User user = new User.Builder()
    .setName("张三")
    .setAge(25)
    .build();
```
#### 场景3：事件监听器 Java GUI经典用法
```java
public class Button {
    private List<ClickListener> listeners = new ArrayList<>();

    public interface ClickListener {
        void onClick();
    }

    public void addListener(ClickListener listener) {
        listeners.add(listener);
    }
}

// 使用方
Button button = new Button();
button.addListener(new Button.ClickListener() {
    @Override
    public void onClick() {
        System.out.println("Button clicked!");
    }
});
// Java 8+ 简化为 Lambda
button.addListener(() -> System.out.println("Button clicked!"));
```

