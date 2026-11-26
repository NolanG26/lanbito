# Spring Boot 全栈开发

> 本章先做一个能返回文字的 HTTP 接口，再看数据库、校验与鉴权各负责什么。

**先学到哪里？** 第一遍读“快速上手 → 构建 RESTful API → 参数校验 → 配置管理”，能解释一次请求怎样变成响应即可。第二遍接数据库；JWT、Redis、测试和部署在需要时再读，不必一次记住所有注解。

**示例环境与边界**：使用 JDK 21、Spring Boot 3.x（`jakarta.*` 包）。只有下面的 `/hello` 示例按给定项目配置即可独立启动；后面的用户系统是分层讲解片段，需要补齐指定文件和依赖，不能把所有代码块粘进同一个文件。每个 `public class` 单独保存为同名 `.java` 文件，方法必须放在类里。

如果 `Long`、`<T>`、`->` 仍不熟悉，可随时回看[基础语法](./01-basics.md)和[面向对象](./02-oop.md)，不需要先学完并发。

## 目录

1. [快速上手](#快速上手)
2. [构建 RESTful API](#构建-restful-api)
3. [Spring Data JPA](#spring-data-jpa)
4. [参数校验](#参数校验)
5. [Spring Security + JWT](#spring-security-jwt)
6. [配置管理](#配置管理)
7. [Redis 缓存集成](#redis-缓存集成)
8. [单元测试](#单元测试)
9. [打包与部署](#打包与部署)

---

## 快速上手

### 创建项目

访问 [https://start.spring.io](https://start.spring.io) 按以下配置：

| 选项 | 值 |
|------|----|
| Project | Maven |
| Language | Java |
| Spring Boot | 选择与本章一致的 3.x 稳定版本；生成器选项可能随时间变化 |
| Java | 21 |
| Dependencies | **第一步只选 Spring Web** |

设置 Group 为 `com.example`、Artifact 为 `demo`，Package name 为 `com.example.demo`。点击 **Generate** 下载 zip，解压后用 IntelliJ IDEA 打开。

第一步不选 JPA、MySQL 或 Security，避免还没有数据库或登录配置就启动失败。它们分别在后面用到时加入；Maven 的 `pom.xml` 相当于依赖清单，`./mvnw` 是项目自带的 Maven 启动脚本。

### 第一个接口：GET /hello

先确认生成的启动文件是 `src/main/java/com/example/demo/DemoApplication.java`：

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

再创建 `src/main/java/com/example/demo/HelloController.java`：

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(@RequestParam(name = "name", defaultValue = "Java") String name) {
        return "你好，" + name;
    }
}
```

在含 `pom.xml` 的项目根目录启动，保持这个终端运行，再开另一个终端访问：

```bash
./mvnw spring-boot:run
```

```bash
curl 'http://localhost:8080/hello?name=Alice'
# 响应正文：你好，Alice

curl 'http://localhost:8080/hello'
# 响应正文：你好，Java
```

这里发生了五件事：

1. `main` 调用 `SpringApplication.run(...)`，启动 Spring 容器和内置 Web 服务器。
2. Spring 扫描启动类所在包及其子包，发现 `@RestController`，替你创建控制器对象。
3. `@GetMapping("/hello")` 把 `GET /hello` 绑定到 `hello` 方法。
4. `@RequestParam` 从查询参数读取 `name`；没有传时使用 `"Java"`。
5. 方法返回的 `String` 直接写进响应正文。**这里返回普通文字，不是 JSON 字符串**；返回对象时，Spring Web 默认可把它转换成 JSON。

`@...` 是注解：它给框架描述这段代码的用途。业务结果仍由你写的 `return` 决定。若端口被占用，可在启动命令后加 `-Dspring-boot.run.arguments=--server.port=9090`，请求地址也改用 `9090`。

### 项目结构对比

```
Express 项目                Spring Boot 项目
├── src/
│   ├── routes/             ├── controller/     ← HTTP 路由层
│   ├── services/           ├── service/        ← 业务逻辑层
│   ├── models/             ├── entity/         ← 数据库实体
│   ├── dto/                ├── dto/            ← 请求/响应数据结构
│   ├── middleware/         ├── config/         ← 配置/拦截器
│   └── app.js              └── Application.java ← 启动入口
├── package.json            ├── pom.xml         ← 依赖管理
└── .env                    └── application.yml ← 配置文件
```

💡 **对比 NestJS**：两者都有控制器和服务层，可借助这个分工理解职责；模块注册、依赖作用域和运行机制仍各有规则。Java 代码实际放在 `src/main/java/com/example/demo/` 及其子包中，上图只对比职责，没有画完整目录。

### 启动入口

下面是前面启动类的简写，用来认注解；无需再创建第二个启动类：

```java
@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 从手动 new 到构造器注入

把“拼问候语”从控制器移到业务类，便于复用。新建 `src/main/java/com/example/demo/HelloService.java`：

```java
package com.example.demo;

import org.springframework.stereotype.Service;

@Service
public class HelloService {
    public String greet(String name) {
        return "你好，" + name;
    }
}
```

用下面内容替换刚才的 `HelloController.java`：

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    private final HelloService helloService;

    public HelloController(HelloService helloService) {
        this.helloService = helloService;
    }

    @GetMapping("/hello")
    public String hello(@RequestParam(name = "name", defaultValue = "Java") String name) {
        return helloService.greet(name);
    }
}
```

#### this.helloService 和参数 helloService 有什么区别？

`HelloService` 是类型名，`helloService` 是我们起的变量名。构造器 `HelloController(HelloService helloService)` 表示“创建控制器时，必须传给我一个服务对象”；构造器与类同名，没有返回类型。

赋值语句可以这样拆开读：

```java
this.helloService = helloService;
//   ↑                 ↑
// 当前控制器的字段    构造器收到的参数
```

它的意思是：**把传进来的服务对象，保存到当前控制器里，供其他方法使用。** 左右两边引用同一个对象，不是各创建了一份服务。

把参数换一个名字更容易看清。下面是上面构造器的等价写法，用来对照，不要再添加第二个同参数类型的构造器：

```java
public HelloController(HelloService service) {
    this.helloService = service;
}
```

`private final HelloService helloService` 声明保存服务的字段；`final` 表示赋值后不能再让这个字段指向另一个对象。构造器的参数只在构造器内可用，保存为字段后，`hello()` 才能通过 `helloService.greet(name)` 使用它。

#### Spring 替我做了哪一步？

普通 Java 中，可以在调用处自己创建并组装对象：

```java
HelloService service = new HelloService();
HelloController controller = new HelloController(service);
```

在本例的 Spring 项目中，框架负责这段创建和传入的工作：

```text
扫描到 @Service 标记的 HelloService → 创建服务对象
                  ↓
扫描到 @RestController 标记的 HelloController
                  ↓
调用控制器构造器，把服务对象传给参数 helloService
                  ↓
构造器执行 this.helloService = helloService，保存服务对象
```

前提是两个类都在扫描范围内。本例与 `DemoApplication` 位于同一个包，满足默认扫描规则；如果类放在扫描范围以外，单独加注解也不会自动注册。

**只有一个构造器时，Spring 默认用它接收依赖，可以省略 `@Autowired`。** 当前只有一个匹配的 `HelloService` 对象，因此 Spring 能确定传入谁；有多个候选对象时还需明确选择，不能只凭同名字段判断。

这就叫依赖注入（DI）：控制器需要服务，由框架提供。由 Spring 负责创建、组装对象的方式叫控制反转（IoC）。框架管理的对象也叫 **Bean**。

#### 用了 Spring，我怎么调用接口？

通过 HTTP 请求调用 Controller。刚才改了 Java 文件，若应用仍在运行，在原终端按 `Ctrl+C` 停止后重新执行 `./mvnw spring-boot:run`；本例没有配置自动重启。

再用浏览器打开 `http://localhost:8080/hello?name=Alice`，或在另一个终端执行：

```bash
curl 'http://localhost:8080/hello?name=Alice'
# 响应正文：你好，Alice
```

请求按下面的顺序流转：

```text
浏览器请求 GET /hello?name=Alice
                  ↓
Spring 根据 @GetMapping("/hello") 找到 hello() 方法
                  ↓
@RequestParam 把 Alice 交给方法参数 name
                  ↓
hello() 调用 helloService.greet(name)
                  ↓
greet() 返回“你好，Alice” → hello() 返回这个字符串
                  ↓
Spring 将字符串写入 HTTP 响应 → 浏览器收到文字
```

这里 `helloService.greet(name)` 仍是你写的普通 Java 方法调用。**Spring 负责创建对象、传入依赖、把请求交给对应方法；方法里的业务逻辑由你编写。**

这两个 Spring 组件默认是单例，通常在应用启动时创建，后续请求复用它们。访问一次 URL 不代表重新调用一次控制器构造器；每次请求的 `name` 保存在方法参数中，不要存进共享字段。

### 和 Midway 是不是差不多？

**是的，Controller 接请求、Service 处理业务、容器注入对象，这部分思路很接近。** 如果写过 Midway，可以先按下面的用途对应起来：

| 要做什么 | Midway 常见写法 | 本例 Spring 写法 |
|----------|----------------|------------------|
| 标记控制器 | `@Controller('/')` | `@RestController` |
| 声明 GET 路由 | `@Get('/hello')` | `@GetMapping("/hello")` |
| 读取查询参数 | `@Query('name')` | `@RequestParam(name = "name")` |
| 把服务交给容器管理 | `@Provide()` | `@Service` |
| 让容器提供服务对象 | 属性上的 `@Inject()` | 构造器参数；只有一个构造器时可省略 `@Autowired` |
| 调用业务方法 | `this.helloService.greet(name)` | `helloService.greet(name)`；也能显式写 `this.helloService.greet(name)` |

例如，在已经配置好的 Midway Web 项目中，可以用下面两个文件对照前面的 Spring 示例；它们不是可以直接用 `node` 运行的独立脚本：

```typescript
// src/service/hello.ts
import { Provide } from '@midwayjs/core';

@Provide()
export class HelloService {
  greet(name: string): string {
    return `你好，${name}`;
  }
}
```

```typescript
// src/controller/hello.ts
import { Controller, Get, Inject, Query } from '@midwayjs/core';
import { HelloService } from '../service/hello';

@Controller('/')
export class HelloController {
  @Inject()
  helloService: HelloService;

  @Get('/hello')
  async hello(@Query('name') name: string): Promise<string> {
    return this.helloService.greet(name || 'Java');
  }
}
```

对于这个例子，两边都把 `GET /hello?name=Alice` 交给控制器，再调用服务，返回 `你好，Alice`。两边都不需要在处理每次请求时手动 `new HelloService()`。

**一个需要注意的区别是对象生命周期：**按 Midway 官方文档的默认配置，Controller 和 `@Provide()` 服务是请求作用域，同一请求链路复用对象，新请求会创建新的实例；本章的 Spring Controller 和 Service 默认是单例，多次请求复用同一个对象。项目可另行配置服务作用域，判断共享状态是否安全时应核对实际配置。

上面对照的是用途：Midway 使用 TypeScript 装饰器，本例 Spring 使用 Java 注解和构造器注入，Midway 也有版本相关的注入方式。异步处理和具体配置仍要分别理解，不能把 Midway 的 `async`/`await` 直接搬进 Java。可对照 [Midway 路由和控制器](https://midwayjs.org/docs/controller)与[依赖注入](https://midwayjs.org/docs/container)继续查阅。

### 核心注解速查表

| 注解 | 作用 | 对应 Express/NestJS |
|------|------|-------------------|
| `@RestController` | 方法返回值写入响应正文，具体格式取决于类型与转换器 | `@Controller()` + `res.json()` |
| `@RequestMapping` | 设置路由前缀 | `router = express.Router()` |
| `@GetMapping` / `@PostMapping` | HTTP 方法路由 | `router.get()` / `router.post()` |
| `@Service` | 标记为服务类（业务逻辑） | `@Injectable()` |
| `@Repository` | 标记为数据访问类 | — |
| `@Component` | 通用 Spring 组件 | `@Injectable()` |
| `@Autowired` | 指定注入点，也可标在构造器上；单构造器通常可省略 | 通过框架提供依赖 |
| `@Value("${key}")` | 注入配置值 | `process.env.KEY` |
| `@Configuration` | 配置类 | — |
| `@Bean` | 在配置类中声明 Bean | — |
| `@Transactional` | 让同一事务的数据库操作一起提交或回滚 | — |
| `@Slf4j` (Lombok) | 注入日志 | `console.log` |

---

## 构建 RESTful API

### 完整 Controller 示例

从这里开始是**用户管理模块的结构示例**，不再属于只依赖 Spring Web 的 `/hello` 小程序。先读路由与调用关系，再回来看每个注解。

| 请求 | 输入从哪里来 | 调用哪个业务方法 | 正常结果 |
|------|-------------|-----------------|----------|
| `GET /api/users/1` | 路径中的 `1` | `getById(1L)` | HTTP 200，用户数据 |
| `GET /api/users?page=0&size=10` | `?` 后的参数 | `list(0, 10, null)` | HTTP 200，第 1 页 |
| `POST /api/users` | JSON 正文 | `create(req)` | HTTP 201，新用户 |
| `PUT /api/users/1` | 路径和 JSON 正文 | `update(1L, req)` | HTTP 200，更新后的用户 |
| `DELETE /api/users/1` | 路径中的 `1` | `delete(1L)` | HTTP 204，无响应正文 |

下例省略 `package`、`import`；需要后文的 `UserService`、`ApiResponse`、`UserDTO`、`CreateUserRequest`、`UpdateUserRequest`。Web 注解来自 `org.springframework.web.bind.annotation`，`@Valid` 来自 `jakarta.validation`；Lombok 需要依赖及注解处理支持，见[面向对象的 Lombok 章节](./02-oop.md#lombok)。

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor  // Lombok：自动生成构造器注入
@Slf4j
public class UserController {

    private final UserService userService;  // ✅ 构造器注入（推荐）

    // GET /api/users?page=0&size=10&keyword=alice
    @GetMapping
    public ApiResponse<Page<UserDTO>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false) String keyword) {
        log.debug("查询用户列表, page={}, size={}", page, size);
        Page<UserDTO> result = userService.list(page, size, keyword);
        return ApiResponse.success(result);
    }

    // GET /api/users/1
    @GetMapping("/{id}")
    public ApiResponse<UserDTO> getById(@PathVariable Long id) {
        return ApiResponse.success(userService.getById(id));
    }

    // POST /api/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)  // 返回 201
    public ApiResponse<UserDTO> create(@Valid @RequestBody CreateUserRequest req) {
        return ApiResponse.created(userService.create(req));
    }

    // PUT /api/users/1
    @PutMapping("/{id}")
    public ApiResponse<UserDTO> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest req) {
        return ApiResponse.success(userService.update(id, req));
    }

    // DELETE /api/users/1
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)  // 返回 204
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

读 `getById` 这一条就能理解其他路由：`@PathVariable Long id` 把路径里的数字交给参数；`userService.getById(id)` 返回一个用户 DTO；`ApiResponse.success(...)` 再包一层响应。`ApiResponse<Page<UserDTO>>` 从里往外读是“用户数据 → 一页用户数据 → 统一响应”。`Page` 来自 Spring Data，页码从 `0` 开始。

控制器只负责接收 HTTP 输入和返回结果；注册邮箱是否重复、密码怎样保存，放在 Service。`@RequiredArgsConstructor` 是 Lombok 对前面手写构造器的简写，为 `final` 字段生成参数。

### 统一响应格式

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(200, "success", data);
    }

    public static <T> ApiResponse<T> created(T data) {
        return new ApiResponse<>(201, "created", data);
    }

    public static ApiResponse<Void> error(int code, String message) {
        return new ApiResponse<>(code, message, null);
    }
}
```

例如 `ApiResponse.success("Alice")` 对应的 JSON 是：

```json
{"code": 200, "message": "success", "data": "Alice"}
```

`<T>` 是数据类型占位符：传字符串时 `T` 是 `String`，传用户时是 `UserDTO`。`Void` 表示这里没有业务数据。**JSON 里的 `code` 不会自动设置 HTTP 状态码**：201/204 由控制器的 `@ResponseStatus` 设置，异常时由下面的 `ResponseEntity` 设置。

### 全局异常处理

```java
@RestControllerAdvice  // 全局异常拦截
@Slf4j
public class GlobalExceptionHandler {
    // 业务异常
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<Void>> handleBusiness(BusinessException e) {
        return ResponseEntity.status(e.getCode())
            .body(ApiResponse.error(e.getCode(), e.getMessage()));
    }

    // 资源不存在
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiResponse<Void> handleNotFound(ResourceNotFoundException e) {
        return ApiResponse.error(404, e.getMessage());
    }

    // 参数校验失败
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Void> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ApiResponse.error(400, message);
    }
}

// 自定义业务异常
@Getter
public class BusinessException extends RuntimeException {
    private final int code;

    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
    }
}

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
```

`@ExceptionHandler` 按异常类型选择方法，例如业务层抛出 `new BusinessException(409, "邮箱已被注册")`，处理器返回 **HTTP 409** 和对应的错误正文。`ResourceNotFoundException` 返回 404；校验失败返回 400。`ResponseEntity` 位于 `org.springframework.http`，这里把业务异常的整数约定为 HTTP 错误状态，勿混入自定义业务编号。

校验消息那段 Stream 可以读成：取出字段错误 → 每个错误变成 `"字段名: 原因"` → 用分号拼成字符串。这里只处理这三种异常，JSON 格式错误、参数类型错误等由框架默认处理；没有用一个兜底处理器把所有失败都改成 500。若项目要求完全统一的错误正文，应再分别处理框架异常。

`GlobalExceptionHandler`、`BusinessException`、`ResourceNotFoundException` 是三个文件，注解来自 Spring MVC；`MethodArgumentNotValidException` 位于 `org.springframework.web.bind`。

---

## Spring Data JPA

JPA 负责把 Java 对象映射到数据库记录；Spring Data JPA 在它上面提供常用查询接口。先给项目加入 **Spring Data JPA、MySQL Driver、Lombok、Validation** 依赖，并准备本地数据库与连接配置（见“配置管理”），否则以下片段不能独立启动。

本节调用链是 `UserController → UserService → UserRepository → MySQL`。写一个 `GET /api/users/1` 时，不是浏览器直接操作 Repository。

### Entity 实体类

实体描述“数据库一行长什么样”。先看 `id/name/email` 三个字段，再看状态和关联。JPA 注解来自 `jakarta.persistence`，审计注解来自 `org.springframework.data.annotation`。

```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_email", columnList = "email", unique = true)
})
@Getter @Setter  // 不自动生成包含密码和关联对象的 toString/equals
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)  // 监听审计事件
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 50)
    private String name;

    @Column(unique = true, nullable = false, length = 100)
    private String email;

    @Column(nullable = false)
    private String password;  // 存储密码哈希，不存明文；哈希不是可解密的密文

    @Enumerated(EnumType.STRING)  // ✅ 存字符串而非数字
    @Column(nullable = false)
    @Builder.Default
    private UserStatus status = UserStatus.ACTIVE;

    @CreatedDate                  // 自动填充创建时间
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate             // 自动填充更新时间
    private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    @Builder.Default
    private List<Article> articles = new ArrayList<>();
}
```

在启动类上加 `@EnableJpaAuditing` 开启时间自动填充。还需单独定义 `enum UserStatus { ACTIVE, DISABLED }`。`Article` 是下一章的文章实体；若只练用户接口，先去掉 `articles` 关联字段及其注解，避免依赖尚未创建的类。

把注解翻译成普通话：`@Entity` 表示“持久化对象”，`@Table` 指定表名，`@Id` 是主键，`@GeneratedValue` 让数据库生成 ID。`@Enumerated(STRING)` 把 `ACTIVE` 这样的名称存入表。`@OneToMany(mappedBy = "author")` 表示一位用户可以有多篇文章，关系由文章的 `author` 字段维护。

`LAZY` 表示关联数据可能在访问时才查询；它不意味着任何时间都能读到关联，离开事务后访问未加载的属性可能失败。接口返回专门的 DTO，避免直接把包含密码、双向关联的实体序列化。

### Repository 接口

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // 方法名即查询！Spring 自动生成 SQL
    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    List<User> findByStatus(UserStatus status);

    List<User> findByStatusOrderByCreatedAtDesc(UserStatus status);

    // 分页查询
    Page<User> findByNameContaining(String keyword, Pageable pageable);

    // 自定义 JPQL
    @Query("SELECT u FROM User u WHERE u.name LIKE %:keyword% OR u.email LIKE %:keyword%")
    Page<User> search(@Param("keyword") String keyword, Pageable pageable);

    // 原生 SQL
    @Query(value = "SELECT * FROM users WHERE created_at >= :date", nativeQuery = true)
    List<User> findRecentUsers(@Param("date") LocalDateTime date);

    // 更新操作（需要 @Modifying + @Transactional）
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") UserStatus status);
}
```

