# 面向对象编程（OOP）

> 这一章解决三个问题：怎么把数据和操作放进一个类、怎么创建对象、怎么读懂别人定义的类型。示例使用 JDK 21，不需要预览功能。

## 先学哪些内容

- **先掌握**：类、构造器、`new`、`this`、实例成员与静态成员、接口、基本泛型。
- **接着学**：继承、枚举、`record`。读懂后就可以进入集合章节。
- **用到再回来看**：Lombok、泛型通配符、类型擦除、设计模式。第一次不用背下来。

本章每个示例独立，建议各建一个文件夹，避免同名的 `Main`、`User` 互相冲突。在该文件夹中保存源文件，并打开终端执行命令。标为“保存为 `Main.java`”的代码可以用 `javac Main.java`、`java Main` 运行；拆成 `User.java` 与 `Main.java` 的示例，把两个文件放在同一个文件夹，用 `javac User.java Main.java` 一起编译，再执行 `java Main`。调用语句放进 `Main` 的 `main` 方法，`import` 放在文件最上面。Lombok 示例需要额外依赖，不能只靠 JDK 运行。

## 目录

1. [类的完整定义](#类的完整定义)
2. [Lombok](#lombok)
3. [继承](#继承)
4. [抽象类 vs 接口](#抽象类-vs-接口)
5. [枚举 enum](#枚举-enum)
6. [泛型](#泛型)
7. [Record 类](#record-类)
8. [常用设计模式](#常用设计模式)

---

## 类的完整定义

### 先读一个完整的小程序

**场景**：保存一个人的姓名，并让这个人“自我介绍”。保存为 `Main.java`：

```java
class User {
    private String name;              // 每个 User 对象各自保存一个姓名

    public User(String name) {        // 构造器：创建对象时执行
        this.name = name;
    }

    public String greet() {           // 方法：返回一段文字
        return "你好，我是" + name;
    }
}

public class Main {
    public static void main(String[] args) {
        User user = new User("小林");
        System.out.println(user.greet());
    }
}
```

输出：`你好，我是小林`。

按执行顺序看：

1. `new User("小林")` 创建一个 `User` 对象，并执行 `User(String name)` 构造器。
2. 构造器参数 `name` 收到 `"小林"`；`this.name` 指当前对象的字段，因此赋值后对象记住了这个姓名。
3. 变量 `user` 保存对这个对象的引用。`user.greet()` 调用该对象的方法，得到返回的字符串。
4. `println` 把这个字符串打印出来。

| 写法 | 读成什么 |
|------|----------|
| `class User` | 定义一种叫 User 的对象类型 |
| `private String name` | 只能在 User 类内部直接访问的字符串字段 |
| `public User(String name)` | 允许外部调用的构造器；名字与类相同，没有返回类型 |
| `public String greet()` | 允许外部调用，返回 String，不接收参数的方法 |
| `return ...;` | 把结果交还给调用者，并结束本次方法调用 |
| `this` | 当前正在操作的这个对象 |

`public` 表示允许从其他类访问，`private` 表示限制在类内部。示例中的 `User` 没有写 `public`，因此能与 `public class Main` 放在同一个文件中；顶层 `public` 类的文件名要与类名一致。

### 字段、构造器、普通方法、静态方法放在一起看

下面是扩展版，**替换前面的小程序**，仍保存为 `Main.java`：

```java
class User {
    private String name;
    private final String email;
    private static int totalCount = 0;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
        totalCount++;
    }

    public static User of(String name, String email) {
        return new User(name, email);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDisplayName() {
        return name + " <" + email + ">";
    }

    public static int getTotalCount() {
        return totalCount;
    }

    @Override
    public String toString() {
        return getDisplayName();
    }
}

public class Main {
    public static void main(String[] args) {
        User first = new User("Alice", "alice@example.com");
        User second = User.of("Bob", "bob@example.com");
        first.setName("小林");

        System.out.println(first.getDisplayName());
        System.out.println(second);
        System.out.println(User.getTotalCount());
    }
}
```

输出依次为：

```text
小林 <alice@example.com>
Bob <bob@example.com>
2
```

- `name`、`email` 是**实例字段**：`first` 改名不会修改 `second`。
- `static totalCount` 是**静态字段**：属于 `User` 类，这两个对象共用同一个计数。这里只演示单线程计数，并发计数另见并发章节。
- `public` 管理访问权限，`static` 表示属于类，是两件事。静态方法可通过 `User.of(...)` 调用，不需要先有 `User` 对象；实例方法通过 `first.getName()` 调用，可以直接使用该对象的字段和 `this`。静态方法没有 `this`，要访问实例字段必须先取得对象。
- `final email` 在构造完成后不能重新赋值。这里没有提供修改邮箱的方法。
- `getName()` 读取姓名，`setName(...)` 修改姓名。`void` 表示方法不返回结果，不能把 `setName(...)` 的调用结果赋给变量。
- `User.of(...)` 是**静态工厂方法**：给创建过程起了一个方法名，内部仍然用 `new`。它是可选写法，并非必须替代构造器。
- `@Override` 是给编译器看的注解，表示这个方法重写了父类的方法。所有普通类都继承 `Object`；重写 `toString()` 后，打印对象时会使用这里返回的文字。

### equals 与 hashCode：什么时候才需要重写

默认的 `equals()` 比较是否为同一个对象。如果业务希望“两个不同对象，只要某些数据相同就算相等”，才需要定义自己的相等规则；**并不是放进集合就必须重写**。

保存为 `Main.java`，演示“邮箱相同就视为同一账号”：

```java
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;

final class Account {
    private final String email;

    public Account(String email) {
        this.email = Objects.requireNonNull(email, "邮箱不能为空");
    }

    @Override
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof Account)) return false;
        Account account = (Account) other;
        return email.equals(account.email);
    }

    @Override
    public int hashCode() {
        return email.hashCode();
    }
}

public class Main {
    public static void main(String[] args) {
        Set<Account> accounts = new HashSet<>();
        accounts.add(new Account("alice@example.com"));
        accounts.add(new Account("alice@example.com"));
        System.out.println(accounts.size()); // 1
    }
}
```

`Objects.requireNonNull(...)` 在参数为 `null` 时抛出异常，避免创建没有邮箱的账号。`instanceof` 检查对象是不是指定类型；`(Account) other` 在检查通过后把它当作 `Account` 使用。`Set<Account>` 表示只存 `Account` 的去重集合，第 3 章会展开。

`HashSet` 会用 `hashCode()` 找到查找范围，再用 `equals()` 判断相等。所以重写 `equals()` 时也要让 `hashCode()` 遵守“相等对象哈希值相同”的规则。这里邮箱是 `final`，避免对象放入哈希集合后相等依据发生变化。哈希值相同不一定表示对象相等。

---

## Lombok

**这一节可以后看。** Lombok 是额外的代码生成工具：编译时根据注解生成 getter、构造器等方法，减少重复代码。先能读懂手写方法，再学它更容易。

### 先看注解替你省了什么

手写版本保存为 `User.java`：

```java
public class User {
    private final String name;

    public User(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}
```

在**已经配置 Lombok 依赖和注解处理器的项目中**，可以用下面的版本替换它：

```java
import lombok.Getter;
import lombok.RequiredArgsConstructor;

@Getter
@RequiredArgsConstructor
public class User {
    private final String name;
}
```

两种写法都支持在 `main` 中调用：

```java
User user = new User("Alice");
System.out.println(user.getName()); // Alice
```

`@Getter` 生成 `getName()`，`@RequiredArgsConstructor` 为未初始化的 `final` 字段及适用的 `@NonNull` 字段生成构造器。注解不是运行时“凭空增加方法”，需要编译工具支持。

### 依赖放在哪里

下面 XML 放在 Maven 项目的 `pom.xml` 的 `<dependencies>` 中。它假设项目已经通过 Spring Boot 等依赖管理指定兼容的 Lombok 版本；独立 Maven 项目还要显式指定版本，并根据构建配置启用注解处理。项目搭建见第 5 章。

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

`optional` 控制依赖向其他项目传递的方式，不代表 Maven 可以忽略这个依赖。IDE 与命令行构建都应能识别注解处理。

### 常用注解速查

| 注解 | 生成的内容或作用 |
|------|------------------|
| `@Getter` / `@Setter` | getter / 可修改字段的 setter |
| `@Data` | getter、非 final 字段的 setter、toString、equals/hashCode、必要参数构造器 |
| `@NoArgsConstructor` | 无参构造器，受 final 字段等约束 |
| `@AllArgsConstructor` | 包含各实例字段参数的构造器 |
| `@RequiredArgsConstructor` | 为未初始化 final 字段、适用的 @NonNull 字段生成构造器 |
| `@Builder` | `builder() ... build()` 形式的创建 API |
| `@Value` | 默认生成 final 类、private final 字段、getter 等；并非深不可变 |
| `@Slf4j` | 生成名为 log 的 SLF4J 日志字段，还需日志依赖 |
| `@ToString` | 生成对象的文字表示 |

### 用 Lombok 重写 User 类

下面是**另一种 User 定义**，不要与前面的 `User` 一起声明。需要 Lombok、SLF4J 以及日志实现：

```java
import java.time.LocalDateTime;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Slf4j
public class User {
    private Long id;
    private String name;
    private String email;

    @Builder.Default
    private LocalDateTime createdAt = LocalDateTime.now();

    public void printInfo() {
        log.info("User: {}", this);
    }
}
```

在该项目的 `main` 或测试中使用：

```java
User user = User.builder()          // 先拿到一个建造者
    .name("Alice")                 // 给待创建对象设置姓名
    .email("alice@example.com")     // 再设置邮箱
    .build();                      // 最后生成 User 对象
System.out.println(user.getName()); // Alice
```

这串点号表示连续调用方法。`@Builder.Default` 让建造者未设置 `createdAt` 时采用字段初始值。`log.info` 中的 `{}` 会被对象的文字表示替换；这不是 Java 的字符串插值语法。

是否用 Lombok 服从项目约定。`@Data` 会生成可修改方法和相等规则，并不适合所有实体；字段含密码等内容时也不能随意生成包含全部字段的 `toString()`。

---

## 继承

**场景**：动物都有名字，狗又有自己的叫声。`extends` 表示狗继承动物已有的字段和行为，并可以重写行为。

下面保存为 `Main.java`：

```java
class Animal {
    protected String name;

    public Animal(String name) {
        this.name = name;
    }

    public String speak() {
        return name + " makes a sound";
    }

    public String describe() {
        return "名字：" + name;
    }
}

class Dog extends Animal {
    public Dog(String name) {
        super(name);
    }

    @Override
    public String speak() {
        return name + " says: Woof!";
    }
}

public class Main {
    public static void main(String[] args) {
        Animal animal = new Dog("Rex");
        System.out.println(animal.speak());
        System.out.println(animal.describe());
    }
}
```

输出：

```text
Rex says: Woof!
名字：Rex
```

1. `new Dog("Rex")` 创建狗；`super(name)` 调用父类构造器，初始化继承的姓名字段。JDK 21 中，显式的 `super(...)` / `this(...)` 构造器调用必须放在构造器第一条语句。
2. `protected` 允许子类使用父类中的字段，也允许同包代码访问。
3. 变量类型虽然是 `Animal`，实际对象是 `Dog`。调用可重写的实例方法 `speak()` 时执行狗的版本，这叫**多态**。
4. `Dog` 没有重写 `describe()`，因此使用父类实现。

Java 类只能继承一个父类，但可以实现多个接口。`private` 方法不能被子类重写，`static` 方法属于类，不能按上面的方式动态分派；`final` 方法不能重写，`final` 类不能继承。无需先学习 JVM 如何实现方法分派。

---

## 抽象类 vs 接口

### 先理解接口：规定“会做什么”

接口可以看成一份约定：“实现我的类，必须提供这些操作”。保存为 `Main.java`：

```java
interface Greeter {
    String greet(String name);
}

class FriendlyGreeter implements Greeter {
    @Override
    public String greet(String name) {
        return "你好，" + name;
    }
}

public class Main {
    public static void main(String[] args) {
        Greeter greeter = new FriendlyGreeter();
        System.out.println(greeter.greet("小林")); // 你好，小林
    }
}
```

`String greet(String name);` 只有方法签名，没有实现。`implements Greeter` 表示这个类承诺提供实现。调用者只需要知道接口，不需要依赖具体类中的其他细节。Java 接口会编译成字节码，TypeScript 的 `interface` 则在编译后被移除。

### 对比表

| 特性 | 抽象类 (`abstract class`) | 接口 (`interface`) |
|------|--------------------------|-------------------|
| 直接 new | 不可以 | 不可以 |
| 使用方式 | 类 extends 一个父类 | 类 implements 一个或多个接口 |
| 实例字段、构造器 | 可以有 | 不可以有；接口字段是 public static final 常量 |
| 方法实现 | 可以有普通方法和抽象方法 | 可有抽象方法、default / static 方法，Java 9+ 也支持 private 方法 |
| 主要用途 | 共享对象状态与部分实现 | 定义可替换的行为约定 |

### 再把抽象类和接口组合起来

下面是上一个小程序的独立替换示例，保存为 `Main.java`：

```java
abstract class Shape {
    protected final String color;

    public Shape(String color) {
        this.color = color;
    }

    public abstract double area(); // 子类负责给出面积的算法

    public void printInfo() {
        System.out.printf("%s图形，面积=%.2f%n", color, area());
    }
}

interface Drawable {
    void draw();

    default String getDescription() {
        return "可绘制的对象";
    }
}

class Circle extends Shape implements Drawable {
    private final double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }

    @Override
    public void draw() {
        System.out.println("画一个半径为 " + radius + " 的圆");
    }
}

public class Main {
    public static void main(String[] args) {
        Circle circle = new Circle("红色", 2);
        circle.printInfo();
        circle.draw();
        System.out.println(circle.getDescription());
    }
}
```

在小数点使用 `.` 的地区设置下，依次输出 `红色图形，面积=12.57`、`画一个半径为 2.0 的圆`、`可绘制的对象`。

`abstract double area()` 没有方法体，必须由非抽象子类实现；`printInfo()` 是共享实现，内部调用 `area()` 时会执行 `Circle` 的版本。`default` 方法给接口提供可继承的默认实现，所以 `Circle` 不需要另写 `getDescription()`。`%.2f` 表示小数保留两位，`%n` 表示换行。

---

## 枚举 enum

枚举表示“一组固定的选项”，例如订单状态。相比任意字符串，它能让编译器发现拼错状态名的问题。

### 基础用法

将枚举保存为 `Day.java`：

```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}
```

在 `main` 中调用：

```java
Day today = Day.MONDAY;
System.out.println(today.name());    // MONDAY
System.out.println(today.ordinal()); // 0：声明顺序从 0 开始
Day[] days = Day.values();           // 所有选项组成的数组
System.out.println(days.length);     // 7
Day day = Day.valueOf("FRIDAY");
System.out.println(day);             // FRIDAY
```

`valueOf` 要求名字精确匹配；传入不存在的名字会抛出异常。不要把 `ordinal()` 当数据库中的稳定业务编码，调整声明顺序会改变它。

### 带字段和方法的枚举

固定选项也能保存数据。下面保存为 `OrderStatus.java`：

```java
public enum OrderStatus {
    PENDING("待支付", false),
    PAID("已支付", true),
    SHIPPED("已发货", true),
    DELIVERED("已收货", true),
    CANCELLED("已取消", false);

    private final String label;
    private final boolean paid;

    OrderStatus(String label, boolean paid) {
        this.label = label;
        this.paid = paid;
    }

    public String getLabel() {
        return label;
    }

    public boolean isPaid() {
        return paid;
    }

    public boolean canCancel() {
        return this == PENDING;
    }
}
```

`PAID("已支付", true)` 表示声明 `PAID` 这个固定实例时，把文字和布尔值传给枚举构造器。不能自己 `new OrderStatus(...)`。

在 `main` 中调用：

```java
OrderStatus status = OrderStatus.PAID;
System.out.println(status.getLabel());  // 已支付
System.out.println(status.isPaid());    // true
System.out.println(status.canCancel()); // false
```

这里“只有待支付可取消”只是示例规则；`paid` 也只是简化状态分类，不是实际支付记录。

### 枚举 + switch（Java 14+ switch 表达式）

接在上一段 `main` 代码后，根据状态选一条提示：

```java
String message = switch (status) {
    case PENDING -> "请尽快完成支付";
    case PAID, SHIPPED -> "订单处理中，请耐心等待";
    case DELIVERED -> "感谢您的购买！";
    case CANCELLED -> "订单已取消";
};
System.out.println(message); // 订单处理中，请耐心等待
```

这里 `->` 表示“命中这个分支就取右侧的值”，不是 Lambda，也不是模式匹配。枚举所有选项都已列出，因此不需要 `default`。同一个枚举常量只有一个实例，可以安全用 `==` 比较；`status` 为 `null` 时这里仍会抛出异常。

---

## 泛型

### 基础语法

**场景**：写一个盒子类，有时装字符串，有时装整数，不想复制两份实现。

下面保存为 `Main.java`：

```java
class Box<T> {
    private final T value;

    public Box(T value) {
        this.value = value;
    }

    public T getValue() {
        return value;
    }
}

public class Main {
    public static void main(String[] args) {
        Box<String> words = new Box<>("Hello");
        Box<Integer> number = new Box<>(42);
        System.out.println(words.getValue());  // Hello
        System.out.println(number.getValue()); // 42
    }
}
```

- `<T>` 声明一个类型参数，`T` 是占位名字，不是一个固定类型。
- `Box<String>` 表示这次用 `String` 替代 `T`，所以 `getValue()` 的返回类型是 `String`。
- `new Box<>(...)` 的空尖括号让编译器根据上下文推断类型，不代表“任何类型都能装”。
- 泛型实参不能直接写基本类型 `int`，应写包装类 `Integer`；`42` 会自动装箱。

TypeScript 也有 `Box<T>` 这样的写法。Java 将返回类型写在方法名前面：`public T getValue()`。

### 泛型方法

类不使用 `<T>`，其中某个方法也可以单独使用它。保存为 `Main.java`：

```java
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static <T> List<T> repeat(T item, int times) {
        List<T> result = new ArrayList<>();
        for (int i = 0; i < times; i++) {
            result.add(item);
        }
        return result;
    }

    public static void main(String[] args) {
        List<String> words = repeat("hello", 3);
        System.out.println(words); // [hello, hello, hello]
    }
}
```

把签名分成几块读：`public static` 表示可通过类调用；`<T>` 声明本方法的类型参数；`List<T>` 是返回类型；`repeat` 是方法名；括号内是参数。这里传入字符串，所以 `T` 被推断成 `String`。重复添加对象时添加的是同一个引用，并没有复制对象。

### 通配符

**进阶，可后看。** `List<Integer>` 不能直接赋值给 `List<Number>`，否则后者可能往里面添加 `Double`，破坏前者“只含 Integer”的约定。通配符用于表达我们实际需要的读写能力。

下面是 `Main` 类中的两个静态方法，需要 `import java.util.List;`：

```java
public static double sum(List<? extends Number> numbers) {
    double total = 0;
    for (Number number : numbers) {
        total += number.doubleValue();
    }
    return total;
}

public static void addNumbers(List<? super Integer> numbers) {
    numbers.add(1);
    numbers.add(2);
}
```

调用语句放在 `main` 中，并加上 `import java.util.ArrayList;`：

```java
System.out.println(sum(List.of(1, 2, 3))); // 6.0
List<Number> numbers = new ArrayList<>();
addNumbers(numbers);
System.out.println(numbers);              // [1, 2]
```

- `? extends Number`：列表元素的具体类型未知，但一定是 `Number` 或其子类；可以当 `Number` 读出，不能随意添加整数或小数。
- `? super Integer`：具体类型是 `Integer` 或其父类；可以添加 `Integer`，但读出来只能保证是 `Object`。

常见口诀是 PECS：提供数据给你读，用 `extends`；接收你写入的数据，用 `super`。“只读/只写”是用途提示，并非容器被强制设置成只读或不能读取。

### 类型擦除

泛型主要由编译器检查。运行时，两个 `ArrayList` 不会仅因泛型参数不同而成为两个不同的类。

下面语句放在 `main` 中，需要 `java.util.List` 和 `java.util.ArrayList`：

```java
List<String> strings = new ArrayList<>();
List<Integer> integers = new ArrayList<>();
System.out.println(strings.getClass() == integers.getClass()); // true
```

这叫类型擦除。它不表示 Java 的所有类型都消失了，也不表示所有泛型信息在所有位置都完全不存在；初学时先记住，不能依赖 `instanceof List<String>` 在运行时检查列表里是否都是字符串。TypeScript 的泛型同样主要用于编译期检查。

---

## Record 类

**场景**：只想携带一组数据，不想手写构造器、读取方法、相等比较和打印方法。JDK 21 可以直接使用 `record`。

保存为 `Main.java`：

```java
record Point(int x, int y) {}

public class Main {
    public static void main(String[] args) {
        Point point = new Point(3, 4);
        System.out.println(point.x());                    // 3
        System.out.println(point.y());                    // 4
        System.out.println(point);                        // Point[x=3, y=4]
        System.out.println(point.equals(new Point(3, 4))); // true
    }
}
```

`record Point(int x, int y)` 声明两个数据分量，Java 自动生成构造器、`x()` / `y()` 读取方法、`equals()`、`hashCode()` 和 `toString()`。读取方法不是 `getX()`；也没有自动生成 `setX()`。

下面是前面 `Point` 定义的**替换版本**，为坐标增加校验和距离计算：

```java
record Point(int x, int y) {
    public Point {
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException("坐标不能为负");
        }
    }

    public double distance() {
        return Math.hypot(x, y);
    }
}
```

`public Point { ... }` 是紧凑构造器，接收声明中的 `x`、`y`；校验通过后 Java 自动完成字段赋值。使用原来的 `main` 再加上 `System.out.println(point.distance());`，会输出 `5.0`；`new Point(-1, 4)` 则抛出异常。

`record` 的字段不能重新赋值，但如果分量是可修改的列表，列表内容仍可能改变，这叫**浅不可变**。需要时应另做防御性复制。TypeScript 的 `readonly` 接口只是编译期约束，也不会替你生成 Java 这里的这些方法。

### DTO 场景对比

DTO 是在接口或各层之间传递数据的对象。

| 方案 | 需要理解的内容 | 可变性 | 常见用途 |
|------|----------------|--------|----------|
| 普通类 | 字段、构造器、方法都由你设计 | 由实现决定 | 有业务行为、需框架特殊支持的对象 |
| Lombok `@Value` | 注解处理、生成规则 | 通常浅不可变 | 使用 Lombok 的项目中的数据对象 |
| `record` | 数据分量、自动方法 | 浅不可变 | 简单 DTO、值对象 |

选择前要看框架要求，例如不能把 `record` 直接当作普通 JPA 实体使用。

---

## 常用设计模式

**这一节用于认出项目里的写法，不要求一次掌握。** 设计模式是组织已有类和方法的常见方式，不是需要背诵的新语法。

### 单例模式

**目的**：让调用方共用一个配置对象。先读最短的枚举版本，保存为 `Main.java`：

```java
enum AppConfig {
    INSTANCE;

    private final String dbUrl = "jdbc:mysql://localhost/mydb";

    public String getDbUrl() {
        return dbUrl;
    }
}

public class Main {
    public static void main(String[] args) {
        AppConfig first = AppConfig.INSTANCE;
        AppConfig second = AppConfig.INSTANCE;
        System.out.println(first == second); // true
        System.out.println(first.getDbUrl()); // jdbc:mysql://localhost/mydb
    }
}
```

`INSTANCE` 是唯一的枚举常量，所以两次取得的是同一个对象。这里仅保存一个地址字符串，没有真的连接数据库；真实应用的数据库连接通常交给连接池管理。

下面是阅读旧项目时可能遇到的**双重检查锁版本**，保存为 `AppConfigLazy.java`。`volatile` 和 `synchronized` 要在第 4 章学习后再理解，不必现在记忆：

```java
public final class AppConfigLazy {
    private static volatile AppConfigLazy instance;

    private AppConfigLazy() {}

    public static AppConfigLazy getInstance() {
        if (instance == null) {
            synchronized (AppConfigLazy.class) {
                if (instance == null) {
                    instance = new AppConfigLazy();
                }
            }
        }
        return instance;
    }
}
```

它在第一次需要时才创建对象；外层判断减少锁操作，锁内再次判断防止重复创建，`volatile` 保证安全发布。单例实例的创建安全，并不代表其中任意可修改字段都自动线程安全。在 Spring 中通常由容器管理对象，不需要自己套用这个实现。

### 建造者模式（Lombok 版）

**目的**：创建参数较多的对象时，通过方法名看清每个值的含义。以下 `HttpRequest.java` 依赖已经配置好的 Lombok：

```java
import java.util.Map;
import lombok.Builder;
import lombok.Getter;

@Builder
@Getter
public class HttpRequest {
    private String url;
    private String method;
    private Map<String, String> headers;
    private String body;
    private int timeoutMs;
}
```

在同一项目的 `main` 中使用：

```java
HttpRequest request = HttpRequest.builder()
    .url("https://api.example.com/users")
    .method("POST")
    .body("{\"name\":\"Alice\"}")
    .timeoutMs(5000)
    .build();
System.out.println(request.getTimeoutMs()); // 5000
```

每次 `.url(...)` / `.method(...)` 配置建造者，`.build()` 才生成对象。此例仅描述请求数据，不会发送 HTTP 请求；`@Builder` 也不会自动校验必填值，未设置的引用字段仍为 `null`。

### 策略模式

**目的**：调用流程不变，只替换其中一个处理规则。下面用“问候语规则”演示，保存为 `Main.java`：

```java
interface GreetingStrategy {
    String greet(String name);
}

class FormalGreeting implements GreetingStrategy {
    @Override
    public String greet(String name) {
        return "您好，" + name;
    }
}

class Greeter {
    private final GreetingStrategy strategy;

    public Greeter(GreetingStrategy strategy) {
        this.strategy = strategy;
    }

    public String sayHello(String name) {
        return strategy.greet(name);
    }
}

public class Main {
    public static void main(String[] args) {
        Greeter formal = new Greeter(new FormalGreeting());
        System.out.println(formal.sayHello("小林")); // 您好，小林

        Greeter casual = new Greeter(name -> "嗨，" + name);
        System.out.println(casual.sayHello("小林")); // 嗨，小林
    }
}
```

`Greeter` 只负责调用 `strategy.greet(name)`，文字如何生成由传入的策略决定。接口只有一个需要实现的抽象方法，所以第二个策略可以用 Lambda 表达式：`name -> "嗨，" + name`。

真实项目里的价格计算、排序方式等，也可以替换成不同策略。没有真实变化需求时，直接写一个方法更简单。

### 观察者模式

**目的**：一个事件发生后，依次通知多个关心它的处理者。下面仅打印通知，不发送真实邮件或短信。保存为 `Main.java`：

```java
import java.util.ArrayList;
import java.util.List;

record Order(String product, int quantity) {}

interface OrderObserver {
    void onOrderCreated(Order order);
}

class OrderService {
    private final List<OrderObserver> observers = new ArrayList<>();

    public void addObserver(OrderObserver observer) {
        observers.add(observer);
    }

    public Order createOrder(String product, int quantity) {
        Order order = new Order(product, quantity);
        for (OrderObserver observer : observers) {
            observer.onOrderCreated(order);
        }
        return order;
    }
}

class EmailNotifier implements OrderObserver {
    @Override
    public void onOrderCreated(Order order) {
        System.out.println("邮件通知：" + order.product());
    }
}

public class Main {
    public static void main(String[] args) {
        OrderService service = new OrderService();
        service.addObserver(new EmailNotifier());
        service.addObserver(order ->
            System.out.println("短信通知：购买了 " + order.quantity() + " 件"));
        service.createOrder("键盘", 1);
    }
}
```

输出：

```text
邮件通知：键盘
短信通知：购买了 1 件
```

`addObserver` 先登记处理者，`createOrder` 创建订单后遍历列表，依次调用每个处理者。这里的 Lambda 对应接口中的 `onOrderCreated(Order order)`。通知在当前线程同步执行；某个处理者抛异常会中断后续通知，不能把它当成已经具备重试、事务或消息队列能力的生产实现。

---

## 自测：能解释这些就可以继续

1. `User.of(...)` 和 `user.getName()`，哪个属于类，哪个属于对象？
2. `Animal animal = new Dog("Rex")` 调用 `speak()`，执行哪个实现？
3. `Box<String>` 中的 `T` 是什么？`Box<int>` 为什么不行？
4. `record Point(int x, int y)` 读取 x 应该写 `point.x`、`point.getX()` 还是 `point.x()`？
5. 观察者示例为什么先调用 `addObserver`，再调用 `createOrder`？

<details>
<summary>展开答案</summary>

1. `User.of(...)` 是静态方法，通过类调用；`user.getName()` 操作一个具体对象。
2. 执行 `Dog.speak()`，因为实际对象是 `Dog`。
3. `T` 在这里替换为 `String`；泛型实参必须是引用类型，整数应使用 `Integer`。
4. `point.x()`，它是 record 自动生成的读取方法。
5. 先登记接收者，创建订单时才有可通知的对象；创建后再登记不会收到之前的通知。

</details>

*← [上一章：基础语法](./01-basics.md) | [下一章：集合框架](./03-collections.md) →*
