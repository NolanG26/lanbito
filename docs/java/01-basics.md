# Java 基础语法完整篇

> 面向有 TypeScript / Node.js 经验的前端工程师

这一章的目标是：**看懂变量声明、条件、循环和方法调用，并能独立运行一个小程序**。先学类型、变量、常用运算、控制流程、数组、字符串和基本异常处理；位运算、带标签的跳转、函数式接口全表可以用到时再看。

## 先跑通一个程序

本章按 **JDK 21** 编写，不需要预览特性或第三方依赖。先把下面的内容保存为 `Main.java`：

```java
public class Main {
    public static void main(String[] args) {
        String name = "Alice";
        int age = 20;
        System.out.println(name + " 明年 " + (age + 1) + " 岁");
    }
}
```

在这个文件所在目录运行：

```bash
javac Main.java
java Main
```

输出为 `Alice 明年 21 岁`。第一条命令编译出 `Main.class`，第二条启动程序；也可以用 JDK 21 的 `java Main.java` 直接运行这个单文件程序。后面若增加其他 `.java` 文件，需要先把它们一起编译，再用 `java Main` 启动，不能继续依赖单文件源码运行。

第一次看到外层结构，先这样读：

| 写法 | 这里的意思 |
|------|------------|
| `public class Main` | 定义一个名为 `Main`、可被外部访问的类；这里的文件名必须是 `Main.java` |
| `{ ... }` | 括号内是这一层包含的代码；类中有方法，方法中有语句 |
| `public static void main(String[] args)` | 程序入口，Java 启动时从这里执行 |
| `static` | 入口属于类，不必先创建 `Main` 对象 |
| `void` | 方法不返回结果；它仍可以打印内容或修改数据 |
| `String[] args` | 启动时传入的字符串数组，本例没有使用 |
| `String name = "Alice";` | 声明字符串变量 `name`，赋值为 `"Alice"`；分号结束语句 |
| `System.out.println(...)` | 调用打印方法，打印括号内的值并换行，可类比 `console.log(...)` |

**后面的代码怎么运行？** 标为普通语句的代码放进上面的 `main`，每个例子单独运行，避免重复声明同名变量。`import` 必须放在类定义前面；方法放在类里面、`main` 外面；独立的 `public class` 放进同名文件。含 `...` 或注明“结构示意”的片段只用于阅读，不能整块复制编译。

## 目录 {#toc}

