# Java 全栈学习指南

> 面向有 JavaScript / TypeScript 经验、刚开始学 Java 的读者。先看懂代码，再逐步写出后端接口。

## 从哪里开始？

建议按 **基础语法 → 面向对象 → 集合 → Spring Boot 入门 → 小项目 → 并发** 的顺序学习。文件编号用于查找；第一次学习时，可以把第 04 章并发放到后面。

| 章节 | 第一次先学什么 | 学完后能做什么 | 可以以后再看 |
|------|---------------|---------------|-------------|
| [01 - 基础语法](./01-basics.md) | 程序入口、类型、方法、条件、循环、数组、字符串 | 写出“筛选偶数并计数”的普通循环 | 位运算、带标签跳转、复杂方法引用 |
| [02 - 面向对象](./02-oop.md) | 类、对象、构造器、`this`、接口、泛型 | 看懂 `new User(...)` 和方法参数、返回值 | Lombok、通配符、设计模式 |
| [03 - 集合与 Stream](./03-collections.md) | `List` / `Map` / `Set`、遍历、简单筛选 | 看懂一组数据如何一步步变成结果 | 复杂收集器、并行流、底层实现 |
| [05 - Spring Boot](./05-spring-boot.md) | 启动项目、一个 GET 接口、分层调用、参数校验 | 从浏览器请求追到 Java 方法和响应 | JWT、Redis、JPA 性能、部署 |
| [06 - 实战与进阶](./06-practice.md) | 分阶段完成最小 CRUD，再接数据库 | 自己增加、查询、修改、删除一条数据 | 完整博客模型、鉴权、面试题、微服务 |
| [04 - 并发编程](./04-concurrency.md) | 线程、共享数据、`join()`、线程池 | 解释“谁在执行、何时完成、谁在等谁” | CAS、复杂 Future 组合、虚拟线程细节 |

第一次学习的达标标准是：**能解释一个小例子，改一处输入，并预测它的结果**。时间可以按自己的理解速度调整。

## 先跑通一个完整程序

本系列统一以 **JDK 21、非预览语法**为基础；Spring 章节使用 Spring Boot 3.x。先确认终端能找到 JDK：

```bash
java --version
javac --version
```

