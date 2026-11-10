# Java, Spring Boot, Microservices & SQL Technical Interview Mastery
### Complete 256-Question Enterprise Interview Guide (2-Year Experienced Developer Standard)

Welcome to the **Master Interview Series (256 Questions)**. This repository houses complete, production-grade interview solutions designed specifically for developers positioning themselves with **2 Years of Professional Experience (2 YOE)**.

Every single question is structured with:
1. **Core Technical Concept & Under-the-Hood Mechanism** (JVM bytecode, heap/stack memory, Spring IoC lifecycle, database execution plans, or network protocols)
2. **Production-Grade Code Example** (Java 17/21, Spring Boot 3, SQL, or Docker/Kubernetes manifests)
3. **2-Year Experienced Developer Interview Pitch**:
   * **Direct Spoken Pitch:** Word-for-word concise explanation to answer the interviewer confidently.
   * **In Our Project / Production Context:** Authentic real-world enterprise context showing practical experience.
   * **Key Pro-Tip & Interview Trap:** Edge cases, hidden gotchas, and performance pitfalls interviewers test for.

---

## 📂 Navigation Index & Modular Sections

| Module | Topic Domain | Questions | Direct Markdown Link |
| :---: | :--- | :---: | :--- |
| **01** | **Modern Java (Java 9 – 21) & Language Evolutions** | Q1 – Q46 | [01_MODERN_JAVA_9_TO_21_DEEP_DIVE.md](./01_MODERN_JAVA_9_TO_21_DEEP_DIVE.md) |
| **02** | **Core Java, Memory Model, Concurrency & Stream API** | Q47 – Q80 | [02_CORE_JAVA_MEMORY_CONCURRENCY_STREAMS.md](./02_CORE_JAVA_MEMORY_CONCURRENCY_STREAMS.md) |
| **03** | **Spring Framework & Spring Boot Core** | Q81 – Q98 | [03_SPRING_BOOT_JPA_TRANSACTIONS.md](./03_SPRING_BOOT_JPA_TRANSACTIONS.md) |
| **04** | **REST API Design, Security, JWT & Performance** | Q99 – Q114 | [04_REST_API_SECURITY_PERFORMANCE.md](./04_REST_API_SECURITY_PERFORMANCE.md) |
| **05** | **SQL, CTEs, Window Functions & Database Optimization** | Q115 – Q138 | [05_SQL_WINDOW_FUNCTIONS_OPTIMIZATION.md](./05_SQL_WINDOW_FUNCTIONS_OPTIMIZATION.md) |
| **06** | **Microservices Architecture & Distributed Systems** | Q139 – Q173 | [06_MICROSERVICES_ARCHITECTURE_ECOSYSTEM.md](./06_MICROSERVICES_ARCHITECTURE_ECOSYSTEM.md) |
| **07** | **API Gateway (Spring Cloud Gateway & Zuul)** | Q174 – Q188 | [07_API_GATEWAY_ROUTING_SECURITY.md](./07_API_GATEWAY_ROUTING_SECURITY.md) |
| **08** | **Inter-Service Communication, OpenFeign & Resilience4j** | Q189 – Q221 | [08_INTER_SERVICE_COMMUNICATION_RESILIENCE4J.md](./08_INTER_SERVICE_COMMUNICATION_RESILIENCE4J.md) |
| **09** | **Distributed Transactions, Saga & Eventual Consistency** | Q222 – Q236 | [09_DISTRIBUTED_TRANSACTIONS_SAGA_EVENTUAL_CONSISTENCY.md](./09_DISTRIBUTED_TRANSACTIONS_SAGA_EVENTUAL_CONSISTENCY.md) |
| **10** | **REST Architectural Constraints & HTTP Semantics** | Q237 – Q256 | [10_REST_ARCHITECTURE_CONSTRAINTS_HTTP_SEMANTICS.md](./10_REST_ARCHITECTURE_CONSTRAINTS_HTTP_SEMANTICS.md) |
| **ALL** | **Consolidated All-in-One Master Guide (9,994 Lines)** | **Q1 – Q256** | [256_JAVA_SPRING_MICROSERVICES_MASTER_INTERVIEW_GUIDE.md](./256_JAVA_SPRING_MICROSERVICES_MASTER_INTERVIEW_GUIDE.md) |

---

## 🎯 Quick Question Breakdown

### Section 1: Modern Java 9 to 21 (Q1 – Q46)
* **Q1–Q5:** Shallow vs Deep Copy, Call by Value vs Reference, Immutable Classes, `module-info.java`, JShell REPL.
* **Q6–Q12:** Private methods in interfaces, Immutable collection factory methods (`List.of` vs `Arrays.asList`), Try-with-resources improvements, Stream `takeWhile()` & `dropWhile()`.
* **Q13–Q19:** `var` keyword rules, method parameter & field limits, null initialization, LVTI limitations, static typing guarantees, Collection `copyOf()`.
* **Q20–Q26:** Switch expressions, `yield`, Java Records (canonical constructors, compact constructors, interfaces, mutability).
* **Q27–Q34:** Helpful NPE messages, Pattern matching for `instanceof`, Text blocks, Sealed classes (`sealed`, `non-sealed`, `final`), Hidden classes.
* **Q35–Q43:** Records vs POJOs, Pattern matching improvements, Virtual Threads (Project Loom), Carrier threads, Platform vs Virtual threads, Record patterns, Switch pattern matching, Structured Concurrency, Scoped Values.
* **Q44–Q46:** Serialization, `Serializable` vs `Externalizable`, `transient` keyword.