`JpaRepository<User, Long>` 的两个类型参数分别是“实体类型”和“主键类型”。继承它就获得 `findById/save/deleteById` 等方法；Spring 在运行时创建实现对象，不需要你手写实现类。

`findByNameContaining("Al", pageable)` 可理解为查询名字包含 `Al` 的一页用户。`@Query` 的 JPQL 写实体名 `User` 和字段名 `createdAt`；`nativeQuery = true` 才写真实表名 `users` 和列名 `created_at`。最后的 `updateStatus` 返回影响行数，不是更新后的用户对象。

### 从代码理解事务

**事务就是把几次数据库操作当成一件事：要么全部提交，要么失败时回滚已经做的修改。** “提交”表示最终保存这些修改，“回滚”表示撤销这个事务中的数据库修改。

比如 A 给 B 转 100 元，需要完成两次数据库修改：A 扣 100 元，B 加 100 元。下面是放在 Spring 管理的 Service 中的方法片段：

```java
@Transactional
public void transfer(long fromId, long toId, BigDecimal amount) {
    // 第一步：修改数据库，让 A 的余额减少
    accountRepository.decreaseBalance(fromId, amount);

    // 第二步：修改数据库，让 B 的余额增加
    accountRepository.increaseBalance(toId, amount);
}
```