两条命令都应显示 21.x。`java` 用于运行程序，`javac` 用于编译代码；若提示找不到 Java Runtime，先按 [JDK 21 安装指南](https://docs.oracle.com/en/java/javase/21/install/)安装并配置 JDK。也可以在 IntelliJ IDEA 中为项目选择或下载 JDK 21；IDE 的项目设置与终端环境要分别核对。

在一个练习目录中创建文件 **`Main.java`**，内容如下：

```java
public class Main {
    public static void main(String[] args) {
        int total = 0;

        for (int n = 1; n <= 10; n++) {
            if (n % 2 == 0) {
                total++;
            }
        }

        System.out.println(total);
    }
}
```

在这个文件所在的目录运行：

```bash
java Main.java
```

预期输出：

```text
5
```

这是 JDK 支持的单文件源码运行方式。也可以分两步执行 `javac Main.java` 和 `java Main`；前一步生成编译结果，后一步运行。以后如果例子拆成 `User.java` 和 `Main.java`，在它们所在目录执行 `javac User.java Main.java`，再执行 `java Main`，先把涉及的源文件一起编译。

### 这段代码怎么读？

先找程序要做的事：**数出 1～10 中有几个偶数**。再从 `main` 方法内部往下看：

| 代码 | 用普通话解释 | 本例中的变化 |
|------|-------------|-------------|
| `int total = 0;` | 准备一个整数变量记录数量 | 一开始是 `0` |
| `int n = 1` | 从数字 1 开始检查 | 当前检查 `1` |
| `n <= 10` | 只要当前数字不超过 10，就继续循环 | 检查 `1～10`，包括 10 |
| `n++` | 每轮结束后，把当前数字加 1 | `1 → 2 → 3 → … → 10` |
| `n % 2 == 0` | 判断除以 2 的余数是否为 0 | `2、4、6、8、10` 符合 |
| `total++` | 找到一个偶数，把数量加 1 | `0 → 1 → 2 → 3 → 4 → 5` |
| `System.out.println(total);` | 把最终数量打印到终端 | 输出 `5` |

最外层的 `public class Main` 定义名为 `Main` 的类，文件名要与它一致。`public static void main(String[] args)` 是这类程序的启动入口：`public` 表示可以公开访问，`static` 允许不创建对象就调用，`void` 表示不返回值，`args` 接收命令行传入的字符串数组。本例没有使用 `args`。

**现在试着改一处：**把 `n <= 10` 改成 `n <= 6`，再运行一次，预期输出 `3`。如果你能解释为什么，接着读第 01 章即可。

## 文档里的代码应该放在哪里？

文档会同时用到完整程序、局部代码和框架片段。运行前先看它属于哪一种：

| 看到的形式 | 应该怎么用 |
|-----------|-----------|
| 带 `public class Main` 和 `main` 的完整程序 | 按标注的文件名保存后运行 |
| `int count = ...;`、`list.add(...)` 等语句 | 放到练习程序的 `main` 方法里；每个例子单独练习，避免同名变量重复声明 |
| `import java.util.List;` | 放在文件顶部、类声明之前；有 `package` 时放在它之后 |
| `class User { ... }` 这样的类定义 | 通常单独放到 `User.java` 中；如果写成 `public class User`，文件名必须与类名一致 |
| 字段或方法定义 | 通常放在类的大括号内；其他方法与 `main` 同级，不能定义在 `main` 方法内部 |
| 带 `@RestController`、`@Entity` 等框架注解 | 放到对应的 Spring 项目文件中，先准备本节要求的依赖和配置 |
| 标注“片段”“伪代码”或假设已有业务对象的例子 | 用来读调用过程；补齐列出的类型、方法和环境后才能运行 |

例如，集合章节常用的导入应该这样放：

```java
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String[] args) {
        List<String> names = new ArrayList<>();
        names.add("Alice");
        System.out.println(names); // [Alice]
    }
}
```

这里 `List<String>` 表示“元素是字符串的列表”，`new ArrayList<>()` 创建实际存数据的对象，`names.add(...)` 往里放一项。第 02、03 章会继续拆开解释。

如果例子还定义了一个方法，它与 `main` 的位置关系如下。可以把这个完整程序单独保存为 `Main.java`：

```java
public class Main {
    public static void main(String[] args) {
        System.out.println(add(2, 3)); // 在 main 内调用，输出 5
    }

    public static int add(int a, int b) { // 在类内、main 外定义
        return a + b;
    }
}
```

调用 `add(2, 3)` 时，把 `2` 和 `3` 传给参数 `a`、`b`，`return` 将相加结果交回调用处。

## 遇到陌生代码，按这四步读

1. **找输入。** 数据从哪里来？是 `1～10`、一个列表，还是 HTTP 请求？
2. **找结果。** 最终要返回什么、打印什么，或者改变什么数据？
3. **拆步骤。** 每次只看一个方法调用，写出这一行之后的数据或状态。
4. **换小数据。** 把 1 亿改成 10，把数据库查询换成两条示例记录，再预测结果。

例如集合章节中的这段代码：

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6);

long count = numbers.stream()
    .filter(n -> n % 2 == 0)
    .count();

