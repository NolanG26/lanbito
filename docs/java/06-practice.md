# 实战项目与进阶路线

> 先做出能请求、能返回的小接口，再逐步加入数据库、登录和业务规则。

**本章目标**：能解释“浏览器发请求 → Controller 接收 → Service 处理 → Repository 读写 → 返回 JSON”这条链路。

- **第一次学必做**：内存版文章 CRUD、一次前后端请求、能区分 400 / 401 / 403 / 404。
- **第二轮再做**：数据库、DTO、作者权限和事务。
- **以后按需看**：评论、标签、缓存、微服务、性能工具和面试题。

运行环境沿用上一章的 **JDK 21 + Spring Boot 3.x + Maven**。本章只有“内存版文章 CRUD”给出了完整业务文件；后面的博客代码用于读懂分层，省略的类和配置会单独说明，不能直接拼成一个完整项目。

## 目录

1. [博客系统后端实战](#博客系统后端实战)
2. [前后端联调](#前后端联调)
3. [进阶学习路线](#进阶学习路线)
4. [面试高频题](#面试高频题)
5. [推荐资源](#推荐资源)

---

## 博客系统后端实战

### 先完成一个内存版文章 CRUD

先不连接 MySQL，也不加登录。沿用上一章只添加 **Spring Web** 的最小项目：启动类放在 `com.example.demo` 包，让 Spring 能扫描到下面的子包。若已经添加 JPA 或 Security，需要先使用独立的最小练习项目，否则会额外要求数据库或登录配置。

创建 `src/main/java/com/example/demo/practice/ArticlePracticeController.java`。下面是整个文件，不是写进 `main()` 的语句：

```java
package com.example.demo.practice;

import java.util.Comparator;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/practice/articles")
public class ArticlePracticeController {
    // 暂时用 Map 代替数据库：key 是文章 ID，value 是文章内容。
    private final ConcurrentHashMap<Long, Article> articles = new ConcurrentHashMap<>();
    private final AtomicLong nextId = new AtomicLong(0);

    // record 自动提供 id() / title() / content()，用于传输数据。
    public record Article(long id, String title, String content) {}
    public record WriteArticleRequest(String title, String content) {}

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Article create(@RequestBody WriteArticleRequest request) {
        checkRequest(request);
        long id = nextId.incrementAndGet();
        Article article = new Article(id, request.title(), request.content());
        articles.put(id, article);
        return article;
    }

    @GetMapping
    public List<Article> list() {
        return articles.values().stream()
            .sorted(Comparator.comparingLong(Article::id))
            .toList();
    }

    @GetMapping("/{id}")
    public Article get(@PathVariable("id") long id) {
        Article article = articles.get(id);
        if (article == null) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "文章不存在");
        }
        return article;
    }

    @PutMapping("/{id}")
    public Article update(@PathVariable("id") long id,
                          @RequestBody WriteArticleRequest request) {
        checkRequest(request);
        Article updated = articles.computeIfPresent(id, (key, oldArticle) ->
            new Article(key, request.title(), request.content()));
        if (updated == null) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "文章不存在");
        }
        return updated;
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable("id") long id) {
        if (articles.remove(id) == null) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "文章不存在");
        }
    }

    private void checkRequest(WriteArticleRequest request) {
        if (request.title() == null || request.title().isBlank()
                || request.content() == null || request.content().isBlank()) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "标题和正文不能为空");
        }
    }
}
```

按下面顺序读，不需要一次记住所有注解：

1. `@RequestMapping` 指定共同路径，`@PostMapping` 等决定 HTTP 方法。
2. `@RequestBody` 把 JSON 转成 `WriteArticleRequest`；`@PathVariable("id")` 从 URL 取 ID。
3. `new Article(...)` 只是创建 Java 对象；`articles.put(id, article)` 才把它保存进 Map。
4. 方法返回对象后，Spring Web 把对象转成 JSON。`@ResponseStatus` 决定 HTTP 状态，返回 `void` 的删除接口没有响应体。
5. `computeIfPresent` 只在 ID 已存在时替换该条目，`(key, oldArticle) -> ...` 是交给 Map 执行的函数；返回 `null` 在这里表示没有找到条目。

这里使用并发 Map 和原子 ID，是因为多个 HTTP 请求可能同时进入同一个 Controller。它们保证单次 Map 操作和 ID 生成安全，不代表多个步骤自动构成事务。列表也不是某一时刻的严格快照。练习数据保存在当前进程，**重启就丢失**；这正是下一步要引入数据库的原因。

在项目根目录启动：

```bash
./mvnw spring-boot:run
```

另开终端，按顺序请求；`-i` 用来同时查看状态码和响应体：

```bash
# 1. 创建：新启动且没有数据时，返回 201，ID 为 1。
curl -i -X POST http://localhost:8080/practice/articles \
  -H 'Content-Type: application/json' \
  -d '{"title":"第一篇文章","content":"你好 Java"}'
# 响应体：{"id":1,"title":"第一篇文章","content":"你好 Java"}

# 2. 列表：返回 200，数组中有刚才的文章。
curl -i http://localhost:8080/practice/articles

# 3. 修改：返回 200，title 变为“修改后的标题”。
curl -i -X PUT http://localhost:8080/practice/articles/1 \
  -H 'Content-Type: application/json' \
  -d '{"title":"修改后的标题","content":"你好 Java"}'

# 4. 删除：返回 204，没有响应体。
curl -i -X DELETE http://localhost:8080/practice/articles/1

# 5. 再查询：返回 404。
curl -i http://localhost:8080/practice/articles/1
```

**这一阶段通过的标准**：不看代码，也能说出五次请求分别调用哪个方法、Map 怎样变化、返回什么状态。这个练习直接返回文章 JSON；后面的正式 API 才使用上一章的 `ApiResponse<T>` 包装，二者不要混着解析。

### 分阶段扩展

| 阶段 | 这一步只解决什么 | 完成后检查什么 |
|------|----------------|----------------|
| 1. 内存 CRUD | HTTP、JSON 和 Java 对象互转 | 上面的五次请求全部符合预期 |
| 2. 拆 Service | Controller 只接收请求，业务规则移到 Service | URL 和响应不变，能沿构造器找到依赖 |
| 3. 接 MySQL / JPA | 把 Map 换成 Repository，数据持久保存 | 重启应用后文章仍能查到；不存在仍是 404 |
| 4. 加 DTO / 校验 | 明确哪些字段能传入、哪些字段能返回 | 空标题是 400；响应不含密码和多余关联 |
| 5. 加登录 / 权限 | 确定当前用户是谁，是否为作者 | 未登录 401；其他作者修改 403；本人成功 |
| 6. 加标签 / 评论 | 学会关联查询和业务规则 | 无效标签被拒绝；草稿不出现在公开列表 |

下面先展示最终目标和数据库版的核心结构。第一次阅读停在能理解 Entity / Repository / Service / DTO 的职责即可，不必同时实现表格里的全部功能。

### 需求分析

| 模块 | 功能 |
|------|------|
| 用户模块 | 注册、登录（JWT）、查看/编辑个人信息 |
| 文章模块 | 发布/编辑/删除文章、分页查询、按标签筛选、浏览量统计 |
| 评论模块 | 发表评论、回复评论（楼中楼）、删除 |
| 标签模块 | 创建/删除标签、查看文章标签 |

### 数据库设计

这是 **MySQL 8 的目标结构**，应在专用练习数据库中使用，尚未由上面的内存示例创建。`users.name` 沿用上一章的 `User.name`；`password` 存密码哈希，不存明文。只做文章阶段时先理解 `users` 和 `articles` 两张表。

- `PRIMARY KEY`：每行的唯一 ID；`AUTO_INCREMENT`：让数据库生成 ID。
- `NOT NULL`：必须有值；`DEFAULT`：未提供时使用的值。
- `articles.user_id → users.id`：文章通过作者 ID 关联用户，不把整个用户对象塞进一个列。
- `article_tags`：一篇文章可有多个标签，一个标签也可用于多篇文章，因此需要中间表。
- `ON DELETE CASCADE`：删除主记录时一并删除关联行；它不会替你完成“当前用户是否有权删除”的检查。

```sql
-- 用户表
CREATE TABLE users (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(50)  NOT NULL,
    email       VARCHAR(100) UNIQUE NOT NULL,
    password    VARCHAR(100) NOT NULL,
    avatar      VARCHAR(255),
    bio         TEXT,
    status      VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 文章表
CREATE TABLE articles (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    title       VARCHAR(200) NOT NULL,
    content     MEDIUMTEXT   NOT NULL,
    summary     VARCHAR(500),
    cover_image VARCHAR(255),
    status      VARCHAR(20)  NOT NULL DEFAULT 'DRAFT',
    view_count  INT          NOT NULL DEFAULT 0,
    user_id     BIGINT       NOT NULL,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FULLTEXT INDEX ft_title_content (title, content)  -- 全文搜索
);

-- 标签表
CREATE TABLE tags (
    id    BIGINT PRIMARY KEY AUTO_INCREMENT,
    name  VARCHAR(50) UNIQUE NOT NULL,
    color VARCHAR(20) DEFAULT '#666666'
);

-- 文章标签关联表（多对多）
CREATE TABLE article_tags (
    article_id BIGINT NOT NULL,
    tag_id     BIGINT NOT NULL,
    PRIMARY KEY (article_id, tag_id),
    FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id)     REFERENCES tags(id)     ON DELETE CASCADE
);

-- 评论表
CREATE TABLE comments (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    content    TEXT    NOT NULL,
    article_id BIGINT  NOT NULL,
    user_id    BIGINT  NOT NULL,
    parent_id  BIGINT  DEFAULT NULL,  -- NULL 表示顶级评论
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id)    REFERENCES users(id),
    FOREIGN KEY (parent_id)  REFERENCES comments(id) ON DELETE CASCADE
);
```

### API 设计

这是计划实现的接口清单，不代表下面已经提供全部实现。公开文章接口只应返回 `PUBLISHED` 状态；修改和删除除了认证，还必须校验作者。登录成功后，服务端从认证上下文取当前用户 ID，不能信任客户端随意传来的 `authorId`。

第五章的 Security 示例默认要求其他接口登录、用户管理接口为管理员；扩展到这张表时，要同步为公开的文章 GET 请求、`/api/users/me` 等设置更具体的规则，并放在较宽泛的规则之前，接口表本身不会改变访问权限。

| 方法 | 路径 | 说明 | 认证 |
|------|------|------|------|
| POST | `/api/auth/register` | 注册 | ❌ |
| POST | `/api/auth/login` | 登录，返回 JWT | ❌ |
| GET | `/api/users/me` | 获取当前用户信息 | ✅ |
| PUT | `/api/users/me` | 更新个人信息 | ✅ |
| GET | `/api/articles` | 文章列表（分页/搜索/按标签） | ❌ |
| GET | `/api/articles/{id}` | 文章详情 | ❌ |
| POST | `/api/articles` | 发布文章 | ✅ |
| PUT | `/api/articles/{id}` | 编辑文章（仅作者） | ✅ |
| DELETE | `/api/articles/{id}` | 删除文章（仅作者） | ✅ |
| GET | `/api/articles/{id}/comments` | 文章评论列表 | ❌ |
| POST | `/api/articles/{id}/comments` | 发表评论 | ✅ |
| DELETE | `/api/comments/{id}` | 删除评论（仅作者） | ✅ |
| GET | `/api/tags` | 标签列表 | ❌ |
| POST | `/api/tags` | 创建标签 | ✅ |

### 核心代码：Article 模块

以下是**数据库版分层片段**，使用 `/api/articles`；前面的内存练习使用 `/practice/articles`，请求时注意对应路径。每个 `public` 类型分别放在同名 `.java` 文件中，按第五章的 `entity` / `repository` / `service` / `dto` 分包，并补相应 `package` / `import`。

| 片段依赖 | 本章提供到什么程度 |
|----------|------------------|
| Spring Data JPA、Validation、MySQL Driver、Lombok | 在第五章相应章节逐项添加；最初的 `/hello` 项目只有 Spring Web，此处未提供完整 `pom.xml` |
| `User` / `UserRepository` | 沿用第五章，用户名字段为 `name` |
| `Tag` / `TagRepository` | 需自行定义 `id`、`name` 、JPA 实体映射，以及继承 `JpaRepository<Tag, Long>` 的接口 |
| `BusinessException` / `ResourceNotFoundException` | 沿用第五章，并确保映射为对应 HTTP 状态 |
| `ArticleController`、登录、分页参数校验 | 本章未提供完整实现；需要接到 Service，不能只粘贴下面几段就请求成功 |
| 时间审计、数据库连接 | 启用 `@EnableJpaAuditing` 并配置数据源 |

#### Entity

Entity 描述“数据库的一行怎样对应 Java 对象”。先只看 `id` / `title` / `author`，其余字段是逐步扩展的。JPA 注解来自 `jakarta.persistence`，审计注解来自 Spring Data；这里使用 Lombok 生成 Getter / Setter。

```java
@Entity
@Table(name = "articles")
@Getter @Setter @NoArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class Article {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String title;

    @Column(nullable = false, columnDefinition = "MEDIUMTEXT")
    private String content;

    @Column(length = 500)
    private String summary;

    private String coverImage;

    @Enumerated(EnumType.STRING)
    private ArticleStatus status = ArticleStatus.DRAFT;

    @Column(nullable = false)
    private Integer viewCount = 0;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User author;

    @ManyToMany
    @JoinTable(
        name = "article_tags",
        joinColumns = @JoinColumn(name = "article_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();

    @CreatedDate @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

读关联时可以把它翻译成：`author` 是一个 `User` 对象，但 `@JoinColumn(name = "user_id")` 告诉 JPA，只把作者 ID 存在 `articles.user_id` 中。`LAZY` 表示按需加载，不保证读取时一定只有一条 SQL。这里不直接返回 Entity，也不使用 `@Data` 自动遍历关联生成 `toString` / `equals`，减少循环引用和额外查询。

状态枚举单独定义为 `public enum ArticleStatus { DRAFT, PUBLISHED }`。`@Enumerated(EnumType.STRING)` 把它保存成 `DRAFT` / `PUBLISHED`，而不是枚举序号。

#### Repository

Repository 负责查询，不负责判断“某人有没有权限修改文章”。`JpaRepository<Article, Long>` 中前一个类型是实体，后一个是主键类型。先理解第一个方法名：按 `status` 筛选，再按 `createdAt` 倒序；返回的 `Page<Article>` 包含这一页数据和分页信息。

```java
@Repository
public interface ArticleRepository extends JpaRepository<Article, Long> {

    // 分页查询（已发布）
    Page<Article> findByStatusOrderByCreatedAtDesc(ArticleStatus status, Pageable pageable);

    // 按标签查询
    @Query("SELECT DISTINCT a FROM Article a JOIN a.tags t WHERE t.id = :tagId AND a.status = :status")
    Page<Article> findByTagId(@Param("tagId") Long tagId,
                              @Param("status") ArticleStatus status, Pageable pageable);

    // 关键词搜索
    @Query("SELECT a FROM Article a WHERE a.status = :status AND " +
           "(a.title LIKE %:kw% OR a.summary LIKE %:kw%)")
    Page<Article> search(@Param("kw") String keyword,
                         @Param("status") ArticleStatus status, Pageable pageable);

    // 某用户的文章
    Page<Article> findByAuthorIdAndStatus(Long authorId, ArticleStatus status, Pageable pageable);
}
```

`@Query` 这里写的是 JPQL：`Article` / `tags` / `status` 是 Java 实体和字段名，不是 SQL 表名。`:status` 通过参数传入枚举，避免把枚举与字符串字面量混比。`Pageable` 决定查第几页、每页几条；Spring Data 的页码从 **0** 开始。

#### Service

下面的业务顺序是：列表先选查询条件；详情先确认已发布；创建先找作者和标签再保存；更新先确认作者再改字段。传入的 `authorId` / `currentUserId` 必须来自服务端认证结果。`@Transactional` 为同一次业务调用建立事务，不能自动覆盖另一个线程里的代码。

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ArticleService {

    private final ArticleRepository articleRepository;
    private final TagRepository tagRepository;
    private final UserRepository userRepository;

    @Transactional(readOnly = true)
    public Page<ArticleDTO> list(int page, int size, Long tagId, String keyword) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());

        Page<Article> articles;
        if (tagId != null) {
            articles = articleRepository.findByTagId(tagId, ArticleStatus.PUBLISHED, pageable);
        } else if (keyword != null && !keyword.isBlank()) {
            articles = articleRepository.search(keyword, ArticleStatus.PUBLISHED, pageable);
        } else {
            articles = articleRepository.findByStatusOrderByCreatedAtDesc(ArticleStatus.PUBLISHED, pageable);
        }
        return articles.map(ArticleDTO::fromEntity);
    }

    @Transactional(readOnly = true)
    public ArticleDetailDTO getById(Long id) {
        Article article = articleRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("文章不存在"));

        if (article.getStatus() != ArticleStatus.PUBLISHED) {
            throw new ResourceNotFoundException("文章不存在");
        }

        return ArticleDetailDTO.fromEntity(article);
    }

    @Transactional
    public ArticleDTO create(CreateArticleRequest req, Long authorId) {
        User author = userRepository.findById(authorId)
            .orElseThrow(() -> new ResourceNotFoundException("作者不存在"));

        Set<Tag> tags = new HashSet<>();
        if (req.tagIds() != null && !req.tagIds().isEmpty()) {
            tags = new HashSet<>(tagRepository.findAllById(req.tagIds()));
            if (tags.size() != req.tagIds().size()) {
                throw new BusinessException(400, "存在无效标签");
            }
        }

        Article article = new Article();
        article.setTitle(req.title());
        article.setContent(req.content());
        article.setSummary(req.summary());
        article.setStatus(req.draft() ? ArticleStatus.DRAFT : ArticleStatus.PUBLISHED);
        article.setAuthor(author);
        article.setTags(tags);

        article = articleRepository.save(article);
        log.info("文章发布成功: id={}", article.getId());
        return ArticleDTO.fromEntity(article);
    }

    @Transactional
    public ArticleDTO update(Long id, UpdateArticleRequest req, Long currentUserId) {
        Article article = articleRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("文章不存在"));

        // 权限校验：只有作者才能修改
        if (!article.getAuthor().getId().equals(currentUserId)) {
            throw new BusinessException(403, "无权限修改此文章");
        }

        article.setTitle(req.title());
        article.setContent(req.content());
        article.setSummary(req.summary());

        return ArticleDTO.fromEntity(articleRepository.save(article));
    }
}
```

读完后用一次具体请求串起来：`POST /api/articles` 带标题和正文 → Controller 校验请求并获取当前用户 ID → `create(req, authorId)` 查作者和标签 → `save(article)` 插入文章 → `fromEntity` 选取响应字段 → Spring 转成 JSON。`save` 不会检查用户权限，权限检查仍由业务代码负责。

这里的筛选规则是“有 `tagId` 就按标签，否则按关键词”，没有实现二者同时筛选。`getById` 也没有实现浏览量累加；不要在只读事务里随手 `runAsync()` 写数据库：原事务不会跟随到异步线程，失败也不会自动反馈给当前请求。浏览量需求应独立设计计数、事务和失败处理后再接入。

#### DTO

DTO 是接口的数据结构。Entity 可能带密码、懒加载关系和数据库细节，不能直接当作所有接口的返回体。这里用 `record` 把“返回哪些字段”写清楚；每个 `public record` 单独一个文件，`fromEntity` 是我们自己写的转换方法，不是 Spring 自动提供的。

```java
// dto/ArticleDTO.java：列表只返回摘要，不带正文。
public record ArticleDTO(Long id, String title, String summary,
                         String status, String authorName, List<String> tags) {
    public static ArticleDTO fromEntity(Article article) {
        List<String> tagNames = article.getTags().stream()
            .map(Tag::getName)
            .sorted()
            .toList();
        return new ArticleDTO(article.getId(), article.getTitle(), article.getSummary(),
            article.getStatus().name(), article.getAuthor().getName(), tagNames);
    }
}
```

`Tag::getName` 相当于 `tag -> tag.getName()`；先从标签对象提取名字，再排序、收成列表。`article.getStatus().name()` 把枚举转成字符串，例如 `PUBLISHED`。映射作者和标签时可能触发 SQL，所以示例在 Service 事务内完成 DTO 转换；列表场景还要按实际 SQL 检查 N+1，不能只看 Java 代码就认为查库一次。

```java
// dto/ArticleDetailDTO.java：用组合复用列表信息，另外增加正文。
public record ArticleDetailDTO(ArticleDTO article, String content) {
    public static ArticleDetailDTO fromEntity(Article article) {
        return new ArticleDetailDTO(ArticleDTO.fromEntity(article), article.getContent());
    }
}
```

详情结果形如 `{"article":{"id":1,"title":"第一篇文章",...},"content":"你好 Java"}`。这里的 `...` 只是说明省略字段，不是实际 JSON。选择组合以后，前端通过 `detail.article.title` 取标题，不能继续假设它在 `detail.title`。

请求类型也要明确。下面是 Service 使用的两个类型，需添加 `jakarta.validation.constraints` 的导入，并在 Controller 入参上用 `@Valid` 触发：

```java
// dto/CreateArticleRequest.java
public record CreateArticleRequest(
    @NotBlank @Size(max = 200) String title,
    @NotBlank String content,
    @Size(max = 500) String summary,
    boolean draft,
    Set<Long> tagIds
) {}

// dto/UpdateArticleRequest.java：本例更新标题、正文和摘要。
public record UpdateArticleRequest(
    @NotBlank @Size(max = 200) String title,
    @NotBlank String content,
    @Size(max = 500) String summary
) {}
```

创建请求例如 `{"title":"第一篇文章","content":"你好 Java","draft":true,"tagIds":[]}`，其结果是保存草稿，公开列表不应出现它。这里未实现发布已有草稿、修改标签、删除文章和评论；按前面的分阶段表逐项补，不要把缺失部分当作已经完成。

---

## 前后端联调

### CORS 配置

先用 `curl` 确认接口本身成功，再接浏览器。前端 `localhost:3000` 请求后端 `localhost:8080` 属于跨源请求，浏览器会检查后端是否允许它。CORS 解决浏览器访问许可，不代替登录或作者权限。

下面是完整配置文件，保存为 `src/main/java/com/example/demo/config/CorsConfig.java`。若已启用 Spring Security，还要在它的 `SecurityFilterChain` 中启用 `.cors(Customizer.withDefaults())`，让安全链使用 Spring MVC 的跨源配置（`Customizer` 来自 `org.springframework.security.config`）。

```java
package com.example.demo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("Content-Type", "Authorization")
            .allowCredentials(false) // 本例只通过 Authorization 请求头传 Token
            .maxAge(3600L);
    }
}
```

`allowedOrigins` 对应浏览器地址栏里的前端来源；`allowedMethods` / `allowedHeaders` 对应请求方式和请求头。`maxAge(3600L)` 的单位是秒，表示预检结果可缓存多久。这里只配置 `/api/**`；若先联调内存练习，把映射改为 `/practice/**`，或为它再增加一条相同规则。若改用 Cookie 登录，需要重新设计凭据、CSRF 和允许来源，不能只改一个布尔值。

### 前端 Axios 封装

这是 **Vite + TypeScript + Axios** 项目的片段，后端应按第五章返回 `ApiResponse<T>`。先理解一条链：`articleApi.getById(1)` 发请求 → Axios 附加 Token → 后端返回 `{"code":200,"message":"success","data":...}` → API 函数取 `response.data.data`。

下面保留 Axios 的正常响应类型，在 API 函数里明确解包，避免拦截器改成返回业务对象后，TypeScript 还误以为它是 `AxiosResponse`。

```typescript
// src/api/request.ts
import axios, { AxiosError } from 'axios'

export const request = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080',
  timeout: 10000,
})

request.interceptors.request.use(config => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

request.interceptors.response.use(
  response => response,
  (error: AxiosError<{ message?: string }>) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      // 页面可在接收到错误后引导用户重新登录。
    }
    return Promise.reject(error) // 让调用方知道失败，不能吞成成功结果。
  },
)

// src/api/article.ts：这些响应类型与上面的 Java record 对应。
import { request } from './request'

type ApiResponse<T> = { code: number; message: string; data: T }
type Article = {
  id: number
  title: string
  summary: string | null
  status: string
  authorName: string
  tags: string[]
}
type ArticleDetail = { article: Article; content: string }
type CreateArticleRequest = {
  title: string
  content: string
  summary?: string
  draft: boolean
  tagIds?: number[]
}
type UpdateArticleRequest = Pick<CreateArticleRequest, 'title' | 'content' | 'summary'>

export const articleApi = {
  async getById(id: number): Promise<ArticleDetail> {
    const response = await request.get<ApiResponse<ArticleDetail>>(`/api/articles/${id}`)
    return response.data.data
  },
  async create(data: CreateArticleRequest): Promise<Article> {
    const response = await request.post<ApiResponse<Article>>('/api/articles', data)
    return response.data.data
  },
  async update(id: number, data: UpdateArticleRequest): Promise<Article> {
    const response = await request.put<ApiResponse<Article>>(`/api/articles/${id}`, data)
    return response.data.data
  },
  async delete(id: number): Promise<void> {
    await request.delete(`/api/articles/${id}`) // 204 没有 data，不能继续解包。
  },
}
```

调用方用 `try/catch` 显示失败信息；HTTP 401 表示需要认证，403 表示身份已确认但没有权限，404 表示资源不存在，400 表示输入不符合要求。`timeout: 10000` 是客户端等待阈值，超时不证明服务端没执行，创建失败后不要不查状态就连续重试。

本例在练习中用 `number` 表示小 ID。Java `long` 的范围大于 JavaScript 安全整数范围；正式 API 若可能返回大 ID，应约定字符串序列化并把 TypeScript 类型改成 `string`。Token 存储方式也只是演示，`localStorage` 可被页面脚本读取，实际项目需结合 XSS 防护和认证方案选择。

---

## 进阶学习路线

能独立完成基本 CRUD 后再按问题选主题，不需要把下面的名词背完才开始写接口。每学一个工具，都先准备“我现在要定位什么问题”，否则很难理解命令输出。

### 阶段六：JVM 原理（2～3 周）

| 主题 | 核心知识点 |
|------|-----------|
| 内存模型 | 堆（年轻代/老年代）、栈（方法栈/本地方法栈）、方法区/元空间、程序计数器 |
| GC 算法 | 标记清除、标记整理、复制算法；分代回收；ZGC/G1 适用场景 |
| 类加载 | 双亲委派模型、类加载器、热加载原理 |
| 性能调优 | `-Xmx`/`-Xms` 参数、GC 日志分析、内存泄漏排查 |

### 阶段七：微服务（1～2 个月）

```
Spring Cloud 生态
├── Spring Cloud Gateway     ← API 网关（路由/限流/认证）
├── Nacos                    ← 服务注册与配置中心（类似 Consul/etcd）
├── OpenFeign                ← 声明式 HTTP 客户端（服务间调用）
├── Sentinel                 ← 限流熔断（类似 Hystrix）
└── Seata                    ← 分布式事务
```

### 阶段八：分布式中间件（持续深入）

| 中间件 | 学习重点 |
|--------|---------|
| MySQL | 索引原理（B+树）、慢查询优化、主从复制、分库分表 |
| Redis | 数据结构、持久化（RDB/AOF）、集群模式、分布式锁、缓存穿透/击穿/雪崩 |
| Kafka | 消息模型、分区/副本、消费者组、消息积压处理 |
| Elasticsearch | 倒排索引、全文检索、聚合分析 |

### 阶段九：性能调优工具

下面列的是命令用途，不要求初学者现在运行。`<pid>` 要替换为目标 Java 进程 ID；`arthas-boot.jar` 需要另行准备。Arthas 的 `trace` / `watch` 在附加进程后的交互终端中执行，`jcmd` 在系统终端执行，不能把整个代码块一次粘贴成脚本。

```bash
# Arthas —— 线上 Java 诊断利器（阿里开源）
# 无需重启即可排查线上问题
java -jar arthas-boot.jar

# 常用命令
trace com.example.UserService getById  # 追踪方法耗时
watch com.example.UserService getById returnObj  # 观察返回值
jcmd <pid> GC.heap_info  # 查看堆信息（适用于本章 JDK 21 环境）
jcmd <pid> Thread.print  # 线程栈（可用于死锁排查）
```

---

## 面试高频题

### Java 基础

**Q1：`==` 和 `equals()` 的区别？**

`==` 比较基本类型的值，或判断两个引用是否指向同一个对象。`equals()` 是方法，默认实现同 `==`，但 `String`、`Integer` 等类重写了它来比较内容。

例如 `Integer a = 127; Integer b = 127;` 后 `a == b` 为 `true`，因为装箱会复用缓存对象。至少 `-128～127` 有缓存，更大的值可能也被缓存，不能依赖某个大数字比较必为 `false`。比较数值用 `equals()`；可能为 `null` 时可用 `Objects.equals(a, b)`。

---

**Q2：`String`、`StringBuilder`、`StringBuffer` 区别？**

| | `String` | `StringBuilder` | `StringBuffer` |
|-|---------|----------------|----------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | ✅（不可变天然安全） | ❌ | ✅（加锁） |
| 拼接特点 | 循环反复拼接可能产生较多临时对象 | 适合循环构建文本 | 单方法同步有开销，性能需测量 |
| 推荐场景 | 常量字符串 | 单线程字符串拼接 | 多线程字符串拼接（少见） |

---

**Q3：`ArrayList` 和 `LinkedList` 区别？**

`ArrayList` 基于数组，随机访问 O(1)，末尾增删均摊 O(1)，中间插入 O(n)。`LinkedList` 基于双向链表，随机访问 O(n)，头尾增删 O(1)。一般先考虑 `ArrayList`，内存占用更少，缓存局部性更好。

---

**Q4：什么是自动装箱/拆箱？有什么坑？**

自动装箱：`int` → `Integer`（自动调用 `Integer.valueOf()`）；拆箱：`Integer` → `int`（自动调用 `.intValue()`）。

**坑**：① 不要依赖 `Integer` 的对象缓存用 `==` 判断数值；② 对 null 的 `Integer` 拆箱会 `NullPointerException`；③ 大量循环中频繁装箱性能差，用基本类型。

---

**Q5：说说 Java 的异常体系？**

```
Throwable
├── Error（严重系统错误，不应捕获：OOM、StackOverflow）
└── Exception
    ├── 受检异常（Checked）：编译器强制处理，如 IOException、SQLException
    └── 非受检异常（Unchecked / RuntimeException）：可不处理，如 NPE、ArrayIndexOutOfBounds
```

业务异常常使用 `RuntimeException`，但要设计由哪一层处理、映射什么状态，并保留失败原因；不要为了减少 `try/catch` 就吞掉异常。

---

### 并发

**Q1：`synchronized` 和 `ReentrantLock` 的区别？**

| | `synchronized` | `ReentrantLock` |
|-|----------------|-----------------|
| 实现 | JVM 内置 | Java 代码实现 |
| 锁释放 | 自动（代码块结束或异常） | 必须手动 `unlock()`（放 finally） |
| 可中断 | ❌ | ✅ `lockInterruptibly()` |
| 超时等待 | ❌ | ✅ `tryLock(timeout)` |
| 公平锁 | ❌ | ✅ `new ReentrantLock(true)` |
| 条件变量 | 一个（`wait/notify`） | 多个（`newCondition()`） |

✅ 简单场景用 `synchronized`，需要超时/可中断/多条件时用 `ReentrantLock`。

---

**Q2：`volatile` 能保证原子性吗？**

**不能保证复合操作原子性**。对同一 `volatile` 变量的写入与后续读取建立可见性和有序性约束，不应理解成“禁止所有重排序”或给出实时到达承诺。`i++` 包含读、加一、写三步，两个线程仍可能覆盖彼此结果。计数可用 `AtomicInteger.incrementAndGet()`，多步不变量可用同一把锁保护。

---

**Q3：线程池的核心参数有哪些？**

`ThreadPoolExecutor` 的 7 个核心参数：

1. `corePoolSize`：核心线程数（通常按需创建，也可预启动；可配置核心线程超时）
2. `maximumPoolSize`：最大线程数
3. `keepAliveTime`：非核心线程的空闲存活时间
4. `unit`：时间单位
5. `workQueue`：任务队列（`LinkedBlockingQueue` / `ArrayBlockingQueue` / `SynchronousQueue`）
6. `threadFactory`：线程工厂（设置线程名等）
7. `rejectedExecutionHandler`：拒绝策略（`AbortPolicy`/`CallerRunsPolicy`/`DiscardPolicy`/`DiscardOldestPolicy`）

---

**Q4：说说 `ThreadLocal` 的使用场景和内存泄漏问题？**

`ThreadLocal` 让每个线程分别保存一个值，不会自动深拷贝对象。常用于当前线程内的用户信息、数据库连接和事务上下文；跨线程执行时不能假设它会自动传递。

**内存泄漏**：`ThreadLocalMap` 的 key 是弱引用（`ThreadLocal`），value 是强引用。`ThreadLocal` 对象被 GC 后，key 变为 null，但 value 在对应条目清理或线程结束前仍可能被 Map 持有，延长对象存活时间。**解决**：使用后调用 `threadLocal.remove()`，尤其是线程池场景（线程复用，不及时清理会污染下一个任务）。

---

**Q5：`ConcurrentHashMap` 和 `HashMap` 的区别？**

`HashMap` 不支持无同步的并发修改。`ConcurrentHashMap` 支持并发访问，但 `if (!map.containsKey(k)) map.put(k, v)` 这种多个调用组合起来仍不是原子操作，需用 `putIfAbsent` / `compute` 等。先理解单次调用与组合操作的边界，再研究 JDK 具体实现；不要凭类名断言所有场景的性能。

---

### Spring

**Q1：Spring IoC 和 DI 是什么？**

**IoC（控制反转）**：对象的创建和依赖管理由 Spring 容器负责，而非程序员手动 `new`。**DI（依赖注入）**：IoC 的实现方式，Spring 在创建对象时自动注入其依赖。✅ 推荐构造器注入（依赖明确、易测试、final 字段）。

---

**Q2：`@Transactional` 的传播行为有哪些？**

| 传播行为 | 说明 | 场景 |
|---------|------|------|
| `REQUIRED`（默认） | 有事务则加入，无则新建 | 最常用 |
| `REQUIRES_NEW` | 总是新建事务，挂起当前事务 | 日志记录（不被外层回滚影响） |
| `SUPPORTS` | 有事务则加入，无则不开启 | 只读查询 |
| `NOT_SUPPORTED` | 挂起当前事务，不使用事务执行 | — |
| `NEVER` | 必须在无事务环境执行 | — |
| `MANDATORY` | 必须在事务中执行，否则抛异常 | — |
| `NESTED` | 通常通过保存点实现局部回滚，需事务管理器支持 | 先确认 JPA / JDBC 配置能否支持 |

默认代理模式下，同一个对象内 A 直接调用 B 会绕过代理，B 声明的事务规则不会重新应用；若 A 已在事务中，B 仍运行于那个已有事务。

---

**Q3：Spring Bean 的生命周期？**

实例化 → 属性填充（依赖注入）→ `BeanNameAware`/`BeanFactoryAware` → `BeanPostProcessor.postProcessBeforeInitialization` → `@PostConstruct` / `InitializingBean.afterPropertiesSet()` → 自定义 init-method → `BeanPostProcessor.postProcessAfterInitialization` → 使用 → `@PreDestroy` / `DisposableBean.destroy()` → 销毁

---

**Q4：`@RestController` 和 `@Controller` 的区别？**

`@RestController` 包含 `@Controller` 和 `@ResponseBody` 的语义：返回值通过消息转换器写入响应体。返回对象常得到 JSON，返回 `String` 可得到普通文本，不是所有结果都必定为 JSON。普通 `@Controller` 的 `String` 返回值通常作为视图名；具体行为还取决于返回类型和注解。

---

**Q5：Spring MVC 的请求处理流程？**

以同步 REST 接口为例：

1. 请求先经过 Servlet Filter（包括配置好的 Security 过滤器），再到 `DispatcherServlet`。
2. Spring 查找目标 Controller，执行 MVC 拦截器的 `preHandle`。
3. 参数解析器读取路径、查询参数和请求体，执行相应校验，再调用 Controller。
4. Controller 调 Service，返回对象；消息转换器在处理返回值时将对象写入响应。
5. 随后进入 `postHandle`，最后在完成阶段执行 `afterCompletion`。对于 `@ResponseBody`，不能依赖 `postHandle` 再修改已经写出的响应体。

异常、异步和流式响应有各自分支，以上只是先建立常见请求的顺序感。

---

## 推荐资源

### 书籍

| 书名 | 阶段 | 推荐理由 |
|------|------|---------|
| 《Java 核心技术》卷一（卡尔） | 入门 | 内容权威详细，覆盖 Java 基础到泛型/集合/并发 |
| 《Effective Java》（布洛克） | 进阶 | 90 条最佳实践，面试必读，每条都是经验精华 |
| 《深入理解 Java 虚拟机》周志明 | 进阶 | 系统学习 JVM；结合实际 JDK 版本理解实现变化 |
| 《Java 并发编程实战》 | 进阶 | 理解并发设计基础；较早成书，虚拟线程等新特性需另补 |
| 《Spring Boot 实战》 | Spring | Spring Boot 官方推荐入门书 |

### 在线资源

| 网站 | 内容 | 推荐理由 |
|------|------|---------|
| [Baeldung.com](https://www.baeldung.com) | Spring/Java 教程 | 按具体问题查教程，留意示例使用的版本 |
| [spring.io/guides](https://spring.io/guides) | Spring 官方指南 | 每个 Guide 15 分钟，覆盖常见集成场景 |
| [LeetCode](https://leetcode.cn) | 算法练习 | 用 Java 刷题，语法熟练后做 Hot 100 |
| [CS Notes](https://cyc2018.xyz) | 面试综合 | 复习知识点，结合当前 JDK 和源码核对 |

### 开源项目（读源码）

| 项目 | 学习重点 |
|------|---------|
| [spring-petclinic](https://github.com/spring-projects/spring-petclinic) | Spring 示例项目，沿一个接口学习分层和测试 |
| [mall](https://github.com/macrozheng/mall) | 电商项目的模块划分；入门阶段先读一个具体接口的调用链 |

## 本章自测

1. **文章存入 Map 后，重启应用还在吗？** 不在；Map 是进程内存，需要持久存储才能跨重启保留。
2. **用户已登录，能改任意文章吗？** 不能；认证只确定身份，Service 还要比较文章作者和当前用户。
3. **`Page<Article>` 转成 `Page<ArticleDTO>` 会再查一页吗？** `map` 本身转换当前页对象；转换中访问懒加载关联可能额外发 SQL，需要检查实际查询。
4. **前端超时后再点一次创建，一定只保存一篇吗？** 不一定；原请求可能仍在服务端执行，要查询结果并为真实业务设计防重复机制。
5. **内存练习和数据库版的响应能用同一段 `.data.data` 解析吗？** 不能直接假设；练习直接返回文章，数据库版约定了 `ApiResponse`，先确认原始响应结构。

*← [上一章：Spring Boot](./05-spring-boot.md) | [进阶：并发编程](./04-concurrency.md) | [返回学习路线](./index.md) →*