`@Transactional` 来自 `org.springframework.transaction.annotation`；`BigDecimal` 来自 `java.math`，这里表示转账金额。`fromId` 是转出账户 ID，`toId` 是转入账户 ID。

这是事务结构示例，不能单独复制运行：假设 Service 已通过构造器注入 `AccountRepository accountRepository`，且它的两个方法都执行数据库更新、参与当前同一个事务。具体实现没有展开；它们需要检查账户、金额、余额和更新结果，失败时抛出异常，不能把“没有更新到记录”也当作成功。完整转账还需要处理并发余额竞争，事务本身不代替这些业务检查。

先按最常见的情况理解：Controller 通过 Spring 注入的 Service 调用 `transfer()`，外层没有已有事务：

```text
Controller 调用 transfer()
          ↓
Spring 开启数据库事务
          ↓
A 扣款 → B 入账
          ↓
方法正常结束 → Spring 尝试提交 → 提交成功，两笔修改最终保存

如果 B 入账时抛出 RuntimeException，并传播出 transfer()：
Spring 回滚事务 → A 的扣款也被撤销
```

**第一步执行成功，不代表第一步已经单独提交。** 对比一下结果：

| 情况 | 没有把两步放进同一事务，且每步各自提交 | 两步在同一事务中 |
|------|--------------------------------------|------------------|
| 两步执行成功，提交也成功 | A 扣钱，B 收到钱 | A 扣钱，B 收到钱 |
| A 扣款成功，B 入账抛出运行时异常 | A 已扣钱，B 没收到 | 回滚，A 的扣款也被撤销 |