System.out.println(count); // 3
```

先看输入 `[1, 2, 3, 4, 5, 6]`，筛选后剩下 `[2, 4, 6]`，最后计数得到 `3`。`stream()` 创建处理流程；`filter(...)` 描述筛选条件；`count()` 触发处理并统计数量。`n -> n % 2 == 0` 相当于“给我一个数字，我判断它是不是偶数”。这与前面的普通循环解决同一个问题。

## 常见符号速查

先记住怎么读，具体规则回到对应章节练习。

| 写法 | 先这样理解 | 在哪里学 |
|------|-----------|---------|
| `int count = 3;` | 类型、变量名、初始值；分号结束一条语句 | 第 01 章 |
| `{ ... }` | 一段代码的范围，例如类、方法或条件分支 | 第 01 章 |
| `method(value)` | 调用方法，把 `value` 作为参数传进去 | 第 01、02 章 |
| `new User("Alice")` | 创建一个 `User` 对象，并调用构造器 | 第 02 章 |
| `this.name = name` | 把参数 `name` 保存到当前对象的字段中 | 第 02 章 |
| `List<String>` | 元素类型为 `String` 的列表；这里的 `< >` 表示类型参数 | 第 02、03 章 |
| `n -> n * 2` | 接收一个参数 `n`，返回它乘以 2 的结果 | 第 01、03 章 |
| `User::getName` | 在这里可理解为 `user -> user.getName()` 的简写 | 第 01、03 章 |
| `@Override` | 告诉编译器，这个方法要重写父类型的方法 | 第 02 章 |
| `@RestController` | 标记这个类由 Spring 用来处理 HTTP 请求 | 第 05 章 |

## 如何借用前端经验？

前端经验可以帮你理解用途，但需要继续核对 Java 的具体规则。

| 熟悉的概念 | Java 中可以联系到什么 | 要留意的差别 |
|-----------|----------------------|-------------|
| TypeScript 类型声明 | Java 变量、方法参数和返回值类型 | Java 也不替你自动校验所有外部输入 |
| JS `number` | `int`、`long`、`double` 等数值类型 | 整数有固定范围；`long` 也不同于任意精度的 JS `bigint` |
| 数组的 `map` / `filter` | Stream 的转换与筛选 | 流通常在终止操作时才处理，且一个流只能消费一次 |
| 类与接口 | Java 的类、继承与接口 | Java 接口通常需要显式 `implements` |
| `Promise` | `CompletableFuture` 的结果组合 | 执行线程、阻塞等待和取消规则不同 |
| Express 路由 | Spring MVC 的 Controller 方法 | 路由注册、对象创建和参数转换由框架参与完成 |
| `package.json`、npm | Maven 的 `pom.xml` 和依赖管理 | Java 代码需要编译；Maven 也管理构建与测试流程 |

Java 和 Node.js 都能开发后端服务。实际性能取决于计算任务、I/O、数据库、运行配置等，不能只按语言判断快慢。学 Java 时，优先理解类型、对象和请求处理过程。

## 每次练习做到哪一步？

用一个短循环完成一节学习：**读用途 → 预测结果 → 运行小例子 → 改一个输入 → 解释变化**。

- **基础阶段：**能用普通循环统计偶数，能说清 `==` 和 `equals()` 的区别。
- **对象阶段：**能创建一个对象，调用它的方法，说明字段和局部变量的区别。
- **集合阶段：**能用循环和 Stream 分别完成一次筛选，说明中间数据如何变化。
- **接口阶段：**能说明一个 URL 如何找到 Controller，参数如何传入，响应如何返回。
- **实战阶段：**能独立完成一条数据的 CRUD，并核对成功和失败响应。
- **并发阶段：**能指出共享变量，说明何时等待任务完成，知道输出顺序可能不确定。

遇到报错时，先核对文件位置、导入、变量定义、JDK 版本和依赖。编译通过只能说明满足编译规则；预期输出、业务行为和线程安全还需要分别验证。

## 推荐资源

- [Java 官方文档](https://docs.oracle.com/en/java/)：查语法与 API 的详细规则。
- [Spring 官方入门指南](https://spring.io/guides)：从一个最小功能开始搭项目。
- LeetCode、Exercism：练基础语法和数据处理；先完成小题即可。
- 《Effective Java》：有实际编码经验后再读，更容易理解其中的取舍。

*最后更新：2026-11-17*
