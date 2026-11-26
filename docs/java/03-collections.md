# 集合框架与 Stream API

> 本章解决一个问题：有一批数据时，怎样保存、查找、修改和统计它们。

**先学**：`List`、`Map`、`Set`，再学 Stream 的 `filter`、`map`、`toList`。能看懂普通循环后，再读集合统计。`Queue`、内部实现、复杂收集器和并行流可以第二遍再看，不需要一次记住所有 API。

**运行约定**：使用 JDK 21。本章代码块通常是独立片段，放在[总览中的 `Main.java`](./index.md) 的 `main` 方法里运行；同一小节注明“接上例”时才沿用变量。不要把所有片段拼进同一个方法，否则会遇到变量重名。本章需在文件顶部补充：

```java
import java.util.*;         // List、Map、Set、ArrayList 等集合工具
import java.util.stream.*;  // Collectors、LongStream 等流工具
```

`import` 只是告诉编译器类型来自哪里，不是在安装依赖；本章工具都由 JDK 提供。`record` 是第二章介绍的数据类，本章的局部 `record` 声明也可以放进 `main`。

## 目录

1. [集合体系概览](#集合体系概览)
2. [List 详解](#list-详解)
3. [Map 详解](#map-详解)
4. [Set 详解](#set-详解)
5. [Queue & Deque](#queue-deque)
6. [Stream API](#stream-api)
7. [Optional](#optional)
8. [不可变集合](#不可变集合)

---

## 集合体系概览

先按需求选容器，不必先背继承树：

| 我想做什么 | 选什么 | 例子 |
|-----------|--------|------|
| 按位置保存一串数据，允许重复 | `List` | 购物清单 `[苹果, 香蕉, 苹果]` |
| 用一个键查对应的值 | `Map` | 商品库存 `{苹果=3, 香蕉=2}` |
| 只保留不同的值 | `Set` | 标签 `[Java, SQL]` |
| 按排队顺序取数据 | `Queue` | 先来先处理的任务 |

`List<String> names = new ArrayList<>();` 可以拆成：

| 写法 | 含义 |
|------|------|
| `List` | 变量承诺支持“列表”的操作，这是接口 |
| `<String>` | 这个列表保存字符串，不能放入整数 |
| `names` | 变量名 |
| `new ArrayList<>()` | 创建真正保存数据的对象；右侧 `<>` 的类型从左侧推断 |

集合的泛型用引用类型：整数列表写 `List<Integer>`，不能写 `List<int>`。保存和读取时 Java 会自动在 `int` 与 `Integer` 之间转换。

下面的树作为以后查阅的地图：

```text
java.util.Collection
├── List（按位置保存、可重复）
│   ├── ArrayList       ← 动态数组，常用默认选择
│   └── LinkedList      ← 链表，同时实现 Deque
├── Set（不重复，顺序由实现决定）
│   ├── HashSet         ← 不承诺遍历顺序
│   ├── LinkedHashSet   ← 保持插入顺序
│   └── TreeSet         ← 按自然顺序或比较器排序
└── Queue（队列）
    ├── ArrayDeque      ← 双端队列，可作栈和普通队列
    ├── PriorityQueue   ← 取出时优先返回最小/最大项
    └── LinkedList

java.util.Map（不继承 Collection，key 唯一）
├── HashMap             ← 不承诺遍历顺序
├── LinkedHashMap       ← 默认保持插入顺序
├── TreeMap             ← 按 key 排序
└── ConcurrentHashMap   ← 提供并发操作，后续并发章节再学
```

对照前端经验：`ArrayList` 类似可增删的 JS 数组；`HashMap` 类似 JS `Map`，但不保证插入顺序；`HashSet` 类似 JS `Set`，也不保证插入顺序。它们只是用途相近，具体规则仍要看 Java API。

---

## List 详解

### ArrayList 常用操作

**任务：维护一份购物清单。** 先只看增加、修改、读取和删除：

```java
List<String> list = new ArrayList<>(); // 开始是 []
list.add("apple");                    // [apple]
list.add("banana");                   // [apple, banana]
list.add(0, "pear");                  // [pear, apple, banana]
list.set(1, "cherry");                // [pear, cherry, banana]

System.out.println(list.get(0));      // pear
System.out.println(list.size());      // 3
System.out.println(list.contains("banana")); // true

String removed = list.remove(0);      // 删除并返回 pear
System.out.println(removed);          // pear
System.out.println(list);             // [cherry, banana]

boolean deleted = list.remove("banana"); // 按值删除首次匹配项
System.out.println(deleted);          // true，表示确实删到了一项
System.out.println(list);             // [cherry]
```

下标从 `0` 开始，所以 `get(0)` 是第一个元素。`get` 读取、`set` 替换、`add` 增加；`get`/`set` 使用不存在的下标会抛出越界异常。

接上例，再补几个常用操作：

```java
list.addAll(List.of("blueberry", "apricot")); // [cherry, blueberry, apricot]
list.removeIf(s -> s.startsWith("b"));        // [cherry, apricot]

System.out.println(list.indexOf("apricot"));  // 1；找不到返回 -1
System.out.println(list.isEmpty());          // false

list.sort(Comparator.naturalOrder());        // [apricot, cherry]，字典序
list.sort(Comparator.reverseOrder());        // [cherry, apricot]，反向字典序
list.sort(Comparator.comparingInt(String::length)); // [cherry, apricot]，长度 6、7
```

`s -> s.startsWith("b")` 表示“对当前字符串 `s`，判断是否以 b 开头”。`removeIf` 删除判断结果为 `true` 的元素。`String::length` 是 `s -> s.length()` 的简写；最后一行的意思是“比较每个字符串的长度”。

子列表与工具方法可在用到时查：

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c", "d"));
List<String> sub = list.subList(1, 3); // [b, c]，包含下标 1，不包含下标 3
sub.set(0, "B");
System.out.println(list);             // [a, B, c, d]

List<String> independent = new ArrayList<>(sub); // 另建容器，不共享列表结构
Collections.reverse(independent);
System.out.println(independent);      // [c, B]
System.out.println(Collections.max(independent)); // c，按字典序比较
Collections.shuffle(independent);     // 随机打乱，结果顺序不固定
```

`subList` 是原列表的一段**视图**，不是独立副本。它的修改能反映到原列表；创建视图后，不要再直接对原列表增删元素并继续使用旧视图。

**常见坑：整数列表的两种 `remove`。** 以下是连续执行，第二次删除基于第一次的结果：

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3));
nums.remove(1);                  // 参数是 int：按下标删，结果 [1, 3]
nums.remove(Integer.valueOf(1)); // 参数是 Integer：按值删，结果 [3]
```

这叫方法重载：方法同名，但参数类型不同，选择的行为也不同。

### 初始化技巧

| 创建方式 | 能增删吗 | 能 `set` 替换吗 | 适合什么 |
|----------|----------|----------------|----------|
| `new ArrayList<>(List.of("a", "b"))` | 能 | 能 | 后面还要修改 |
| `List.of("a", "b")` | 不能 | 不能 | 固定内容，只读取 |
| `Arrays.asList("a", "b")` | 不能 | 能 | 固定长度的列表 |

它们都能用来遍历，区别在修改能力。`List.of` 还不允许 `null`。想练习增删，优先使用第一种。

### ArrayList vs LinkedList

下面的 `O(1)` 表示操作成本通常不随元素数量增加；`O(n)` 表示数据越多通常越费时，`n` 是元素个数。

| 操作 | ArrayList | LinkedList | 原因 |
|------|-----------|------------|------|
| 按下标读取 `get(i)` | O(1) | O(n) | 链表需要沿节点寻找 |
| 末尾追加 `add(e)` | 均摊 O(1) | O(1) | ArrayList 偶尔需要扩容 |
| 头部插入 `add(0,e)` | O(n) | O(1) | 数组需要移动后面的元素 |
| 按下标在中间插入 | O(n) | O(n) | 链表也要先找到位置 |

日常列表先选 `ArrayList`。如果需求是两端入队、出队，先看后面的 `ArrayDeque`，不要仅凭“链表插入快”就替换实现。

---

## Map 详解

### HashMap 常用操作

**任务：按商品名查询库存。** `Map<String, Integer>` 的两个类型分别是“键的类型”和“值的类型”。

```java
Map<String, Integer> stock = new HashMap<>();
stock.put("apple", 3);            // apple 对应 3
stock.put("banana", 2);
stock.put("apple", 5);            // 同一个 key 只能有一份，覆盖成 5
stock.putIfAbsent("apple", 9);    // 已有非 null 值，不覆盖，仍是 5

System.out.println(stock.get("apple"));             // 5
System.out.println(stock.get("pear"));              // null，没找到
System.out.println(stock.getOrDefault("pear", 0));  // 0，没找到时用默认值
System.out.println(stock.containsKey("banana"));    // true

stock.remove("banana", 2); // 仅在值仍为 2 时删除
System.out.println(stock.size());  // 1
```

`get` 返回 `Integer`，缺失时返回 `null`。不要直接把可能缺失的结果赋给 `int`：自动拆箱 `null` 会触发 `NullPointerException`。`getOrDefault` 只处理“键不存在”；如果你主动存了 `null`，仍可能取到 `null`。本章库存示例只保存非空数字。

**遍历时，先理解“一条键值对”：** 接上例，下面会输出 `apple -> 5`。

```java
for (Map.Entry<String, Integer> entry : stock.entrySet()) {
    String product = entry.getKey(); // 这条记录的键
    Integer amount = entry.getValue(); // 这条记录的值
    System.out.println(product + " -> " + amount);
}

// 同样的工作，改用 Lambda 写法
stock.forEach((product, amount) -> System.out.println(product + " -> " + amount));
```

`entrySet()` 给出所有键值对；`Map.Entry<String, Integer>` 就是一条键值对的类型。`for (类型 变量 : 集合)` 表示逐个取出元素。第二种写法中 `(product, amount)` 是每次收到的两个参数。

**进阶：按现有值计算新值。** 先观察每一步的结果，不必马上记住方法名：

```java
Map<String, Integer> stock = new HashMap<>();
stock.put("apple", 3);

stock.computeIfPresent("apple", (key, oldValue) -> oldValue + 1); // apple=4
stock.computeIfAbsent("pear", key -> 0); // pear 原来没有值，计算并存入 0
stock.merge("apple", 2, (oldValue, added) -> oldValue + added); // apple=6
stock.replaceAll((key, value) -> value * 2); // apple=12，pear=0

Map<String, Integer> extra = Map.of("banana", 2);
stock.putAll(extra); // 加入 banana=2；同名键会被覆盖
```

`computeIfAbsent` 在键缺失或值为 `null` 时调用计算函数；计算结果为 `null` 就不写入。`computeIfPresent` 只处理已有非空值。`merge` 在没有非空旧值时直接放入第二个参数，否则调用合并函数。后两者的计算函数如果返回 `null`，会删除该键；本例的函数都返回数字。

### 计数模式（高频用法）

**任务：统计每个单词出现几次。** 先用最直观的循环：

```java
List<String> words = List.of("apple", "banana", "apple");
Map<String, Integer> freq = new HashMap<>();

for (String word : words) {
    int oldCount = freq.getOrDefault(word, 0); // 第一次遇到时按 0 次算
    freq.put(word, oldCount + 1);             // 把新次数写回
}
System.out.println(freq.get("apple"));  // 2
System.out.println(freq.get("banana")); // 1
```

每一轮的变化是：`{apple=1}` → `{apple=1, banana=1}` → `{apple=2, banana=1}`。这是内容示意，`HashMap` 的打印顺序不固定。

同一任务也能用 `merge`，**重新创建一张空表**后执行：

```java
List<String> words = List.of("apple", "banana", "apple");
Map<String, Integer> freq = new HashMap<>();
for (String word : words) {
    freq.merge(word, 1, Integer::sum);
}
```

`Integer::sum` 等价于 `(oldCount, added) -> oldCount + added`。第一次放 `1`，再次遇到就把旧次数加 `1`。不要在已经统计完成的 `freq` 上再执行一次，否则次数会重复累加。

学完后面的 `groupingBy` 再回来看第三种写法：

```java
List<String> words = List.of("apple", "banana", "apple");
Map<String, Long> freq = words.stream()
    .collect(Collectors.groupingBy(word -> word, Collectors.counting()));
// 按单词本身分组，再数每组数量：apple=2L，banana=1L
```

这里统计结果是 `Long`，所以声明为 `Map<String, Long>`，不能换成 `Map<String, Integer>`。

### HashMap 原理简介

这一节可以后看。先记住：`HashMap` 通过键的哈希值寻找存储位置，再用 `equals()` 区分键，通常查找很快。

- 不同键可能落入同一个桶，这叫哈希冲突，不要求它们的 `hashCode()` 完全相同。
- 在常见 OpenJDK 实现中，冲突较多且容量足够时，桶内结构可能由链表变为红黑树；不能只记“超过 8 个必定变树”。
- 自定义对象作键时，`equals()` 与 `hashCode()` 必须遵守一致性约定，且放入后不要修改参与这两个方法计算的字段。
- 多线程共同修改不是本章默认场景；需要并发时再学习 `ConcurrentHashMap` 以及复合操作的原子性，不能简单认为换一个类就解决所有同步问题。

### LinkedHashMap vs TreeMap

**任务：让遍历结果有确定顺序。** 相同数据放进不同实现，按键遍历时会不同：

```java
Map<String, Integer> linked = new LinkedHashMap<>();
linked.put("banana", 2);
linked.put("apple", 1);
linked.put("cherry", 3);
System.out.println(linked.keySet()); // [banana, apple, cherry]，插入顺序

Map<String, Integer> tree = new TreeMap<>(linked);
System.out.println(tree.keySet());   // [apple, banana, cherry]，按键排序
```

`keySet()` 只取键；`new TreeMap<>(linked)` 把已有键值对复制进另一个容器。

`TreeMap` 还能按键取范围：

```java
TreeMap<Integer, String> scores = new TreeMap<>();
scores.put(59, "Alice");
scores.put(75, "Bob");
scores.put(90, "Carol");

System.out.println(scores.subMap(60, 90)); // {75=Bob}，包含 60、不包含 90
System.out.println(scores.headMap(60));   // {59=Alice}，小于 60
System.out.println(scores.tailMap(90));   // {90=Carol}，大于等于 90
```

这些范围结果也是视图。这里用分数作键只是演示范围查询；真实排名中可能有同分，不能用一个 `Map<Integer, String>` 保存所有人。

---

## Set 详解

**任务：给标签去重。** `Set` 不按下标读取，重点是“是否存在”：

```java
Set<String> tags = new HashSet<>();
System.out.println(tags.add("Java")); // true，首次加入
System.out.println(tags.add("Java")); // false，重复，不再加入
System.out.println(tags.size());      // 1
System.out.println(tags.contains("Java")); // true
```

如果还关心顺序，选择相应的实现：

```java
List<String> withDups = List.of("b", "a", "b", "c", "a");
Set<String> unique = new HashSet<>(withDups);       // 只有 a、b、c，顺序不承诺
Set<String> ordered = new LinkedHashSet<>(withDups);
Set<String> sorted = new TreeSet<>(withDups);

System.out.println(ordered); // [b, a, c]，保留第一次出现的顺序
System.out.println(sorted);  // [a, b, c]，按字典序排序
List<String> deduped = new ArrayList<>(ordered); // 需要下标时再转回列表
```

`new LinkedHashSet<>(withDups)` 会逐个放入原列表元素，重复项被忽略。自定义对象去重也需要定义正确的相等规则；`HashSet` 使用 `equals`/`hashCode`，`TreeSet` 根据比较结果是否为 `0` 判断重复。

集合运算可以想成两张名单的合并、共同项和差异：

```java
Set<String> a = new HashSet<>(List.of("x", "y", "z"));
Set<String> b = new HashSet<>(List.of("y", "z", "w"));

Set<String> union = new HashSet<>(a);
union.addAll(b); // 并集：包含 x、y、z、w

Set<String> intersection = new HashSet<>(a);
intersection.retainAll(b); // 交集：只留下 y、z

Set<String> diff = new HashSet<>(a);
diff.removeAll(b); // 差集：a 中有、b 中没有，剩下 x
```

这三个方法都会修改调用它的集合，所以每次先复制 `a`，避免第一步就破坏后面还要用的原数据。

---

## Queue & Deque

这一节可第二遍再读。**队列像排队，先进去的先出来；栈像叠盘子，最后放上去的先拿走。**

```java
Queue<String> queue = new ArrayDeque<>();
queue.offer("first");
queue.offer("second");

System.out.println(queue.peek()); // first，只看不拿，仍有两项
System.out.println(queue.poll()); // first，取走队头，只剩 second
System.out.println(queue.poll()); // second，取走后为空
System.out.println(queue.poll()); // null，已经没有元素
System.out.println(queue.peek()); // null
```

`offer` 尝试入队，`peek` 查看，`poll` 取走。某些有容量限制的队列会在装满时让 `offer` 返回 `false`；`ArrayDeque` 会自动扩容，没有这种固定容量限制，并且不允许放入 `null`。

```java
Deque<String> stack = new ArrayDeque<>();
stack.push("a");
stack.push("b");
System.out.println(stack.peek()); // b，只看栈顶
System.out.println(stack.pop());  // b，拿走栈顶
System.out.println(stack.pop());  // a
// stack.pop();                  // 再取会抛 NoSuchElementException
System.out.println(stack.poll()); // null；想在空时返回 null，可使用 poll
```

`Deque` 读作“双端队列”，两端都能操作；`push`/`pop` 使用同一端来实现栈。新代码通常用 `ArrayDeque` 替代旧的 `Stack` 类。

**优先级队列**按优先级取出，而非按加入时间：

```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(3);
minHeap.offer(1);
minHeap.offer(2);
System.out.println(minHeap.poll()); // 1
System.out.println(minHeap.poll()); // 2
System.out.println(minHeap.poll()); // 3

PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.addAll(List.of(3, 1, 2));
System.out.println(maxHeap.poll()); // 3，反过来优先取最大值
```

`PriorityQueue` 只保证每次取出的头部符合优先级；直接遍历或打印整个队列，不保证得到完整排序。

自定义任务也能排序。下面用小型数据类把所需字段完整定义出来：

```java
record Task(String name, int priority) {}
PriorityQueue<Task> tasks = new PriorityQueue<>(
    Comparator.comparingInt(Task::priority).reversed()
);
tasks.offer(new Task("普通任务", 1));
tasks.offer(new Task("紧急任务", 9));
System.out.println(tasks.poll().name()); // 紧急任务
```

`Task::priority` 等价于 `task -> task.priority()`；`comparingInt` 按这个整数比较，`reversed()` 反转顺序，因此数字较大的先取出。实际项目要先约定数字大还是数字小表示更紧急。

---

## Stream API

Stream 用来描述“这批数据要经过哪些处理”。它不是保存数据的容器；`List` 保存原始数据，Stream 描述处理过程。

### JS vs Java Stream 完整对比

| JS 数组方法 | Java Stream | 类型 | 含义 |
|------------|-------------|------|------|
| `.filter(fn)` | `.filter(fn)` | 中间操作 | 只保留符合条件的 |
| `.map(fn)` | `.map(fn)` | 中间操作 | 把每一项转换成另一项 |
| `.flatMap(fn)` | `.flatMap(fn)` | 中间操作 | 把每项展开，再接成一条流 |
| `.reduce(fn, init)` | `.reduce(init, fn)` | 终止操作 | 累积成一个结果 |
| `.forEach(fn)` | `.forEach(fn)` | 终止操作 | 对元素执行操作，结束这条流 |
| `.find(fn)` | `.filter(fn).findFirst()` | 终止操作 | 找到第一项，结果可能为空 |
| `.some(fn)` / `.every(fn)` | `.anyMatch(fn)` / `.allMatch(fn)` | 终止操作 | 判断是否有任意项/所有项满足条件 |
| `.slice(s, e)` | `.skip(s).limit(e-s)` | 中间操作 | 跳过前 s 项，最多取 e-s 项 |
| `[...new Set(arr)]` | `.distinct()` | 中间操作 | 去重 |
| `.sort(fn)` | `.sorted(comparator)` | 中间操作 | 对流排序，不改原列表 |
| `.length` | `.count()` | 终止操作 | 统计流中元素数量 |

### 中间操作和终止操作（重点）

把 Stream 想成一条“数据处理流水线”：

- **中间操作**：登记处理规则，并返回另一条 Stream，所以还能继续 `.filter()`、`.map()`。例如 `filter`、`map`、`sorted`、`distinct`。
- **终止操作**：按前面登记的规则处理数据，拿到最终结果，并结束这条 Stream。例如 `toList()`、`count()`、`collect()`、`forEach()`、`reduce()`。

“终止”不是 Java 程序结束，而是**这条 Stream 不能再继续使用**。终止操作通常返回列表、数字、`Optional` 或 `void`，不再返回 Stream。

```java
List<Integer> numbers = List.of(1, 2, 3, 4);

Stream<Integer> pipeline = numbers.stream()
    .filter(n -> n % 2 == 0); // 中间操作：只登记“保留偶数”规则，还没开始遍历

List<Integer> evens = pipeline.toList(); // 终止操作：现在才执行，得到 [2, 4]
System.out.println(evens);
// pipeline.count(); // 错误：同一条 Stream 已经终止，不能再次使用
```

如果只写中间操作，没有终止操作，规则不会真正执行：

```java
numbers.stream()
    .filter(n -> {
        System.out.println("检查 " + n);
        return n % 2 == 0;
    }); // 没有 count、toList 等终止操作，不会打印“检查”
```

需要再次处理原数据时，重新调用 `numbers.stream()` 创建一条新 Stream。

### 基础用法

**任务：保留偶数，把它们平方，再留下大于 20 的结果。** 先写普通循环：

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6);
List<Integer> result = new ArrayList<>();
for (int n : nums) {
    if (n % 2 == 0) {
        int square = n * n;
        if (square > 20) {
            result.add(square);
        }
    }
}
System.out.println(result); // [36]
```

把每一步翻译成 Stream，得到同一个结果：

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6);
List<Integer> result = nums.stream() // 从列表建立处理流程
    .filter(n -> n % 2 == 0)         // 保留 2、4、6
    .map(n -> n * n)                 // 转换成 4、16、36
    .filter(n -> n > 20)             // 只留下 36
    .toList();                       // 终止操作：执行前面的规则，得到结果列表
System.out.println(result);         // [36]
System.out.println(nums);           // [1, 2, 3, 4, 5, 6]，原列表未改
```

`n -> n * n` 是一个函数：输入当前数字 `n`，返回它的平方。每个 `n` 都是该步骤当前收到的元素，并不是一个共享计数器。注释中的中间数字是帮助理解的逻辑结果，Stream 不会为每一行先建一个中间列表。

记住三个区别：

1. **惰性处理**：只有 `filter`/`map` 时只是描述流程，遇到终止操作才处理；不应依赖回调里的打印或副作用，因为某些终止操作可以优化掉不需要的步骤。
2. **一条流只消费一次**：执行完 `toList()` 后不能再对同一个 Stream 调用 `count()`；可以重新从原来的 `nums.stream()` 建一条新流。
3. **结果的修改能力不同**：`toList()` 返回不可修改的列表。要继续增删，明确写 `.collect(Collectors.toCollection(ArrayList::new))`；`Collectors.toList()` 的规范不承诺具体实现或可修改性。

### 聚合操作

**任务：把多个数汇总成一个值。** `mapToInt` 把包装类流转换为专门处理 `int` 的流，之后可以调用 `sum`、`average`、`max`：

```java
List<Integer> nums = List.of(1, 2, 3, 4);
int sum = nums.stream().mapToInt(n -> n).sum(); // 10
OptionalDouble avg = nums.stream().mapToInt(n -> n).average(); // 内含 2.5
OptionalInt max = nums.stream().mapToInt(n -> n).max();        // 内含 4

System.out.println(sum);             // 10
System.out.println(avg.orElse(0.0)); // 2.5
System.out.println(max.orElse(0));   // 4
```

`n -> n` 返回数字本身，这里同时发生 `Integer` 到 `int` 的拆箱，也可写 `Integer::intValue`。空列表的和可以定义为 `0`，但没有最大值、平均值，因此后两者返回“可能为空”的容器；`orElse` 指定空时的替代值，后面的 Optional 小节会继续讲。

自定义累积可以用 `reduce`，理解时先想成循环：

```java
List<Integer> nums = List.of(1, 2, 3, 4);
int product = 1;
for (int n : nums) {
    product = product * n;
}
System.out.println(product); // 24

int sameProduct = nums.stream().reduce(1, (acc, n) -> acc * n);
System.out.println(sameProduct); // 24
```

`1` 是初始值；`acc` 是累积结果，依次变成 `1 → 2 → 6 → 24`。空流返回初始值。这里没有调用并行流，先理解顺序累积即可。

接上例，也可以只问数量或真假：

```java
long count = nums.stream().filter(n -> n > 2).count(); // 2
boolean anyLarge = nums.stream().anyMatch(n -> n > 3); // true，存在 4
boolean allLarge = nums.stream().allMatch(n -> n > 3); // false，1 就不满足
```

`anyMatch`/`allMatch` 得到确定答案后可以提前结束，不需要总把所有元素都处理完。

### 字符串操作

**任务：过滤单词、变成大写，然后拼接。**

```java
List<String> words = List.of("hello", "world", "java", "stream");
List<String> processed = words.stream()
    .filter(word -> word.length() > 4) // hello、world、stream
    .map(String::toUpperCase)          // HELLO、WORLD、STREAM
    .sorted()                         // HELLO、STREAM、WORLD
    .toList();
System.out.println(processed);        // [HELLO, STREAM, WORLD]

String joined = words.stream().collect(Collectors.joining(", "));
System.out.println(joined); // hello, world, java, stream

String bracketed = words.stream().collect(Collectors.joining(", ", "[", "]"));
System.out.println(bracketed); // [hello, world, java, stream]
```

`String::toUpperCase` 等价于 `word -> word.toUpperCase()`；`joining` 的三个参数依次是分隔符、整体前缀、整体后缀。`collect` 按收集器提供的规则汇总元素，这里汇总为一个字符串。

### flatMap（展平嵌套集合）

**任务：把几组数字合成一份列表。** 输入是“两层列表”，输出是“一层列表”：

```java
List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3, 4));
List<Integer> flat = new ArrayList<>();
for (List<Integer> group : nested) { // 第一轮 [1, 2]，第二轮 [3, 4]
    flat.addAll(group);
}
System.out.println(flat); // [1, 2, 3, 4]
```

同一任务的 Stream 写法，独立运行：

```java
List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3, 4));
List<Integer> flat = nested.stream()
    .flatMap(group -> group.stream()) // 每个小列表变成流，再接成一条流
    .toList();
System.out.println(flat); // [1, 2, 3, 4]
```

`group -> group.stream()` 可以简写为 `List::stream`。`map` 是“一项变一项”，这里若用 `map(group -> group.stream())` 会得到“装着小流的流”；`flatMap` 会继续把这些小流展开。

应用到订单也一样，下面把所需数据完整定义出来，不依赖未知的 `orders` 或 `Item`：

```java
record Order(List<String> products) {}
List<Order> orders = List.of(
    new Order(List.of("茶", "面包")),
    new Order(List.of("茶", "牛奶"))
);
List<String> allProducts = orders.stream()
    .flatMap(order -> order.products().stream()) // 茶、面包、茶、牛奶
    .distinct()                                // 去重：茶、面包、牛奶
    .toList();
System.out.println(allProducts); // [茶, 面包, 牛奶]
```

### Collectors 收集器（重点！）

这一节先掌握“分组”和“每组计数”，其余操作留作查阅。**任务：把员工名单按部门整理。** 为了方便手算，先只用三人：

```java
record Employee(String name, String dept, double salary) {}
List<Employee> employees = List.of(
    new Employee("Alice", "Engineering", 10000),
    new Employee("Bob", "Engineering", 12000),
    new Employee("Carol", "Marketing", 8000)
);
```

`record` 自动提供 `name()`、`dept()`、`salary()` 读取方法。工资使用 `double` 仅为演示统计 API；真实金额计算应按业务精度规则选择 `BigDecimal` 或最小货币单位整数。

接上例，先用循环分组：

```java
Map<String, List<Employee>> byDept = new HashMap<>();
for (Employee employee : employees) {
    String dept = employee.dept();
    if (!byDept.containsKey(dept)) {
        byDept.put(dept, new ArrayList<>()); // 第一次遇到部门，先给它一份空名单
    }
    byDept.get(dept).add(employee);         // 把当前员工加入该部门名单
}
System.out.println(byDept.get("Engineering").size()); // 2
```

`Map<String, List<Employee>>` 从外向内读：“按字符串部门名查到一份员工列表”。这就解释了为什么类型里会出现两层尖括号。

接上面的员工数据，换一个变量名写 Stream 版本：

```java
Map<String, List<Employee>> grouped = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept));
// Engineering 对应 [Alice, Bob] 两位员工；Marketing 对应 [Carol]

Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
System.out.println(countByDept.get("Engineering")); // 2
System.out.println(countByDept.get("Marketing"));   // 1
```

`groupingBy` 的第一个参数告诉它“按什么分组”，`Employee::dept` 就是 `employee -> employee.dept()`。不传第二个参数时收集各组员工；传入 `counting()`，则把每组名单换成每组人数。

以下都**接同一份员工数据**，可以按需求选读：

```java
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept,
        Collectors.averagingDouble(Employee::salary)));
// Engineering=11000.0，Marketing=8000.0

Map<String, Optional<Employee>> topByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept,
        Collectors.maxBy(Comparator.comparingDouble(Employee::salary))));
// Engineering 的 Optional 内含 Bob；Marketing 的内含 Carol

Map<String, Double> salaryByName = employees.stream()
    .collect(Collectors.toMap(Employee::name, Employee::salary));
// Alice=10000.0，Bob=12000.0，Carol=8000.0

Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(employee -> employee.salary() > 10000));
// true 组是 Bob；false 组是 Alice、Carol

DoubleSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingDouble(Employee::salary));
System.out.println(stats.getMax());     // 12000.0
System.out.println(stats.getAverage()); // 10000.0
System.out.println(stats.getSum());     // 30000.0
```

读嵌套调用时先读外层：`groupingBy(..., averagingDouble(...))` 是“先按部门分组，再对组内工资求平均”。`maxBy` 要一个比较规则，`comparingDouble(Employee::salary)` 就是按工资比较。

`toMap` 的两个函数分别产生键和值。上例假设名字唯一；同名员工会导致重复键异常。真实业务应选唯一 ID 作键，或明确提供重复键合并规则。`partitioningBy` 固定分为 `true`/`false` 两组，而 `groupingBy` 可以有任意多个部门。

### 并行流

下面这段代码的作用是：**从 1 到 1 亿中，数出有多少个偶数**，结果为 **5000 万**。

```java
long count = LongStream.rangeClosed(1, 100_000_000L) // 生成 1～1 亿的整数序列，包含两端
    .parallel()                                    // 允许多个线程分工处理
    .filter(n -> n % 2 == 0)                        // 只保留偶数
    .count();                                      // 统计留下了多少个数：50000000
```

逐步理解：

- `LongStream`：用于处理 `long` 整数的流；这里的数字按需处理，不会先创建一个存满 1 亿个数字的列表。
- `100_000_000L`：下划线只是方便阅读，数值就是 `100000000`；末尾的 `L` 表示 `long` 类型。
- `n -> n % 2 == 0`：对每个数 `n`，判断它除以 2 的余数是否为 0，是就保留。例如 `1～10` 经过筛选后剩下 `2、4、6、8、10`，计数为 `5`。
- `.parallel()`：类似几个人分工数数，最后合计结果。去掉这一行就变成顺序流，统计结果相同。
- `.count()`：触发整个流的处理并返回数量，结果合并由流框架负责，不需要自己在多个线程之间累加计数。

**为什么这个筛选适合并行？** 判断只依赖当前的 `n`，不修改共享变量，这就是这里所说的“无状态”。应避免在并行操作中随意修改外部计数器或共享列表，否则可能发生线程安全问题。`count()`、`sum()` 等内置统计操作本身可以安全用于并行流；普通收集器也不必都是线程安全的，规范的 `collect()` 会协调各部分结果的合并。

⚠️ **注意**：并行不保证更快，拆分任务和合并结果也有开销，小数据量或计算简单时可能更慢；是否使用应以实际测量为准。并行也不等于丢失顺序：`forEach()` 不保证执行顺序，需要按流的相遇顺序执行时可用 `forEachOrdered()`，但可能限制并行收益。

这个例子用于理解流的执行过程；如果只是求 `1～1 亿` 的偶数个数，直接计算 `100_000_000L / 2` 即可，无需遍历。

---

## Optional

**任务：表达“查找结果可能没有值”。** `Optional<String>` 是一个可能装着字符串、也可能为空的容器，主要用于方法返回值；不是要求把所有变量和字段都换成 Optional。

先看普通空值判断：

```java
String name = null;
String displayName;
if (name == null) {
    displayName = "UNKNOWN";
} else {
    displayName = name.toUpperCase();
}
System.out.println(displayName); // UNKNOWN
```

同一逻辑的 Optional 写法：

```java
String name = null; // 改成 "Alice"，观察结果变成 ALICE
String displayName = Optional.ofNullable(name)
    .map(String::toUpperCase)
    .orElse("UNKNOWN");
System.out.println(displayName); // UNKNOWN
```

`ofNullable` 把可能为空的值放进容器；有值才执行 `map`，没有值就一路保持空，最后 `orElse` 给默认值。用途近似 `name?.toUpperCase() ?? 'UNKNOWN'`，但 Java 的返回类型和调用方式不同。

### 读懂一条完整的 Optional 链

下面把“根据 ID 查用户，再取姓名并转大写”写成一个完整例子，保存为 `Main.java`，用 JDK 21 执行 `java Main.java`：

```java
import java.util.Optional;

public class Main {
    static User getUserById(long id) {
        return id == 1 ? new User("Alice") : null;
    }

    public static void main(String[] args) {
        String result = Optional.ofNullable(getUserById(1))
            .map(User::getName)
            .map(String::toUpperCase)
            .orElse("UNKNOWN");

        System.out.println(result); // ALICE

        String missing = Optional.ofNullable(getUserById(2))
            .map(User::getName)
            .map(String::toUpperCase)
            .orElse("UNKNOWN");

        System.out.println(missing); // UNKNOWN
    }
}

record User(String name) {
    // record 默认生成 name()；这里额外提供 getName()，模拟普通 JavaBean
    public String getName() {
        return name;
    }
}
```

按类型变化读这条链：

| 代码 | 此时的类型 | 作用 |
|------|------------|------|
| `Optional.ofNullable(getUserById(1))` | `Optional<User>` | 用户存在就装入；返回 `null` 就得到空容器 |
| `.map(User::getName)` | `Optional<String>` | 有用户时调用 `user.getName()`，取出姓名 |
| `.map(String::toUpperCase)` | `Optional<String>` | 有姓名时调用 `name.toUpperCase()` |
| `.orElse("UNKNOWN")` | `String` | 有值就取值，空了就返回默认文本 |

这里的两个方法引用可以先改写成 Lambda：

```java
.map(User::getName)       // 等价于 .map(user -> user.getName())
.map(String::toUpperCase) // 等价于 .map(name -> name.toUpperCase())
```

如果 `getUserById(2)` 返回 `null`，第一个 `map` 和第二个 `map` 都会跳过，不会调用空对象的方法，最后得到 `"UNKNOWN"`。如果 `getName()` 自己返回 `null`，`Optional.map` 也会把结果视为空。

不用 Optional 时，逻辑大致是：

```java
long id = 2;
User user = getUserById(id);
String result;
if (user == null || user.getName() == null) {
    result = "UNKNOWN";
} else {
    result = user.getName().toUpperCase();
}
```

这个例子展示的是“可能没有结果时如何继续转换”，不是要求所有字段和局部变量都使用 Optional。

| 创建方式 | 结果 |
|----------|------|
| `Optional.of("hello")` | 有值的容器；传 `null` 会抛异常 |
| `Optional.empty()` | 空容器 |
| `Optional.ofNullable(name)` | `name` 为 `null` 时为空，否则装入该值 |

下面是独立示例，展示有值、过滤后为空，以及明确要求必须有值：

```java
Optional<String> name = Optional.of("Alice");
name.ifPresent(value -> System.out.println("Hello " + value)); // Hello Alice

String longName = name.filter(value -> value.length() > 10).orElse("太短了");
System.out.println(longName); // 太短了

String required = name.orElseThrow(() -> new IllegalArgumentException("姓名不能为空"));
System.out.println(required); // Alice；如果 name 为空才抛出异常
```

`ifPresent` 用于“有值时做一件事”，`filter` 用于“有值且满足条件才保留”，`orElseThrow` 用于“没有值就失败”。不要直接对空容器调用 `get()`，那会抛异常；先 `isPresent()` 再 `get()` 并非语法错误，但简单场景通常用这些方法更直接。

**`orElse` 和 `orElseGet` 的区别在参数何时计算：**

```java
Optional<String> name = Optional.of("Alice");
String first = name.orElse(new String("default"));
String second = name.orElseGet(() -> new String("default"));
// first、second 都是 Alice
// 第一行仍先创建了 default 字符串；第二行的函数没有执行
```

这里特意用 `new String` 演示求值时机，不是推荐这样创建字符串。默认值是固定文本时 `orElse("default")` 就足够；默认值需要查询或昂贵计算时，考虑把计算放入 `orElseGet` 的函数。

---

## 不可变集合

**先区分“容器不能改”和“里面的对象不能改”。** `List.of` 等方法创建的集合不能增删或替换元素，但并不保证元素对象的内部字段也不可修改。

```java
List<String> list = List.of("a", "b", "a"); // 保留顺序，允许重复
Set<String> set = Set.of("x", "y", "z");   // 不承诺顺序，重复参数会抛异常
Map<String, Integer> map = Map.of("one", 1, "two", 2);
// list.add("c"); // UnsupportedOperationException
// map.put("three", 3); // UnsupportedOperationException

Map<String, Integer> entries = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2)
);
```

`List.of`、`Set.of`、`Map.of` 都不接收 `null`；`Map.of` 也不允许重复键。`Map.of` 最多接收 10 对键值，更多时用 `Map.ofEntries`，每个 `Map.entry` 表示一对键值；上面用两对演示语法。

**只读视图与独立快照也不同：**

```java
List<String> mutable = new ArrayList<>(List.of("a"));
List<String> readOnly = Collections.unmodifiableList(mutable);
List<String> copy = List.copyOf(mutable);

mutable.add("b");
System.out.println(readOnly); // [a, b]，视图跟着原列表变化
System.out.println(copy);     // [a]，快照内容不跟随原列表的增删
// readOnly.add("c");         // 不能通过只读视图修改
```

`copyOf` 复制的是元素引用，不会深拷贝元素对象。`final List<String>` 只限制变量不能改指向另一个列表，也不等于限制列表内容。方法要返回只读快照时可考虑 `List.copyOf`，要返回可继续修改的列表则明确创建 `ArrayList`。

---

## 动手检查：能解释这些结果就可以继续

先自己回答，再展开答案。无需背住所有收集器名称。

<details>
<summary>1. List.of(1, 2, 3) 能直接 remove(0) 吗？想删除该怎么创建？</summary>

不能，它不可修改。使用 `new ArrayList<>(List.of(1, 2, 3))` 创建可修改列表，再 `remove(0)` 删除下标 0 的元素，剩下 `[2, 3]`。

</details>

<details>
<summary>2. 对 [1, 2, 3, 4] 先 filter(n -> n % 2 == 0)，再 map(n -> n * 10)，结果是什么？原列表会变吗？</summary>

调用 `toList()` 后结果为 `[20, 40]`，原列表不变。`filter` 保留 `2、4`，`map` 分别乘以 10。

</details>

<details>
<summary>3. 用一张 Map 统计 a、b、a，merge(word, 1, Integer::sum) 每轮后是什么？</summary>

内容依次为 `{a=1}`、`{a=1, b=1}`、`{a=2, b=1}`。`HashMap` 不承诺这些键的打印顺序。

</details>

<details>
<summary>4. Optional.ofNullable(null).orElse("空") 会返回什么？</summary>

返回 `"空"`。`ofNullable` 得到空容器，`orElse` 提供默认值；如果用 `Optional.of(null)`，则会在创建时抛异常。

</details>

---

*← [上一章：面向对象](./02-oop.md) | [建议下一步：Spring Boot](./05-spring-boot.md) | [选读：并发编程](./04-concurrency.md) →*
