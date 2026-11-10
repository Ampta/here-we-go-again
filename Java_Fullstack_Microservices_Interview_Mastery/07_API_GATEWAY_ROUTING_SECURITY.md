# Section 7: API Gateway (Spring Cloud Gateway & Zuul) (Q174 – Q188)

---

### Q174: What is an API Gateway?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** An architectural pattern and specialized reverse proxy server that serves as the **single unified entry point** for all external client traffic (web, mobile, third-party partners) into a microservices ecosystem.
* **Under the Hood:**
  * Sits at the edge of the private network (North-South boundary).
  * Intercepts incoming HTTP requests, terminates TLS/SSL, enforces authentication and rate limits, and routes the traffic to the appropriate internal downstream microservice.
  * Masks internal microservice topology, port numbers, and private network IP addresses from external exposure.

#### 2. Architecture Blueprint
```
+-------------------------------------------------------------------------+
|                  CLIENTS (Web Browser, iOS, Android)                    |
+------------------------------------+------------------------------------+
                                     | HTTPS (Public Internet)
+------------------------------------v------------------------------------+
|                         API GATEWAY (Edge Proxy)                        |
|  [SSL Termination | JWT Validation | Rate Limiting | Dynamic Routing]   |
+---------+--------------------------+--------------------------+---------+
          |                          |                          |
          | (Private Network)        | (Private Network)        | (Private Network)
          v                          v                          v
+-------------------+      +-------------------+      +-------------------+
|  Order Service    |      |  Payment Service  |      | Inventory Service |
|    (:8081)        |      |    (:8082)        |      |    (:8083)        |
+-------------------+      +-------------------+      +-------------------+
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "An API Gateway is a reverse proxy that acts as the single public entry point for all client requests into our microservices architecture. It abstracts internal service topology, terminates public SSL, and centralizes cross-cutting concerns like authentication, rate limiting, and request routing so individual microservices don't duplicate boilerplate security logic."
* **In Our Project / Production Context:** "We use Spring Cloud Gateway as our API Gateway. Clients make all requests to `https://api.mycompany.com`, and the gateway validates user JWT tokens and dynamically routes requests to our internal Kubernetes pods."
* **Pro-Tip & Interview Trap:** "Don't confuse an API Gateway with a standard Load Balancer: An AWS ALB or Nginx performs layer 4/7 load balancing, whereas an API Gateway understands microservice semantics, integrates with service discovery registries (Eureka), and executes custom programmatic Java filters."

---

### Q175: Why do we use API Gateway in Microservices?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Critical Problems Solved by an API Gateway:**
  1. **Prevents Client-to-Microservice Direct Coupling:** Without a gateway, mobile apps would have to maintain direct connections to 20 different service hostnames and ports.
  2. **Centralizes Cross-Cutting Concerns:** Offloads authentication, JWT validation, CORS headers, logging, and rate limiting to the edge.
  3. **Protocol Translation:** Can translate external public HTTP/REST (JSON) into internal high-performance gRPC (Protobuf).
  4. **API Composition / Aggregation:** Can aggregate responses from three services (e.g., User, Orders, and Recommendations) into a single composite JSON response, saving mobile battery and bandwidth.
  5. **Security Perimeter (Attack Surface Reduction):** Only the gateway's port 443 is exposed to the internet; all microservices live in private subnets with no public IPs.

