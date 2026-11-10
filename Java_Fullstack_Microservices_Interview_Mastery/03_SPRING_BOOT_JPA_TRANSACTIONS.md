# Section 3: Spring Framework & Spring Boot Core (Q81 – Q98)

---

### Q81: What is the difference between IoC and DI?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Inversion of Control (IoC):** A broad architectural design principle in software engineering where the control of program flow, object creation, and lifecycle management is transferred (inverted) from the application code to an underlying framework or container.
* **Dependency Injection (DI):** A **specific design pattern** that implements IoC. Instead of a class instantiating its dependencies directly using `new ServiceImpl()`, the dependencies are "injected" into the class from the outside by the IoC container at runtime.
* **Under the Hood in Spring:** The Spring IoC container (ApplicationContext) reads bean configuration metadata (annotations or XML), instantiates beans using reflection or CGLIB proxies, resolves dependencies, and injects them into target components via constructors or setters before managing their entire lifecycle.

#### 2. Production Code Example
```java
// Without DI / Without IoC: Tightly coupled, hard to test
public class OrderServiceWithoutDI {
    private PaymentGateway paymentGateway = new StripePaymentGateway(); // Direct instantiation!
}

// With DI / With IoC: Loosely coupled, testable, Spring-managed
@Service
public class OrderService {
    private final PaymentGateway paymentGateway;

    // Dependency Injection via Constructor
    @Autowired
    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "IoC is the overarching design principle where control over object creation and lifecycle is inverted from the developer's code to the framework. Dependency Injection is the specific mechanism Spring uses to implement IoC by providing dependent objects to a class from the outside, typically via constructor injection, enabling loose coupling and easy unit testing."
* **In Our Project / Production Context:** "In our microservices, DI allows us to inject mock repositories and payment clients in JUnit tests without modifying the business service classes."
* **Pro-Tip & Interview Trap:** "Don't say 'IoC and DI are the same thing'. Be precise: IoC is the architectural principle ('Hollywood Principle: Don't call us, we'll call you'); DI is the concrete pattern that implements it."

---

### Q82: What is the difference between BeanFactory and ApplicationContext?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | `BeanFactory` (Basic Container) | `ApplicationContext` (Advanced Enterprise Container) |
| :--- | :--- | :--- |
| **Hierarchy** | Root interface for Spring IoC (`org.springframework.beans.factory`). | Sub-interface extending `BeanFactory` (`org.springframework.context`). |
| **Bean Instantiation** | **Lazy by default:** Instantiates beans on-demand when `getBean()` is called. | **Eager by default:** Pre-instantiates all singleton beans at application startup. |
| **Enterprise Features** | Minimalist; basic DI and lifecycle management. | Rich enterprise features: AOP integration, MessageSource (i18n), Event Publishing, WebApplicationContext. |
| **Annotation Support** | Requires manual registration of `BeanPostProcessor`s. | Automatically detects and registers all annotation processors (`@Autowired`, `@Value`). |
| **Usage** | Resource-constrained legacy devices / small applets. | **Standard container for all modern Spring and Spring Boot apps.** |

#### 2. Production Code Example
```java
// 1. BeanFactory (Legacy / Minimal)
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// Beans instantiated lazily only when requested

// 2. ApplicationContext (Production Spring Boot Standard)
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        // ConfigurableApplicationContext is created and eagerly initializes all singletons
        ConfigurableApplicationContext context = SpringApplication.run(Application.class, args);
        
        // Event Publishing capability of ApplicationContext:
        context.publishEvent(new OrderCreatedEvent("ORD-101"));
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`BeanFactory` is Spring's fundamental IoC container that instantiates beans lazily on demand. `ApplicationContext` is a feature-rich sub-interface extending `BeanFactory` that pre-instantiates all singleton beans eagerly at startup and adds enterprise capabilities like AOP integration, internationalization, and application event publishing. Spring Boot exclusively uses `ApplicationContext`."
* **In Our Project / Production Context:** "Eager initialization in `ApplicationContext` ensures that any configuration errors, missing beans, or cyclic dependencies fail fast during container startup rather than failing at runtime when a customer hits an endpoint."
* **Pro-Tip & Interview Trap:** "Remember that `ApplicationContext` catches misconfigured beans at startup because it is eager. If you use `BeanFactory`, a misconfigured bean will remain hidden until that specific bean is invoked at runtime!"

---

### Q83: What are the different bean scopes in Spring?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Spring Bean Scopes (`@Scope`):**
  1. **`singleton` (Default):** Exactly **one shared bean instance** per Spring IoC Container.
  2. **`prototype`:** A **new bean instance** is created every time the bean is requested from the container or injected.
  3. **`request` (Web-aware):** A single bean instance per HTTP request lifecycle. Destroyed when request completes.
  4. **`session` (Web-aware):** A single bean instance per HTTP Session lifecycle. Destroyed when session invalidates.
  5. **`application` (Web-aware):** A single bean instance per `ServletContext` lifecycle (similar to singleton, but scoped to ServletContext).
  6. **`websocket` (Web-aware):** A single bean instance per WebSocket session lifecycle.