### Section 2: Core Java, Memory & Concurrency (Q47 – Q80)
* **Q47–Q53:** JVM bytecode execution internals, Abstract class instantiation & constructors, Default/static interface methods, Method Hiding vs Overriding, Composition vs Inheritance, Marker interfaces, IS-A vs HAS-A.
* **Q54–Q60:** Stack vs Heap memory, Iterator vs ListIterator, `ConcurrentModificationException`, Comparable vs Comparator, Fail-fast vs Fail-safe, Checked vs Unchecked exceptions, `throw` vs `throws`.
* **Q61–Q67:** Multiple catch blocks, Custom exceptions, Thread vs Runnable, Thread lifecycle states, Synchronization monitor locks, Deadlock prevention, `synchronized` vs `volatile`.
* **Q68–Q73:** `ExecutorService` thread pool tuning, Functional interfaces (`Predicate`, `Function`, `Consumer`, `Supplier`), SAM rule with default methods, Method references, Lambdas vs Anonymous Inner Classes.
* **Q74–Q80:** Collection vs Stream, `map` vs `flatMap`, Intermediate vs Terminal operations, Sequential vs Parallel streams, `Optional` best practices, `orElse` vs `orElseGet` eager evaluation trap.

### Section 3: Spring Framework & Spring Boot Core (Q81 – Q98)
* **Q81–Q87:** IoC vs DI, BeanFactory vs ApplicationContext, Bean scopes (Singleton vs Prototype destruction trap), Field vs Constructor injection, Bean lifecycle phases, `@PostConstruct` & `@PreDestroy`.
* **Q88–Q92:** `application.properties` vs `.yml`, Spring Boot Actuator endpoints, `CommandLineRunner` vs `ApplicationRunner`, Multi-environment profiles (`application-{profile}.yml`).
* **Q93–Q98:** Lazy vs Eager loading, N+1 query problem (`JOIN FETCH`, `@EntityGraph`), `@Transactional` AOP proxies and self-invocation trap, `persist()` vs `merge()` vs `save()`, L1 vs L2 caching, Pagination & Sorting with `Pageable` and `Slice`.

### Section 4: REST API Design & Security (Q99 – Q114)
* **Q99–Q103:** PathVariable vs RequestParam, API versioning strategies, Pagination design, Synchronous vs Asynchronous REST APIs, HTTP 500 vs 503 error handling.
* **Q104–Q110:** Springdoc OpenAPI / Swagger UI setup, JWT authentication filter implementation, Defense-in-depth API security, Role-based authorization (`@PreAuthorize`), Multipart file uploads to S3, Postman API testing with Newman, Advanced OpenAPI security schemes.
* **Q111–Q114:** Request/response logging with `ContentCachingRequestWrapper`, Idempotency Key pattern with Redis, API performance optimization, Timeouts and downstream failure handling.