#### 2. Production Code Example (Gateway Route Consolidation)
```yaml
# Spring Cloud Gateway Route Configuration
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service # Route to Eureka service discovery
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=1
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We use an API Gateway to prevent client applications from directly coupling to dozens of changing internal service endpoints. It drastically simplifies architecture by centralizing cross-cutting security, rate limiting, and CORS policies at the edge, reduces mobile latency through request aggregation, and protects internal services behind a secure private subnet."
* **In Our Project / Production Context:** "Before implementing the API Gateway, our frontend team had to manage different domain URLs and handle authentication tokens across 6 different backends. Centralizing traffic behind Spring Cloud Gateway reduced frontend integration bugs by over 50%."
* **Pro-Tip & Interview Trap:** "Watch out for the **'Gateway as a Monolith' antipattern**: Never put heavy business logic or complex database queries inside the API Gateway! The gateway should remain lightweight and focus purely on routing, security, and edge transformations."

---

### Q176: What is the difference between API Gateway and Eureka Server?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | Eureka Server (Service Registry) | API Gateway (Traffic Router) |
| :--- | :--- | :--- |
| **Primary Role** | **Address Book / Registry:** Maintains a dynamic database of active service names, IP addresses, and ports. | **Traffic Controller / Proxy:** Intercepts actual client HTTP traffic and forwards it to target microservices. |
| **Data Flow** | Handles **metadata only** (heartbeats, registrations, IP lookups). Never processes client payload data. | Handles **all client traffic** (routes HTTP requests, processes headers, buffers responses). |
| **Client Exposure** | Strictly internal; **never exposed to the public internet**. | Publicly accessible edge server; exposed to internet clients. |
| **Relationship** | The Gateway **queries Eureka** to discover where services are running (`lb://order-service`). | The Gateway relies on Eureka's registry to perform client-side load balancing. |