- [1. 类型系统](#type-system)
- [2. 变量与常量](#variables)
- [3. 运算符](#operators)
- [4. 控制流程](#control-flow)
- [5. 数组](#arrays)
- [6. 字符串操作](#strings)
- [7. Lambda 表达式](#lambda)
- [8. 异常处理](#exceptions)

---

## 1. 类型系统 {#type-system}

Java 的变量有确定的类型。例如 `int age = 20` 表示 `age` 用来保存整数，不能再赋值为字符串 `"二十"`。编译器会检查这些类型约束；用 `var` 时由编译器推断类型，也不会变成动态类型。

💡 **对比 TS**：两者都有静态类型检查。Java 的类和部分类型信息也存在于运行时，但这不代表所有输入都合法，例如外部数据仍需校验，`null` 仍可能导致错误，泛型参数通常会擦除。

### 1.1 8种基本类型（Primitive Types）

| 类型 | 大小 | 范围 | 默认值 | 对比 TS |
|------|------|------|--------|---------|
| `byte` | 1字节 | -128 ~ 127 | 0 | — |
| `short` | 2字节 | -32768 ~ 32767 | 0 | — |
| `int` | 4字节 | -2^31 ~ 2^31-1 | 0 | `number` |
| `long` | 8字节 | -2^63 ~ 2^63-1 | 0L | 固定范围整数，不等同于任意精度的 `bigint` |
| `float` | 4字节 | 单精度浮点 | 0.0f | — |
| `double` | 8字节 | 双精度浮点 | 0.0 | `number` |
| `boolean` | 语言规范未固定存储大小 | true / false | false | `boolean` |
| `char` | 2字节 | 一个 UTF-16 代码单元 | '\u0000' | 不一定能表示一个完整字符，如部分 emoji 需两个 `char` |

表中的默认值针对**字段和数组元素**。方法中的局部变量必须先赋值才能读取。

下面是普通语句，先记住常用的 `int`、`long`、`double`、`boolean`，其余用到时查表：

```java
// 基本类型声明示例
byte age = 25;
short year = 2024;
int count = 1_000_000;      // Java 7+ 支持数字分隔符，提高可读性
long population = 8_000_000_000L;  // 超出 int 范围的整数字面量要加 L
float price = 9.99f;        // float 字面量必须加 f
double pi = 3.14159265358979;
boolean isActive = true;
char grade = 'A';           // 注意：单引号，不是双引号
```

> ⚠️ **注意**：Java 的 `int` 是 32 位，不像 JS 的 `number` 是 64 位浮点数。处理大数时用 `long`，或者 `BigInteger`/`BigDecimal`。

> ⚠️ **注意**：`float` 和 `double` 不能精确表示许多十进制小数。需要精确金额时，常用 `BigDecimal` 或约定单位的整数，并明确舍入规则。

下面先在文件顶部加入 `import java.math.BigDecimal;`，再把计算放入 `main`。`new` 表示创建对象，`BigDecimal("0.1")` 是用字符串构造精确十进制数；`x.add(y)` 返回相加后的新值。

```java
// 金融计算的坑
double a = 0.1 + 0.2;
System.out.println(a);  // 0.30000000000000004 ← 经典浮点精度问题

// ✅ 推荐：金融场景用 BigDecimal
BigDecimal x = new BigDecimal("0.1");
BigDecimal y = new BigDecimal("0.2");
System.out.println(x.add(y));  // 0.3 ✓
```

### 1.2 包装类（Wrapper Classes）

每个基本类型都有对应的**引用类型（包装类）**：

先理解 `int` 与 `Integer`：前者直接表示整数值；后者是对象类型，可以为 `null`，也可以用在 `List<Integer>` 这样的泛型中。`Integer.parseInt("123")` 表示“调用 `Integer` 提供的转换方法，把字符串变成整数”，不是访问某个对象的字段。

| 基本类型 | 包装类 |
|---------|-------|
| `int` | `Integer` |
| `long` | `Long` |
| `double` | `Double` |
| `boolean` | `Boolean` |
| `char` | `Character` |
| `byte` | `Byte` |
| `short` | `Short` |
| `float` | `Float` |

```java
// 包装类的常用场景
Integer num = Integer.valueOf(42);
int parsed = Integer.parseInt("123");     // String → int
String str = Integer.toString(42);        // int → String
int max = Integer.MAX_VALUE;              // 2147483647
int min = Integer.MIN_VALUE;              // -2147483648

// 进制转换
String binary = Integer.toBinaryString(10);  // "1010"
String hex = Integer.toHexString(255);       // "ff"
int fromHex = Integer.parseInt("ff", 16);    // 255
```

### 1.3 自动装箱与拆箱（Autoboxing / Unboxing）

Java 会自动在基本类型和包装类之间转换：

这一例在文件顶部加入 `import java.util.List;` 和 `import java.util.ArrayList;`。`List<Integer>` 可以读成“元素类型为 `Integer` 的列表”，`<>` 是类型参数的位置，第二章会展开解释。

```java
// 自动装箱：int → Integer
Integer a = 5;          // 等价于 Integer a = Integer.valueOf(5);

// 自动拆箱：Integer → int
int b = a;              // 等价于 int b = a.intValue();

// 集合中必须用包装类（集合不能存基本类型）
List<Integer> list = new ArrayList<>();
list.add(1);   // 自动装箱
list.add(2);
int sum = list.get(0) + list.get(1);  // 自动拆箱
```

> ⚠️ **常见坑：== 比较包装类**

```java
Integer x = 127;
Integer y = 127;
System.out.println(x == y);   // true  ← 因为 [-128, 127] 有缓存

Integer p = 200;
Integer q = 200;
System.out.println(p == q);   // 常见默认配置下为 false；缓存范围可能扩展，不依赖这个结果

// ✅ 推荐：比较包装类用 equals()
System.out.println(p.equals(q));  // true ✓
```

`==` 在两个包装对象之间比较“是不是同一个对象”，不适合比较数字内容。`p.equals(q)` 比较内容，但要求 `p` 非空；允许两端为空时可用 `java.util.Objects.equals(p, q)`。同样，`Integer missing = null; int value = missing;` 会在拆箱时抛出 `NullPointerException`。

### 1.4 String 类型

`String` 在 Java 中是**引用类型**，但行为很特殊：

```java
String name = "Alice";          // 字符串字面量（存储在常量池）
String name2 = new String("Alice");  // 堆上的新对象（不推荐）

// String 是不可变的（immutable）
String s = "hello";
s.toUpperCase();          // 返回新字符串，原 s 不变！
System.out.println(s);   // "hello"

String upper = s.toUpperCase();
System.out.println(upper);  // "HELLO"
```

---

## 2. 变量与常量 {#variables}

### 2.1 变量声明

💡 **对比 JS**：Java 每个变量都必须声明类型（除非用 `var`），没有 `undefined`。

```java
// Java 变量声明
int age = 25;
String name = "Alice";
boolean isStudent = true;

// Java 10+ 局部变量类型推断（类似 TS 的类型推断）
var message = "Hello";      // 推断为 String
var count = 42;             // 推断为 int
var list = new ArrayList<String>();  // 推断为 ArrayList<String>
```

这里的 `ArrayList` 需要 `import java.util.ArrayList;`。即使使用 `var`，`count` 仍是 `int`，不能执行 `count = "四十二"`。

> ⚠️ **注意**：`var` 只能用于**局部变量**，不能用于字段（类成员变量）。

```typescript
// JS/TS 对比
let age: number = 25;
const name: string = "Alice";
let isStudent: boolean = true;
```

### 2.2 常量

```java
// Java 常量用 final 关键字
final int MAX_SIZE = 100;
final String APP_NAME = "MyApp";
```

类常量可以由其他代码通过类名访问。下面的定义保存为 `Config.java`，与 `Main.java` 放在同一目录：

```java
public class Config {
    public static final int TIMEOUT_MS = 5000;
    public static final String BASE_URL = "https://api.example.com";
}
```

在 `Main.main` 中使用：

```java
System.out.println(Config.TIMEOUT_MS);   // 5000
```

这次有两个源文件，在它们所在的目录执行 `javac Main.java Config.java`，再执行 `java Main`。

```typescript
// TS 对比
const MAX_SIZE = 100;           // TS const
const APP_NAME: string = "MyApp";
```

💡 **对比 JS**：Java 的 `final` 类似 TS 的 `const`，但 `final` 对对象只是引用不可变，对象内部属性仍可修改（和 `const` 完全一样）。

### 2.3 作用域

作用域是“一个名字在哪些位置能用”。下面是一个完整类，保存为 `ScopeExample.java`；在 `Main.main` 中执行 `new ScopeExample().method();`。运行 `javac Main.java ScopeExample.java`、`java Main`，输出 `30`。

```java
public class ScopeExample {
    // 类字段（实例变量）
    private String name = "Alice";
    
    // 静态字段（类变量）
    private static int count = 0;
    
    public void method() {
        // 局部变量
        int x = 10;
        
        if (x > 5) {
            int y = 20;  // if 块内的局部变量
            System.out.println(x + y);
        }
        // System.out.println(y);  // 编译错误！y 超出作用域
    }
}
```

`private` 表示字段只允许类内部访问。`name` 随每个对象各自保存，`static count` 在这个类的对象之间共享；`x`、`y` 则只在各自的大括号范围内可用。此例没有修改 `name` 和 `count`，只用它们展示三种声明位置。

### 2.4 读懂和调用一个方法

方法就是可以反复调用的一段代码。将下面的方法放在 `Main` 类内、`main` 外面：

```java
public static int add(int a, int b) {
    return a + b;
}
```

从左向右读：`public` 允许外部访问；`static` 允许通过类来调用；第一个 `int` 是返回值类型；`add` 是方法名；`(int a, int b)` 是两个整数参数；`return` 把计算结果交给调用方。

在 `main` 中调用：

```java
int total = add(2, 3);
System.out.println(total); // 5
```

先把 `2`、`3` 分别交给 `a`、`b`，计算 `a + b`，再将返回的 `5` 赋给 `total`。如果返回类型写成 `void`，就不能这样用变量接收结果。

---

## 3. 运算符 {#operators}

### 3.1 算术运算符

`%` 是取余：`10 % 3` 等于 `1`，因为 `10 = 3 × 3 + 1`。`(double) a` 是把 `a` 转成小数类型再参与运算，不会改变变量 `a` 的类型。

```java
int a = 10, b = 3;

System.out.println(a + b);   // 13
System.out.println(a - b);   // 7
System.out.println(a * b);   // 30
System.out.println(a / b);   // 3  ← 整数除法，截断小数！
System.out.println(a % b);   // 1

// 浮点除法
System.out.println(10.0 / 3);    // 3.3333...
System.out.println((double) a / b);  // 3.3333... （强制转换）
```

> ⚠️ **常见坑：整数除法**

```java
int result = 5 / 2;
System.out.println(result);  // 2，不是 2.5！

// ✅ 推荐：确保至少一个操作数是浮点数
double result2 = 5.0 / 2;      // 2.5
double result3 = (double) 5 / 2; // 2.5
```

### 3.2 == vs equals() — 最重要的区别！

💡 **对比 JS**：JS 的 `===` 对字符串做值比较，Java 的 `==` 对引用类型是**引用比较**。

```java
// 基本类型：== 比较值
int x = 5, y = 5;
System.out.println(x == y);   // true ✓

// 引用类型：== 判断是否引用同一个对象
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);       // false！两个不同对象
System.out.println(s1.equals(s2));  // true ✓ 比较内容

// 字符串字面量的特殊情况（常量池）
String s3 = "hello";
String s4 = "hello";
System.out.println(s3 == s4);       // true（共享常量池中的同一对象）
```

> ⚠️ **常见坑：比较字符串内容应使用 equals()**

```java
String input = new String("admin"); // 模拟内容相同、对象不同的输入
System.out.println(input == "admin");      // false
System.out.println("admin".equals(input)); // true

String missing = null;
System.out.println("admin".equals(missing)); // false，不会抛异常
// missing.equals("admin") 会因 missing 为 null 抛出异常
```

`null` 表示没有对象，不能对它调用实例方法。`equals()` 如何比较由类决定；`String` 已实现了按字符串内容比较，自定义类见第二章。

### 3.3 逻辑运算符

```java
boolean a = true, b = false;

// 短路：左边已经能确定结果时，不计算右边
System.out.println(a && b);   // false
System.out.println(a || b);   // true
System.out.println(!a);       // false

// 短路特性
int x = 0;
if (x != 0 && 10 / x > 1) {  // x != 0 为 false，后面不执行，不会除以零
    System.out.println("never");
}
```

这里最后的 `if` 不会打印。与 JS 的差别是，Java 的 `&&`、`||` 要求两边为布尔值，结果也是布尔值；不能用它们返回字符串或对象。

### 3.4 位运算符

**可后看。** 位运算按二进制的每一位计算；`0b` 表示后面按二进制读取。初学常规业务代码时，先会用 `&&`、`||`，不要把它们与 `&`、`|` 混淆。

```java
int a = 0b1010;  // 10
int b = 0b1100;  // 12

System.out.println(a & b);   // 0b1000 = 8  (按位与)
System.out.println(a | b);   // 0b1110 = 14 (按位或)
System.out.println(a ^ b);   // 0b0110 = 6  (按位异或)
System.out.println(~a);      // -11          (按位取反)
System.out.println(a << 1);  // 20           (左移)
System.out.println(a >> 1);  // 5            (右移，保留符号)
System.out.println(a >>> 1); // 5            (无符号右移)
```

### 3.5 三元运算符

读法是“条件成立取问号后面的值，否则取冒号后面的值”。

```java
int x = 5;
String result = x > 3 ? "big" : "small";  // 和 JS 完全一样
System.out.println(result);  // "big"
```

---

## 4. 控制流程 {#control-flow}

### 4.1 if / else if / else

```java
int score = 85;

if (score >= 90) {
    System.out.println("A");
} else if (score >= 80) {
    System.out.println("B");
} else if (score >= 70) {
    System.out.println("C");
} else {
    System.out.println("D");
}
// 输出：B
```

💡 **对比 JS**：语法完全一样，但 Java 的条件**必须是 boolean**，不能像 JS 那样用 truthy/falsy。

```java
int x = 0;
// ❌ Java 不允许：if (x) { }  — 编译错误！
// ✅ Java 必须：if (x != 0) { }
```

### 4.2 switch 表达式（Java 14+）

传统 switch 语句（老写法，有 fall-through 坑）：

```java
// 老式 switch（容易写漏 break）
int day = 3;
switch (day) {
    case 1:
        System.out.println("Monday");
        break;  // ⚠️ 忘写 break 会 fall-through！
    case 2:
        System.out.println("Tuesday");
        break;
    default:
        System.out.println("Other");
}
```

> ⚠️ **常见坑：忘写 break**

```java
// ✅ 推荐：Java 14+ 的 switch 表达式（箭头语法，无 fall-through）
String dayName = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 3 -> "Wednesday";
    case 4 -> "Thursday";
    case 5 -> "Friday";
    case 6, 7 -> "Weekend";   // 多个 case 合并
    default -> "Invalid";
};
System.out.println(dayName);  // "Wednesday"
```

这段沿用前例的 `day = 3`。`switch` 选择匹配的分支，箭头右侧给出结果，最后将 `"Wednesday"` 赋给 `dayName`；这里的 `->` 是 switch 规则，不是 Lambda。

```java
// switch 也可以用于多行逻辑（用 yield 返回值）
int result = switch (day) {
    case 1, 2, 3, 4, 5 -> {
        System.out.println("Weekday");
        yield day * 10;  // 给这个 switch 表达式一个结果，不是退出整个方法
    }
    default -> 0;
};
```

仍取 `day = 3` 时，先打印 `Weekday`，再使 `result` 等于 `30`。

### 4.3 for 循环

`for (int i = 0; i < 5; i++)` 按“初始化一次 → 判断条件 → 执行循环体 → `i++` 加一 → 再判断”的顺序执行。`for (int n : numbers)` 则是“每次从数组中取一个值，命名为 `n`”。

```java
// 传统 for 循环
for (int i = 0; i < 5; i++) {
    System.out.println(i);  // 0 1 2 3 4
}

// 增强 for 循环（for-each）— 类似 JS 的 for...of
int[] numbers = {1, 2, 3, 4, 5};
for (int n : numbers) {
    System.out.println(n);
}

// 遍历集合
List<String> names = List.of("Alice", "Bob", "Charlie"); // 顶部 import java.util.List;
for (String name : names) {
    System.out.println(name);
}
```

💡 **对比 JS**：Java 的增强 for 对应 JS 的 `for...of`，不过 Java 没有 `for...in`（对象属性遍历用 Map）。

### 4.4 while / do-while

`while` 先判断，再执行；`do-while` 先执行，再判断。两段的输出分别是 `0～4` 和 `j = 0`、`j = 1`、`j = 2`。不要忘了改变循环条件中的变量，否则可能一直循环。

```java
// while
int i = 0;
while (i < 5) {
    System.out.println(i);
    i++;
}

// do-while（至少执行一次）
int j = 0;
do {
    System.out.println("j = " + j);
    j++;
} while (j < 3);
```

### 4.5 break / continue / 带标签的跳转

```java
// break：跳出最近的循环
for (int i = 0; i < 10; i++) {
    if (i == 5) break;
    System.out.print(i + " ");  // 0 1 2 3 4
}

// continue：跳过本次迭代
for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) continue;
    System.out.print(i + " ");  // 1 3 5 7 9
}

// 带标签的 break（跳出嵌套循环）— JS 也支持但很少用
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (i == 1 && j == 1) break outer;
        System.out.println(i + "," + j);
    }
}
// 输出：0,0  0,1  0,2  1,0
```

前两种先掌握即可。最后一例的 `outer:` 是给外层循环起名，`break outer` 会结束两层循环。

---

## 5. 数组 {#arrays}

### 5.1 一维数组

`int[]` 表示整数数组，`new int[5]` 表示创建 5 个位置。下标从 `0` 开始，因此第 5 个位置是 `arr[4]`；赋值不会自动扩容。

```java
// 声明并初始化
int[] nums = {1, 2, 3, 4, 5};
String[] names = {"Alice", "Bob", "Charlie"};

// 声明并分配空间（默认值为 0/false/null）
int[] arr = new int[5];  // [0, 0, 0, 0, 0]

// 访问和修改
arr[0] = 10;
arr[4] = 50;
System.out.println(arr.length);  // 5（不是 .length()，是属性）

// 遍历
for (int i = 0; i < nums.length; i++) {
    System.out.println(nums[i]);
}

// 增强 for
for (int n : nums) {
    System.out.println(n);
}
```

> ⚠️ **注意**：Java 数组长度固定！创建后无法 push/pop。需要动态大小用 `ArrayList`。

> ⚠️ **注意**：访问越界会抛出 `ArrayIndexOutOfBoundsException`。

### 5.2 二维数组

先把它理解为“数组里放数组”：`matrix[1]` 取第 2 行，`matrix[1][2]` 再取这一行的第 3 个值。嵌套循环的外层依次取行，内层依次取该行的值。

```java
// 声明二维数组（矩阵）
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

System.out.println(matrix[1][2]);  // 6（第2行第3列）
System.out.println(matrix.length);     // 3（行数）
System.out.println(matrix[0].length);  // 3（列数）

// 遍历二维数组
for (int[] row : matrix) {
    for (int val : row) {
        System.out.printf("%3d", val);
    }
    System.out.println();
}

// 不规则数组（每行长度可以不同）
int[][] jagged = new int[3][];
jagged[0] = new int[]{1};
jagged[1] = new int[]{2, 3};
jagged[2] = new int[]{4, 5, 6};
```

`printf("%3d", val)` 打印一个至少占 3 列的整数，因此三行会对齐；输出依次是 `1 2 3`、`4 5 6`、`7 8 9`。`matrix[0].length` 仅代表第 1 行长度，不能代表所有不规则数组的列数。

### 5.3 Arrays 工具类

`Arrays` 是 JDK 提供的工具类，先导入，再在 `main` 中调用。`sort(arr)` **修改原数组**；`copyOf` 和 `copyOfRange` 返回新数组，后者范围含起点、不含终点，所以索引 `[1, 4)` 对应 `2、3、4`。

```java
import java.util.Arrays;

int[] arr = {5, 3, 1, 4, 2};

// 排序
Arrays.sort(arr);
System.out.println(Arrays.toString(arr));  // [1, 2, 3, 4, 5]

// 二分查找（数组必须已排序）
int idx = Arrays.binarySearch(arr, 3);
System.out.println(idx);  // 2

// 复制数组
int[] copy = Arrays.copyOf(arr, arr.length);
int[] partial = Arrays.copyOfRange(arr, 1, 4);  // [2, 3, 4]

// 填充
int[] filled = new int[5];
Arrays.fill(filled, 7);  // [7, 7, 7, 7, 7]

// 比较
System.out.println(Arrays.equals(arr, copy));  // true
```

💡 **对比 JS**：`Arrays.sort()` 对应 JS 的 `Array.prototype.sort()`，但 JS sort 默认转字符串排序（坑！），Java sort 对数字是正确的数值排序。

---

## 6. 字符串操作 {#strings}

### 6.1 String 不可变性

```java
String s = "hello";
s.toUpperCase();        // 返回新字符串，s 本身不变
s.replace("l", "r");   // 返回新字符串，s 本身不变

System.out.println(s);  // 还是 "hello"

// ✅ 正确使用
String upper = s.toUpperCase();
String replaced = s.replace("l", "r");
```

💡 **对比 JS**：JS 的字符串也是不可变的，这一点完全一样。

### 6.2 常用字符串方法

先运行几个最常用的操作，下面都是放进 `main` 的普通语句：

```java
String text = "  Hello, World!  ";
String cleaned = text.strip(); // 返回去掉两端空白的新字符串
System.out.println(cleaned);   // Hello, World!
System.out.println(text.length()); // 17，原字符串仍有两端空格
System.out.println(cleaned.substring(7, 12)); // World
System.out.println(cleaned.contains("World")); // true
```

`text.strip()` 读作“对 text 指向的字符串调用 strip 方法”。`substring(7, 12)` 截取下标 7 到 11，**不含下标 12**。连续写 `text.strip().substring(7, 12)`，就是先用第一个方法的返回值，再调用第二个方法。

其他方法作为速查即可。表中 `text` 仍是 `"  Hello, World!  "`；想查看结果可写 `System.out.println(表中的表达式);`。

| 需求 | 表达式 | 结果或含义 |
|------|--------|------------|
| 去掉两端空白 | `text.trim()` / `text.strip()` | 本例都是 `"Hello, World!"`；`strip` 按 `Character.isWhitespace` 判断空白 |
| 转大写 / 小写 | `text.toUpperCase()` / `text.toLowerCase()` | 返回新字符串 |
| 第一次 / 最后一次找到 o | `text.indexOf("o")` / `text.lastIndexOf("o")` | `6` / `10`，未找到返回 `-1` |
| 判断开头 / 结尾 | `text.startsWith("  H")` / `text.endsWith("  ")` | 都是 `true` |
| 从下标截到末尾 | `text.strip().substring(7)` | `"World!"` |
| 普通替换 | `text.replace("World", "Java")` | `"  Hello, Java!  "` |
| 按正则替换 | `text.replaceAll("\\s+", " ")` | 连续空白合成一个空格，包括两端空白 |
| 连接多个字符串 | `String.join(", ", "a", "b", "c")` | `"a, b, c"` |
| 长度是否为零 | `text.isEmpty()` | `false`；`"".isEmpty()` 才是 `true` |
| 是否为空或全是空白 | `"  ".isBlank()` | `true` |
| 字符串转整数 | `Integer.parseInt("42")` | 整数 `42`；非法格式会抛异常 |
| 值转字符串 | `String.valueOf(42)` | 字符串 `"42"` |
| 取下标处的 char | `text.charAt(2)` | `'H'` |
| 转 char 数组 | `text.toCharArray()` | 依次包含字符串各 UTF-16 代码单元 |

分割字符串后得到的是数组。顶部加 `import java.util.Arrays;`，打印数组内容时使用 `Arrays.toString`：

```java
String csv = "a,b,c,d";
String[] parts = csv.split(",");
System.out.println(parts[1]); // b
System.out.println(Arrays.toString(parts)); // [a, b, c, d]
```

`split` 的参数也是正则表达式。例如按点号分隔需写 `split("\\.")`，不能直接把 `"."` 当成普通点号。

### 6.3 StringBuilder — 可变字符串

当需要频繁拼接字符串时，用 `StringBuilder`：

```java
// 循环中反复拼接：结果越来越长，会反复复制内容
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // 具体对象分配由编译器和运行时决定
}

// ✅ 高效写法
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
String result2 = sb.toString();

// StringBuilder 的常用方法
StringBuilder sb2 = new StringBuilder("Hello");
sb2.append(", World");          // Hello, World
sb2.insert(5, "!");             // Hello!, World
sb2.delete(5, 6);               // Hello, World
sb2.reverse();                  // dlroW ,olleH
sb2.replace(0, 5, "Java");      // 替换子串
System.out.println(sb2.length()); // 11
```

这两段都生成 `01234567891011...999`。`append(i)` 向同一个构建器追加内容，最后用 `toString()` 取出 `String`。循环中大量拼接可优先考虑 `StringBuilder`；少量 `"姓名: " + name` 很清楚，不必机械改写，Java 编译器和运行时也会优化字符串连接。

### 6.4 字符串格式化

```java
String name = "Alice";
int age = 25;
double score = 98.5;

// 方式1：String.format（类似 C 的 printf）
String msg1 = String.format("姓名: %s, 年龄: %d, 分数: %.1f", name, age, score);
// "姓名: Alice, 年龄: 25, 分数: 98.5"

// 方式2：printf（直接打印，不返回字符串）
System.out.printf("Name: %s, Age: %d%n", name, age);

// 方式3：文本块（Java 15+）
String json = """
        {
            "name": "%s",
            "age": %d
        }
        """.formatted(name, age);

// 方式4：字符串连接（简单场景）
String msg2 = "姓名: " + name + ", 年龄: " + age;
```

文本块用于书写多行字符串，**不会像 JS 模板字符串那样自动插入变量**。下面的 Java 示例用 `.formatted(name, age)` 依次填入 `%s` 和 `%d`。`%s` 表示字符串、`%d` 表示整数、`%.1f` 表示保留一位小数、`%n` 表示换行。

```javascript
// JS 模板字符串
const msg = `姓名: ${name}, 年龄: ${age}`;
```

```java
// Java 文本块（Java 15+）
String msg = """
    姓名: %s, 年龄: %d
    """.formatted(name, age);
```

---

## 7. Lambda 表达式 {#lambda}

### 7.1 基本语法

Lambda 是把“一小段待执行的行为”交给变量或其他方法。语法类似 JS 箭头函数，但 Java 用 `->`，并且需要一个约定参数和返回值的**函数式接口**作为类型。

先看最小例子，顶部导入 `java.util.function.Predicate`，在 `main` 中运行：

```java
Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4)); // true
System.out.println(isEven.test(3)); // false
```

从右侧看：`n` 是传入的数，`n % 2 == 0` 判断它是不是偶数。从左侧看：`Predicate<Integer>` 约定“接收一个 `Integer`，返回 `boolean`”，`isEven` 是变量名。**赋值只保存行为，调用 `test(4)` 时才执行判断**。这里的 `.test(...)` 是这个接口约定的方法名，不是特殊语法。

下面两种 Lambda 写法表达同一个判断，任选其一：

```java
Predicate<Integer> isEven = n -> n % 2 == 0;
// 带大括号时，返回结果要显式写 return
Predicate<Integer> isEvenWithBlock = n -> {
    return n % 2 == 0;
};
```

再对照几个 JS 写法，先看行为，不必一次记住全部接口名：

```javascript
// JS 箭头函数
const add = (a, b) => a + b;
const double = x => x * 2;
const greet = () => console.log("Hello");
```

```java
// 文件顶部：import java.util.function.BinaryOperator;
//           import java.util.function.Function;
// Java Lambda（需要配合函数式接口使用）
// 形式：(参数) -> 表达式  或  (参数) -> { 代码块; }

// 等价形式
BinaryOperator<Integer> add = (a, b) -> a + b;
Function<Integer, Integer> double_ = x -> x * 2;
Runnable greet = () -> System.out.println("Hello");

System.out.println(add.apply(2, 3)); // 5
System.out.println(double_.apply(4)); // 8
greet.run(); // Hello
```

`Runnable` 的 `run()` 没有参数也没有返回值，所以写 `()`；`Function<Integer, Integer>` 的两个类型分别是输入类型和输出类型。函数式接口只有一个需要实现的抽象方法，Lambda 提供的就是该方法的行为。接口本身会在第二章解释。

### 7.2 常用函数式接口

以下作为速查。`T`、`U`、`R` 是类型占位名，使用时换成 `String`、`Integer` 等具体引用类型；不是三个必须记住的 Java 类型。

```java
import java.util.function.*;

// Function<T, R>：接收 T，返回 R（类比 JS: x => result）
Function<String, Integer> strLen = s -> s.length();
System.out.println(strLen.apply("hello"));  // 5

// Predicate<T>：接收 T，返回 boolean（类比 JS: x => boolean）
Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4));   // true
System.out.println(isEven.test(3));   // false

// Consumer<T>：接收 T，无返回（类比 JS: x => void）
Consumer<String> printer = s -> System.out.println(s);
printer.accept("Hello!");

// Supplier<T>：无参数，返回 T（类比 JS: () => value）
Supplier<Double> random = () -> Math.random();
System.out.println(random.get());

// BiFunction<T, U, R>：接收两个参数
BiFunction<Integer, Integer, Integer> multiply = (a, b) -> a * b;
System.out.println(multiply.apply(3, 4)); // 12
```

调用名可以这样记：转换用 `apply`，判断用 `test`，接收并处理用 `accept`，获取一个值用 `get`。`random.get()` 每次调用才生成一个随机数，所以没有固定输出。

### 7.3 方法引用

**先会写 Lambda，再看这个简写。** 如果 Lambda 只是把参数交给某个现成方法，可以用 `::` 引用那个方法。`System.out::println` 相当于 `name -> System.out.println(name)`，不是现在就打印。

```java
import java.util.Arrays;
import java.util.List;
import java.util.ArrayList;
import java.util.function.Function;
import java.util.function.Supplier;

// 方法引用是 Lambda 的简写形式
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");

// Lambda 写法
names.forEach(name -> System.out.println(name));

// 方法引用写法（::）
names.forEach(System.out::println);

// 静态方法引用
Function<String, Integer> parse = Integer::parseInt;
System.out.println(parse.apply("42"));  // 42

// 实例方法引用
String prefix = "Hello, ";
Function<String, String> greet = prefix::concat;
System.out.println(greet.apply("Alice"));  // "Hello, Alice"

// 构造器引用
Supplier<ArrayList<String>> newList = ArrayList::new;
ArrayList<String> empty = newList.get(); // 此时才执行 new ArrayList<String>()
System.out.println(empty.size()); // 0
```

前两次遍历都会依次打印 `Charlie`、`Alice`、`Bob`。`Integer::parseInt` 表示已有静态方法，`prefix::concat` 表示已有对象的方法，`ArrayList::new` 表示构造器；它们仍需要合适的接口类型来确定参数和返回值。

---

## 8. 异常处理 {#exceptions}

异常表示“当前步骤无法按正常方式继续”。例如把 `"abc"` 转成整数失败，就不会凭空返回一个可用的整数，而是抛出异常。

### 8.1 异常层级结构

先记住常见的 `Exception` 和 `RuntimeException`；层级表用来查名字，不必背下来。

```text
Throwable
├── Error（通常表示运行环境等严重问题，普通业务代码一般不捕获）
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception
    ├── RuntimeException（非受检异常，编译器不强制声明或捕获）
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   └── IllegalArgumentException
    │       └── NumberFormatException
    └── 其他 Exception 子类通常是受检异常
        ├── IOException
        ├── SQLException
        └── ParseException
```

`NumberFormatException` 也是一种 `IllegalArgumentException`。捕获父类异常时，也能捕获对应子类；同一个 `try` 的多个 `catch` 必须把更具体的类型放前面。

### 8.2 try-catch-finally

把下面的普通语句放进 `main`：

```java
try {
    System.out.println("开始转换");
    int age = Integer.parseInt("abc");
    System.out.println("年龄是 " + age); // 转换失败，走不到这里
} catch (NumberFormatException e) {
    System.out.println("请输入合法整数");
} finally {
    System.out.println("本次转换结束");
}
System.out.println("继续执行后面的代码");
```

输出顺序：

```text
开始转换
请输入合法整数
本次转换结束
继续执行后面的代码
```

按流程读：`try` 尝试执行；发生异常后跳过其余语句，寻找匹配的 `catch`；变量 `e` 是这次异常对象；最后执行 `finally`。`e.getMessage()` 可读取错误说明，但具体文字可能随 JDK 改变。

`catch` 应处理自己能处理的失败；这里把非法输入转换成提示，是为了演示流程。如果无法处理，应把错误交给调用方，不能捕获后假装成功。`finally` 在正常控制流程中用于清理，但 JVM 被强制终止等情况下不保证执行。

如果多种异常要采用同一种处理方式，可写 `catch (IOException | SQLException e)`；这只是语法示意，需要对应的导入，而且 `try` 中必须确实有可能抛出这些受检异常。

### 8.3 受检异常 vs 非受检异常

差别是**编译器是否要求你交代处理方式**，不是“一个严重、一个不严重”。

| 类型 | 编译器的要求 | 常见处理 |
|------|--------------|----------|
| 受检异常，如 `IOException` | 调用方必须捕获，或者方法用 `throws` 继续声明 | 文件不存在时提示，或交由上层决定 |
| 非受检异常，如 `IllegalArgumentException` | 不强制捕获或声明 | 参数不合法时拒绝操作，定位并修正调用问题 |

下面的方法放进 `Main` 类中、`main` 外；顶部导入 `java.io.IOException`、`java.nio.file.Files` 和 `java.nio.file.Path`：

```java
public static String readFile(String path) throws IOException {
    return Files.readString(Path.of(path));
}
```

读法：方法名 `readFile`，接收路径字符串，返回文件内容字符串；`throws IOException` 告诉调用方“读取可能失败，需要处理”。它**不会立即抛异常**，实际失败时才会抛出。

在 `Main.java` 所在目录创建 `hello.txt`，写入 `Hello Java`，然后从同一目录运行程序。在 `main` 中调用：

```java
try {
    String content = readFile("hello.txt");
    System.out.println(content); // 文件内容：Hello Java
} catch (IOException e) {
    System.err.println("读取失败: " + e.getMessage());
}
```

如果文件不存在，则进入 `catch`，不打印正常结果。`System.err` 是错误输出流。也可以让上层继续处理：

```java
// 放在类中、main 外面；调用 readAndPrint 的代码也需要处理 IOException
public static void readAndPrint() throws IOException {
    System.out.println(readFile("hello.txt"));
}
```

再看非受检异常，下面的方法同样放在类中：

```java
public static int divide(int a, int b) {
    if (b == 0) {
        throw new IllegalArgumentException("除数不能为零");
    }
    return a / b;
}
```

`throw new ...` 创建并**实际抛出**异常，后面的 `return` 不再执行。`divide(10, 2)` 返回 `5`；`divide(10, 0)` 会抛出异常。若在 `main` 中不捕获，该异常会结束这次主线程执行，并输出错误堆栈。

### 8.4 自定义异常

当现成的异常类型不足以表达业务含义，可以定义自己的异常。保存为 `UserNotFoundException.java`，与 `Main.java` 放在同一目录：

```java
public class UserNotFoundException extends RuntimeException {
    private final Long userId;

    public UserNotFoundException(Long userId) {
        super("User not found: " + userId);
        this.userId = userId;
    }

    public Long getUserId() {
        return userId;
    }
}
```

先只看三件事：`extends RuntimeException` 表示它是一种非受检异常；`super(...)` 将错误说明交给父类构造器；字段 `userId` 保存额外信息。类、继承和 `this` 的细节在第二章讲解。

在 `Main` 中加入这个方法，模拟只存在用户 1，不需要数据库：

```java
public static String findUserName(long id) {
    if (id == 1L) {
        return "Alice";
    }
    throw new UserNotFoundException(id);
}
```

再在 `main` 中调用：

```java
try {
    System.out.println(findUserName(999L));
} catch (UserNotFoundException e) {
    System.out.println("用户 " + e.getUserId() + " 不存在");
}
```

在文件所在目录执行 `javac Main.java UserNotFoundException.java`、`java Main`，输出 `用户 999 不存在`。这里方法返回的是用户名，因此没有依赖尚未定义的 `User` 或 `userRepository`。

### 8.5 try-with-resources（自动关闭资源）

文件、连接等资源用完后需要关闭，垃圾回收并不能代替及时释放它们。把支持 `AutoCloseable` 的资源写在 `try (...)` 中，退出这个代码块时会按声明的逆序关闭资源，包括中途抛异常的情况。

下面是普通语句。顶部导入 `java.io.BufferedReader`、`java.io.IOException`、`java.nio.file.Files`、`java.nio.file.Path`；沿用前面的 `hello.txt`：

```java
try (BufferedReader reader = Files.newBufferedReader(Path.of("hello.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
} catch (IOException e) {
    System.err.println("读取失败: " + e.getMessage());
}
```

把 `while` 的条件拆开看：`readLine()` 读取一行；`line = ...` 保存读到的内容；`!= null` 判断是否到文件末尾。每轮打印一行；没有更多内容时结束循环，随后自动关闭 `reader`。不必再手写一层 `finally` 调用 `close()`。

---

## 自测：先猜，再运行

1. `System.out.println(5 / 2);` 与 `System.out.println(5.0 / 2);` 各输出什么？
2. `String name = "alice"; name.toUpperCase();` 执行后，`name` 是什么？
3. `Predicate<Integer> isEven = n -> n % 2 == 0;` 这一行是否已经判断过某个具体数字？
4. `throw` 与 `throws` 有什么区别？

<details>
<summary>查看答案</summary>

1. 分别是 `2` 与 `2.5`，第一个进行整数除法。
2. 仍是 `"alice"`。要保存新值，需要 `name = name.toUpperCase();`。
3. 没有，它只是定义行为；调用 `isEven.test(4)` 时才判断 `4`。
4. `throw` 实际抛出一个异常对象；方法签名中的 `throws` 声明可能向外抛出的异常类型。

</details>

## 总结对比表

| 概念 | JavaScript/TypeScript | Java |
|------|----------------------|------|
| 类型检查 | 编译期（TS）/ 运行时（JS） | 编译期 |
| 数字类型 | `number`（64位浮点） | `int`/`long`/`double` 等 |
| 字符串 | `string`，不可变 | `String`，不可变 |
| 字符串比较 | `===` | `.equals()` |
| 空值 | `null` / `undefined` | `null` |
| 常量 | `const` | `final` |
| 类型推断 | TS 自动推断 | `var`（Java 10+） |
| 错误处理 | `try-catch-finally` | `try-catch-finally`（+受检异常） |
| 函数作为值 | 箭头函数 | Lambda 表达式 |
| 数组 | 动态大小 `Array` | 固定大小，动态用 `ArrayList` |

---

*← [返回首页](./index.md) | [下一章：面向对象编程 →](./02-oop.md)*
