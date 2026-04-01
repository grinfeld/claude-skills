---
name: senior-spring-java-dev
description: Use when writing, reviewing, or designing Spring Boot / Spring Framework Java code. Triggers on Spring Boot applications, REST APIs with Spring MVC, Spring Data JPA repositories, Spring Security configuration, @Bean/@Configuration classes, Spring tests (@SpringBootTest, @WebMvcTest, @DataJpaTest), or any project that imports org.springframework. Always use this skill when the user mentions Spring, Spring Boot, Spring MVC, Spring Data, Spring Security, Spring Cloud, or asks to build a Java service/API — even if they don't say "Spring" explicitly.
---

**REQUIRED BACKGROUND:** Apply all principles from `java-senior-dev` first (which itself requires `senior-dev`). This skill adds Spring-specific patterns on top.

## Layered Architecture — Enforce the Boundary

Spring apps live or die by layer separation. Each layer has one job:

```
Controller  →  Service  →  Repository
     ↓             ↓             ↓
  HTTP/DTO      Domain        Persistence
```

Rules:
- Controllers know nothing about persistence; they speak DTOs / request/response records
- Services contain business logic; they operate on domain objects
- Repositories are interfaces — never instantiate JPA entities in a controller
- Never inject a `Repository` directly into a `Controller` — always go through a `Service`

## Configuration: Constructor Injection Only

```java
// ❌ Field injection — hides dependencies, breaks testability
@Service
public class OrderService {
    @Autowired
    private OrderRepository repo;
}

// ✅ Constructor injection — explicit, final, testable without container
@Service
public class OrderService {
    private final OrderRepository repo;
    private final PaymentGateway gateway;

    public OrderService(OrderRepository repo, PaymentGateway gateway) {
        this.repo = repo;
        this.gateway = gateway;
    }
}
```

`@Autowired` on a constructor is optional since Spring 4.3 when there's only one constructor — omit it.

## REST Controllers

```java
// ✅ Idiomatic Spring MVC controller
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor  // Lombok OK here for constructor — or write it manually
public class OrderController {

    private final OrderService orderService;

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(OrderResponse::from)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@RequestBody @Valid CreateOrderRequest request) {
        return OrderResponse.from(orderService.create(request));
    }
}
```

- Use `record` for request/response DTOs — immutable and concise
- Always validate input with `@Valid` + Bean Validation (`@NotNull`, `@Size`, etc.)
- Return `ResponseEntity<T>` when status code is conditional; use `@ResponseStatus` for fixed codes
- Version APIs in the path (`/api/v1/`) from day one

## Exception Handling

Centralize all HTTP error mapping in one place — not scattered across controllers:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(EntityNotFoundException ex) {
        return new ErrorResponse(ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return new ErrorResponse(message);
    }
}

public record ErrorResponse(String message) {}
```

Never catch exceptions in controllers just to re-wrap them into `ResponseEntity` — that's `@RestControllerAdvice`'s job.

## Spring Data JPA

```java
// ✅ Repository interface — Spring generates the implementation
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByActiveTrue();

    // Custom query when derived methods get unreadable
    @Query("SELECT u FROM User u WHERE u.createdAt > :since AND u.active = true")
    List<User> findActiveCreatedAfter(@Param("since") Instant since);
}
```

**Entity design:**

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    // Use Instant for timestamps — timezone-aware
    @CreationTimestamp
    private Instant createdAt;
}
```

- `@OneToMany` default fetch is `LAZY` — keep it that way; eager fetching is a common N+1 source
- Use `@Transactional` on the **service** layer, not the repository layer
- Use `@Transactional(readOnly = true)` for read-only operations — tells Hibernate to skip dirty checking

## `@Transactional` — Where and Why

```java
@Service
public class OrderService {

    @Transactional(readOnly = true)
    public Optional<Order> findById(Long id) {
        return orderRepository.findById(id);
    }

    @Transactional
    public Order create(CreateOrderRequest request) {
        var order = new Order(request.customerId(), request.items());
        orderRepository.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order.getId()));
        return order;
    }
}
```