所以这段代码中的 `@Transactional` 可以先读成：“Spring，请把这个方法里的数据库操作放在一个事务里。” Spring 负责开启事务、提交或回滚，你负责写数据库操作和业务失败的判断。

如果调用方已经开启了事务，`@Transactional` 默认会加入它，由外层决定最终提交；不能仅凭这个方法返回，就认为修改已经提交。JPA 的 `save()`、执行 SQL、把修改刷新到数据库，也都不等于最终提交；提交本身仍可能失败。

#### 哪些失败会回滚？

- **默认规则**：`RuntimeException` 或 `Error` 传播到 Spring 的事务代理时触发回滚。代理可以先理解为“Spring 包在方法外面、负责事务处理的一层调用”。
- **受检异常**：例如 `IOException`，默认不因它回滚。需要时可把注解写成 `@Transactional(rollbackFor = IOException.class)`；希望包含所有 `Exception` 子类时可用 `rollbackFor = Exception.class`，仍要按业务选择。
- **捕获异常后不再抛出**：Spring 可能看到方法正常返回而尝试提交，不能认为打印了错误就会回滚；某些底层错误也可能已经把事务标记为只能回滚。
- **返回 `false` 或影响行数为 `0`**：这些返回值本身不会触发回滚。若它们表示业务失败，需要由代码判断并抛出相应异常。

