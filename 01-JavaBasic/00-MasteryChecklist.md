# 精通清单 · Java 基础

> **使用说明**
> 1. 合上资料，录音回答每题 → 对照三级标准自评（及格 / 良好 / 精通）
> 2. 未达「精通」的题 → 回锚点材料补 → **3 天后重答**
> 3. 单题标准：**及格** = 核心结论 + 主干要点；**良好** = 及格 + 能解释「为什么」+ 一个应用/坑；**精通** = 良好 + 源码级细节 + 连环追问不卡壳
>
> **本板块通关线**：5 题全部达到「精通」档（同见根目录全局标准）

---

## 1. String 为什么不可变？`new String("a")` 创建几个对象？

**锚点**：JDK 源码 `String.java` / javadoc

- [ ] **及格**：说出不可变三层含义——① String 类是 final 的；② `value` 是 final 的（引用不可换）；③ 不暴露修改方法，所有「修改」操作返回新对象
- [ ] **良好**：及格 + 能解释不可变的好处（常量池复用安全、线程安全、可安全做 HashMap key、类加载安全）+ 说出 JDK9+ 内部由 `char[]` 改为 `byte[]` + `coder` 字段（英文场景省一半内存）
- [ ] **精通**：良好 + 能应对追问——反射能否破坏不可变（能，`setAccessible`，但属暴力破坏）、`intern()` 原理（常量池有则返回、无则入池）、`new String("a")` 创建 1 个还是 2 个对象（取决于常量池是否已有）、常量池位置（JDK6 永久代 / JDK7+ 堆）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 2. `equals` 与 `hashCode` 的约定？违反会怎样？

**锚点**：`Object.java` 注释 / 《Effective Java》第 11 条

- [ ] **及格**：说出约定——equals 相等则 hashCode 必相等；hashCode 相等 **不要求** equals 相等；重写 equals 必须重写 hashCode
- [ ] **良好**：及格 + 能解释为什么（HashMap/HashSet 先按 hashCode 定位桶，再按 equals 比内容）+ 说出 equals 的 5 条性质（自反/对称/传递/一致/与 null 比较返回 false）
- [ ] **精通**：良好 + 能举出违反约定的实际后果（只重写 equals 不重写 hashCode → 两个「相等」对象 hashCode 不同、落在不同桶 → `get` 找不到）+ 能说出面试经典陷阱（如用可变对象做 key、`hashCode` 用可变字段计算）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 3. 泛型为什么叫「类型擦除」？桥接方法是什么？

**锚点**：javadoc 泛型章节 / 《Java 泛型》/ `ClassSignature` 字节码

- [ ] **及格**：编译后泛型类型被擦除为 `Object`（或上界类型），**运行时没有泛型**；`List<String>` 编译后与 `List` 相同
- [ ] **良好**：及格 + 能解释为什么要擦除（**向后兼容**旧代码/字节码）+ 桥接方法的作用（保证**多态**不被破坏，如 `Comparable` 的 `compareTo(Object)` 桥接到 `compareTo(Integer)`）
- [ ] **精通**：良好 + 能应对追问——反射到底能不能拿到泛型（能，`Signature` 属性保留，Gson `TypeToken` 的原理）、擦除的坑（不能 `new T()`、不能 `instanceof T`、静态字段不能依赖类型参数、`List<String>` 与 `List<Integer>` 运行时不区分）

**自评记录**：___（档位） 日期：___ 复答：___

---

## 4. `Integer` 缓存池？自动装箱的坑？

**锚点**：`Integer.java` 源码 `valueOf()` / javadoc

- [ ] **及格**：`Integer` 缓存范围 **-128 ~ 127**；`valueOf()` 走缓存；`==` 比较两个装箱对象有陷阱
- [ ] **良好**：及格 + 能解释为什么缓存（高频小整数复用，省对象）+ `==` vs `equals` 的区别 + 哪些包装类有缓存（`Byte`/`Short`/`Integer`/`Long` 全范围，`Character` 部分，`Float`/`Double` 无）
- [ ] **精通**：良好 + 能应对追问——缓存范围可调（`-XX:AutoBoxCacheMax`）、`new Integer(1)` vs `Integer.valueOf(1)` 的区别（前者强制新对象，后者走缓存）、`Integer a = 100; Integer b = 100; a == b` 是多少

**自评记录**：___（档位） 日期：___ 复答：___

---

## 5. 异常体系？受检异常 vs 非受检异常？

**锚点**：javadoc `Throwable` / 《Java 核心技术》异常章节

- [ ] **及格**：说出体系——`Throwable` → `Error` / `Exception`；`Exception` → 受检异常 / `RuntimeException`（非受检）；受检异常必须处理（throws/捕获），非受检不需要
- [ ] **良好**：及格 + 能解释为什么设计受检异常（编译期强制处理，防止忽略错误）+ `try-with-resources` 的原理（`AutoCloseable`，自动 close）+ 异常性能开销（`new` 异常时填充堆栈，循环中抛异常慢）
- [ ] **精通**：良好 + 能应对追问——自定义异常怎么设计（何时继承 Exception vs RuntimeException）、异常链（`initCause`）、`finally` 中 `return` 的陷阱（会吞掉 try 中的 return/异常）、受检异常在函数式接口（`Runnable`）里的坑

**自评记录**：___（档位） 日期：___ 复答：___

---

> **通关检查**：本板块 5 题是否全部达到「精通」档？是 → 进入下一板块；否 → 标记未达标的题号：___