#### 2. Interaction Diagram
```
1. Microservices register with Eureka:
[ Order Service :8081 ] ----(Register: IP=10.0.1.5:8081)----> [ Eureka Server ]
[ Payment Service :8082 ] --(Register: IP=10.0.1.9:8082)----> [ Eureka Server ]

2. Gateway queries Eureka & routes client traffic:
[ Client Request ] ---> [ API Gateway ] <--(Lookup: "order-service" = 10.0.1.5:8081)-- [ Eureka ]
                               |
                               +----(Forward HTTP)----> [ Order Service: 10.0.1.5:8081 ]
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Eureka Server is a service registry that maintains a dynamic directory of microservice instances and their IP addresses. The API Gateway is the reverse proxy that handles incoming client traffic. The Gateway consults Eureka to discover the live IP addresses of downstream services and routes client requests to them."
* **In Our Project / Production Context:** "When a new Order Service container spins up in our cluster, it registers its dynamic port with Eureka. Spring Cloud Gateway polls Eureka every 30 seconds and immediately begins routing `/api/orders` requests to the new instance."
* **Pro-Tip & Interview Trap:** "A common junior misunderstanding: 'Does client traffic flow through Eureka?' Absolutely NOT! Eureka is purely an out-of-band registry. All actual user traffic flows through the API Gateway directly to the microservices."

---

### Q177: What is request routing?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** The fundamental mechanism by which an API Gateway evaluates incoming HTTP request attributes and determines which internal downstream microservice should receive the request.
* **Spring Cloud Gateway Routing Architecture:**
  * **Route:** The basic building block consisting of an `ID`, a destination `URI`, a collection of `Predicates`, and a collection of `Filters`.
  * **Predicates:** Conditions evaluated against the HTTP request (Path, Method, Headers, Query Params, Host, Cookies). If all predicates evaluate to `true`, the route is matched.
  * **Filters:** Interceptors applied before forwarding the request or before returning the response.

#### 2. Production Code Example (Spring Cloud Gateway Configuration)
```yaml
spring:
  cloud:
    gateway:
      routes:
        # Route 1: Directing Order traffic based on Path and Method
        - id: order-service-route
          uri: lb://order-service # Dynamic discovery via Eureka
          predicates:
            - Path=/api/v1/orders/**
            - Method=GET,POST,PUT,DELETE
          filters:
            - AddRequestHeader=X-Gateway-Source, EdgeGateway

        # Route 2: Directing Admin traffic with Query Parameter and Header check
        - id: admin-service-route
          uri: lb://admin-service
          predicates:
            - Path=/api/v1/admin/**
            - Header=X-Admin-Role, SuperAdmin
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Request routing is the process where the API Gateway evaluates incoming HTTP requests against configured predicates—such as URL paths, HTTP methods, or headers—and forwards matching traffic to the appropriate downstream service destination URI, using service discovery load balancing (`lb://service-name`)."
* **In Our Project / Production Context:** "Our Spring Cloud Gateway uses path-based routing: requests to `/api/v1/orders/**` are routed to the Order Service, and `/api/v1/payments/**` to the Payment Service, while stripping unnecessary gateway prefixes before reaching internal services."
* **Pro-Tip & Interview Trap:** "Order matters in route definitions! Spring Cloud Gateway evaluates routes in the order they are declared in configuration. Specific routes must always be declared before generic wildcard catch-all routes."

---

### Q178: What is load balancing?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** The process of distributing incoming network traffic efficiently and evenly across a pool of multiple healthy backend server instances to optimize resource utilization, maximize throughput, and prevent any single server from becoming a bottleneck.
* **Server-Side Load Balancing vs Client-Side Load Balancing:**
  * **Server-Side Load Balancing (Hardware / Cloud ALB):** Client sends requests to a centralized load balancer (e.g., AWS ALB, F5). The load balancer picks an instance and forwards the packet.
  * **Client-Side Load Balancing (Spring Cloud LoadBalancer):** The client (API Gateway or calling microservice) possesses the list of available instances from Eureka and uses an internal algorithm (Round Robin or Random) to select the target instance directly before making the HTTP call.
* **Common Load Balancing Algorithms:** Round Robin, Weighted Round Robin, Least Connections, Random, and IP Hash.

#### 2. Architectural Comparison Diagram
```
Server-Side Load Balancing:
[ Client ] ---> [ Hardware Load Balancer ] ---> [ Instance 1 ]
                                            ---> [ Instance 2 ]

Client-Side Load Balancing (Spring Cloud):
[ API Gateway / FeignClient ] 
  (Queries Eureka -> Picks Instance locally via Round-Robin)
       |
       +-------------------------------------> [ Instance 2 ]
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Load balancing distributes incoming traffic across multiple healthy service instances to ensure high availability and prevent resource saturation. In Spring Cloud, we use Spring Cloud LoadBalancer for client-side load balancing, where the calling service queries Eureka, caches instance addresses, and distributes requests locally using algorithms like Round Robin without an extra hardware hop."
* **In Our Project / Production Context:** "When our Payment Service scales to 4 instances, Spring Cloud Gateway balances incoming checkout requests across all 4 pods using Round Robin, ensuring no single pod exceeds CPU thresholds."
* **Pro-Tip & Interview Trap:** "Note that **Netflix Ribbon** is completely deprecated and removed in modern Spring Cloud! Spring Cloud LoadBalancer is the official, modern non-blocking replacement."

---

### Q179: What is the purpose of Spring Cloud Gateway?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Context:** Built by the Spring team on top of **Spring 5, Spring Boot 3, Spring WebFlux, and Project Reactor**.
* **Under the Hood (Non-Blocking Reactive Engine):**
  * Powered by an embedded **Netty** server (asynchronous event loop), rather than traditional blocking servlet containers like Tomcat.
  * Uses a small, fixed pool of event-loop worker threads (typically matching CPU core count).
  * Threads never block waiting for downstream responses; when a remote microservice is called, the thread is released back to the Netty event loop to serve other incoming connections.
  * When downstream data arrives, Netty resumes execution via reactive callbacks (`Mono` and `Flux`).
* **Purpose:** Provides a high-performance, non-blocking, developer-friendly API Gateway with deep Spring ecosystem integration (Actuator, Security, Sleuth/Micrometer).

#### 2. Production Code Example (Fluent Java DSL Route Configuration)
```java
@Configuration
public class GatewayRoutesConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service", r -> r.path("/orders/**")
                .filters(f -> f.stripPrefix(1)
                               .addResponseHeader("X-Powered-By", "Spring-Cloud-Gateway"))
                .uri("lb://order-service"))
            .route("payment-service", r -> r.path("/payments/**")
                .filters(f -> f.circuitBreaker(c -> c.setName("paymentCB")))
                .uri("lb://payment-service"))
            .build();
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Spring Cloud Gateway is the modern, official API Gateway framework in the Spring ecosystem. Built on non-blocking reactive architectures using Project Reactor, WebFlux, and Netty, it handles high volumes of concurrent connections with minimal thread overhead, providing routing, custom filter pipelines, and native Resilience4j circuit breaker integration."
* **In Our Project / Production Context:** "We migrated from legacy Zuul 1 to Spring Cloud Gateway. Under load tests with 10,000 concurrent WebSocket and HTTP connections, Spring Cloud Gateway used 70% less memory and handled 3x more throughput without thread exhaustion."
* **Pro-Tip & Interview Trap:** "Spring Cloud Gateway requires `spring-boot-starter-webflux`! If you accidentally include `spring-boot-starter-web` (Tomcat) on its classpath, Spring Boot will fail to start with a configuration conflict."

---

### Q180: What is the difference between Spring Cloud Gateway and Zuul?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Architectural Criteria | Netflix Zuul 1.x (Legacy) | Spring Cloud Gateway (Modern Standard) |
| :--- | :--- | :--- |
| **I/O Processing Model** | **Blocking I/O:** Thread-per-connection model based on standard Servlet API. | **Non-Blocking Reactive I/O:** Event-driven architecture based on Netty and Project Reactor. |
| **Underlying Web Server** | Traditional Servlet Container (Tomcat / Jetty). | Embedded Netty Event Loop. |
| **Concurrency & Memory** | High memory usage; threads block during slow downstream calls, causing thread pool starvation. | Minimal memory; event loop threads never block, handling thousands of concurrent requests effortlessly. |
| **WebSocket Support** | Poor / Not natively supported. | **First-class native WebSocket routing support.** |
| **Maintenance Status** | **Deprecated in Spring Cloud**; maintenance mode. | **Active development**; official standard in Spring Cloud. |

#### 2. Thread Model Comparison Diagram
```
Zuul 1 (Blocking Thread-per-Request):
Client 1 ---> [ Worker Thread 1 (Blocks 2000ms on Downstream) ] ---> CPU Stall!
Client 2 ---> [ Worker Thread 2 (Blocks 2000ms on Downstream) ] ---> Thread Exhaustion!

Spring Cloud Gateway (Non-Blocking Netty Event Loop):
Client 1 --+
Client 2 --+---> [ 1 Netty Event Loop Thread ] ---> Dispatches async I/O and immediately
Client 3 --+                                       serves next connection without blocking!
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Zuul 1 was built on a blocking, thread-per-request servlet architecture where slow downstream responses blocked worker threads, leading to thread pool starvation under high concurrency. Spring Cloud Gateway was built from scratch on reactive Netty and Project Reactor, allowing a small fixed pool of non-blocking event-loop threads to process tens of thousands of concurrent requests efficiently with full WebSocket support."
* **In Our Project / Production Context:** "We decommissioned our old Zuul gateway when slow third-party API calls caused Tomcat worker threads to max out at 200 threads, bringing down all routes. Spring Cloud Gateway's reactive model eliminated thread starvation completely."
* **Pro-Tip & Interview Trap:** "Netflix did eventually build Zuul 2 on Netty, but the Spring team decided not to integrate it and instead built Spring Cloud Gateway natively for seamless alignment with Spring Boot 2/3 and WebFlux."

---

### Q181: What are Gateway Filters?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** Interceptors within Spring Cloud Gateway that allow inspecting and modifying incoming HTTP requests before forwarding them to downstream microservices, or inspecting and mutating outgoing HTTP responses before returning them to clients.
* **The Filter Chain Lifecycle:**
  * **'Pre' Filter Logic:** Executes *before* the request is routed downstream (e.g., verifying auth tokens, logging headers, adding correlation IDs, rate limiting).
  * **'Post' Filter Logic:** Executes *after* the downstream microservice has returned a response (e.g., adding security headers, calculating latency, compressing payloads).

#### 2. Production Code Example (Custom GatewayFilterFactory)
```java
@Component
public class CorrelationIdGatewayFilterFactory 
        extends AbstractGatewayFilterFactory<CorrelationIdGatewayFilterFactory.Config> {

    public CorrelationIdGatewayFilterFactory() { super(Config.class); }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // 1. PRE-FILTER LOGIC: Check or generate Correlation ID
            String correlationId = exchange.getRequest().getHeaders().getFirst("X-Correlation-ID");
            if (correlationId == null || correlationId.isBlank()) {
                correlationId = UUID.randomUUID().toString();
            }

            // Mutate request to add header downstream
            ServerWebExchange mutatedExchange = exchange.mutate()
                .request(r -> r.header("X-Correlation-ID", correlationId))
                .build();

            // 2. POST-FILTER LOGIC: Add correlation ID to client response headers
            return chain.filter(mutatedExchange).then(Mono.fromRunnable(() -> {
                mutatedExchange.getResponse().getHeaders().add("X-Correlation-ID", correlationId);
            }));
        };
    }

    public static class Config { /* Configuration properties */ }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Gateway Filters are interceptors in Spring Cloud Gateway that execute pre- and post-processing logic around routed requests. Pre-filters inspect and modify incoming requests—such as adding authentication headers or correlation IDs—before forwarding them downstream. Post-filters inspect and modify the outgoing HTTP response—such as injecting security headers or tracking latency—before sending it to the client."