- `@Transactional` belongs on services, not repositories or controllers
- Spring's proxy-based AOP means `@Transactional` **only fires on external calls** — calling a `@Transactional` method from within the same bean bypasses the proxy; extract to another bean if needed

## Configuration Properties

Prefer `@ConfigurationProperties` over `@Value` for related settings:

```java
// ✅ Typed, validated, IDE-friendly
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
    @NotBlank String apiUrl,
    @NotNull Duration timeout,
    int maxRetries
) {}

// Register it
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { ... }
```

```yaml
# application.yml
app:
  payment:
    api-url: https://pay.example.com
    timeout: 5s
    max-retries: 3
```

Use `application.yml` over `application.properties` for structured config. Use profiles (`application-prod.yml`) for environment-specific overrides.

## Spring Security

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)          // stateless APIs: disable CSRF
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

- Never store passwords in plaintext — use `BCryptPasswordEncoder`
- Prefer JWT / OAuth2 resource server over session-based auth for stateless APIs
- Method-level security (`@PreAuthorize`) for fine-grained access control on service methods

## Testing

### Slice tests — test one layer at a time

```java
// Controller test — no full context, no DB
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService orderService;  // stub the service layer

    @Test
    void returnsNotFoundWhenOrderMissing() throws Exception {
        when(orderService.findById(42L)).thenReturn(Optional.empty());

        mvc.perform(get("/api/v1/orders/42"))
            .andExpect(status().isNotFound());
    }
}

// Repository test — real DB (H2 or Testcontainers), no web layer
@DataJpaTest
class UserRepositoryTest {
    @Autowired UserRepository repo;

    @Test
    void findsUserByEmail() {
        repo.save(new User("alice@example.com"));
        assertThat(repo.findByEmail("alice@example.com")).isPresent();
    }
}
```

### Integration test — full context

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class OrderIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

Use Testcontainers for integration tests — never H2 as a substitute for Postgres in prod parity tests.

### Test slice preference

| What you're testing | Use |
|--------------------|-----|
| Controller request/response mapping | `@WebMvcTest` |
| JPA queries and mappings | `@DataJpaTest` |
| Service business logic | Plain JUnit + Mockito (no Spring context) |
| Full flow with real DB | `@SpringBootTest` + Testcontainers |

## Observability

```java
// ✅ Use Micrometer for metrics (Spring Boot Actuator auto-configures)
@Service
public class PaymentService {
    private final Counter paymentCounter;

    public PaymentService(MeterRegistry registry) {
        this.paymentCounter = Counter.builder("payments.processed")
            .tag("currency", "usd")
            .register(registry);
    }

    public void process(Payment payment) {
        paymentCounter.increment();
        // ...
    }
}
```

- Enable `/actuator/health` and `/actuator/info` — never expose full actuator to the internet
- Use structured logging (logback with JSON encoder in prod) — avoid `System.out.println`
- Propagate trace IDs via Micrometer Tracing (replaces Sleuth in Spring Boot 3)

## Spring Boot Pitfalls

| Pitfall | Fix |
|---------|-----|
| `@Autowired` field injection | Constructor injection — always |
| `@Transactional` on controller | Move transaction boundary to service |
| Calling `@Transactional` method within same bean | Extract to a separate Spring bean |
| `FetchType.EAGER` on associations | Keep `LAZY`; fetch explicitly when needed |
| `@SpringBootTest` for every test | Use slices (`@WebMvcTest`, `@DataJpaTest`) for speed |
| H2 for integration tests | Testcontainers with the real DB engine |
| Secrets in `application.yml` | Externalize via env vars or a secrets manager |
| `@Value` for groups of related properties | Use `@ConfigurationProperties` records |
| Catching and swallowing in controllers | Centralize in `@RestControllerAdvice` |
| No API versioning | Version from day one (`/api/v1/`) |