#### 注解生效和回滚的范围

本章使用 Spring 默认的代理方式。应像前面的 Controller 一样，通过 Spring 注入的 Service 对象调用事务方法。自己 `new` 一个 Service，或同一个对象内部用 `this.transfer(...)` 调用，不会触发这个方法的事务代理；如果调用方已经有事务，内部操作仍可能参与那个事务。

**回滚的是参与同一个事务的数据库修改，不是把整个 Java 程序倒放。** 已经发出的短信、HTTP 请求，以及日志、内存对象和缓存修改，不会因为数据库回滚就自动撤销。

接下来的 Service 示例中，`@Transactional(readOnly = true)` 表示这段事务以查询为主，向框架和数据库表达只读意图；它不是访问权限控制，也不保证一定更快。普通的 `@Transactional` 则用于创建、修改、删除等业务操作。

### Service 层

先只读 `getById`：数据库查 ID → 没找到则抛异常 → 找到后转 DTO。这比一次记住整个类更容易。`PasswordEncoder` 的 Bean 见 Security 配置；暂时不做注册功能时，可先只保留查询相关方法和 `UserRepository` 依赖。

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Transactional(readOnly = true)  // 声明只读意图；不是权限控制，也不保证一定更快
    public Page<UserDTO> list(int page, int size, String keyword) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        Page<User> users = keyword != null && !keyword.isBlank()
            ? userRepository.search(keyword, pageable)
            : userRepository.findAll(pageable);
        return users.map(this::toDTO);
    }

    @Transactional(readOnly = true)
    public UserDTO getById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户 " + id + " 不存在"));
        return toDTO(user);
    }

    @Transactional  // 让这次业务操作在同一事务中完成
    public UserDTO create(CreateUserRequest req) {
        if (userRepository.existsByEmail(req.getEmail())) {
            throw new BusinessException(409, "邮箱已被注册");
        }
        User user = User.builder()
            .name(req.getName())
            .email(req.getEmail())
            .password(passwordEncoder.encode(req.getPassword()))
            .build();
        user = userRepository.save(user);
        log.info("用户注册成功: id={}", user.getId());
        return toDTO(user);
    }

    @Transactional
    public UserDTO update(Long id, UpdateUserRequest req) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户 " + id + " 不存在"));
        user.setName(req.getName());
        return toDTO(userRepository.save(user));
    }

    @Transactional
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("用户 " + id + " 不存在");
        }
        userRepository.deleteById(id);
    }

    private UserDTO toDTO(User user) {
        return UserDTO.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .status(user.getStatus())
            .createdAt(user.getCreatedAt())
            .build();
    }
}
```

分页参数目前尚未增加边界校验；实际接口要拒绝负页码、限制单页大小，避免非法输入变成业务异常或超大查询。这里的 `this::toDTO` 等价于 `user -> toDTO(user)`，把一页中的每位用户转换成响应数据。`orElseThrow(() -> new ...)` 的意思是“查不到时才创建并抛出异常”。

下例是 `UserDTO.java`，只保留可返回给客户端的字段，故意不含 `password`：

```java
@Data @Builder
public class UserDTO {
    private Long id;
    private String name;
    private String email;
    private UserStatus status;
    private LocalDateTime createdAt;
}
```

注册时的“先查邮箱再保存”能给常见重复请求友好提示，但两个并发请求仍可能一起通过检查；数据库唯一约束才是最后保障。这里尚未将唯一约束冲突转换为统一的 409；实际项目要补对应异常处理，不能宣称已解决并发注册问题。

### 解决 N+1 查询问题

```java
// 可能的 N+1：先查 100 个 User，再逐个加载未初始化的 articles
List<User> users = userRepository.findAll();
users.forEach(u -> System.out.println(u.getArticles().size())); // 未缓存且未批量加载时可能逐个查询

// 方案1：在 UserRepository 中声明需要一起加载的关联
@EntityGraph(attributePaths = {"articles"})
@Query("SELECT u FROM User u")
List<User> findAllWithArticles();

