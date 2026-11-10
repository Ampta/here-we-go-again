# Section 4: REST API Design, Security, JWT & Performance (Q99 – Q114)

---

### Q99: What is the difference between Path Parameter and Query Parameter?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | Path Parameter (`@PathVariable`) | Query Parameter (`@RequestParam`) |
| :--- | :--- | :--- |
| **HTTP URL Structure** | Part of the **hierarchical resource path**: `/api/orders/{id}` | Appended after the `?` delimiter as key-value pairs: `/api/orders?status=PENDING` |
| **Primary Semantic Purpose** | **Identifies a specific resource** directly. | **Filters, sorts, paginates, or modifies** a collection of resources. |
| **Optionality** | **Mandatory** by default (part of the URL route structure). | **Optional** by default (`required = false`), with fallback default values. |
| **Caching Impact** | Each unique path represents a unique cacheable resource URL. | Query string variations can be ignored or stripped by intermediate CDNs. |

#### 2. Production Code Example
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    // Path Variable: Identifies a single order resource
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDTO> getOrderById(@PathVariable("orderId") UUID orderId) {
        return ResponseEntity.ok(orderService.findById(orderId));
    }

    // Query Parameters: Filters and paginates orders
    @GetMapping
    public ResponseEntity<Page<OrderDTO>> searchOrders(
            @RequestParam(name = "status", required = false) String status,
            @RequestParam(name = "minAmount", required = false, defaultValue = "0") BigDecimal minAmount,
            @RequestParam(name = "page", defaultValue = "0") int page,
            @RequestParam(name = "size", defaultValue = "20") int size) {
        return ResponseEntity.ok(orderService.search(status, minAmount, PageRequest.of(page, size)));
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Path parameters identify a specific resource within the URI hierarchy, such as an order ID (`/orders/{id}`), and are mandatory by default. Query parameters appear after the question mark (`/orders?status=PAID&page=0`) and are used to filter, sort, and paginate collections, and are typically optional."
* **In Our Project / Production Context:** "In our eCommerce APIs, we use path variables for hierarchical resource identity (`/customers/{customerId}/addresses/{addressId}`) and query parameters for filtering catalogs by price, category, and sorting."
* **Pro-Tip & Interview Trap:** "Interview trap: 'Can a path variable be optional?' In Spring MVC, you can make a path variable optional by defining multiple route mappings (e.g., `@GetMapping({"/items", "/items/{id}"})`), but semantic REST best practices discourage optional path variables—use query parameters instead."

---

### Q100: What is API versioning?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Definition:** The practice of managing changes to an API contract over time to ensure backward compatibility and avoid breaking existing client integrations when making breaking schema changes.
* **4 Major API Versioning Strategies:**
  1. **URI Path Versioning (Most Popular):** `/api/v1/orders` vs `/api/v2/orders`. Highly visible, cache-friendly, easy to route at the API Gateway.
  2. **Custom Request Header Versioning:** `X-API-VERSION: 1` vs `X-API-VERSION: 2`. Keeps URIs clean; requires custom routing logic.
  3. **Query Parameter Versioning:** `/api/orders?version=1`. Simple for testing in browsers, but can complicate CDN caching.
  4. **Content Negotiation / Media Type Versioning (Accept Header):** `Accept: application/vnd.company.v1+json`. Follows strict RESTful HATEOAS philosophy; more complex to test in browsers.

#### 2. Production Code Example
```java
@RestController
@RequestMapping("/api")
public class VersionedOrderController {

    // 1. URI Path Versioning (Recommended in Microservices)
    @GetMapping("/v1/orders/{id}")
    public ResponseEntity<OrderV1DTO> getOrderV1(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrderV1(id));
    }

    @GetMapping("/v2/orders/{id}")
    public ResponseEntity<OrderV2DTO> getOrderV2(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrderV2(id)); // Contains new customer breakdown
    }

    // 2. Request Header Versioning
    @GetMapping(value = "/orders/{id}", headers = "X-API-VERSION=2")
    public ResponseEntity<OrderV2DTO> getOrderHeaderV2(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrderV2(id));
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "API versioning allows developers to evolve API contracts and introduce breaking changes without disrupting existing clients. Common strategies include URI path versioning, request headers, query parameters, and media type content negotiation. In our microservices, we standardize on URI path versioning (`/api/v1`) because it is explicit, easy to configure in API Gateways, and works seamlessly with CDNs."
* **In Our Project / Production Context:** "When redesigning our payment response to support multi-currency installments, we introduced `/api/v2/payments`. We supported `/v1` alongside `/v2` for six months with deprecation headers before decommissioning v1."
* **Pro-Tip & Interview Trap:** "Never bump an API version for non-breaking changes! Adding an optional new field or a new endpoint is backward-compatible and does NOT warrant a new major version."

---

### Q101: What is pagination in REST API?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Purpose:** Prevents server memory exhaustion and network latency spikes by returning large datasets in discrete, manageable subsets (pages) rather than returning 1,000,000 rows at once.
* **Pagination Strategies:**
  1. **Offset-Based Pagination:** Uses `page` and `size` parameters translated to SQL `LIMIT size OFFSET (page * size)`. Easy to implement, but suffers from performance degradation on deep pages ($O(N)$ table scans) and duplicate/missing row issues during concurrent insertions.
  2. **Keyset / Cursor-Based Pagination:** Uses the ID or timestamp of the last seen record as a cursor: `WHERE id > lastSeenId ORDER BY id ASC LIMIT 20`. Consistently fast ($O(1)$ index seek) regardless of dataset size; ideal for mobile feeds.
* **REST Response Contract:** Must return both the data items and navigation metadata (`pageNumber`, `pageSize`, `totalElements`, `totalPages`, `isLast`).

#### 2. Production Code Example
```java
// Standardized Page Response Wrapper
public record PagedResponse<T>(
    List<T> content,
    int pageNumber,
    int pageSize,
    long totalElements,
    int totalPages,
    boolean last
) {
    public static <T> PagedResponse<T> from(Page<T> page) {
        return new PagedResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isLast()
        );
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Pagination divides large collections into bounded pages to optimize server memory, database I/O, and client rendering. Offset pagination uses page and size query parameters, while cursor-based pagination uses a unique record pointer for $O(1)$ performance on large datasets. In our APIs, we wrap results in a standard `PagedResponse` envelope containing the payload and navigation metadata."
* **In Our Project / Production Context:** "On our audit log search endpoint containing 40 million records, offset pagination at page 500 caused 10-second database timeouts. We migrated to cursor-based pagination using the last `auditLogId`, reducing query times to under 20 milliseconds."
* **Pro-Tip & Interview Trap:** "Always enforce a strict `max-size` clamp on query parameters! If a client passes `?size=1000000`, your API must cap it at 100 to prevent denial-of-service memory crashes."

---

### Q102: What is the difference between synchronous and asynchronous API calls?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| Criteria | Synchronous API Communication | Asynchronous API Communication |
| :--- | :--- | :--- |
| **Client Thread State** | **Blocking:** Client thread waits idly until the server finishes processing and returns HTTP 200/201. | **Non-blocking:** Server accepts request, returns **HTTP 202 Accepted**, and processes in background. |
| **Latency & Timeout Risk** | High risk of HTTP connection timeouts if operations take >3-5 seconds. | Near-zero timeout risk; initial response returns in milliseconds. |
| **Result Delivery** | Immediate in the HTTP response body. | Delivered via **Polling** (checking `/status/{jobId}`), **Webhooks**, or **WebSocket/SSE**. |
| **Typical Use Cases** | Read queries, user login, quick checkout validations. | Heavy report generation, batch CSV imports, video transcoding. |

#### 2. Production Code Example
```java
@RestController
@RequestMapping("/api/reports")
public class AsyncReportController {
    private final AsyncReportService reportService;

    public AsyncReportController(AsyncReportService reportService) {
        this.reportService = reportService;
    }

    // Asynchronous API: Returns HTTP 202 Accepted with polling location
    @PostMapping("/generate")
    public ResponseEntity<Void> triggerReportGeneration(@RequestBody ReportRequestDTO request) {
        String jobId = UUID.randomUUID().toString();
        reportService.processReportAsync(jobId, request); // Triggers @Async background worker

        URI statusUri = ServletUriComponentsBuilder.fromCurrentContextPath()
                .path("/api/reports/status/{jobId}")
                .buildAndExpand(jobId)
                .toUri();

        return ResponseEntity.accepted().location(statusUri).build(); // 202 Accepted
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "In a synchronous API call, the client blocks waiting for the server to finish processing and return the final response, which is ideal for real-time CRUD operations. In an asynchronous API, the server validates the request and immediately returns HTTP 202 Accepted with a status tracking URI, processing the heavy work in the background. Results are delivered via polling or webhooks."
* **In Our Project / Production Context:** "Our monthly financial statement generation takes 30 seconds. Instead of a blocking HTTP call that caused gateway timeouts, we built an async API that returns HTTP 202 and notifies users via email and webhook when the PDF is ready."
* **Pro-Tip & Interview Trap:** "When designing an asynchronous API, always return an HTTP `Location` header in the `202 Accepted` response pointing to the status check endpoint, conforming to RFC 7231."

---

### Q103: What is the difference between 500 and 503?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
| HTTP Status Code | Name | Root Cause & Meaning | Client Retry Behavior |
| :--- | :--- | :--- | :--- |
| **HTTP 500** | **Internal Server Error** | An **unhandled exception or unexpected software bug** occurred inside the application code (e.g., `NullPointerException`, unhandled DB syntax error). | **Do NOT retry immediately without changes.** Retrying will likely hit the same bug. |
| **HTTP 503** | **Service Unavailable** | The server is **temporarily unable to handle the request** due to temporary overload, scheduled maintenance, or an open Circuit Breaker. | **Safe to retry after backoff.** Often accompanied by a `Retry-After: 30` header. |

#### 2. Production Code Example
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 500: Unexpected internal software defect
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception ex) {
        logger.error("Unhandled internal server bug: ", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("INTERNAL_SERVER_ERROR", "An unexpected error occurred."));
    }

    // 503: Temporary outage / Downstream Circuit Breaker tripped
    @ExceptionHandler(CallNotPermittedException.class) // Resilience4j Open Circuit
    public ResponseEntity<ErrorResponse> handleCircuitBreakerOpen(CallNotPermittedException ex) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .header(HttpHeaders.RETRY_AFTER, "60") // Retry after 60 seconds
                .body(new ErrorResponse("SERVICE_UNAVAILABLE", "Payment gateway is temporarily down. Please retry later."));
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "HTTP 500 indicates an unexpected server-side software bug or unhandled exception within the application code, and clients should not retry immediately. HTTP 503 indicates a temporary server outage, resource exhaustion, or tripped circuit breaker, signaling to clients that the service is temporarily unavailable and they should retry after a backoff period using the `Retry-After` header."
* **In Our Project / Production Context:** "When our payment service's Resilience4j circuit breaker transitions to `OPEN` due to downstream Stripe timeouts, our fallback method returns HTTP 503 with a `Retry-After: 30` header instead of generic 500s."
* **Pro-Tip & Interview Trap:** "Never return HTTP 500 for business validation failures or expected errors! A missing user is 404, invalid input is 400, and unauthenticated access is 401. 500 must be reserved strictly for unexpected technical failures."

---

### Q104: How do you generate API documentation using Swagger/OpenAPI?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Modern Standard:** In Spring Boot 3, **Springdoc OpenAPI** (`springdoc-openapi-starter-webmvc-ui`) is the modern standard (Springfox Swagger 2 is completely deprecated).
* **How it Works Under the Hood:**
  * Springdoc inspects Spring MVC annotations (`@RestController`, `@GetMapping`, `@Valid`) via reflection at runtime.
  * It automatically generates an **OpenAPI 3.0 JSON specification** at `/v3/api-docs`.
  * It serves the interactive **Swagger UI** web interface at `/swagger-ui/index.html`.
* **Standard OpenAPI Annotations:**
  * `@OpenAPIDefinition`: Defines global API metadata (title, version, terms).
  * `@Operation`: Summarizes endpoint purpose.
  * `@ApiResponse`: Documents expected HTTP response codes (200, 400, 404).
  * `@Schema`: Adds descriptions and examples to DTO fields.

#### 2. Production Code Example
```xml
<!-- pom.xml dependency -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

```java
@Tag(name = "Order Management", description = "APIs for creating and querying customer orders")
@RestController
@RequestMapping("/api/v1/orders")
public class OrderDocumentationController {

    @Operation(summary = "Get order by ID", description = "Retrieves full details of an active order")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Order retrieved successfully"),
        @ApiResponse(responseCode = "404", description = "Order not found with given ID")
    })
    @GetMapping("/{id}")
    public ResponseEntity<OrderResponseDTO> getOrderById(
            @Parameter(description = "Primary key ID of order", example = "101") 
            @PathVariable Long id) {
        return ResponseEntity.ok(orderService.findById(id));
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "In Spring Boot 3, we generate API documentation using the `springdoc-openapi` starter, which dynamically scans our controllers and models to generate an OpenAPI 3.0 spec and serves the interactive Swagger UI at `/swagger-ui.html`. We enhance documentation using `@Operation`, `@ApiResponses`, and `@Schema` annotations on our controllers and DTOs."
* **In Our Project / Production Context:** "Our Swagger UI serves as the single source of truth for our Angular frontend team, allowing them to test endpoints and inspect JSON payload schemas directly during development."
* **Pro-Tip & Interview Trap:** "In production, always restrict or disable `/swagger-ui.html` and `/v3/api-docs` using Spring Security or `springdoc.swagger-ui.enabled=false` so unauthorized external users cannot inspect your entire internal API attack surface."

---

### Q105: How do you implement JWT authentication?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **JWT Structure:** Three Base64Url-encoded parts separated by dots (`.`): `Header.Payload.Signature`.
  * **Header:** Algorithm (`HS256`, `RS256`) and token type (`JWT`).
  * **Payload:** Claims (Subject `sub`, Expiration `exp`, custom roles).
  * **Signature:** Cryptographic signature verifying that the token was not tampered with (`HMACSHA256(header + "." + payload, secret)`).
* **Spring Security Filter Pipeline:**
  1. Client sends token in header: `Authorization: Bearer <jwt_token>`.
  2. Custom `JwtAuthenticationFilter` (extending `OncePerRequestFilter`) intercepts the request.
  3. Filter extracts token, parses claims, and validates expiration and cryptographic signature using `jjwt` library.
  4. If valid, extracts username and roles, creates a `UsernamePasswordAuthenticationToken`, and sets it into `SecurityContextHolder.getContext().setAuthentication(...)`.
  5. The filter chain proceeds to the controller.

#### 2. Production Code Example
```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        final String jwt = authHeader.substring(7);
        final String username = jwtService.extractUsername(jwt);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);

            if (jwtService.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken); // Authenticated!
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We implement stateless JWT authentication using a custom `OncePerRequestFilter`. The filter extracts the Bearer token from the `Authorization` header, cryptographically validates the signature and expiration using the `jjwt` library, extracts user claims and roles, and registers a `UsernamePasswordAuthenticationToken` in Spring's `SecurityContextHolder` so Spring Security can enforce role-based access."
* **In Our Project / Production Context:** "Our Auth Service issues short-lived 15-minute access JWTs signed with RS256 private keys alongside 7-day refresh tokens stored in Redis, allowing downstream microservices to verify access tokens locally using the public key."
* **Pro-Tip & Interview Trap:** "Never store sensitive data (like plain text passwords or credit card numbers) in a JWT payload! The payload is merely Base64-encoded, NOT encrypted, and anyone can decode and view its contents."

---

### Q106: How do you secure REST APIs?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Defense-in-Depth REST API Security Checklist:**
  1. **Transport Layer Security (HTTPS/TLS):** Enforce TLS 1.3 across all communication to prevent man-in-the-middle sniffing.
  2. **Stateless Authentication:** Secure endpoints using OAuth2 / OpenID Connect (OIDC) or JWT Bearer tokens.
  3. **Role-Based Authorization (RBAC):** Apply `@PreAuthorize("hasRole('ADMIN')")` on sensitive endpoints.
  4. **Strict CORS Policy:** Restrict allowed frontend origins; never allow wildcard `Access-Control-Allow-Origin: *` with credentials.
  5. **CSRF Protection:** Disable CSRF for purely stateless token-based REST APIs, but keep it enabled if using browser session cookies.
  6. **Input Validation & Sanitization:** Use Bean Validation (`@Valid`, `@Pattern`) to block SQL injection and Cross-Site Scripting (XSS).
  7. **Rate Limiting & Throttling:** Guard against DDoS and brute force attacks using Redis Bucket4j or API Gateway rate limits.
  8. **Security Headers:** Enforce `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, and `X-Frame-Options: DENY`.

#### 2. Production Code Example
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http, JwtAuthenticationFilter jwtAuthFilter) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable) // Disabled for stateless JWT APIs
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Securing REST APIs requires a defense-in-depth strategy: enforcing HTTPS everywhere, implementing stateless JWT authentication, enforcing method-level RBAC via `@PreAuthorize`, configuring restrictive CORS policies, validating all input payloads with Jakarta validation, and applying rate-limiting at the API Gateway to prevent abuse."
* **In Our Project / Production Context:** "In our banking microservices, we enforce Spring Security with stateless JWT validation, configure Content-Security-Policy headers, and run daily OWASP dependency checks in our Jenkins pipeline."
* **Pro-Tip & Interview Trap:** "Why do we disable CSRF in stateless REST APIs? CSRF attacks exploit automatic browser cookie transmission. If your REST API uses the `Authorization: Bearer <token>` header stored in memory, browsers do not attach it automatically, making CSRF impossible."

---

### Q107: How do you implement role-based authorization?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Spring Security RBAC Architecture:**
  * **Authorities vs Roles:** In Spring Security, a Role is simply an authority prefixed with `ROLE_` (e.g., `hasRole('ADMIN')` checks for `GrantedAuthority("ROLE_ADMIN")`). `hasAuthority('READ_PRIVILEGE')` checks for the exact raw string without prefix.
  * **URL-Based Authorization:** Configured inside the `SecurityFilterChain` bean using `.requestMatchers("/admin/**").hasRole("ADMIN")`.
  * **Method-Level Authorization:** Enabled via `@EnableMethodSecurity`. Uses Spring EL expressions (`@PreAuthorize`, `@PostAuthorize`, `@Secured`) evaluated by Spring AOP before method execution.

#### 2. Production Code Example
```java
@Configuration
@EnableMethodSecurity(prePostEnabled = true) // Enables @PreAuthorize
public class MethodSecurityConfig { /* Configuration */ }

@RestController
@RequestMapping("/api/v1/accounts")
public class AccountController {

    // 1. Role-based check (Implicitly checks for authority 'ROLE_ADMIN')
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteAccount(@PathVariable Long id) {
        accountService.delete(id);
        return ResponseEntity.noContent().build();
    }

    // 2. Fine-grained Authority / Permission check
    @PutMapping("/{id}/freeze")
    @PreAuthorize("hasAuthority('ACCOUNT_WRITE') and hasRole('MANAGER')")
    public ResponseEntity<Void> freezeAccount(@PathVariable Long id) {
        accountService.freeze(id);
        return ResponseEntity.ok().build();
    }

    // 3. Dynamic SpEL Check: User can only access their own account
    @GetMapping("/{userId}/profile")
    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    public ResponseEntity<UserProfileDTO> getProfile(@PathVariable Long userId) {
        return ResponseEntity.ok(accountService.getProfile(userId));
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We implement role-based authorization in Spring Security by enabling `@EnableMethodSecurity` and using `@PreAuthorize` annotations on controller and service methods. Under the hood, Spring evaluates SpEL expressions against the `Authentication` object's `GrantedAuthority` collection in the `SecurityContext`. We also configure URL-level path matchers in `SecurityFilterChain`."
* **In Our Project / Production Context:** "We map roles from our JWT claims into Spring's `SimpleGrantedAuthority`. We use fine-grained SpEL checks like `#customerId == authentication.principal.id` to prevent Insecure Direct Object References (IDOR)."
* **Pro-Tip & Interview Trap:** "Remember that `hasRole('ADMIN')` automatically checks for `ROLE_ADMIN` in the authorities list. If your JWT token has `ADMIN` without the prefix, ensure your JWT converter prefixes it, or use `hasAuthority('ADMIN')` instead."

---

### Q108: How do you handle file uploads through REST APIs?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **HTTP Protocol:** Multipart file uploads use the `multipart/form-data` content type defined in RFC 7578.
* **Spring Boot Mechanism:**
  * Uses `StandardServletMultipartResolver`.
  * Spring maps incoming file parts to the `MultipartFile` interface.
  * Temporary files are buffered to `/tmp` if they exceed the memory threshold.
* **Production Best Practices:**
  * **Never store uploaded files on the local container disk!** In containerized microservices (Docker/Kubernetes), container disks are ephemeral and isolated across replicas.
  * **Stream directly to Cloud Object Storage:** Upload files directly to AWS S3, Google Cloud Storage, or Azure Blob Storage using SDK streaming.
  * **Validate strictly:** Verify file extension, MIME type (inspecting magic bytes with Apache Tika, not just trusting `file.getContentType()`), and enforce maximum file size limits in `application.yml`.

#### 2. Production Code Example
```yaml
# application.yml: Enforcing Multipart limits
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 15MB
```

```java
@RestController
@RequestMapping("/api/v1/documents")
public class FileUploadController {
    private final S3StorageService storageService;

    public FileUploadController(S3StorageService storageService) {
        this.storageService = storageService;
    }

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<FileUploadResponseDTO> uploadDocument(
            @RequestParam("file") MultipartFile file,
            @RequestParam("category") String category) throws IOException {

        if (file.isEmpty()) {
            throw new IllegalArgumentException("Cannot upload empty file");
        }

        // Validate allowed content types
        List<String> allowedTypes = List.of("application/pdf", "image/png", "image/jpeg");
        if (!allowedTypes.contains(file.getContentType())) {
            throw new UnsupportedMediaTypeException("Invalid file format: " + file.getContentType());
        }

        // Upload to S3 and return public/presigned URL
        String fileUrl = storageService.upload(file.getInputStream(), file.getOriginalFilename(), file.getSize());
        return ResponseEntity.status(HttpStatus.CREATED).body(new FileUploadResponseDTO(fileUrl));
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We handle file uploads using Spring's `MultipartFile` with `multipart/form-data` endpoints. We configure strict upload size limits in `application.yml`, validate MIME types using magic bytes, and stream files directly to cloud object storage like AWS S3 using `file.getInputStream()` rather than saving them to local container disks."
* **In Our Project / Production Context:** "For our KYC verification service, users upload identification documents. We validate that files are PDFs or PNGs under 5MB, upload them to an encrypted S3 bucket, and store the resulting S3 key in PostgreSQL."
* **Pro-Tip & Interview Trap:** "Never rely solely on `file.getOriginalFilename()` for validation! An attacker can upload a malicious executable named `exploit.php.jpg`. Always validate the file's underlying magic bytes and generate a random UUID for the storage key."

---

### Q109: How do you test APIs using Postman?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Postman Capabilities in Modern Development:**
  1. **Collections & Environments:** Group endpoints by microservice and define dynamic variables (`{{baseUrl}}`, `{{bearerToken}}`) for Dev, QA, and Staging.
  2. **Pre-Request Scripts:** Execute JavaScript prior to request dispatch (e.g., fetching a fresh JWT auth token or generating HMAC request signatures).
  3. **Tests Tab (Assertions):** Uses Chai.js BDD assertion syntax to validate HTTP status codes, response times, headers, and JSON body schemas.
  4. **Newman CLI:** Headless Postman runner that executes collections in Jenkins/GitHub Actions CI/CD pipelines.

#### 2. Production Code Example
```javascript
// Postman Tests Script Example:
pm.test("Status code is 201 Created", function () {
    pm.response.to.have.status(201);
});

pm.test("Response time is less than 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});

pm.test("Order payload matches schema", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property("orderId");
    pm.expect(jsonData.status).to.eql("PENDING");
    
    // Store generated orderId in environment for downstream requests:
    pm.environment.set("currentOrderId", jsonData.orderId);
});
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "In Postman, we organize endpoints into Collections structured by microservice domain and parameterize URLs using Environment variables. We write automated test assertions in the Tests tab using JavaScript to verify status codes, response payloads, and latency thresholds, and we use Newman to run these collections automatically in our CI/CD regression pipelines."
* **In Our Project / Production Context:** "Our authentication collection has a pre-request script that automatically logs in and sets `{{authToken}}` across the environment, allowing our QA engineers to run end-to-end regression suites with a single click."
* **Pro-Tip & Interview Trap:** "Mention using **Newman** (`newman run collection.json -e env.json`): interviewers love hearing that you know how to run Postman collections headlessly inside CI/CD automation pipelines."

---

### Q110: How do you generate API documentation using Swagger/OpenAPI? (Advanced Customization)

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Beyond Basics:** A production OpenAPI specification requires **Global Security Schemes (JWT Bearer Auth)**, **Grouping APIs by Domain**, and **Custom Type Mappings**.
* **Global Security Configuration:** By registering an `OpenAPI` bean with a `SecurityScheme` of type `HTTP` and scheme `bearer`, the Swagger UI renders an **'Authorize' button**, allowing developers to input their JWT token once and test all protected endpoints seamlessly.
* **API Grouping (`GroupedOpenApi`):** Divides large enterprise APIs into logical modules (e.g., Public APIs vs Internal Admin APIs).

#### 2. Production Code Example
```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        final String securitySchemeName = "BearerAuth";
        return new OpenAPI()
            .info(new Info()
                .title("Enterprise Commerce API")
                .version("v1.0")
                .description("Production REST API with JWT Bearer Authentication"))
            .addSecurityItem(new SecurityRequirement().addList(securitySchemeName))
            .components(new Components()
                .addSecuritySchemes(securitySchemeName, new SecurityScheme()
                    .name(securitySchemeName)
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")
                    .description("Enter your JWT token: Bearer <token>")));
    }

    // Grouping Endpoints in Swagger UI
    @Bean
    public GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
            .group("1-public-api")
            .pathsToMatch("/api/v1/public/**")
            .build();
    }

    @Bean
    public GroupedOpenApi adminApi() {
        return GroupedOpenApi.builder()
            .group("2-admin-api")
            .pathsToMatch("/api/v1/admin/**")
            .build();
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "In production, we customize Springdoc OpenAPI by configuring an `OpenAPI` configuration bean that registers global JWT Bearer security schemes, adding an Authorize button to Swagger UI. We also configure `GroupedOpenApi` beans to partition public customer endpoints and restricted internal admin endpoints into clean, separate Swagger dropdowns."
* **In Our Project / Production Context:** "Configuring the JWT SecurityScheme in our OpenAPI configuration allowed our QA and frontend teams to test authenticated endpoints directly in Swagger UI without having to copy-paste Bearer tokens into header fields manually."
* **Pro-Tip & Interview Trap:** "Remember that if an endpoint should be public and not require the Authorize padlock in Swagger UI, you can place `@SecurityRequirements()` (with empty value) on that specific method to override the global security scheme."

---

### Q111: How do you log API requests and responses?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **The Challenge:** In Java Servlet architecture, the `HttpServletRequest` and `HttpServletResponse` input/output streams can **only be read once**! If a servlet filter logs the request body by reading `request.getInputStream()`, the subsequent `@RestController` method will fail with an `IOException: Stream closed` or `Required request body is missing`.
* **The Solution:** Wrap the request and response in Spring's **`ContentCachingRequestWrapper`** and **`ContentCachingResponseWrapper`**.
* **Under the Hood:**
  * The caching wrappers buffer the stream bytes in memory as they are read/written.
  * After the filter chain executes, the filter reads the cached byte buffers and logs the payloads.
  * **Crucial Step:** You MUST call `responseWrapper.copyBodyToResponse()` before the filter exits, otherwise an empty HTTP response will be returned to the client!

#### 2. Production Code Example
```java
@Component
public class RequestResponseLoggingFilter extends OncePerRequestFilter {
    private static final Logger log = LoggerFactory.getLogger(RequestResponseLoggingFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        // Wrap to allow multiple reads of request and response streams
        ContentCachingRequestWrapper reqWrapper = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper respWrapper = new ContentCachingResponseWrapper(response);

        long startTime = System.currentTimeMillis();
        try {
            filterChain.doFilter(reqWrapper, respWrapper); // Execute controller logic
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            logPayload(reqWrapper, respWrapper, duration);
            respWrapper.copyBodyToResponse(); // MANDATORY: Flushes response body to client!
        }
    }

    private void logPayload(ContentCachingRequestWrapper req, ContentCachingResponseWrapper resp, long duration) {
        String reqBody = new String(req.getContentAsByteArray(), StandardCharsets.UTF_8);
        String respBody = new String(resp.getContentAsByteArray(), StandardCharsets.UTF_8);

        log.info("HTTP {} {} | Status: {} | Time: {}ms | Request: {} | Response: {}",
                req.getMethod(), req.getRequestURI(), resp.getStatus(), duration, 
                maskSensitive(reqBody), maskSensitive(respBody));
    }

    private String maskSensitive(String body) {
        return body.replaceAll("("password"\s*:\s*")[^"]+(")", "$1*****$2");
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "Logging request and response bodies requires wrapping the servlet request and response in `ContentCachingRequestWrapper` and `ContentCachingResponseWrapper` because raw servlet input streams can only be read once. We log the method, URI, status code, latency, and sanitized payloads inside a `OncePerRequestFilter`, and always call `copyBodyToResponse()` to flush the cached response back to the client."
* **In Our Project / Production Context:** "Our logging filter captures request-response payloads alongside the distributed `X-Correlation-ID`, while using regex masking to redact sensitive fields like credit card numbers and passwords before sending logs to Logstash/ELK."
* **Pro-Tip & Interview Trap:** "The #1 trap developers fall into: forgetting `respWrapper.copyBodyToResponse()`. If you omit that call, your client will receive an HTTP 200 with an empty body every time!"

---

### Q112: How do you handle duplicate requests?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **The Problem:** Network timeouts, user double-clicks, or automated client retries can send identical `POST` requests (e.g., charging a card twice).
* **The Solution: Idempotency Key Pattern:**
  1. The client generates a unique UUID (Idempotency Key) and attaches it as an HTTP header: `Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d`.
  2. The server intercepts the request in a Filter or AOP Aspect.
  3. The server attempts an atomic `SET key value NX EX 120` in **Redis** (set if not exists with a TTL).
  4. **If Key Already Exists:** A request with that key is either currently processing or has already finished. If finished, the server immediately returns the **cached response** without re-executing the business logic.
  5. **If Key is New:** The server locks the key, processes the transaction, caches the final HTTP response in Redis, and returns it to the client.

#### 2. Production Code Example
```java
@Aspect
@Component
public class IdempotentAspect {
    private final StringRedisTemplate redisTemplate;

    public IdempotentAspect(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Around("@annotation(idempotent)")
    public Object enforceIdempotency(ProceedingJoinPoint joinPoint, Idempotent idempotent) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        String idempotencyKey = request.getHeader("Idempotency-Key");

        if (idempotencyKey == null || idempotencyKey.isBlank()) {
            throw new BadRequestException("Missing mandatory Idempotency-Key header");
        }

        String redisKey = "idempotency:" + idempotencyKey;
        // Atomic acquire lock with 5 minute expiration
        Boolean isNew = redisTemplate.opsForValue().setIfAbsent(redisKey, "PROCESSING", Duration.ofMinutes(5));

        if (Boolean.FALSE.equals(isNew)) {
            String cachedResult = redisTemplate.opsForValue().get(redisKey);
            if ("PROCESSING".equals(cachedResult)) {
                throw new ConflictException("Concurrent duplicate request in progress. Please wait.");
            }
            return deserializeResponse(cachedResult); // Return previously cached response!
        }

        try {
            Object result = joinPoint.proceed();
            redisTemplate.opsForValue().set(redisKey, serializeResponse(result), Duration.ofMinutes(60));
            return result;
        } catch (Exception e) {
            redisTemplate.delete(redisKey); // Release key on business failure to permit retry
            throw e;
        }
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We prevent duplicate request processing using the Idempotency Key pattern. Clients attach a unique `Idempotency-Key` header with mutation requests. Our service checks this key in Redis using atomic `setIfAbsent()`. If the key exists, we return the cached response; if it is new, we process the transaction and store the resulting response in Redis for subsequent duplicates."
* **In Our Project / Production Context:** "In our checkout payment API, user double-clicks previously created duplicate credit card authorization charges. Implementing the Idempotency Key pattern in an AOP aspect eliminated 100% of duplicate payment charges."
* **Pro-Tip & Interview Trap:** "Always set an expiration (TTL) on idempotency keys (e.g., 24 to 48 hours) so Redis memory does not grow unbounded."

---

### Q113: How do you improve API performance?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **Multi-Layered Performance Optimization Strategies:**
  1. **Caching:**
     * **Client/CDN Caching:** Use HTTP caching headers (`Cache-Control: max-age=3600`, `ETag`).
     * **Application Caching:** Cache database reads in Redis using Spring Cache (`@Cacheable`).
  2. **Database Optimization:**
     * Eliminate N+1 queries using `JOIN FETCH` or `@EntityGraph`.
     * Add composite B-Tree indexes on frequently filtered/sorted columns.
     * Configure optimal connection pool sizes in HikariCP.
  3. **Payload Compression & Serialization:**
     * Enable GZIP compression in `application.yml`: `server.compression.enabled=true`.
     * Return targeted DTOs with only necessary fields rather than exposing full entity graphs.
  4. **Asynchronous Processing:**
     * Offload non-critical side effects (sending confirmation emails, updating audit logs) to `@Async` threads or Kafka messaging topics.
  5. **Pagination:** Enforce pagination across all collection endpoints.

#### 2. Production Code Example
```yaml
# application.yml: Performance Configurations
server:
  compression:
    enabled: true
    mime-types: text/html,text/xml,text/plain,text/css,application/javascript,application/json
    min-response-size: 2048 # 2KB

spring:
  cache:
    type: redis
    redis:
      time-to-live: 10m
```

```java
@Service
public class ProductCatalogService {
    private final ProductRepository productRepo;

    public ProductCatalogService(ProductRepository productRepo) {
        this.productRepo = productRepo;
    }

    // Spring Cache: Hits Redis first, skips database on subsequent calls
    @Cacheable(value = "products", key = "#id")
    public ProductDTO getProductById(Long id) {
        return productRepo.findById(id).map(ProductDTO::from).orElseThrow();
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We optimize REST API performance across multiple layers: caching hot reads in Redis with `@Cacheable`, enabling HTTP GZIP compression, eliminating N+1 queries with `JOIN FETCH`, enforcing pagination on all collections, and offloading asynchronous work like emails to Kafka. These combined optimizations maximize throughput and keep latencies under 50ms."
* **In Our Project / Production Context:** "Our product details endpoint was hitting the database for every single page load. Introducing a Redis cache with a 10-minute TTL and enabling GZIP response compression reduced our p99 latency from 450ms to 25ms."
* **Pro-Tip & Interview Trap:** "Always mention **HikariCP connection pool tuning**: often an API is slow not because of Java code, but because worker threads are blocked waiting for an available database connection from an undersized connection pool."

---

### Q114: How do you handle API timeout and failure?

#### 1. Core Technical Concept & Under-the-Hood Mechanism
* **The Hazard:** Calling a slow downstream service without timeouts can tie up server worker threads, leading to thread pool exhaustion, cascading failures, and total system outage.
* **Key Defensive Mechanisms:**
  1. **Strict Client Timeouts:** Always configure explicit **Connect Timeout** (time to establish TCP connection) and **Read Timeout** (time waiting for data packets) on HTTP clients (`RestTemplate`, `WebClient`, `OpenFeign`).
  2. **Circuit Breaker (Resilience4j):** Trips to `OPEN` when failure/timeout rates exceed a threshold (e.g., 50%), immediately failing fast and executing a fallback without hitting the failing downstream service.
  3. **Retry with Exponential Backoff:** Automatically retry transient network errors (e.g., HTTP 503, connect timeouts) with jitter to avoid thundering herd problems.
  4. **Graceful Fallbacks:** Return cached data, default values, or empty lists to degrade gracefully rather than crashing the user experience.

#### 2. Production Code Example
```java
@Service
public class ExternalPricingClient {
    private final RestClient restClient;

    public ExternalPricingClient() {
        // Enforcing strict timeouts on RestClient (Spring Boot 3)
        var factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofMillis(1000)); // 1 sec connect timeout
        factory.setReadTimeout(Duration.ofMillis(2500));    // 2.5 sec read timeout

        this.restClient = RestClient.builder().requestFactory(factory).build();
    }

    @CircuitBreaker(name = "pricingService", fallbackMethod = "pricingFallback")
    @Retry(name = "pricingService")
    public PricingDTO fetchDynamicPricing(String sku) {
        return restClient.get()
                .uri("https://pricing.internal/api/v1/sku/{sku}", sku)
                .retrieve()
                .body(PricingDTO.class);
    }

    // Graceful Fallback method executed when downstream times out or circuit is OPEN
    public PricingDTO pricingFallback(String sku, Throwable ex) {
        System.err.println("Downstream Pricing service failed: " + ex.getMessage() + ". Returning base price.");
        return new PricingDTO(sku, BigDecimal.valueOf(99.99), "FALLBACK_BASE_PRICE");
    }
}
```

#### 3. 2-Year Experienced Developer Interview Pitch
* **Direct Spoken Pitch:** "We handle API timeouts and downstream failures by configuring strict connect and read timeouts on our HTTP clients, combining them with Resilience4j Circuit Breakers and Retries. If a downstream service is down or timing out, the circuit breaker trips open to prevent thread starvation and executes a fallback method that returns cached or default data to maintain application availability."
* **In Our Project / Production Context:** "In our checkout service, our third-party address verification API experienced intermittent 10-second latency spikes that starved our Tomcat thread pool. Adding a 2-second read timeout and a Resilience4j circuit breaker with fallback address acceptance eliminated our thread starvation issues."
* **Pro-Tip & Interview Trap:** "Never configure retries without exponential backoff and jitter! Retrying instantly on a struggling server will cause a 'thundering herd' effect and push the struggling server into complete collapse."