### Section 5: SQL, Window Functions & Optimization (Q115 – Q138)
* **Q115–Q123:** Common Table Expressions (CTEs), Recursive CTEs for hierarchy trees, Subqueries (scalar, table), Correlated vs Non-correlated subqueries, `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, Comparative ranking matrix, `PARTITION BY`.
* **Q124–Q126:** Finding duplicate records, Deleting duplicates keeping one record, Finding Nth-highest salary with `DENSE_RANK()`.
* **Q127–Q134:** Slow SQL query optimization, `EXPLAIN ANALYZE` execution plans, Deadlocks and wait-for graphs, COMMIT vs ROLLBACK, Transaction isolation levels and read anomalies, Composite indexes and Leftmost Prefix rule, `EXISTS` vs `IN` with NULLs, Stored Procedures vs Functions.
* **Q135–Q138:** Search API debouncing with RxJS (`debounceTime`, `distinctUntilChanged`, `switchMap`), Angular unnecessary API call optimization (`shareReplay(1)`, OnPush, `trackBy`), Point-in-Time Recovery (PITR) & soft deletes, Slow stored procedure troubleshooting and parameter sniffing.

### Section 6: Microservices Architecture Fundamentals (Q139 – Q173)
* **Q139–Q147:** Microservices definition and core characteristics, Monolith vs Microservices trade-offs, Advantages & Disadvantages, Enterprise adoption drivers, SOA vs Microservices, Loose coupling, High cohesion and low coupling.
* **Q148–Q158:** Inter-service communication paradigms, Synchronous vs Asynchronous, REST vs Messaging (Kafka), Service Discovery & Registration, Centralized Configuration, Distributed Tracing (Micrometer Tracing + Zipkin), Fault tolerance, Granular Horizontal Pod Autoscaling, Domain-Driven Design (DDD) Bounded Contexts.
* **Q159–Q165:** Creating Spring Boot microservices, Role of Spring Boot, Process isolation, Configuring ports, Polyglot persistence, Database-per-service pattern, Why direct database access is prohibited.
* **Q166–Q173:** Standardized exception handling (RFC 7807 ProblemDetails), Shared configurations, Centralized JSON logging (ELK), Zero-trust security & mTLS, Multi-service Kubernetes deployments, Service communication management, Failure handling, Observability with the Four Golden Signals (Prometheus/Grafana).

### Section 7: API Gateway (Q174 – Q188)
* **Q174–Q180:** API Gateway role and reverse proxy architecture, Why use an API Gateway, API Gateway vs Eureka Server, Request routing predicates, Load balancing, Spring Cloud Gateway reactive architecture, Spring Cloud Gateway vs Netflix Zuul 1.x.
* **Q181–Q183:** Gateway filters (Pre/Post lifecycle), Global Filters vs Gateway Filters, Declarative YAML vs Java DSL route configuration.
* **Q184–Q188:** Edge JWT authentication filter, Global CORS handling, Access restriction & Kubernetes NetworkPolicies, Redis Token Bucket rate limiting, Handling API Gateway single points of failure (SPOF) with multi-AZ ALB clustering.

### Section 8: Inter-Service Communication & Resilience4j (Q189 – Q221)
* **Q189–Q196:** Synchronous vs Asynchronous service communication, `RestTemplate` vs `WebClient` vs `OpenFeign`, Declarative REST clients, Spring 6 HTTP Interfaces, `@FeignClient` configuration and fallback factories.
* **Q197–Q204:** Request header propagation (Feign `RequestInterceptor`), Connect and Read timeout tuning, Downstream service outage handling, Service-to-service authentication (OAuth2 Client Credentials & mTLS), Load balancing types, Client-side load balancing, Spring Cloud LoadBalancer, Fault tolerance pillars.
* **Q205–Q213:** Resilience4j architecture vs Hystrix, Circuit Breaker mechanics, Circuit Breaker states, CLOSED / OPEN / HALF_OPEN lifecycle, Retry pattern, RateLimiter, Bulkhead (Semaphore vs ThreadPool), TimeLimiter with async futures, Fallback methods.
* **Q214–Q218:** Retry vs Circuit Breaker, Why you shouldn't retry every request, Handling slow external APIs, Cascading failure prevention, Configuring Resilience4j in Spring Boot.
* **Q219–Q221:** Dead Letter Queue (DLQ) pattern, Event-driven architecture with Kafka, Implementing Kafka Producers and Consumers with manual acknowledgment in Spring Boot.

### Section 9: Distributed Transactions & Saga Pattern (Q222 – Q236)
* **Q222–Q227:** Distributed transactions complexity, Why transactions are difficult in microservices (Dual-Write problem), Saga pattern mechanics, Choreography vs Orchestration Sagas, Why Two-Phase Commit (2PC) is obsolete, Eventual consistency and inconsistency lag.
* **Q228–Q230:** Database-per-service implications, Can microservices share a database, Maintaining data consistency across services.
* **Q231–Q233:** Transactional Outbox pattern with Debezium CDC, Command Query Responsibility Segregation (CQRS) with Elasticsearch, ACID vs BASE distributed models.
* **Q234–Q236:** Handling failed payments in Sagas, Compensating transactions design, Avoiding duplicate transaction processing with the Idempotent Consumer pattern.

### Section 10: REST Architecture & HTTP Semantics (Q237 – Q256)
* **Q237–Q241:** REST architectural style and 6 constraints, REST API vs RESTful Web Service (Richardson Maturity Model), The six constraints in detail, Stateless communication, Why REST APIs must be stateless.
* **Q242–Q245:** Resource-based API design (Nouns vs Verbs), Resource vs Endpoint distinction, REST vs SOAP, HATEOAS (Level 3 hypermedia links).
* **Q246–Q250:** Content negotiation (`Accept` and `Content-Type`), Idempotency in REST APIs, Idempotent HTTP methods matrix, Why POST is non-idempotent, Safe vs Idempotent methods.
* **Q251–Q256:** Synchronous vs Asynchronous REST APIs (HTTP 202 Accepted), API versioning necessity, Backward compatibility and Postel's Law, URI vs URL vs URN, HTTP headers taxonomy, Handling JSON and XML multi-format payloads.

---

## 💡 How to Use This Guide in Interviews
* **Memorize the "Direct Spoken Pitch"**: It gives you a crisp 3-sentence summary to say immediately when asked the question.
* **Back it up with "In Our Project"**: Follow up directly with real-world context ("In our microservices project, we faced this when...").
* **Score Bonus Points with "Pro-Tip & Interview Trap"**: Proactively mention the common traps before the interviewer even asks them. This immediately marks you as an engineer with genuine hands-on production experience.