// 方案2：JOIN FETCH（与上面二选一，不能重复声明同名方法）
@Query("SELECT DISTINCT u FROM User u LEFT JOIN FETCH u.articles")
List<User> findAllWithArticles();
```

“N+1”指 1 次主查询加上 N 次关联查询，实际次数还受缓存和批量加载配置影响。两种方法都表达“取用户时也加载文章”；`EntityGraph` 的实际 SQL 由 JPA 实现决定，需看 SQL 验证。示例首两行放在事务内的业务方法中，后面的接口声明放 Repository。集合关联与分页组合时还可能造成行膨胀或内存分页，不应直接套到所有列表。

---

## 参数校验

```java
// 先在 pom.xml 添加 spring-boot-starter-validation
// Spring Web 本身不等于已经包含 Validation
// 约束注解：jakarta.validation.constraints.*

@Data
public class CreateUserRequest {

    @NotBlank(message = "姓名不能为空")
    @Size(min = 2, max = 50, message = "姓名长度须在 2-50 之间")
    private String name;

    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;

    @NotBlank(message = "密码不能为空")
    @Size(min = 8, max = 100, message = "密码长度至少 8 位")
    private String password;
}
```

更新请求目前只允许改姓名，另存为 `UpdateUserRequest.java`：

```java
@Data
public class UpdateUserRequest {
    @NotBlank(message = "姓名不能为空")
    @Size(min = 2, max = 50, message = "姓名长度须在 2-50 之间")
    private String name;
}
```

`@RequestBody` 先把 JSON 转为请求对象，`@Valid` 再检查字段约束，通过后才进入业务方法。例如向 `POST /api/users` 发送 `{"name":"Bob","email":"not-an-email","password":"password123"}`，会得到 HTTP 400，`create()` 不会执行；这里的错误正文由前面的校验异常处理器生成。

`@NotBlank` 检查空白字符串，`@Email` 检查格式，`@Size` 检查字符长度。校验邮箱格式并不证明邮箱真实存在，也不代替数据库唯一约束。示例密码长度仅演示校验语法；后面使用 BCrypt 时还需按其限制校验 UTF-8 字节数（最多 72 字节），不能把字符数直接当成字节数。

---

## Spring Security + JWT

**进阶，先跳过也可以。** Spring Security 在控制器之前检查请求；JWT 是一种签名令牌格式，不是加密保险箱，客户端可以读取载荷。本节仅展示“已获得令牌后，如何校验并建立身份”，**没有实现登录、注册、令牌撤销或完整权限系统**。

阅读顺序：登录成功签发令牌 → 客户端通过 `Authorization: Bearer ...` 发送 → 过滤器验证签名/过期时间 → 加载用户 → 写入当前请求的身份 → 权限规则决定是否放行。不能把前端传来的用户 ID 直接当作已验证身份。

### pom.xml 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

这里只用固定版本说明三件套必须匹配，不代表这个版本永远是最新推荐。`jjwt-api` 用来编译；`impl` 和 `jackson` 在运行时提供实现与 JSON 支持。只有 API 包通常无法完成真实签发/解析。

### JWT 工具类

```java
@Component
public class JwtUtil {

    private final SecretKey key;
    private final long expiration;

    public JwtUtil(@Value("${jwt.secret}") String secret,
                   @Value("${jwt.expiration:86400000}") long expiration) {
        // 在创建组件时检查配置，配置错误应让启动失败
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
        if (expiration <= 0) {
            throw new IllegalArgumentException("jwt.expiration 必须大于 0");
        }
        this.expiration = expiration;
    }

    public String generateToken(String email) {
        return Jwts.builder()
            .subject(email)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(key)
            .compact();
    }

    public String extractEmail(String token) {
        String email = getClaims(token).getSubject();
        if (email == null || email.isBlank()) {
            throw new JwtException("令牌缺少 subject");
        }
        return email;
    }