* **In Our Project / Production Context:** "We implemented a custom gateway filter that checks for an `X-Correlation-ID` header; if missing, it generates a UUID and injects it into both the downstream request and client response headers for end-to-end tracing."
* **Pro-Tip & Interview Trap:** "In reactive WebFlux filters, never use `ThreadLocal`! Reactive pipelines switch threads frequently. Request state must be stored in the reactive context or the `ServerWebExchange.getAttributes()` map."

---

### Q182: What is the difference between Global Filter and Gateway Filter?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | Global Filter (`GlobalFilter`) | Gateway Filter (`GatewayFilter`) |
| :--- | :--- | :--- |
| **Scope of Application** | Applied **globally to EVERY single route** in the gateway without exception. | Applied **selectively to specific individual routes** in configuration. |
| **Configuration Requirement** | Automatically detected and registered as a Spring bean. No YAML route mapping needed. | Must be explicitly referenced inside route definitions in `application.yml` or Java DSL. |
| **Typical Use Cases** | Global telemetry logging, global distributed tracing, system-wide metrics, global CORS. | Route-specific tasks: `StripPrefix`, route-specific auth, custom request body transformations. |
| **Interface Implemented** | `org.springframework.cloud.gateway.filter.GlobalFilter` | `org.springframework.cloud.gateway.filter.GatewayFilter` |