#### 2. Production Code Example
```java
@Component
@Scope("singleton") // Default: One instance shared across all threads
public class GlobalConfigCache { /* ... */ }

@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) // New instance every injection
public class ReportGeneratorTask { /* Stateful per task */ }

@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserRequestContext {
    private String traceId;
    private String jwtToken;
    // Stateful per HTTP request
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Spring provides six bean scopes: singleton (the default, one instance per IoC container) and prototype (a new instance per injection). In web-aware applications, it adds request (per HTTP request), session (per user session), application (per ServletContext), and websocket scopes. In production, 95% of our service and repository beans are singletons."
* **In Our Project / Production Context:** "We use request-scoped beans with scoped proxies to capture incoming HTTP headers, user tenant IDs, and distributed trace IDs so they can be injected into down-stack service classes without polluting method signatures."
* **Pro-Tip & Interview Trap:** "A famous interview question: 'What happens when you inject a prototype bean into a singleton bean?' The prototype bean is **injected only once** when the singleton is instantiated! To get a new prototype instance every time, you must use `@Lookup` method injection or an `ObjectProvider<T>`."

---

### Q84: What is the difference between Singleton and Prototype scope?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Feature | Singleton Scope (Default) | Prototype Scope |
| :--- | :--- | :--- |
| **Instance Count** | **One single instance** per Spring IoC Container. | A **new instance created every time** requested or injected. |
| **Instantiation Timing** | Created eagerly at application startup (by `ApplicationContext`). | Created lazily upon request or injection. |
| **Statefulness** | **Must be stateless!** Shared across concurrent threads. | Can be stateful; isolated to the requester. |
| **Lifecycle Destruction** | Fully managed: Spring calls `@PreDestroy` / `DisposableBean`. | **Spring does NOT manage destruction!** Spring creates it and lets client/GC manage it. |
| **Memory Footprint** | Extremely low; single heap object reused. | Higher; repeated allocations can stress Garbage Collection. |

#### 2. Production Code Example
```java
@Configuration
public class ScopeDemoConfig {
    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
    public AuditService auditService() {
        return new AuditService(); // Created once at startup
    }

    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public BatchWorker batchWorker() {
        return new BatchWorker(); // Created new every call
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Singleton is the default Spring scope where one instance exists per container, initialized eagerly at startup and fully managed through destruction. Prototype creates a new instance on every request or injection, is suitable for stateful objects, and crucially, Spring does NOT manage prototype destruction lifecycle—`@PreDestroy` is not invoked by Spring on prototype beans."
* **In Our Project / Production Context:** "Our controllers and services are singletons and strictly stateless. For stateful multipart file parsing where worker threads hold mutable byte buffers, we define prototype beans."
* **Pro-Tip & Interview Trap:** "Remember the destruction trap: Spring calls `@PostConstruct` on prototype beans, but **NEVER calls `@PreDestroy`** on prototype beans! The client code is responsible for cleaning up prototype bean resources."

---

### Q85: What is the difference between field injection and constructor injection?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | Field Injection (`@Autowired` on field) | Constructor Injection (Recommended) |
| :--- | :--- | :--- |
| **Immutability** | **Cannot make fields `final`.** Fields must be mutable. | **Supports `final` fields.** Guarantees true immutability. |
| **Unit Testing** | Painful; requires reflection or Mockito `ReflectionTestUtils` to inject mocks. | Simple and clean; mocks can be passed directly via standard constructor in unit tests. |
| **Safety & NPEs** | Can cause `NullPointerException` if instantiated outside Spring (e.g., via `new`). | Fails fast at compile-time/instantiation time if dependencies are missing. |
| **Circular Dependencies** | Masks circular dependencies at startup; causes hidden runtime errors. | Immediately detects circular dependencies at startup with a `BeanCurrentlyInCreationException`. |
| **Spring Recommendation** | **Strongly discouraged** by the official Spring team. | **Official Best Practice** since Spring 4.x. |

#### 2. Production Code Example
```java
// Anti-Pattern: Field Injection
@Service
public class BadOrderService {
    @Autowired
    private PaymentClient paymentClient; // Mutable, hidden dependency, hard to test
}

// Production Best Practice: Constructor Injection with Lombok or Java Record
@Service
public class BestOrderService {
    private final PaymentClient paymentClient;
    private final NotificationClient notificationClient;

    // Single constructor: @Autowired is implicit since Spring 4.3!
    public BestOrderService(PaymentClient paymentClient, NotificationClient notificationClient) {
        this.paymentClient = Objects.requireNonNull(paymentClient);
        this.notificationClient = Objects.requireNonNull(notificationClient);
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Constructor injection is the official best practice in Spring. It allows dependencies to be declared as `final` ensuring immutability, makes dependencies explicit, makes plain JUnit unit testing trivial without Spring or reflection, and detects circular dependencies at startup. Field injection hides dependencies, violates encapsulation, prevents immutability, and makes unit tests fragile."
* **In Our Project / Production Context:** "We enforce constructor injection across our entire microservice codebase using `@RequiredArgsConstructor` from Lombok. In our unit tests, we instantiate classes directly via `new OrderService(mockPayment, mockRepo)` without needing slow Spring context boots."
* **Pro-Tip & Interview Trap:** "Since Spring 4.3, if a class has only one constructor, the `@Autowired` annotation is completely optional and Spring injects dependencies automatically."

---

### Q86: What is Bean Life Cycle in Spring?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Step-by-Step Spring Bean Lifecycle:**
  1. **Instantiation:** Spring instantiates the bean class (via reflection or constructor).
  2. **Populate Properties:** Spring injects all dependencies (`@Autowired`, values).
  3. **Aware Interfaces:** If implemented, Spring invokes:
     * `BeanNameAware.setBeanName()`
     * `BeanFactoryAware.setBeanFactory()`
     * `ApplicationContextAware.setApplicationContext()`
  4. **BeanPostProcessor (Before Initialization):** `postProcessBeforeInitialization()` runs (handles `@PostConstruct` via `CommonAnnotationBeanPostProcessor`).
  5. **Initialization:**
     * `@PostConstruct` annotated method.
     * `InitializingBean.afterPropertiesSet()`.
     * Custom `init-method` declared on `@Bean(initMethod = "...")`.
  6. **BeanPostProcessor (After Initialization):** `postProcessAfterInitialization()` runs. **This is where Spring creates dynamic CGLIB / JDK Dynamic Proxies for `@Transactional`, `@Async`, and `@Aspect`!**
  7. **Ready for Use:** Bean is fully initialized and serves application requests.
  8. **Destruction:** When container shuts down:
     * `@PreDestroy` annotated method.
     * `DisposableBean.destroy()`.
     * Custom `destroyMethod`.

#### 2. Production Code Example
```java
@Component
public class LifecycleLoggingBean implements BeanNameAware, InitializingBean, DisposableBean {

    @Override
    public void setBeanName(String name) {
        System.out.println("1. BeanNameAware: Bean name is " + name);
    }

    @PostConstruct
    public void initAnnotation() {
        System.out.println("2. @PostConstruct executed");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("3. InitializingBean.afterPropertiesSet executed");
    }

    @PreDestroy
    public void preDestroyAnnotation() {
        System.out.println("4. @PreDestroy executed before shutdown");
    }

    @Override
    public void destroy() {
        System.out.println("5. DisposableBean.destroy executed");
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "The Spring Bean Lifecycle begins with instantiation, followed by property and dependency injection. Then Aware interfaces are called, followed by `BeanPostProcessor` before-initialization, `@PostConstruct`, and `InitializingBean`. Then `BeanPostProcessor` after-initialization runs—where AOP proxies for transactions are wrapped. When shutting down, `@PreDestroy` and `DisposableBean` handle resource cleanup."
* **In Our Project / Production Context:** "Knowing that AOP proxies are created in `postProcessAfterInitialization` explained why calling a `@Transactional` method from `@PostConstruct` failed to run inside an active transaction: the transaction proxy hadn't wrapped the bean yet!"
* **Pro-Tip & Interview Trap:** "A classic gotcha: When does Spring create `@Transactional` proxies? During the `postProcessAfterInitialization()` phase of `BeanPostProcessor`."

---

### Q87: What is the purpose of @PostConstruct and @PreDestroy?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Context:** Standard annotations from `jakarta.annotation` (formerly `javax.annotation`).
* **`@PostConstruct`:**
  * Runs **once immediately after** the bean is instantiated and all dependencies have been injected.
  * Used for initialization tasks that require injected dependencies (e.g., warming up local caches, establishing network handshakes, validating configuration properties).
* **`@PreDestroy`:**
  * Runs **once immediately before** the bean is destroyed and removed from the ApplicationContext upon container shutdown.
  * Used for graceful resource cleanup (e.g., releasing database connection pools, flushing in-memory buffers, closing socket streams, stopping background worker threads).

#### 2. Production Code Example
```java
@Component
public class CacheWarmerManager {
    private final ProductRepository productRepository;
    private final Map<String, Product> localCache = new ConcurrentHashMap<>();

    public CacheWarmerManager(ProductRepository productRepository) {
        this.productRepository = productRepository;
        // In constructor, dependencies exist, but post-processing is not complete
    }

    @PostConstruct
    public void warmUpCache() {
        System.out.println("Executing @PostConstruct: Pre-loading top products into memory...");
        productRepository.findTopSelling().forEach(p -> localCache.put(p.getId(), p));
    }

    @PreDestroy
    public void cleanUp() {
        System.out.println("Executing @PreDestroy: Flushing metrics and clearing cache...");
        localCache.clear();
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`@PostConstruct` executes once after dependency injection completes, allowing us to perform initialization logic that relies on injected dependencies. `@PreDestroy` executes just before the bean is destroyed during container shutdown, providing a hook for graceful resource cleanup like closing connections or flushing buffers."
* **In Our Project / Production Context:** "We use `@PostConstruct` to warm up static currency exchange rates from our Redis cache upon service startup, and `@PreDestroy` to gracefully drain our internal thread pools when Kubernetes sends a `SIGTERM` signal."
* **Pro-Tip & Interview Trap:** "Never perform heavy initialization inside the constructor! In the constructor, Spring hasn't applied AOP proxies, environment variables, or post-processors. Always use `@PostConstruct` for post-injection logic."

---

### Q88: What is the difference between application.properties and application.yml?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | `application.properties` | `application.yml` (YAML) |
| :--- | :--- | :--- |
| **Format & Syntax** | Flat key-value pairs separated by `=` or `:`. Requires repeating prefixes. | Hierarchical structure based on indentation (spaces, never tabs). |
| **Readability** | Verbose and repetitive for deeply nested configs. | Clean, concise, and eliminates repetitive prefix names. |
| **Complex Structures** | Lists and maps are clunky: `app.servers[0]=1.1.1.1`. | Native, clean support for lists (`- item`) and nested maps. |
| **Multi-Profile Documents** | Requires separate files: `application-dev.properties`. | Supports multi-profile documents in a single file via `---` separators. |
| **Annotation Loading** | Supported by `@PropertySource`. | **NOT supported by `@PropertySource` directly** (requires custom `PropertySourceFactory`). |

#### 2. Production Code Example
```yaml
# application.yml: Clean hierarchical indentation
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orderdb
    username: postgres
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5

---
# Multi-profile document in single YAML file
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`application.properties` uses flat, repetitive key-value pairs, whereas `application.yml` uses hierarchical indentation which eliminates repetitive prefixes and supports multi-profile documents separated by `---`. In modern Spring Boot development, YAML is preferred for main application configs, though `@PropertySource` cannot load YAML files without custom factories."
* **In Our Project / Production Context:** "We use `application.yml` across all microservices for clear visual nesting of database, Kafka, and security configs, separating Dev, Staging, and Prod profiles with Spring Boot 2.4+ `spring.config.activate.on-profile`."
* **Pro-Tip & Interview Trap:** "YAML is strictly indentation-sensitive! Mixing tabs and spaces will prevent Spring Boot from starting with a `ParserException`."

---

### Q89: What is Spring Boot Actuator?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Context:** A production-ready sub-project of Spring Boot (`spring-boot-starter-actuator`).
* **Purpose:** Exposes operational telemetry, health checks, metrics, and application internals via HTTP REST endpoints or JMX.
* **Key Endpoints:**
  * `/actuator/health`: Application health status (UP/DOWN), database connectivity, disk space, and custom health indicators. Crucial for Kubernetes **Liveness and Readiness probes**.
  * `/actuator/metrics`: JVM memory usage, GC pauses, CPU utilization, HTTP request latency (Micrometer metrics exported to Prometheus/Grafana).
  * `/actuator/info`: Git commit hash, build version, and application description.
  * `/actuator/env`: Environment properties and configuration properties.
  * `/actuator/loggers`: View and dynamically alter log levels at runtime without restarting!

#### 2. Production Code Example
```yaml
# application.yml: Exposing Actuator Endpoints Safely
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus, info
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true # Enables /health/liveness and /health/readiness for Kubernetes
```

```java
// Custom Health Indicator in Java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean gatewayReachable = checkGatewayPing();
        if (gatewayReachable) {
            return Health.up().withDetail("Gateway", "Stripe API Available").build();
        }
        return Health.down().withDetail("Error", "Stripe API Timeout").build();
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Spring Boot Actuator brings production-ready features to applications by exposing HTTP endpoints for health monitoring, metrics, and audit logs. It integrates with Micrometer and Prometheus for observability and powers Kubernetes liveness and readiness probes to manage container restarts and traffic routing."
* **In Our Project / Production Context:** "In our EKS Kubernetes deployment, the readiness probe points to `/actuator/health/readiness` to prevent user traffic from reaching pods until our database connection pool and Kafka consumers are fully established."
* **Pro-Tip & Interview Trap:** "Security hazard! Never expose `include: "*"` in production without Spring Security! Sensitive endpoints like `/env`, `/heapdump`, and `/beans` can leak database credentials and secret API keys to the public."

---

### Q90: What is the purpose of CommandLineRunner?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Context:** An interface provided by Spring Boot (`org.springframework.boot.CommandLineRunner`).
* **Purpose:** Used to execute a block of code **once immediately after the Spring ApplicationContext is fully loaded and started**, but before the main startup method exits.
* **Signature:** `void run(String... args) throws Exception;`
* **Under the Hood:** Receives the raw, unparsed command-line arguments passed to `main(String[] args)`.
* **Execution Order:** If multiple runners exist, their execution sequence can be controlled using the `@Order(1)` annotation.

#### 2. Production Code Example
```java
@Component
@Order(1) // Runs first
public class DatabaseSeederCommandLineRunner implements CommandLineRunner {
    private final RoleRepository roleRepository;

    public DatabaseSeederCommandLineRunner(RoleRepository roleRepository) {
        this.roleRepository = roleRepository;
    }

    @Override
    public void run(String... args) throws Exception {
        System.out.println("Executing CommandLineRunner with raw args count: " + args.length);
        if (roleRepository.count() == 0) {
            roleRepository.save(new Role("ROLE_ADMIN"));
            roleRepository.save(new Role("ROLE_USER"));
            System.out.println("Default roles seeded successfully.");
        }
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`CommandLineRunner` is a callback interface in Spring Boot with a `run(String... args)` method that executes once immediately after the application context starts. It accepts raw command-line string arguments and is typically used for startup seeding, pre-flight system checks, or running one-off batch migration scripts."
* **In Our Project / Production Context:** "In our local Docker development profile, we use a `CommandLineRunner` to automatically seed mock customer data and admin accounts into our H2 database on startup."
* **Pro-Tip & Interview Trap:** "If an unhandled exception is thrown inside `CommandLineRunner.run()`, Spring Boot will abort startup and shut down the JVM."

---

### Q91: What is the purpose of ApplicationRunner?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Context:** An interface provided by Spring Boot (`org.springframework.boot.ApplicationRunner`).
* **Purpose:** Similar to `CommandLineRunner`, it executes code once right after Spring Boot starts up.
* **Key Difference from `CommandLineRunner`:**
  * `CommandLineRunner` receives raw `String... args`.
  * `ApplicationRunner` receives an `ApplicationArguments` object, which has already parsed the command-line options into **Option Arguments** (e.g., `--env=prod`, `--debug`) and **Non-Option Arguments** (e.g., raw file paths).

#### 2. Production Code Example
```java
@Component
public class AppConfigApplicationRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Checking for option arguments (--export=true)
        if (args.containsOption("export")) {
            List<String> values = args.getOptionValues("export");
            System.out.println("Export mode enabled with option: " + values);
        }

        // Checking non-option arguments
        List<String> nonOptionArgs = args.getNonOptionArgs();
        System.out.println("Non-option arguments: " + nonOptionArgs);
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`ApplicationRunner` is the modern alternative to `CommandLineRunner`. Both execute right after application startup, but `ApplicationRunner` provides an `ApplicationArguments` parameter that cleanly parses command-line flags into option arguments (prefixed with `--`) and non-option arguments, eliminating manual string parsing."
* **In Our Project / Production Context:** "When our scheduled microservice runs as an ephemeral Kubernetes CronJob, we use `ApplicationRunner` to parse `--jobName=EOD_RECONCILIATION` arguments and trigger the appropriate batch processor."
* **Pro-Tip & Interview Trap:** "Both `ApplicationRunner` and `CommandLineRunner` can coexist in the same application. You can order their execution using the `@Order` annotation."

---

### Q92: How do you handle multiple environments such as Dev, QA, and Production?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Spring Profiles Mechanism (`@Profile`):**
  * Allows segregating parts of application configuration and bean wiring to be available only in specific runtime environments.
* **Configuration File Structure:**
  * `application.yml` (Common shared base configurations).
  * `application-dev.yml` (Local H2/PostgreSQL, verbose SQL logging, mock third-party clients).
  * `application-qa.yml` (Shared staging database, test payment gateways).
  * `application-prod.yml` (Production Aurora/PostgreSQL, connection pooling, strict security, error-only logs).
* **Activating Profiles in Production:**
  * Via Environment Variable: `SPRING_PROFILES_ACTIVE=prod`.
  * Via JVM Argument: `java -jar -Dspring.profiles.active=prod app.jar`.
  * In Container manifests (Kubernetes Deployment YAML).

#### 2. Production Code Example
```java
// Bean configured only in dev environment
@Component
@Profile("dev")
public class MockEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String msg) {
        System.out.println("[DEV MOCK] Logged email to: " + to);
    }
}

// Bean configured only in production
@Component
@Profile("prod")
public class AwsSesEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String msg) {
        // Real AWS SES API Call
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We manage environments in Spring Boot using Spring Profiles and profile-specific YAML files (`application-dev.yml`, `application-prod.yml`). Base configs live in `application.yml`, and environment overrides are activated at runtime using the `SPRING_PROFILES_ACTIVE` environment variable. We also use `@Profile` on beans to swap real third-party integrations with mocks in lower environments."
* **In Our Project / Production Context:** "In our CI/CD pipeline, Jenkins deploys our Docker containers to Kubernetes, where each Helm chart injects `SPRING_PROFILES_ACTIVE=qa` or `prod` as a pod environment variable alongside Kubernetes Secrets for database passwords."
* **Pro-Tip & Interview Trap:** "Never commit production credentials into `application-prod.yml`! In production, environment-specific YAML files should contain placeholders that resolve against environment variables or secret managers (e.g., `${DB_PASSWORD}`)."

---

### Q93: What is lazy loading and eager loading?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Eager Loading (`FetchType.EAGER`):**
  * Associated entities or collections are fetched **immediately** alongside the parent entity using a SQL `JOIN`.
  * Default for: `@ManyToOne` and `@OneToOne`.
  * Risk: Can cause massive Cartesian products and severe performance bottlenecks when fetching large graphs.
* **Lazy Loading (`FetchType.LAZY`):**
  * Associated entities or collections are **not** loaded immediately.
  * Instead, Hibernate injects a dynamic runtime bytecode **Proxy** (e.g., ByteBuddy) or an uninitialized collection wrapper.
  * The actual SQL `SELECT` is issued on-demand only when a child property or getter is explicitly accessed (e.g., `parent.getChildren().size()`).
  * Default for: `@OneToMany` and `@ManyToMany`.
  * Risk: Throws `LazyInitializationException` if the child is accessed after the Hibernate `Session` / `@Transactional` scope has closed.

#### 2. Production Code Example
```java
@Entity
public class Customer {
    @Id
    private Long id;

    // Best Practice: Always make relationships LAZY
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();

    // Default for @ManyToOne is EAGER, best practice is to explicitly switch to LAZY:
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "account_manager_id")
    private AccountManager accountManager;
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Eager loading fetches child associations immediately using SQL joins when the parent entity is loaded. Lazy loading injects a proxy and defers the database query until the child association is explicitly accessed. In production, we configure `FetchType.LAZY` across all entity relationships to avoid fetching massive graphs, and we fetch needed associations explicitly using join fetches."
* **In Our Project / Production Context:** "We changed all `@ManyToOne` associations from their default `EAGER` to `LAZY`. This reduced our initial query payload by 60% across our customer dashboard queries."
* **Pro-Tip & Interview Trap:** "Beware of `LazyInitializationException: could not initialize proxy - no Session`. This occurs when you try to access a lazy relationship outside of an active `@Transactional` boundary (e.g., in a controller or Jackson JSON serializer)."

---

### Q94: What is the N+1 query problem?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **The Cause:** Occurs when an ORM like Hibernate executes **1 initial query** to fetch $N$ parent records, and then executes **$N$ additional individual queries** to fetch the child association for each parent record.
* **Under-the-Hood Scenario:**
  1. Initial query: `SELECT * FROM department;` (Returns 100 departments).
  2. Iterating through each department to get its employees triggers: `SELECT * FROM employee WHERE department_id = ?;` (Executed 100 times!).
  3. Total queries executed: $1 + 100 = 101$ queries.
* **Solutions to Fix N+1:**
  1. **`JOIN FETCH` in JPQL:** Performs an inner or left join and populates parent and children in 1 single query.
  2. **`@EntityGraph`:** Declaratively specifies which attributes to fetch eagerly for that specific query.
  3. **Hibernate Batch Fetching (`@BatchSize(size = 25)`):** Fetches children in batches using an SQL `WHERE department_id IN (?, ?, ...)`.

#### 2. Production Code Example
```java
public interface DepartmentRepository extends JpaRepository<Department, Long> {

    // Solution 1: JOIN FETCH in JPQL (Resolves N+1 in 1 single query)
    @Query("SELECT d FROM Department d LEFT JOIN FETCH d.employees")
    List<Department> findAllWithEmployeesJoinFetch();

    // Solution 2: @EntityGraph (Clean and declarative)
    @EntityGraph(attributePaths = {"employees"})
    @Query("SELECT d FROM Department d")
    List<Department> findAllWithEmployeesEntityGraph();
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "The N+1 query problem occurs when Hibernate executes 1 query to fetch $N$ parent records, and then issues $N$ subsequent queries to fetch child relationships for each record, degrading database performance. We solve it by using `JOIN FETCH` in JPQL or `@EntityGraph` in Spring Data JPA to fetch the parent and child records in a single optimized SQL join."
* **In Our Project / Production Context:** "In our order history API, fetching 50 orders generated 51 SQL queries because order items were fetched in a loop. Adding `JOIN FETCH o.items` consolidated it into a single SQL query, reducing API response time from 1.2 seconds to 80 milliseconds."
* **Pro-Tip & Interview Trap:** "Be cautious with `JOIN FETCH` on multiple `@OneToMany` collections simultaneously: it creates a Cartesian Product, which can duplicate memory rows and cause Hibernate to throw `MultipleBagFetchException`."

---

### Q95: What is the purpose of @Transactional?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Context:** Declarative transaction management in Spring (`@org.springframework.transaction.annotation.Transactional`).
* **Under-the-Hood Mechanism (Spring AOP Proxy):**
  1. Spring wraps the `@Transactional` bean in a dynamic proxy (CGLIB or JDK Dynamic Proxy).
  2. When a transactional method is called, the proxy's `TransactionInterceptor` intercepts the call.
  3. It obtains a database connection from the `DataSourceTransactionManager`, sets `autoCommit = false`, and binds the connection to the current thread via `TransactionSynchronizationManager`.
  4. The business method executes.
  5. If the method completes successfully, the interceptor calls `connection.commit()`.
  6. If an **unchecked exception (`RuntimeException` or `Error`)** is thrown, it calls `connection.rollback()`.
* **Important Attributes:**
  * `propagation`: Defines transaction boundaries (`REQUIRED`, `REQUIRES_NEW`, etc.).
  * `isolation`: Isolation levels (`READ_COMMITTED`, `REPEATABLE_READ`, etc.).
  * `readOnly = true`: Optimizes Hibernate by setting the Session FlushMode to `MANUAL`, skipping dirty checking.
  * `rollbackFor = Exception.class`: Forces rollback on checked exceptions.

#### 2. Production Code Example
```java
@Service
public class BankTransferService {
    private final AccountRepository accountRepository;

    public BankTransferService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional(
        propagation = Propagation.REQUIRED, 
        isolation = Isolation.READ_COMMITTED, 
        rollbackFor = Exception.class // Ensures rollback even on checked exceptions!
    )
    public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        from.debit(amount);
        to.credit(amount);

        // Both updates commit together. If an exception occurs, both roll back!
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`@Transactional` provides declarative transaction management using Spring AOP proxies. It opens a database transaction, manages commit upon successful method completion, and rolls back upon `RuntimeException`. We configure `readOnly = true` on query methods to disable Hibernate dirty checking for a performance boost, and use `rollbackFor = Exception.class` when handling checked exceptions."
* **In Our Project / Production Context:** "In our checkout flow, `@Transactional` coordinates deducting customer balances, creating order records, and reserving inventory. If third-party payment fails, the entire transaction rolls back atomically."
* **Pro-Tip & Interview Trap:** "Self-invocation trap! Calling a `@Transactional` method from another method **inside the same class** bypasses the Spring AOP proxy (`this.method()`), meaning **no transaction is started**! The call must come from an external bean to pass through the proxy."

---

### Q96: What is the difference between persist(), merge(), and save()?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Method | Provider | Under-the-Hood Mechanism | Entity State Transition | Return Value |
| :--- | :--- | :--- | :--- | :--- |
| **`persist()`** | JPA `EntityManager` | Takes a **transient** instance and makes it **managed**. Issues an `INSERT` statement on flush/commit. | Transient -> Managed. | `void` (Modifies the passed object directly). |
| **`merge()`** | JPA `EntityManager` | Takes a **detached** instance, finds or loads the managed entity from DB/Cache, copies the state onto it. | Detached -> Detached (Original remains detached; returns **new managed instance**). | Returns the **managed copy**. |
| **`save()`** | Spring Data JPA `CrudRepository` | Convenience abstraction. Checks `entityInformation.isNew()`. If `true`, it calls `em.persist()`; if `false`, it calls `em.merge()`. | Handles both new and existing entities dynamically. | Returns the managed entity instance. |

#### 2. Production Code Example
```java
@Repository
public class EntityLifecycleDemo {
    @PersistenceContext
    private EntityManager em;

    public void demoPersistAndMerge() {
        // 1. persist(): Must be a new entity
        User newUser = new User("Alice");
        em.persist(newUser); // newUser is now MANAGED. Returns void.

        // 2. merge(): Used for detached entities
        User detachedUser = new User("Bob");
        detachedUser.setId(10L); // Pre-existing ID
        User managedUser = em.merge(detachedUser); 
        // detachedUser is STILL DETACHED! managedUser is the one tracked by Hibernate!
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "`persist()` is a JPA method that transitions a new transient entity into the managed state and returns void. `merge()` is a JPA method that copies the state of a detached entity into an active persistence context and returns a new managed copy. Spring Data's `save()` is a higher-level wrapper that checks whether the entity is new: it calls `persist()` for new entities and `merge()` for existing entities."
* **In Our Project / Production Context:** "In Spring Data JPA, when saving existing entities with custom generated IDs, developers sometimes forget that `save()` calls `merge()`, which executes an extra `SELECT` before issuing the `UPDATE`. Understanding this allowed us to optimize high-throughput batch imports by implementing `Persistable`."
* **Pro-Tip & Interview Trap:** "With `em.merge(entity)`, remember that the passed object remains **detached**! Any subsequent changes to `entity` will not be tracked by Hibernate unless you use the returned reference: `User managed = em.merge(entity);`."

---

### Q97: What is the difference between first-level and second-level cache?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Feature | First-Level Cache (L1) | Second-Level Cache (L2) |
| :--- | :--- | :--- |
| **Scope & Lifecycle** | **Session / EntityManager Scope:** Bound strictly to the current database transaction. | **SessionFactory / Cluster Scope:** Shared across all sessions, transactions, and threads. |
| **Default Status** | **Mandatory & Enabled by default.** Cannot be turned off. | **Optional & Disabled by default.** Must be explicitly configured. |
| **Storage Location** | Local JVM heap within the active Hibernate `Session`. | Configurable external caching provider (Ehcache, Hazelcast, Redis). |
| **Concurrency Hazard** | None; accessed only by the single thread of that session. | Requires concurrency strategies (`READ_ONLY`, `NONSTRICT_READ_WRITE`, `TRANSACTIONAL`). |
| **Clearing Mechanism** | `session.clear()`, `session.evict(entity)`. | `sessionFactory.getCache().evictAll()`. |

#### 2. Production Code Example
```yaml
# application.yml: Configuring L2 Cache with Redisson / Ehcache
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.redisson.hibernate.RedissonRegionFactory
```

```java
// Enabling L2 caching on read-heavy, rarely-changing entity
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class CountryCode {
    @Id
    private String code;
    private String countryName;
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "First-level cache is the mandatory Hibernate Session cache enabled by default and scoped to a single transaction, ensuring that calling `findById()` multiple times within the same transaction hits memory rather than the database. Second-level cache is an optional, shared cache at the SessionFactory level, backed by providers like Redis or Ehcache, that caches entities across multiple transactions and application instances."
* **In Our Project / Production Context:** "We enabled Hibernate Second-Level Cache with Redis for our global tax rates and currency lookup tables. This reduced database read queries by 35% on our high-traffic checkout microservices."
* **Pro-Tip & Interview Trap:** "Be aware of memory leaks with L1 cache during batch processing! When inserting 100,000 records in a single transaction, the L1 cache stores every entity in memory, leading to `OutOfMemoryError`. You must periodically call `entityManager.flush()` and `entityManager.clear()`."

---

### Q98: How do you implement pagination and sorting using Spring Data JPA?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Spring Data JPA Support:** Built into `PagingAndSortingRepository` and `JpaRepository`.
* **Key Components:**
  * `Pageable`: Interface defining pagination requests (`PageRequest.of(page, size, Sort)`).
  * `Sort`: Handles multi-column sorting (`Sort.by("createdDate").descending()`).
  * `Page<T>`: Returns the page content **along with total elements and total pages** (executes an additional `SELECT count(*)` query).
  * `Slice<T>`: Returns only whether a **next page exists** (`hasNext()`), fetching `size + 1` elements **without executing an expensive `count(*)` query** (ideal for mobile infinite scrolling).

#### 2. Production Code Example
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Repository method accepting Pageable
    Page<Order> findByStatus(String status, Pageable pageable);
    
    // Slice for infinite scrolling without count(*) overhead
    Slice<Order> findByCustomerId(Long customerId, Pageable pageable);
}

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderRepository orderRepository;

    public OrderController(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @GetMapping
    public ResponseEntity<Page<OrderResponseDTO>> getOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "desc") String direction) {

        Sort sort = direction.equalsIgnoreCase("desc") ? 
            Sort.by(sortBy).descending() : Sort.by(sortBy).ascending();
            
        Pageable pageable = PageRequest.of(page, size, sort);
        
        Page<OrderResponseDTO> dtoPage = orderRepository.findByStatus("COMPLETED", pageable)
                                                        .map(OrderResponseDTO::fromEntity);
        return ResponseEntity.ok(dtoPage);
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "In Spring Data JPA, we implement pagination and sorting by passing a `Pageable` instance—created via `PageRequest.of(page, size, Sort)`—into repository query methods. It returns a `Page<T>` containing records, total counts, and metadata. When building infinite-scroll mobile feeds, we return a `Slice<T>` instead of `Page<T>` to avoid executing expensive `count(*)` queries on large tables."
* **In Our Project / Production Context:** "On our customer transactions table containing 20 million rows, `Page<T>` caused severe latency because `SELECT COUNT(*)` took over 3 seconds. Switching to `Slice<T>` eliminated the count query entirely, reducing API response times to under 50 milliseconds."
* **Pro-Tip & Interview Trap:** "Remember that Spring Data page numbers are **0-indexed** by default (`page=0` is the first page). Always validate that `size` has a safe maximum (e.g., max 100) to prevent malicious users from requesting `size=1000000` and crashing server memory."