    private Claims getClaims(String token) {
        return Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}
```

`subject` 保存当前例子中的邮箱，`issuedAt` 是签发时间，`expiration` 是到期时间；`compact()` 生成最终字符串。`parseSignedClaims` 会检查签名和过期时间，失败时抛出 JJWT 异常，不能把“能解码”当成“验证通过”。

上面的 `Jwts/Claims` 来自 `io.jsonwebtoken`，`Keys` 来自 `io.jsonwebtoken.security`，`Decoders` 来自 `io.jsonwebtoken.io`；`Date` 来自 `java.util`，`SecretKey` 来自 `javax.crypto`。构造器在启动时解码密钥、检查有效期，配置错误不会被误报为某个用户的登录失败。配置中的 `jwt.secret` 要提供 **Base64 编码的随机 HMAC 密钥**，解码后至少 32 字节；不要使用示例短口令。`jwt.expiration` 的单位是毫秒，`86400000` 为 24 小时。

### JWT 过滤器

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String header = req.getHeader("Authorization");

        if (header != null && header.startsWith("Bearer ")) {
            try {
                String token = header.substring(7);  // 去掉 "Bearer " 前缀
                if (token.isBlank()) {
                    res.sendError(HttpServletResponse.SC_UNAUTHORIZED);
                    return;
                }
                String email = jwtUtil.extractEmail(token);  // 解析时同时验证令牌
                UserDetails user = userDetailsService.loadUserByUsername(email);
                if (!user.isEnabled() || !user.isAccountNonLocked()
                        || !user.isAccountNonExpired() || !user.isCredentialsNonExpired()) {
                    res.sendError(HttpServletResponse.SC_UNAUTHORIZED);
                    return;
                }
                var auth = new UsernamePasswordAuthenticationToken(
                    user, null, user.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException | UsernameNotFoundException e) {
                SecurityContextHolder.clearContext();
                res.sendError(HttpServletResponse.SC_UNAUTHORIZED);
                return;  // 失败后不要继续进入控制器
            }
        }

        chain.doFilter(req, res);
    }
}
```

`substring(7)` 的 `7` 来自前缀 `"Bearer "` 的长度。`SecurityContextHolder` 保存当前请求经过验证的身份；`chain.doFilter` 把请求交给下一环。验证失败提前返回 401，不能继续访问控制器。

此类还依赖你实现的 `UserDetailsService`：根据邮箱查用户，把数据库状态和权限转换为 `UserDetails`。上面的 `isEnabled` 等检查依赖该实现正确映射账号状态。Servlet 类型来自 `jakarta.servlet`，`OncePerRequestFilter` 来自 `org.springframework.web.filter`。这里没有登录端点，单独复制过滤器并不会得到一个能登录的系统。

### Security 配置

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .exceptionHandling(errors -> errors
                .authenticationEntryPoint((req, res, e) -> res.sendError(401))
                .accessDeniedHandler((req, res, e) -> res.sendError(403)))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()  // 放行登录注册
                .requestMatchers("/hello", "/api/public/**").permitAll()
                .requestMatchers("/api/users/**").hasRole("ADMIN")
                .anyRequest().authenticated()                 // 其他需要认证
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

`SecurityFilterChain` 是一组按顺序执行的安全规则，不是控制器。这里把用户管理限定为管理员，`UserDetailsService` 必须正确提供 `ROLE_ADMIN` 才能通过；`authenticated()` 只表示已登录，不会自动实现“只能编辑自己的文章”，资源归属仍要在业务层检查。

此例关闭 CSRF 的前提是客户端显式用 Authorization 请求头传令牌，服务端不依赖浏览器自动附带的认证 Cookie。使用 Cookie 登录时要重新设计 CSRF 防护。过滤器标了 `@Component`，实际接入还需避免它同时被 Servlet 容器和 Security 重复注册（可用禁用自动注册的 `FilterRegistrationBean`）；省略了这部分集成代码。

401 表示缺少有效身份，403 表示已有身份但没有权限。`sendError` 的错误正文不等同于上面的 `ApiResponse`，若前端依赖统一格式，还需实现相应 JSON 错误响应。

---

## 配置管理

配置文件放在 `src/main/resources/`。下面是**三个独立文件**；每块前的注释就是文件名，不是把它们拼成同一个 YAML。`dev/prod` 是环境配置名称，不是 Java 关键字。只有接入数据库或 JWT 后，才需要加入对应配置。

```yaml
# application.yml
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}  # 默认 dev，可用环境变量覆盖
  jpa:
    open-in-view: false  # 关闭 OSIV，避免懒加载在视图层触发查询
    show-sql: false

jwt:
  secret: ${JWT_SECRET}  # 必须由环境注入 Base64 密钥，不提供公共默认值
  expiration: 86400000
```

```yaml
# application-dev.yml
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://localhost:3306/mydb_dev?serverTimezone=Asia/Shanghai
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: update  # 开发环境自动建表
    show-sql: true
```

```yaml
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # 生产环境只验证，不修改表结构
```

`${DB_PASSWORD}` 的意思是读取环境变量，缺少时启动会报错；`${SPRING_PROFILES_ACTIVE:dev}` 表示没提供时默认用 `dev`。`ddl-auto: update` 仅用于可丢弃的本地练习库，不是数据库迁移方案；`validate` 只校验现有结构，需要你先建表。

### @ConfigurationProperties（强类型配置）

```java
@ConfigurationProperties(prefix = "app")
@Data
@Component
public class AppProperties {
    private String name = "MyApp";
    private Upload upload = new Upload();

    @Data
    public static class Upload {
        private String path = "/tmp/uploads";
        private long maxSize = 10 * 1024 * 1024L;  // 10 MiB
    }
}
```

```yaml
app:
  name: 博客系统
  upload:
    path: /data/uploads
    max-size: 52428800  # 50 MiB
```

`@ConfigurationProperties(prefix = "app")` 把 `app.*` 配置绑定到对象字段；`max-size` 对应 Java 的 `maxSize`。例如 YAML 的 `52428800` 会覆盖默认值，这里的单位是字节（50 MiB）。内部 `Upload` 类把上传相关配置归在一起；`static` 表示创建它不必先创建外部对象。

绑定通常发生在启动阶段，不代表修改 YAML 会自动刷新运行中的对象。先用这种写法读懂配置，再考虑动态配置。

---

## Redis 缓存集成

**可选进阶。** 缓存把重复查询的结果暂存起来，减少查数据库；它引入了“数据库和缓存可能不一致”的新问题。先确保无缓存的接口正确，再添加这一层。本节假设已有本地 Redis、用户模块、DTO 的序列化配置，以及下面的依赖；本教程没有替你启动 Redis。

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
    redis:
      time-to-live: 5m  # 明确单位：5 分钟
```

```java
@SpringBootApplication
@EnableCaching  // 开启缓存
public class Application { ... }

@Service
@RequiredArgsConstructor
public class UserService {

    // 其余构造器依赖、toDTO 等沿用前文，不能作为独立类编译
    private final UserRepository userRepository;

    @Transactional(readOnly = true)
    @Cacheable(value = "users", key = "#id")  // 查询时缓存
    public UserDTO getById(Long id) {
        // 缓存未命中时执行；命中时由代理直接返回缓存值
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户 " + id + " 不存在"));
        return toDTO(user);
    }

    @Transactional
    @CacheEvict(value = "users", key = "#id")  // 删除时清除缓存
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("用户 " + id + " 不存在");
        }
        userRepository.deleteById(id);
    }

    @Transactional
    @CachePut(value = "users", key = "#result.id")  // 更新时刷新缓存
    public UserDTO update(Long id, UpdateUserRequest req) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户 " + id + " 不存在"));
        user.setName(req.getName());
        return toDTO(userRepository.save(user));
    }
}
```

以 `getById(1L)` 为例：缓存没有 `users::1` → 执行方法查数据库 → 保存返回的 DTO；再次调用时若缓存未过期，代理直接返回它。`@CacheEvict` 在正常删除后清理对应项，`@CachePut` 无论是否命中都执行更新，并将新返回值写回缓存。没有找到用户时，仍沿用前面的异常与 HTTP 404 约定。

`#id` 和 `#result.id` 是 Spring 表达式：分别取参数 ID 与返回对象 ID。缓存代码仍需前面的 `toDTO` 和更新方法；省略的 `Application { ... }` 只表示在现有启动类上追加 `@EnableCaching`，不是合法的完整 Java 类。默认序列化未必适合 DTO，应明确配置并验证，不能仅加注解就声称可用。

