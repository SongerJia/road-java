# 01-考点：protected 为什么"子类能访问，却不能通过父类实例访问"？

## 别名问法
- protected 修饰的方法，不同包的子类能调用吗？
- 为什么跨包子类里 `new Parent().hook()` 编译报错？
- protected 的访问范围到底怎么算？
- 子类访问父类 protected 成员有什么限制？

## 正文锚点（写作指引，内容区自写）
- 本地材料：`Object Oriented.md` §private/protected（访问矩阵 + new Parent() 报错案例）
- 一句话表层结论：protected 允许**子类在自己体内**访问继承来的成员，但限定类型的类型必须是"当前子类 S 或 S 的子类"——父类实例的限定类型是 C，不在许可范围
- 机制层骨架（三 case 对照，第三 case 是本地材料没写透的增量）：
  - `this.hook()` → simple name，发生在 S 体内 → ✅
  - `new Child().hook()` → 限定类型 = Child = S → ✅（JLS 允许 S 的子类）
  - `new Parent().hook()` → 限定类型 = Parent = C ≠ S 及其子类 → ❌
- 为什么层骨架：JLS 原话 "A subclass S is regarded as being **responsible for the implementation** of objects of class C"——protected 是子类实现自己职责的扩展点，不是操作任意 C 实例的后门；若允许限定类型为 C，protected 就退化成 public
- 现实印证：JDK 的 ArrayList#removeRange 是 protected——外部删除区间的公开姿势是 `list.subList(from, to).clear()`，clear 内部才调 removeRange；SO 高票引 Effective Java Item 40 讲这个设计取舍

## 追问链
- [[protected 与 default（包私有）的矩阵差异？跨包的 default 能见吗？]] 待答
- [[protected 暴露实现给子类，和继承破坏封装（白盒复用）什么关系？]] 待答 → 独立成篇 [[A1-09]]
- [[为什么 JDK 把 removeRange 设计成 protected 而不是 public？API 设计上怎么权衡？]] 待答
- [[record 的自动不可变封装，与本篇的封装边界有什么呼应？]] 待答 → 独立成篇 [[A1-02]]

## 验证

