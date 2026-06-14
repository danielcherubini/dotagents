# Java Code Review Guide

Java review focus: Java 21/25 features, Spring Boot 4 best practices, concurrency (virtual threads), JPA performance optimization, and code maintainability.

## Table of Contents

- [Modern Java Features (21/25+)](#modern-java-features-2125)
- [Stream API & Optional](#stream-api--optional)
- [Spring Boot Best Practices](#spring-boot-best-practices)
- [JPA & Database Performance](#jpa--database-performance)
- [Concurrency & Virtual Threads](#concurrency--virtual-threads)
- [Lombok Usage Guidelines](#lombok-usage-guidelines)
- [Exception Handling](#exception-handling)
- [Testing Guidelines](#testing-guidelines)
- [Review Checklist](#review-checklist)

---

## Modern Java Features (21/25+)

### Records

```java
// ❌ Traditional POJO/DTO: Lots of boilerplate
public class UserDto {
    private final String name;
    private final int age;

    public UserDto(String name, int age) {
        this.name = name;
        this.age = age;
    }
    // getters, equals, hashCode, toString...
}

// ✅ Use Record: Concise, immutable, semantically clear
public record UserDto(String name, int age) {
    // Compact constructor for validation
    public UserDto {
        if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
    }
}
```

### Switch Expressions and Pattern Matching

```java
// ❌ Traditional switch: Easy to miss break, verbose and error-prone
String type = "";
switch (obj) {
    case Integer i: // Java 16+
        type = String.format("int %d", i);
        break;
    case String s:
        type = String.format("string %s", s);
        break;
    default:
        type = "unknown";
}

// ✅ Switch expression: No fall-through risk, requires return value
String type = switch (obj) {
    case Integer i -> "int %d".formatted(i);
    case String s  -> "string %s".formatted(s);
    case null      -> "null value"; // Java 21 handles null
    default        -> "unknown";
};

// ✅ Pattern matching with when clauses (Java 21+, finalized in Java 25)
// When clauses allow complex conditions directly in the pattern
String result = switch (obj) {
    case Integer i when i > 0 -> "positive integer";
    case Integer i when i < 0 -> "negative integer";
    case Integer i           -> "zero";
    case String s when !s.isBlank() -> "non-empty string: " + s;
    case null, default -> "unknown";
};
```

### Primitive Types in Pattern Matching (Java 25)

JEP 507 allows `instanceof` and `switch` patterns to match primitive types directly — no more autoboxing!

```java
// ❌ Autoboxing required for primitive type checks
public boolean isPositive(Object value) {
    if (value instanceof Integer i) {
        return i > 0; // auto-unboxes, but creates Integer object
    }
    return false;
}

// ✅ Primitive type patterns (Java 25): direct matching without boxing
public boolean isPositive(Object value) {
    return switch (value) {
        case int i when i > 0 -> true;   // matches int directly, no boxing!
        default                -> false;
    };
}

// ✅ Primitive patterns in switch with type narrowing
int compute(Object input) {
    return switch (input) {
        case byte b  -> b;
        case short s -> s;
        case int i   -> i;
        case long l  -> (int) l;
        default      -> throw new IllegalArgumentException("Not a number");
    };
}
```

### Module Import Declarations (Java 25 Preview)

JEP 511 introduces `module-import` for cleaner module declarations:

```java
// ❌ Traditional module-info.java: Verbose requires + exports
module com.example.app {
    requires java.logging;
    requires java.sql;
    exports com.example.api;
}

// ✅ Module imports (Java 25 Preview): concise, auto-resolves transitive deps
module com.example.app {
    import java.logging;   // Auto-requires + re-exports transitively
    import java.sql;
    exports com.example.api;
}
```

> 💡 This is a preview feature. Use `--enable-preview` to compile and test it.

### Text Blocks

```java
// ❌ Concatenating SQL/JSON strings
String json = "{\n" +
              "  \"name\": \"Alice\",\n" +
              "  \"age\": 20\n" +
              "}";

// ✅ Text blocks: What you see is what you get
String json = """
    {
      "name": "Alice",
      "age": 20
    }
    """;

// ✅ Text blocks with .formatted() for dynamic content (Java 25+)
String sql = """
    INSERT INTO users (name, email)
    VALUES (%s, %s)
    """.formatted(name, email);
```

---

## Stream API & Optional

### Avoid Stream Abuse

```java
// ❌ Simple loop doesn't need Stream (performance overhead + poor readability)
items.stream().forEach(item -> {
    process(item);
});

// ✅ Just use for-each for simple cases
for (var item : items) {
    process(item);
}

// ❌ Extremely complex Stream chains
List<Dto> result = list.stream()
    .filter(...)
    .map(...)
    .peek(...)
    .sorted(...)
    .collect(...); // Hard to debug

// ✅ Split into meaningful steps
var filtered = list.stream().filter(...).toList();
// ...
```

### Optional Correct Usage

```java
// ❌ Using Optional as parameter or field (serialization issues, adds complexity)
public void process(Optional<String> name) { ... }
public class User {
    private Optional<String> email; // Not recommended
}

// ✅ Optional only for return values
public Optional<User> findUser(String id) { ... }

// ❌ Using isPresent() + get() when you already have Optional
Optional<User> userOpt = findUser(id);
if (userOpt.isPresent()) {
    return userOpt.get().getName();
} else {
    return "Unknown";
}

// ✅ Use functional API
return findUser(id)
    .map(User::getName)
    .orElse("Unknown");
```

---

## Spring Boot Best Practices

### Dependency Injection (DI)

```java
// ❌ Field injection (@Autowired)
// Drawbacks: Harder to test (needs reflection), hides excessive dependencies, poor immutability
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo;
}

// ✅ Constructor injection
// Benefits: Dependencies clear, easy unit testing (Mock), fields can be final
@Service
public class UserService {
    private final UserRepository userRepo;

    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
// 💡 Tip: Combined with Lombok @RequiredArgsConstructor for simpler code, but watch for circular dependencies
```

### REST API Versioning (Spring Boot 4 / Spring Framework 7)

Spring Boot 4 introduces first-class support for REST API versioning via `@RestControllerVersioned`:

```java
// ✅ Spring Boot 4: Built-in API versioning with @RestControllerVersioned
@RestControllerVersioned(path = "/api/users", versions = {"v1", "v2"})
public class UserController {

    // Methods in v1 controller
    @GetMapping("/users/{id}")
    public UserDto getUserV1(@PathVariable Long id) { ... }
}

// ✅ Versioning strategies supported:
// - URI path:   /api/users/v1/...
// - Header:     Accept-Version: v2
// - Parameter:  /api/users?version=v2
// - Media type: Accept: application/vnd.app.v2+json
```

### Configuration Management

```java
// ❌ Hardcoded configuration values
@Service
public class PaymentService {
    private String apiKey = "sk_live_12345";
}

// ❌ Using @Value scattered throughout code
@Value("${app.payment.api-key}")
private String apiKey;

// ✅ Use @ConfigurationProperties for type-safe configuration
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(String apiKey, int timeout, String url) {}
```

### Jackson 3 Migration (Spring Boot 4)

Spring Boot 4 defaults to **Jackson 3** (`tools.jackson.*` namespace). Review annotations and configurations:

```java
// ❌ Jackson 2 annotations (still works but deprecated in SB 4)
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.databind.ObjectMapper;

// ✅ Jackson 3 annotations (Spring Boot 4 default)
import tools.jackson.annotation.JsonProperty;
import tools.jackson.databind.json.JsonMapper;

// ✅ @JsonComponent renamed to @JacksonComponent (SB 4)
// @JsonComponent → @JacksonComponent
// @JsonMixin → @JacksonMixin
// ObjectMapper → JsonMapper
```

> 💡 Use `spring-boot-properties-migrator` during migration to catch property name changes. Set `spring.jackson.use-jackson2-defaults=true` temporarily if needed.

---

## JPA & Database Performance

### N+1 Query Problem

```java
// ❌ FetchType.EAGER or lazy loading in loops
// Entity definition
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER) // Dangerous!
    private List<Order> orders;
}

// Business code
List<User> users = userRepo.findAll(); // 1 SQL
for (User user : users) {
    // If Lazy, this triggers N SQL queries
    System.out.println(user.getOrders().size());
}

// ✅ Use @EntityGraph or JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### Transaction Management

```java
// ❌ Opening transactions at Controller layer (database connection held too long)
// ❌ Adding @Transactional to private methods (AOP doesn't take effect)
@Transactional
private void saveInternal() { ... }

// ✅ Add @Transactional on Service layer public methods
// ✅ Mark read operations explicitly with readOnly = true (performance optimization)
@Service
public class UserService {
    @Transactional(readOnly = true)
    public User getUser(Long id) { ... }

    @Transactional
    public void createUser(UserDto dto) { ... }
}
```

### Entity Design

```java
// ❌ Using Lombok @Data on Entity
// @Data generates equals/hashCode containing all fields, may trigger lazy loading causing performance issues or exceptions
@Entity
@Data
public class User { ... }

// ✅ Use only @Getter, @Setter
// ✅ Custom equals/hashCode (usually based on ID)
@Entity
@Getter
@Setter
public class User {
    @Id
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return id != null && id.equals(((User) o).id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

---

## Concurrency & Virtual Threads

### Virtual Threads (Java 21+)

```java
// ❌ Traditional thread pool for large I/O blocking tasks (resource exhaustion)
ExecutorService executor = Executors.newFixedThreadPool(100);

// ✅ Use virtual threads for I/O-intensive tasks (high throughput)
// Spring Boot 3.2+ / 4.x: Enable with spring.threads.virtual.enabled=true
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// In virtual threads, blocking operations (DB queries, HTTP requests) consume almost no OS thread resources
```

### Virtual Threads Caveats

| Pitfall | Solution |
|---------|----------|
| `synchronized` blocks pin virtual threads | Use `ReentrantLock` instead of `synchronized` |
| CPU-bound work in virtual threads | Use platform threads for CPU-heavy tasks |
| Thread-local state issues | Avoid heavy ThreadLocal usage with virtual threads |
| Deadlock detection harder | Use structured concurrency or explicit lock ordering |

```java
// ❌ Bad: synchronized pins the virtual thread to an OS thread
synchronized(lock) {
    blockingIOCall();  // Pins thread!
}

// ✅ Good: ReentrantLock allows unparking
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    blockingIOCall();
} finally {
    lock.unlock();
}
```

### Thread Safety

```java
// ❌ SimpleDateFormat is not thread-safe
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// ✅ Use DateTimeFormatter (Java 8+)
private static final DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");

// ❌ HashMap may cause infinite loops or data loss in multi-threaded environments
// ✅ Use ConcurrentHashMap
Map<String, String> cache = new ConcurrentHashMap<>();
```

---

## Lombok Usage Guidelines

```java
// ❌ Overusing @Builder makes it impossible to enforce required field validation
@Builder
public class Order {
    private String id; // Required
    private String note; // Optional
}
// Caller might forget id: Order.builder().note("hi").build();

// ✅ For critical business objects, consider manually writing Builder or constructor to ensure invariants
// Or add validation logic in build() method (Lombok @Builder.Default, etc.)
```

---

## Exception Handling

### Global Exception Handling

```java
// ❌ Try-catch everywhere swallowing exceptions or just printing logs
try {
    userService.create(user);
} catch (Exception e) {
    e.printStackTrace(); // Should not be used in production
    // return null; // Swallows exception, upper layers don't know what happened
}

// ✅ Custom exceptions + @ControllerAdvice (Spring Boot 4 ProblemDetail)
public class UserNotFoundException extends RuntimeException { ... }

@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail handleNotFound(UserNotFoundException e) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
    }
}
```

---

## Testing Guidelines

### Unit Tests vs Integration Tests

```java
// ❌ Unit tests depending on real database or external services
@SpringBootTest // Starts entire Context, slow
public class UserServiceTest { ... }

// ✅ Unit tests use Mockito
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service;

    @Test
    void shouldCreateUser() { ... }
}

// ✅ Integration tests use Testcontainers
@Testcontainers
@SpringBootTest
class UserRepositoryTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    // ...
}
```

---

## Review Checklist

### Basics and Conventions
- [ ] Follows Java 21/25 features (Switch expressions, Records, text blocks, primitive type patterns)
- [ ] Avoids deprecated classes (Date, Calendar, SimpleDateFormat)
- [ ] Stream API used appropriately (not overused for simple loops)?
- [ ] Optional used only for return values, not for fields or parameters

### Spring Boot 4 Specific
- [ ] Constructor injection used instead of @Autowired field injection
- [ ] Configuration properties use @ConfigurationProperties
- [ ] REST API versioning uses @RestControllerVersioned (Spring Boot 4)
- [ ] Jackson 3 annotations (`tools.jackson.*`) used instead of Jackson 2 (`com.fasterxml.jackson.*`)
- [ ] Correct Spring Boot 4 starters used (`spring-boot-starter-webmvc` not `spring-boot-starter-web`)
- [ ] Undertow NOT used (removed in Spring Boot 4; use Tomcat/Jetty/Reactor Netty)
- [ ] Controller responsibilities are single, business logic pushed to Service
- [ ] Global exception handling uses @ControllerAdvice / ProblemDetail

### Database & Transactions
- [ ] Read operation transactions marked with `@Transactional(readOnly = true)`
- [ ] Checked for N+1 queries (EAGER fetch or loop calls)
- [ ] Entity classes don't use @Data, correctly implemented equals/hashCode
- [ ] Database indexes cover query conditions

### Concurrency and Performance
- [ ] I/O-intensive tasks considering virtual threads?
- [ ] Thread-safe classes used correctly (ConcurrentHashMap vs HashMap)?
- [ ] Lock granularity reasonable? Avoiding I/O operations within locks

### Maintainability
- [ ] Critical business logic has sufficient unit tests
- [ ] Logging appropriate (use Slf4j, avoid System.out)
- [ ] Magic values extracted to constants or enums