这些注解通常同样通过代理生效；类内直接互调可能绕过缓存。上例只说明基本读写顺序，没有解决并发更新、事务回滚或缓存雪崩等问题。

---

## 单元测试

先分清测的是什么：普通单元测试只验证一个类；`MockMvc` 的 Web 层测试还验证路由、参数转换和 JSON。`@SpringBootTest` 会加载整个应用，通常不是最小单元测试。

先为最前面的问候业务写一个无需启动服务器的测试，保存为 `src/test/java/com/example/demo/HelloServiceTest.java`（生成项目默认的 `spring-boot-starter-test` 提供 JUnit）：

```java
package com.example.demo;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;

class HelloServiceTest {
    @Test
    void greet_shouldIncludeName() {
        HelloService service = new HelloService();
        String actual = service.greet("Alice");
        assertEquals("你好，Alice", actual);
    }
}
```

读法是“准备对象 → 用固定输入调用 → 断言实际结果等于预期”。测试失败会显示预期值和实际值，不能用打印输出代替断言。

下面再测试最初的 `/hello` 请求，保存为同目录的 `HelloControllerTest.java`。用独立的 MockMvc，不启动 Spring 容器或 Security 过滤器：

```java
package com.example.demo;

import java.nio.charset.StandardCharsets;
import org.junit.jupiter.api.Test;
import org.springframework.http.converter.StringHttpMessageConverter;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

class HelloControllerTest {
    @Test
    void hello_shouldReturnGreeting() throws Exception {
        MockMvc mockMvc = MockMvcBuilders
            .standaloneSetup(new HelloController(new HelloService()))
            .setMessageConverters(new StringHttpMessageConverter(StandardCharsets.UTF_8))
            .build();

        mockMvc.perform(get("/hello").param("name", "Alice"))
            .andExpect(status().isOk())
            .andExpect(content().string("你好，Alice"));
    }
}
```

`perform` 模拟请求，`andExpect` 检查状态码和正文。独立 MockMvc 不加载 Boot 的自动配置，这里显式使用 UTF-8 字符串转换器，避免中文响应按其他编码写出。这里用的是**完成构造器注入改造后的控制器**；若你保留第一版无依赖控制器，改用 `new HelloController()`。

在项目根目录执行：

```bash
./mvnw test
```

预期两个测试通过。这个结果只证明问候业务和控制器在测试中符合预期，**没有验证真实端口、数据库、JWT 或生产配置**。后续用户业务可以用 Mockito 替换 `UserRepository` 再测 Service；需要鉴权的 Web 测试必须明确提供测试身份，不能凭一个未实现的 `getTestToken()` 跳过认证。

---

## 打包与部署

### 打包为可执行 Jar

这是操作命令说明，不代表教程已替你打包或部署。先在本地项目确认测试通过，再生成 Jar：

```bash
# 编译、运行测试并打包
./mvnw clean package

# 运行
java -jar target/myapp-0.0.1-SNAPSHOT.jar

# 指定环境
java -jar target/myapp.jar --spring.profiles.active=prod

# 指定端口
java -jar target/myapp.jar --server.port=9090

# 设置 JVM 初始/最大堆内存，并非通用最优值
java -Xms256m -Xmx512m -jar target/myapp.jar
```

`target/` 是 Maven 输出目录，Jar 名称由项目名称和版本决定，替换示例文件名后再执行。`java -jar` 启动已有 Jar；`--server.port=9090` 是应用参数，`-Xmx512m` 是 JVM 参数，位置不同。`-DskipTests` 会跳过执行测试，不应把这种打包成功称为测试通过。

### Dockerfile

```dockerfile
# 多阶段构建，减小镜像体积
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline  # 缓存依赖层
COPY src ./src
RUN mvn clean package

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENV SPRING_PROFILES_ACTIVE=prod
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

```bash
# 构建镜像
docker build -t myapp:latest .

# 运行容器
docker run -d \
  -p 8080:8080 \
  -e DB_URL \
  -e DB_USERNAME \
  -e DB_PASSWORD \
  -e JWT_SECRET \
  --name myapp \
  myapp:latest
```

第一阶段用 Maven 镜像编译，第二阶段只带 JRE 和 Jar 运行；`COPY --from=builder` 把编译结果复制过去。`-p 8080:8080` 把宿主机端口映射到容器端口。运行前需在当前终端准备这些环境变量，`-e NAME` 才会传入它们；不要把真实密码写在文档或命令历史里。容器中的数据库地址、Redis 地址必须从容器网络可达，`localhost` 通常指容器自己。

此处未提供生产部署编排、网络、数据库初始化和密钥管理；镜像构建成功也不代表这些连接已通过验证。

## 学完后自测

1. `@RestController` 会自动帮你查询数据库吗？
2. `ApiResponse` 里的 `code=404` 能否单独让 HTTP 返回 404？
3. `UserRepository` 接口没有实现类，谁提供实现？
4. 为什么不直接把 `User` 实体返回给前端？
5. 给方法加 `@Cacheable` 就一定会命中 Redis 吗？

::: details 参考答案

1. 不会。它把方法映射到请求、把结果写入正文；查询要由你调用 Service/Repository。
2. 不能。要设置真实响应状态，例如 `ResponseEntity.status(404)` 或 `@ResponseStatus`。
3. Spring Data 在运行时创建代理实现，前提是实体、数据源和扫描配置正确。
4. 实体可能包含密码和懒加载/双向关联；DTO 明确限定对外字段。
5. 不一定。还要启用缓存、配置 Redis 与序列化，并从代理调用；未命中时会执行方法。

:::

---

*← [前置：集合与 Stream](./03-collections.md) | [下一章：实战项目](./06-practice.md) →*