#### 2. Production Code Example
```java
// 1. Global Filter: Automatically applies to ALL routes
@Component
public class GlobalRequestLatencyLoggingFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long duration = System.currentTimeMillis() - startTime;
            System.out.println("Path: " + exchange.getRequest().getPath() + " executed in " + duration + "ms");
        }));
    }

    @Override
    public int getOrder() { return -1; } // High priority
}
```

```yaml
# 2. Gateway Filter: Configured selectively on specific routes
spring:
  cloud:
    gateway:
      routes:
        - id: payment-route
          uri: lb://payment-service
          predicates:
            - Path=/payments/**
          filters:
            - StripPrefix=1 # Gateway Filter applied ONLY to payment-route
            - name: RequestRateLimiter # Gateway Filter applied ONLY to payment-route
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Global Filters apply to every single route passing through the gateway automatically and are used for universal concerns like distributed tracing, global audit logging, and security headers. Gateway Filters are declared selectively on specific route configurations, allowing fine-grained transformations like `StripPrefix` or custom rate limiting on sensitive routes."
* **In Our Project / Production Context:** "Our global filter injects OpenTelemetry trace IDs across all requests universally, while our payment route specifically attaches a custom `PaymentRateLimiter` gateway filter."
* **Pro-Tip & Interview Trap:** "Order matters! You can control the combined execution order of both Global Filters and Gateway Filters by implementing the `Ordered` interface or using the `@Order` annotation."

---

### Q183: How do you configure routes in Spring Cloud Gateway?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Two Primary Approaches:**
  1. **Declarative Configuration via `application.yml` (Recommended for Operations):**
     * Clean, externalized, and can be dynamically refreshed via Spring Cloud Config without recompiling Java code.
  2. **Programmatic Configuration via Java DSL (`RouteLocatorBuilder`):**
     * Type-safe, compiled, supports complex runtime conditional logic and programmatic lambda transformations.

#### 2. Production Code Example
```yaml
# Approach 1: Declarative YAML Configuration
spring:
  cloud:
    gateway:
      routes:
        - id: catalog-service-route
          uri: lb://catalog-service
          predicates:
            - Path=/api/v1/products/**
            - Method=GET
          filters:
            - RewritePath=/api/v1/(?<segment>.*), /${segment}
            - AddResponseHeader=X-Cache-Status, HIT
```

```java
// Approach 2: Programmatic Java DSL Configuration
@Configuration
public class GatewayRouteBuilderConfig {

    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service-route", r -> r
                .path("/api/v1/users/**")
                .and()
                .method("GET", "POST")
                .filters(f -> f.stripPrefix(2)
                               .addRequestHeader("X-Internal-Secret", "SecretToken123"))
                .uri("lb://user-service"))
            .build();
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We configure routes in Spring Cloud Gateway either declaratively in `application.yml` or programmatically using the Java `RouteLocatorBuilder` DSL. Each route consists of an ID, target destination URI (using `lb://service-name`), predicates that match request criteria like Path or Method, and filters that manipulate requests and responses."
* **In Our Project / Production Context:** "We use declarative YAML configuration in our Git repository backed by Spring Cloud Config Server, allowing our DevOps team to route traffic or add deprecation headers without touching application source code."
* **Pro-Tip & Interview Trap:** "Remember the **`StripPrefix`** filter! If a client calls `/api/v1/orders/10` and your backend service expects `/orders/10`, forgetting `StripPrefix=2` will cause the backend service to return an HTTP 404."

---

### Q184: How do you implement JWT authentication at API Gateway?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Architecture:**
  1. Incoming HTTP request hits the API Gateway with `Authorization: Bearer <jwt>`.
  2. A custom reactive `AuthenticationFilter` intercepts the request before routing.
  3. The filter checks against an unsecured endpoint whitelist (e.g., `/auth/login`, `/auth/register`).
  4. If secured, the filter extracts the token and cryptographically validates the signature and expiration using the Identity Provider's public key (RSA `RS256`).
  5. If valid, the filter extracts user ID and roles, injects them into downstream headers (`X-User-Id`, `X-User-Roles`), and routes the request.
  6. If invalid or expired, it **short-circuits immediately with HTTP 401 Unauthorized**, preventing unauthorized traffic from ever touching internal microservices.

#### 2. Production Code Example (Reactive Gateway Auth Filter)
```java
@Component
public class JwtAuthenticationGatewayFilterFactory 
        extends AbstractGatewayFilterFactory<JwtAuthenticationGatewayFilterFactory.Config> {

    private final JwtUtils jwtUtils;

    public JwtAuthenticationGatewayFilterFactory(JwtUtils jwtUtils) {
        super(Config.class);
        this.jwtUtils = jwtUtils;
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            // 1. Whitelist public paths
            if (request.getURI().getPath().contains("/api/v1/auth/")) {
                return chain.filter(exchange);
            }

            // 2. Validate Authorization header
            if (!request.getHeaders().containsKey(HttpHeaders.AUTHORIZATION)) {
                return onError(exchange, HttpStatus.UNAUTHORIZED, "Missing Authorization Header");
            }

            String authHeader = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                return onError(exchange, HttpStatus.UNAUTHORIZED, "Invalid Bearer Token Format");
            }

            String token = authHeader.substring(7);
            if (!jwtUtils.isTokenValid(token)) {
                return onError(exchange, HttpStatus.UNAUTHORIZED, "Invalid or Expired JWT Token");
            }

            // 3. Propagate claims downstream as HTTP headers
            String userId = jwtUtils.extractUserId(token);
            String roles = jwtUtils.extractRoles(token);

            ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                    .header("X-User-Id", userId)
                    .header("X-User-Roles", roles)
                    .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());
        };
    }

    private Mono<Void> onError(ServerWebExchange exchange, HttpStatus status, String err) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(status);
        return response.setComplete(); // Short-circuits request!
    }

    public static class Config {}
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We implement edge JWT authentication using a custom reactive GatewayFilter. The filter validates the JWT cryptographic signature and expiration at the gateway. If valid, it extracts the claims and injects them as trusted internal headers like `X-User-Id` and `X-User-Roles` before forwarding downstream. If invalid, it short-circuits immediately with HTTP 401, protecting internal services."
* **In Our Project / Production Context:** "By terminating JWT validation at our API Gateway, downstream services like Order and Payment don't need to duplicate token parsing or query auth servers; they simply read the trusted `X-User-Id` header injected by the gateway."
* **Pro-Tip & Interview Trap:** "Security hazard! Prevent **Header Spoofing**: Ensure your API Gateway strips any pre-existing `X-User-Id` or `X-User-Roles` headers sent by external clients before applying the filter, so malicious clients cannot impersonate other users."

---

### Q185: How do you handle CORS at API Gateway?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Cross-Origin Resource Sharing (CORS):** A browser security mechanism that restricts web applications running at one origin from requesting resources from a different origin.
* **Pre-Flight `OPTIONS` Requests:** Browsers automatically issue an HTTP `OPTIONS` request before making mutating requests (`POST`, `PUT`, `DELETE`).
* **Why at the Gateway:** In microservices, handling CORS inside individual microservices leads to inconsistent headers and broken pre-flight responses. Centralizing CORS at the API Gateway ensures consistent, unified cross-origin policies.

#### 2. Production Code Example (Spring Cloud Gateway Global CORS Configuration)
```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins:
              - "https://app.mycompany.com"
              - "https://admin.mycompany.com"
            allowed-methods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowed-headers:
              - Authorization
              - Content-Type
              - X-Correlation-ID
            allow-credentials: true
            max-age: 3600 # Cache pre-flight response for 1 hour
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We configure CORS centrally at the API Gateway level using Spring Cloud Gateway's `globalcors` configuration. The gateway handles pre-flight `OPTIONS` requests, verifies allowed origins, headers, and credentials, and returns standard CORS headers. This eliminates the need to configure CORS in individual microservices and avoids duplicate header conflicts."
* **In Our Project / Production Context:** "When our Angular frontend made cross-origin calls to different microservices, developers previously added `@CrossOrigin` annotations haphazardly, resulting in duplicate `Access-Control-Allow-Origin` headers that crashed Chrome. Centralizing CORS in the gateway solved all CORS issues permanently."
* **Pro-Tip & Interview Trap:** "Never configure `allow-credentials: true` with wildcard `allowed-origins: "*"`! Browsers will reject the response with a strict security error. When credentials (cookies/auth headers) are allowed, origins must be explicitly listed."

---

### Q186: How do you restrict access to specific Microservices?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Defense-in-Depth Service Restriction Architecture:**
  1. **Gateway Path Restriction:** Only expose public customer endpoints in gateway route configurations. Internal admin services or batch workers are omitted from gateway routes entirely.
  2. **Role-Based Path Filtering:** Enforce role checks in the gateway (e.g., rejecting `/api/admin/**` unless the user holds `ROLE_ADMIN`).
  3. **Network Isolation (Kubernetes NetworkPolicies):** In Kubernetes, apply `NetworkPolicy` resources so pods in the internal namespace only accept TCP traffic originating from the API Gateway pod IP pool.
  4. **Mutual TLS (mTLS):** Enforce Istio mTLS with service authorization policies (`AuthorizationPolicy`).

#### 2. Production Code Example (Kubernetes NetworkPolicy)
```yaml
# Kubernetes NetworkPolicy: Restricting Payment Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-restriction
spec:
  podSelector:
    matchLabels:
      app: payment-service
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: order-service     # Allow calls ONLY from Order Service
    - podSelector:
        matchLabels:
          app: api-gateway       # Allow calls ONLY from API Gateway
    ports:
    - protocol: TCP
      port: 8080
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We restrict microservice access across multiple layers: at the application layer, the API Gateway routes only whitelisted public endpoints and blocks internal paths. At the network infrastructure layer, Kubernetes NetworkPolicies restrict pod-to-pod traffic so sensitive services like Payments only accept connections from authorized pods like the Order Service."
* **In Our Project / Production Context:** "Our Billing and Reporting microservices are internal-only; they have no route mappings in Spring Cloud Gateway and are physically isolated in a private Kubernetes namespace unreachable from outside the cluster."
* **Pro-Tip & Interview Trap:** "Don't rely solely on application-level gateway routing for security! If an attacker compromises a single container inside the cluster, they could query other internal microservices directly via cluster DNS unless Kubernetes NetworkPolicies are in place."

---

### Q187: How do you implement rate limiting?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **The Token Bucket Algorithm:**
  * A bucket has a maximum capacity of tokens (burst capacity).
  * Tokens are replenished at a constant rate per second (replenish rate).
  * Every incoming request consumes 1 token.
  * If the bucket is empty, the request is rejected immediately with **HTTP 429 Too Many Requests**.
* **Spring Cloud Gateway Implementation:**
  * Uses the built-in `RequestRateLimiter` gateway filter backed by **Redis** and **Lua scripts** for atomic, distributed token evaluation across multiple gateway instances.
  * Configured with a `KeyResolver` bean that defines how rate limits are scoped (per IP address, per authenticated user ID, or per API client key).

#### 2. Production Code Example (Redis Rate Limiting)
```java
@Configuration
public class RateLimiterConfig {

    // Rate limits by Authenticated User ID (or falls back to IP address)
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            if (userId != null && !userId.isBlank()) {
                return Mono.just(userId);
            }
            return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
        };
    }
}
```

```yaml
# application.yml: Redis Token Bucket Rate Limiter
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10 # 10 requests per second
                redis-rate-limiter.burstCapacity: 20 # Allows bursts up to 20 requests
                key-resolver: "#{@userKeyResolver}"
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We implement distributed rate limiting at the API Gateway using Spring Cloud Gateway's `RequestRateLimiter` filter backed by Redis. Under the hood, it executes atomic Lua scripts implementing the Token Bucket algorithm. We define a custom `KeyResolver` bean to enforce limits per authenticated user ID or IP address, returning HTTP 429 when limits are exceeded."
* **In Our Project / Production Context:** "To prevent scraping on our catalog search API, we configured a rate limit of 20 requests per second with a burst capacity of 30 based on client IP addresses, successfully blocking aggressive web scrapers."
* **Pro-Tip & Interview Trap:** "Always return standard HTTP rate-limiting headers: `X-RateLimit-Remaining`, `X-RateLimit-Replenish-Rate`, and `Retry-After` so clients know when they can safely resume requests."

---

### Q188: What happens if API Gateway goes down?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **The Single Point of Failure (SPOF) Hazard:**
  * Because all client traffic routes through the API Gateway, if the gateway crashes, **the entire system becomes completely inaccessible to the outside world**, even if all internal microservices and databases are 100% healthy!
* **Architectural High Availability (HA) Solutions:**
  1. **Redundant Horizontal Clustering:** Never run a single gateway instance! Deploy at least 3-5 gateway pod replicas in Kubernetes distributed across multiple Availability Zones.
  2. **External Cloud Load Balancer (AWS ALB / Cloudflare):** An AWS Application Load Balancer distributes public traffic across all active gateway pods and continuously monitors their health via `/actuator/health`.
  3. **Multi-Region DNS Failover (AWS Route 53):** If an entire AWS region experiences a catastrophic gateway outage, Route 53 health checks automatically failover DNS traffic to a secondary standby region.

#### 2. High-Availability Architecture Diagram
```
+--------------------------------------------------------------------------+
|                  PUBLIC CLIENTS (Internet Traffic)                       |
+------------------------------------+-------------------------------------+
                                     |
+------------------------------------v-------------------------------------+
|         AWS ROUTE 53 (Global DNS Latency & Failover Routing)             |
+------------------------------------+-------------------------------------+
                                     |
+------------------------------------v-------------------------------------+
|        AWS APPLICATION LOAD BALANCER (Multi-AZ Health Checks)            |
+------------------+-------------------+------------------+----------------+
                   |                   |                  |
                   v                   v                  v
         +-------------------+ +-------------------+ +-------------------+
         | Gateway Pod 1     | | Gateway Pod 2     | | Gateway Pod 3     |
         | (AZ us-east-1a)   | | (AZ us-east-1b)   | | (AZ us-east-1c)   |
         +-------------------+ +-------------------+ +-------------------+
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "If the API Gateway goes down, it creates a catastrophic Single Point of Failure where all external client communication is cut off. To prevent this, we never run a single gateway instance; we deploy multiple gateway replicas across different Availability Zones behind an AWS Application Load Balancer with automated health checks, backed by Route 53 multi-region DNS failover."
* **In Our Project / Production Context:** "In our EKS cluster, our Spring Cloud Gateway runs with a minimum of 4 replicas distributed across 3 AWS Availability Zones. If an individual pod crashes, the ALB reroutes traffic in milliseconds while Kubernetes automatically provisions a replacement pod."
* **Pro-Tip & Interview Trap:** "A favorite senior architectural question: 'How do you prevent the API Gateway from becoming a SPOF?' Answer with: **'Clustering across AZs + External Cloud Load Balancer (ALB) + Health Checks + DNS Failover'**."
