> **模块边界**：线程安全集合（`ConcurrentHashMap` / `CopyOnWriteArrayList` / `BlockingQueue` 等）→ 并发模块；`Stream` / `Optional` 等流式 API → 新特性模块。本模块只讲 `java.util` 下的常规集合。 **与基础模块的分工**：`equals`/`hashCode` 的**契约本身**在基础模块，本模块只讲它在集合中的**后果**。

**① 集合框架总览**

1. 集合框架两大分支：Collection 与 Map
2. Iterable / Iterator / ListIterator 接口
3. 集合 vs 数组的区别与选择
4. 集合框架中的设计模式（迭代器模式、适配器模式）

**② Collection 接口族** 5. Collection 接口的通用方法 6. Collection 的 Java 8 默认方法（forEach / removeIf / stream） 7. List 的语义：有序、可重复、有索引 8. Set 的语义：不可重复 9. Queue / Deque 的语义：先进先出与双端 10. 各接口方法的返回值语义（add 返回 boolean、put 返回旧值）

**③ List 实现** 11. ArrayList：动态数组、扩容机制（1.5 倍） 12. LinkedList：双向链表、同时实现 Deque 13. Vector：同步、历史遗留 14. ArrayList vs LinkedList 的选型 15. `Arrays.asList` 与 `List.of` 的坑 16. ListIterator：双向遍历与遍历中增删

**④ Set 实现** 17. HashSet：基于 HashMap 18. LinkedHashSet：插入序 19. TreeSet：基于 TreeMap、有序 20. 三者对比与选型

**⑤ Queue / Deque 实现** 21. ArrayDeque：循环数组、不允许 null 22. PriorityQueue：二叉堆、只保证队首最小、遍历不保证有序、siftUp/siftDown 23. Queue 方法差异（add/offer、remove/poll、element/peek）

**⑥ Map 实现** 24. HashMap 底层结构：数组 + 链表 + 红黑树 25. HashMap 的 put / get 流程 26. HashMap 的哈希扰动函数 27. 容量为什么是 2 的幂、tableSizeFor 取整 28. HashMap 扩容机制、负载因子 0.75 的由来、resize 的 `(e.hash & oldCap)` 重哈希优化 29. 树化与退化阈值（8 / 6，且 table ≥ 64 才树化） 30. HashMap 的 null 处理与 Hashtable / ArrayDeque 的差异 31. HashMap 冲突处理：JDK 8 尾插、链表转树条件 32. HashMap 的 JDK 7 vs JDK 8 差异（头插/尾插、死循环 vs 数据覆盖、结构演进） 33. LinkedHashMap：插入序 / 访问序、LRU 实现（accessOrder + removeEldestEntry） 34. TreeMap：红黑树、自然序 vs Comparator、不允许 null key（自然序时） 35. Hashtable：同步、历史遗留 36. HashMap vs Hashtable 37. HashMap 线程不安全的原因 38. Map 的三种遍历（keySet / values / entrySet）与 Map.Entry 39. containsKey / containsValue / getOrDefault / putIfAbsent

**⑦ equals / hashCode 与集合** 40. equals / hashCode 契约在集合中的影响 41. 为什么重写 equals 必须重写 hashCode

**⑧ 迭代与遍历** 42. Iterator 原理与 for-each 43. fail-fast 机制（modCount） 44. fail-safe 迭代器（并发集合，交界并发模块） 45. 遍历中删除元素的正确方式（iterator.remove） 46. Spliterator（了解，衔接 Stream）

**⑨ 视图与转换** 47. 集合视图：keySet / values / subList 是视图而非拷贝 48. 集合与数组互转：toArray 与 Arrays.asList

**⑩ 工具类与不可变集合** 49. Collections 工具类 50. Collections.synchronizedXxx（同步包装） 51. Arrays 工具类 52. 不可变集合的创建 53. 集合的序列化

**⑪ 排序与比较** 54. Comparable vs Comparator 55. 排序算法实现（TimSort / 双轴快排）与排序稳定性

**⑫ 性能与复杂度** 56. 各实现的时间复杂度对比