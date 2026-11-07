# Full-Stack Angular & Spring Boot Security Guide (Interview Preparation)

> **Purpose of this Document:**
> You already know the backend inside out (Spring Boot, `SecurityConfig`, `JwtUtils`, TOTP 2FA, `AuthController`, `SupportTicketController`). 
> This document explains the **Angular frontend** you built (`loan-frontend`) from scratch—translating frontend concepts into Java/Spring terms you already know, walking through your exact code, and giving you ready-to-speak interview answers.
> 
> In addition, **Section 9** provides an exhaustive, masterclass-level breakdown of **every single skill listed in your resume's Technical Skills section**, explaining:
> 1. Real application feature context in this Loan Management System
> 2. Why we need it & what problem it solves
> 3. How it works under the hood
> 4. Concrete code snippets from your application

---

## Table of Contents
1. [The "Spring Boot to Angular" Mental Dictionary](#1-the-spring-boot-to-angular-mental-dictionary)
2. [Angular Fundamentals (Zero-Knowledge Primer)](#2-angular-fundamentals-zero-knowledge-primer)
3. [Deep-Dive 1: Reactive Forms (Login, 2FA & Support Tickets)](#3-deep-dive-1-reactive-forms-login-2fa--support-tickets)
4. [Deep-Dive 2: Session & Token Management (`TokenService`)](#4-deep-dive-2-session--token-management-tokenservice)
5. [Deep-Dive 3: Protecting Routes with Route Guards](#5-deep-dive-3-protecting-routes-with-route-guards)
6. [Deep-Dive 4: HTTP Interceptors (JWT Injection & Global 401 Handling)](#6-deep-dive-4-http-interceptors-jwt-injection--global-401-handling)
7. [Deep-Dive 5: The End-to-End 2FA Flow (UI + Backend Handshake)](#7-deep-dive-5-the-end-to-end-2fa-flow-ui--backend-handshake)
8. [Top 10 Interview Questions & Ready-to-Speak Answers](#8-top-10-interview-questions--ready-to-speak-answers)
9. [Resume Technical Skills Masterclass (Every Skill Explained with Code)](#9-resume-technical-skills-masterclass)
   - [9.1 Languages](#91-languages-java-typescript-sql-javascript-html5-css3)
   - [9.2 Backend & Security](#92-backend--security-spring-boot-spring-security-spring-data-jpa-hibernate-jwt-totp-2fa-bcrypt-rbac)
   - [9.3 Frontend](#93-frontend-angular-angular-material-reactive-forms-route-guards-rxjs-signals-bootstrap)
   - [9.4 API & Web Services](#94-api--web-services-restful-apis-http-interceptors-jakarta-validation-swagger-postman)
   - [9.5 Database](#95-database-mysql)
   - [9.6 Build & Dev Tools](#96-build--dev-tools-git-maven-docker)
   - [9.7 Core Concepts](#97-core-concepts-oop-mvc-dependency-injection-session-management-exception-handling-clean-architecture)

---

## 1. The "Spring Boot to Angular" Mental Dictionary

If you know Spring Boot, you already understand 80% of Angular's architecture. They use the exact same design patterns:

| Spring Boot Concept | Angular Concept | What It Does |
| :--- | :--- | :--- |
| **`@Component` / `@RestController`** | **`@Component` (Standalone)** | Controls the view. Combines TypeScript logic, an HTML template, and CSS. |
| **`@Service` & `@Autowired`** | **`@Injectable()` & `inject()`** | Singleton service injected via Dependency Injection across components. |
| **`OncePerRequestFilter` / FilterChain** | **`HttpInterceptorFn` (`jwtInterceptor`)** | Intercepts every outgoing HTTP request to add headers (e.g. `Bearer <token>`) and catches incoming errors. |
| **`SecurityContextHolder`** | **`TokenService` (Signals + `localStorage`)** | Stores authenticated user data and tokens for the active session. |
| **`authorizeHttpRequests()` (URL rules)** | **Route Guards (`canActivate: [authGuard]`)** | Gatekeeper that checks if a user is logged in before allowing navigation to a page. |
| **DTO (`@Valid LoginRequest`)** | **`FormGroup` & `Validators` (Reactive Forms)** | Validates user input on the client before making the HTTP call. |
| **Java Streams / `CompletableFuture`** | **RxJS `Observable` (`.pipe()`, `.subscribe()`)** | Handles asynchronous HTTP requests and streaming data in Angular. |

---

## 2. Angular Fundamentals (Zero-Knowledge Primer)

### A. What is a Standalone Component?
In modern Angular (Angular 17/18+), every page or UI widget is a **Standalone Component**. It does not need complex module files (`NgModule`).
```typescript
@Component({
  selector: 'app-login',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, MatButtonModule], // Imports what it needs
  templateUrl: './login.component.html',
  styleUrl: './login.component.css'
})
export class LoginComponent {
  // Component logic here
}
```

### B. Dependency Injection: `inject()` vs `@Autowired`
In Spring Boot, you write:
```java
@Autowired private AuthService authService;
// or constructor injection with Lombok @RequiredArgsConstructor
```
In Angular, you write:
```typescript
private readonly authService = inject(AuthService);
private readonly router = inject(Router);
```

### C. What are Angular Signals? (State Management)
Think of a **Signal** as an atomic, reactive variable. Whenever its value changes, Angular automatically re-renders only the parts of the HTML that display it—no manual DOM manipulation needed!
```typescript
// Define a signal with initial value false
readonly busy = signal<boolean>(false);

// Read the value (notice the parentheses)
if (this.busy()) { ... }

// Update the value
this.busy.set(true);
```

### D. What are Observables and RxJS?
Every `HttpClient` call in Angular (e.g. `http.post()`) returns an **`Observable`**. 
- An Observable is like a conveyor belt carrying the future response from your Spring Boot API.
- You listen to it by calling `.subscribe({ next: (data) => ..., error: (err) => ... })`.
- You can transform or inspect it before subscribing using `.pipe(tap(...), finalize(...))`.

---

## 3. Deep-Dive 1: Reactive Forms (Login, 2FA & Support Tickets)

### Why Reactive Forms?
Instead of binding inputs directly to plain variables (Template-Driven), **Reactive Forms** build a TypeScript object tree (`FormGroup` containing `FormControl`s). 
- It gives programmatic control over validation, error states, and submission disabling.
- It prevents invalid requests from hitting your Spring Boot backend.

### Implementation: The Two Forms in `LoginComponent`
Located in: `loan-frontend/src/app/features/auth/pages/login/login.component.ts`

```typescript
// 1. Credentials Form (Email & Password)
readonly form: FormGroup = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  password: ['', Validators.required]
});

// 2. 2FA Form (6-digit TOTP code)
readonly mfaForm: FormGroup = this.fb.group({
  otpCode: ['', [Validators.required, Validators.pattern('^[0-9]{6}$')]]
});
```

### How Form Validation & Submission Works:
```typescript
submit(): void {
  // If user left fields empty or format is wrong, stop immediately!
  if (this.form.invalid || this.busy()) {
    this.form.markAllAsTouched(); // Highlights invalid fields in red in HTML
    return;
  }

  this.busy.set(true); // Shows progress bar
  const { email, password } = this.form.value; // Extracts clean JSON

  this.authService.login({ email, password })
    .pipe(finalize(() => this.busy.set(false)))
    .subscribe({
      next: (res) => {
        if (res.data?.mfaRequired) {
          // Switch to 2FA step!
          this.mfaData.set(res.data);
          this.step.set('2fa');
        } else {
          this.navigateAfterLogin();
        }
      },
      error: (err) => {
        this.error.set(err.error?.message || 'Login failed.');
      }
    });
}
```

### Support Ticket Form (`SupportDashboardComponent`)
Located in: `loan-frontend/src/app/features/support/pages/support-dashboard.component.ts`
```typescript
readonly ticketForm: FormGroup = this.fb.group({
  subject: ['', [Validators.required, Validators.minLength(5)]],
  category: ['General', Validators.required],
  description: ['', [Validators.required, Validators.minLength(10)]]
});
```

---

## 4. Deep-Dive 2: Session & Token Management (`TokenService`)

Located in: `loan-frontend/src/app/core/auth/token.service.ts`

In a browser, when a user refreshes the page, memory is wiped. To persist the logged-in session, tokens must be stored in browser `localStorage`. `TokenService` is the single source of truth for session state.

### Key Responsibilities of `TokenService`:
1. **Saving Session on Login / 2FA Success:**
   ```typescript
   saveSession(auth: AuthResponse): void {
     localStorage.setItem('lendflow_access_token', auth.accessToken);
     localStorage.setItem('lendflow_refresh_token', auth.refreshToken);
     localStorage.setItem('lendflow_user', JSON.stringify(auth));

     // Update reactive signals so navigation bars update immediately
     this.currentUser.set(auth);
     this.isLoggedIn.set(true);
   }
   ```
2. **Role Checking via Computed Signals:**
   ```typescript
   readonly isOfficer = computed(() => {
     const role = this.currentUser()?.role || '';
     return role.includes('OFFICER') || role.includes('ADMIN');
   });

   readonly isCustomer = computed(() => !this.isOfficer());
   ```
3. **Logout / Clear Session:**
   ```typescript
   clearSession(): void {
     localStorage.removeItem('lendflow_access_token');
     localStorage.removeItem('lendflow_refresh_token');
     localStorage.removeItem('lendflow_user');
     this.currentUser.set(null);
     this.isLoggedIn.set(false);
   }
   ```

---

## 5. Deep-Dive 3: Protecting Routes with Route Guards

In Angular, routes are defined in `app.routes.ts`. Without guards, anyone could type `http://localhost:4200/customer/dashboard` in the browser URL and view private screens.

A **Route Guard (`CanActivateFn`)** is a security function that runs **before** a route loads. If it returns `true`, the page opens. If it returns `false`, it redirects the user elsewhere.

### 1. `authGuard` (Blocks Unauthenticated Users)
Located in: `loan-frontend/src/app/core/guards/auth.guard.ts`
```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const tokenService = inject(TokenService);
  const router = inject(Router);

  // If access token exists in localStorage, grant entry!
  if (tokenService.getAccessToken()) {
    return true;
  }

  // Otherwise, kick to login page and remember where they wanted to go
  router.navigate(['/auth/login'], {
    queryParams: { returnUrl: state.url }
  });
  return false;
};
```

### 2. `customerGuard` & `officerGuard` (Role-Based Access Control)
Located in: `loan-frontend/src/app/core/guards/customer.guard.ts`
```typescript
export const customerGuard: CanActivateFn = () => {
  const tokenService = inject(TokenService);
  const router = inject(Router);

  // Only customers can access customer routes
  if (tokenService.isCustomer()) {
    return true;
  }

  // If an Officer tries to access customer routes, redirect to officer dashboard
  router.navigate(['/officer/dashboard']);
  return false;
};
```

### 3. `guestGuard` (Public Auth Pages Only)
Prevents an already logged-in user from seeing the login/register screen again. If already logged in, it redirects them straight into their dashboard.

---

## 6. Deep-Dive 4: HTTP Interceptors (JWT Injection & Global 401 Handling)

An **HTTP Interceptor** is the frontend equivalent of Spring's `OncePerRequestFilter`. Every single request sent by Angular's `HttpClient` passes through interceptors before hitting the network.

### 1. `jwtInterceptor`: Attaching the Bearer Token
Located in: `loan-frontend/src/app/core/http/jwt.interceptor.ts`

**Why clone?** In Angular, HTTP requests are immutable. To add a header, you must `.clone()` the request with new headers.

```typescript
export const jwtInterceptor: HttpInterceptorFn = (req, next) => {
  const tokenService = inject(TokenService);
  const token = tokenService.getAccessToken();

  const isApiUrl = req.url.startsWith('/api') || req.url.startsWith(environment.apiUrl);

  // DO NOT attach Bearer tokens to public endpoints (login, register, 2fa)
  const isPublicAuth =
    req.url.includes('/auth/login') ||
    req.url.includes('/auth/register') ||
    req.url.includes('/auth/verify-2fa') ||
    req.url.includes('/auth/forgot-password');

  if (token && isApiUrl && !isPublicAuth) {
    req = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}` // <--- Injects standard JWT Bearer header
      }
    });
  }

  return next(req); // Passes request down the pipeline
};
```

### 2. `errorInterceptor`: Global Exception Handling & Session Expiry
Located in: `loan-frontend/src/app/core/http/error.interceptor.ts`

Instead of writing `try/catch` in every single component, the `errorInterceptor` intercepts HTTP error status codes globally:
```typescript
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const snackBar = inject(MatSnackBar);
  const router = inject(Router);
  const tokenService = inject(TokenService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = 'An unexpected error occurred.';

      switch (error.status) {
        case 401: // Unauthorized (Token expired or invalid)
          if (!req.url.includes('/auth/login')) {
            errorMessage = 'Your session has expired. Please log in again.';
            tokenService.clearSession(); // Erase expired JWT
            router.navigate(['/auth/login']); // Auto-redirect to login
          }
          break;

        case 403: // Forbidden
          errorMessage = 'Access denied. You do not have permission.';
          break;

        case 500: // Backend crash
          errorMessage = 'Internal server error. Our team has been notified.';
          break;
      }

      // Display popup toast to the user
      snackBar.open(errorMessage, 'Close', { duration: 4500 });
      return throwError(() => error);
    })
  );
};
```

---

## 7. Deep-Dive 5: The End-to-End 2FA Flow (UI + Backend Handshake)

```
User (Browser)               Angular Frontend                  Spring Boot Backend
     |                               |                                  |
     | 1. Enter email + password     |                                  |
     |------------------------------>| POST /api/v1/auth/login          |
     |                               |--------------------------------->| Validate password
     |                               |<---------------------------------| Return: { mfaRequired: true,
     |                               |   preAuthToken: "xyz...",        |          preAuthToken, qrCodeDataUri }
     | 2. Display QR Code & OTP Form |                                  |
     |<------------------------------|                                  |
     |                               |                                  |
     | 3. Enter 6-digit TOTP code    |                                  |
     |------------------------------>| POST /api/v1/auth/verify-2fa     |
     |                               |      (preAuthToken, otpCode)     |
     |                               |--------------------------------->| Verify TOTP with SamStevens TOTP
     |                               |<---------------------------------| Return: { accessToken, refreshToken, role }
     |                               |                                  |
     |                               | 4. TokenService.saveSession()    |
     |                               |    (saves to localStorage)       |
     |                               | 5. Router.navigate('/customer')  |
     | 6. Shows Customer Dashboard   |                                  |
     |<------------------------------|                                  |
```

---

## 8. Top 10 Interview Questions & Ready-to-Speak Answers

### Q1: "How do you protect routes from unauthorized access in Angular?"
> **Your Answer:**  
> *"I implemented functional route guards using Angular's `CanActivateFn`. The master guard is `authGuard`, which checks `TokenService` for a valid JWT access token in `localStorage`. If absent, it halts navigation and redirects to `/auth/login`, passing the target URL as a query parameter (`returnUrl`) so the user is returned to their desired page after login. For role-based protection, I created `customerGuard` and `officerGuard` to ensure users cannot access dashboards outside their role permissions."*

### Q2: "What is an HTTP Interceptor, and how did you use it?"
> **Your Answer:**  
> *"An HTTP Interceptor is the frontend counterpart of a Spring Security `OncePerRequestFilter`. It intercepts all outgoing requests and incoming responses. I built two interceptors:  
> 1. `jwtInterceptor`: Checks if the target URL is our backend API and not a public auth endpoint (like `/login` or `/register`). It clones the immutable request and adds the `Authorization: Bearer <accessToken>` header.  
> 2. `errorInterceptor`: Catches backend HTTP errors globally. For instance, if the backend returns a `401 Unauthorized` because the token expired, it automatically clears the session and redirects the user to the login page with a notification."*

### Q3: "Why did you use Reactive Forms instead of Template-Driven Forms?"
> **Your Answer:**  
> *"I used Reactive Forms (`FormGroup`, `FormBuilder`, and `Validators`) because they are synchronous, explicit, and manage form state entirely in TypeScript rather than the template. This allowed strict validation before sending network requests—such as email format validation, minimum lengths, and regex patterns for the 6-digit TOTP code (`^[0-9]{6}$`). It also lets us easily disable buttons while asynchronous HTTP calls are in-flight to prevent duplicate submissions."*

### Q4: "How does the Two-Factor Authentication (2FA) flow work on the frontend?"
> **Your Answer:**  
> *"When the user submits valid credentials, the backend responds with `mfaRequired: true`, a short-lived `preAuthToken`, and a Base64 QR code data URI. The Angular component uses an Angular Signal to switch the view from `'credentials'` to `'2fa'`. We sanitize the QR code URI using Angular's `DomSanitizer` so Google Authenticator can scan it. Once the user submits the 6-digit OTP, the frontend calls `/auth/verify-2fa` with the pre-auth token. Upon successful validation, the backend issues the full access and refresh tokens, which `TokenService` stores in `localStorage`."*

### Q5: "How does the Support Ticket frontend work?"
> **Your Answer:**  
> *"The Support Ticket module features a `SupportDashboardComponent` built with Angular Material table and Reactive Forms. Customers can fill out a form with a subject, category, and issue description. Upon submission, it calls `SupportService.createTicket()`. The component maintains reactive signals for ticket list loading and toggling the ticket creation modal, linking tickets directly to the customer's authenticated account."*

### Q6: "Why do you need to clone an HTTP request in the JWT Interceptor?"
> **Your Answer:**  
> *"In Angular, `HttpRequest` objects are immutable by design to prevent side effects in the interceptor chain. To modify headers or query parameters, you must call `req.clone({ setHeaders: { ... } })` to produce a new modified copy of the request before passing it to `next(req)`."*

### Q7: "What are Angular Signals, and why are they used in modern Angular?"
> **Your Answer:**  
> *"Signals are Angular's reactive primitives introduced in Angular 16/17 for fine-grained state management. In our `TokenService`, we use signals like `currentUser = signal(...)` and computed signals like `isOfficer = computed(...)`. When the user logs in or out, updating the signal automatically notifies and re-renders only the dependent template elements without triggering heavy zone-wide change detection."*

### Q8: "How do you avoid sending JWT tokens to endpoints where they aren't needed?"
> **Your Answer:**  
> *"In our `jwtInterceptor`, we check the request URL against an excluded list of public authentication endpoints (such as `/auth/login`, `/auth/register`, `/auth/verify-email`, `/auth/forgot-password`). If the URL matches, we skip adding the Authorization header so unneeded or stale tokens are not transmitted."*

### Q9: "How do you handle expired sessions (401 errors) across the application?"
> **Your Answer:**  
> *"Inside `errorInterceptor`, we pipe through RxJS `catchError`. If an `HttpErrorResponse` has status code 401 and the failed call was not the login endpoint itself, we know the user's session expired. We call `tokenService.clearSession()`, notify the user with an Angular Material `MatSnackBar`, and route them to `/auth/login`."*

### Q10: "If an interviewer asks: 'Did you build both the backend and the frontend for this?'"
> **Your Answer:**  
> *"Yes. On the backend, I implemented the Spring Boot `AuthController` with Spring Security, JJWT, and SamStevens TOTP for 2FA, along with the `SupportTicketController`. On the frontend, I developed the corresponding Angular standalone components, managing form validation via Reactive Forms, protecting navigation via Route Guards, and automating JWT transmission via HTTP interceptors."*

---

## 9. Resume Technical Skills Masterclass

This section breaks down **every single skill** listed in your resume's Technical Skills section:
- **Application Feature**: Where and how it is used in this Loan Management System.
- **Why we need it**: The architectural problem it solves.
- **How it works**: Internal execution model and lifecycle.
- **Code Snippet**: Production-ready code directly from your project.

---

### 9.1 Languages: Java, TypeScript, SQL, JavaScript, HTML5, CSS3

#### 1. Java (8/17/21)
- **Application Feature**: Powers the entire Spring Boot backend microservice (`loan-service`), driving loan origination, authentication, TOTP generation, database transactions, and business logic.
- **Why we need it**: Enterprise stability, strong static type safety, high concurrency, and seamless integration with the Spring ecosystem. Java 17/21 brings Records, pattern matching, enhanced `java.time` APIs (`Instant`, `ChronoUnit`), and virtual threads for high-throughput I/O.
- **How it works**: Compiles to bytecode (`.class` files) executed by the Java Virtual Machine (JVM). Uses the Java Memory Model (Heap/Stack) and garbage collection for automated memory management.
- **Code Snippet**:
  ```java
  // Using Java 17+ Instant & ChronoUnit for token expiry calculation
  Instant verificationTokenExpiry = Instant.now().plus(24, ChronoUnit.HOURS);
  
  // Using Java Streams to transform Entities into DTOs
  List<SupportTicketResponse> responseList = supportTicketRepository.findAll().stream()
      .map(this::mapToResponse)
      .toList();
  ```

#### 2. TypeScript
- **Application Feature**: Powers the entire frontend (`loan-frontend`), enforcing compile-time types across models, services, guards, and components.
- **Why we need it**: Plain JavaScript fails at runtime when accessing undefined object properties (e.g. `res.data.mfaRequired`). TypeScript catches type mismatches, missing fields, and incorrect method signatures at compile time before code ever runs in the browser.
- **How it works**: A statically typed superset of JavaScript developed by Microsoft. The Angular CLI transpile TypeScript down to standard ES2022 JavaScript using `tsc`/`vite`/`esbuild`.
- **Code Snippet**:
  ```typescript
  // Strict interface matching backend Spring Boot AuthResponse DTO
  export interface AuthResponse {
    userId: number;
    email: string;
    role: 'CUSTOMER' | 'OFFICER' | 'ADMIN';
    accessToken: string;
    refreshToken: string;
    expiresIn: number;
  }
  ```

#### 3. SQL
- **Application Feature**: Stores relational customer data, user credentials, loan accounts, and support tickets in MySQL.
- **Why we need it**: Banking and loan management requires ACID transactions (Atomicity, Consistency, Isolation, Durability) to guarantee financial data integrity, foreign key relational constraints, and zero data corruption.
- **How it works**: Structured Query Language executed by the MySQL relational engine with B-tree indexing on indexed columns (`email`, `customer_id`, `ticket_id`).
- **Code Snippet**:
  ```sql
  -- Foreign key linking support ticket to customer and loan account
  CREATE TABLE support_tickets (
    ticket_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    loan_account_id BIGINT NULL,
    subject VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    status VARCHAR(50) DEFAULT 'OPEN',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_ticket_customer FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
  );
  ```

#### 4. JavaScript
- **Application Feature**: Underlying runtime execution engine for all Angular client-side code running in the user's browser.
- **Why we need it**: The only native programming language universally supported by web browsers for asynchronous networking, local storage, and DOM manipulation.
- **How it works**: Operates on a single-threaded non-blocking event loop using the call stack, Web APIs, microtask queue (Promises), and macrotask queue (`setTimeout`).

#### 5. HTML5 & CSS3
- **Application Feature**: Structures and styles the responsive portal pages (login card, 2FA QR code modal, ticket submission drawer, Angular Material tables).
- **Why we need it**: Delivers semantic UI layout, accessible form controls, clean visual hierarchy, and responsive layouts that look great on both desktop and mobile devices.
- **How it works**: The browser parses HTML5 into a Document Object Model (DOM) tree and CSS3 into a CSS Object Model (CSSOM), then calculates layout geometry and paints pixels.

---

### 9.2 Backend & Security: Spring Boot, Spring Security, Spring Data JPA, Hibernate, JWT, TOTP (2FA), BCrypt, RBAC

#### 1. Spring Boot
- **Application Feature**: The foundational framework of the `loan-service` microservice hosting REST endpoints, dependency injection, and data access.
- **Why we need it**: Eliminates massive XML boilerplate through convention-over-configuration, provides embedded Tomcat (no manual WAR deployments), and simplifies dependency versions via starter POMs.
- **How it works**: Annotating the main class with `@SpringBootApplication` enables component scanning, auto-configuration (`@EnableAutoConfiguration`), and externalized configuration (`application.yml`).
- **Code Snippet**:
  ```java
  @SpringBootApplication
  public class LoanServiceApplication {
      public static void main(String[] args) {
          SpringApplication.run(LoanServiceApplication.class, args);
      }
  }
  ```

#### 2. Spring Security
- **Application Feature**: Protects all backend endpoints against unauthorized access, enforces stateless session management, and wires the custom JWT authentication filter.
- **Why we need it**: Standardized, battle-tested framework for enterprise authentication, authorization, CSRF protection, and CORS management.
- **How it works**: Implements a chain of Servlet filters (`SecurityFilterChain`). Each HTTP request passes through filters like `UsernamePasswordAuthenticationFilter`, `JwtAuthenticationFilter`, and `FilterSecurityInterceptor`.
- **Code Snippet**:
  ```java
  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
      http
          .csrf(AbstractHttpConfigurer::disable)
          .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/api/v1/auth/**").permitAll()
              .requestMatchers("/api/v1/officer/**").hasAnyRole("OFFICER", "ADMIN")
              .anyRequest().authenticated()
          )
          .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
      return http.build();
  }
  ```

#### 3. Spring Data JPA
- **Application Feature**: Powers all database repositories (`UserRepository`, `SupportTicketRepository`, `CustomerRepository`).
- **Why we need it**: Eliminates repetitive boilerplate SQL queries. You define interfaces, and Spring Data JPA automatically synthesizes runtime queries from method names (`findByEmail`, `existsByPanNumber`).
- **How it works**: Uses dynamic Java proxies to inspect repository interfaces at startup and generate underlying JPQL queries executed by Hibernate.
- **Code Snippet**:
  ```java
  public interface SupportTicketRepository extends JpaRepository<SupportTicket, Long> {
      // Derived query method automatically parses entity properties
      List<SupportTicket> findByCustomer_CustomerId(Long customerId);
  }
  ```

#### 4. Hibernate
- **Application Feature**: Object-Relational Mapping (ORM) framework bridging Java entities (`SupportTicket`, `User`, `Customer`) to relational database tables.
- **Why we need it**: Developers manipulate clean Java objects instead of writing raw JDBC `ResultSet` parsing, SQL dialects, or foreign key joins manually.
- **How it works**: Manages a persistence context (`Session`), entity lifecycle states (Transient, Persistent, Detached, Removed), first-level caching, automatic dirty checking on `@Transactional` commits, and lazy loading (`FetchType.LAZY`).
- **Code Snippet**:
  ```java
  @Entity
  @Table(name = "support_tickets")
  @Getter @Setter
  public class SupportTicket extends BaseEntity {
      @Id
      @GeneratedValue(strategy = GenerationType.IDENTITY)
      private Long ticketId;

      @ManyToOne(fetch = FetchType.LAZY)
      @JoinColumn(name = "customer_id", nullable = false)
      private Customer customer;

      private String status = "OPEN";
  }
  ```

#### 5. JWT (JSON Web Tokens)
- **Application Feature**: Used for stateless session tokens (`accessToken`, `preAuthToken`) in the authentication module.
- **Why we need it**: Unlike traditional stateful session IDs stored in server memory (which break when scaling horizontally across multiple servers), JWTs are self-contained and cryptographically signed. Any server instance can verify the token without shared database lookups.
- **How it works**: A JWT consists of three Base64URL-encoded parts separated by dots: `Header.Payload.Signature`. The server verifies the signature using a secret HMAC-SHA256 key (`HS256`).
- **Code Snippet**:
  ```java
  public String generateToken(String email, Role role) {
      return Jwts.builder()
          .subject(email)
          .claim("role", role.name())
          .issuedAt(new Date())
          .expiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
          .signWith(getSigningKey(), Jwts.SIG.HS256)
          .compact();
  }
  ```

#### 6. TOTP (Time-Based One-Time Password / 2FA)
- **Application Feature**: Provides Two-Factor Authentication via Google Authenticator or Microsoft Authenticator during login and lost-2FA recovery.
- **Why we need it**: Protects customer financial and loan records even if their password is leaked, phished, or compromised.
- **How it works**: Implements RFC 6238. Both the server and the authenticator app share a secret Base32 key and the current Unix epoch time divided into 30-second intervals (`floor(time / 30)`). Applying HMAC-SHA1 produces the identical 6-digit passcode on both devices without requiring SMS or cellular networks.
- **Code Snippet**:
  ```java
  // Generates shared secret and QR code URI for Google Authenticator
  String secret = totpService.generateSecret();
  String qrCodeDataUri = totpService.getQrCodeDataUri(secret, user.getEmail());

  // Verifies the user's 6-digit code against current time step
  boolean isValid = totpService.verifyCode(user.getMfaSecret(), otpCode);
  ```

#### 7. BCrypt
- **Application Feature**: Hashes user passwords in `AuthServiceImpl` before saving them to the database.
- **Why we need it**: Storing plain text or weak MD5/SHA-256 hashes is vulnerable to rainbow table attacks. BCrypt includes a cryptographic salt and is deliberately computationally slow (adaptive work factor) to resist brute-force GPU attacks.
- **How it works**: Uses a salt (random string) combined with the password and runs it through multiple rounds of the Blowfish block cipher ($2^{\text{cost}}$ iterations).
- **Code Snippet**:
  ```java
  // Encoding password upon customer registration
  user.setPassword(passwordEncoder.encode(request.getPassword()));

  // Verifying password during login
  boolean matches = passwordEncoder.matches(rawPassword, user.getPassword());
  ```

#### 8. RBAC (Role-Based Access Control)
- **Application Feature**: Segregates permissions between `CUSTOMER`, `OFFICER`, and `ADMIN` roles across backend endpoints and frontend views.
- **Why we need it**: Ensures customers can only view their own loan applications and tickets, while loan officers can approve/reject loans and resolve tickets.
- **How it works**: Roles are stored in an Enum, mapped to Spring Security `GrantedAuthority` (`ROLE_CUSTOMER`), embedded in the JWT payload claims, and verified via URL matchers and route guards.
- **Code Snippet**:
  ```java
  public enum Role {
      CUSTOMER,
      OFFICER,
      ADMIN
  }
  ```

---

### 9.3 Frontend: Angular, Angular Material, Reactive Forms, Route Guards, RxJS, Signals, Bootstrap

#### 1. Angular (v17/v18+)
- **Application Feature**: Single Page Application (SPA) framework powering the loan management and customer service portal.
- **Why we need it**: Delivers instant, fluid page transitions without full-page browser reloads, combined with modular component architecture and built-in routing.
- **How it works**: Compiles HTML templates and TypeScript classes into efficient JavaScript DOM manipulation routines using the Angular rendering pipeline.

#### 2. Angular Material
- **Application Feature**: Powers polished UI components including `<mat-table>` for tickets, `<mat-form-field>` for inputs, and `MatSnackBar` for error toasts.
- **Why we need it**: Standardized, Google Material Design-compliant accessible components with built-in accessibility (A11y), responsive animations, and theme support.
- **Code Snippet**:
  ```html
  <mat-form-field appearance="outline" class="w-full">
    <mat-label>Subject</mat-label>
    <input matInput formControlName="subject" placeholder="Brief summary of your inquiry">
    <mat-error *ngIf="ticketForm.get('subject')?.hasError('required')">Subject is required</mat-error>
  </mat-form-field>
  ```

#### 3. Reactive Forms
- **Application Feature**: Manages input validation, error highlighting, and payload binding in `LoginComponent`, `RegisterComponent`, and `SupportDashboardComponent`.
- **Why we need it**: Provides programmatic, immutable, synchronous form state management in TypeScript rather than loose template bindings.
- **How it works**: Maintains a hierarchical model of `FormGroup` and `FormControl` objects. Value changes and validation status bubble up the tree synchronously.
- **Code Snippet**:
  ```typescript
  readonly form: FormGroup = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]]
  });
  ```

#### 4. Route Guards (`CanActivateFn`)
- **Application Feature**: Protects private routes (`/customer/*`, `/officer/*`) from unauthenticated access and ensures role separation.
- **Why we need it**: Prevents unauthorized users from navigating directly to restricted portal URLs via the browser address bar.
- **How it works**: Intercepts Angular Router navigation before route activation; returns `true` to permit or `false` with a redirect (`router.navigate`).
- **Code Snippet**:
  ```typescript
  export const authGuard: CanActivateFn = (route, state) => {
    const tokenService = inject(TokenService);
    const router = inject(Router);
    if (tokenService.getAccessToken()) return true;
    router.navigate(['/auth/login'], { queryParams: { returnUrl: state.url } });
    return false;
  };
  ```

#### 5. RxJS (Reactive Extensions for JavaScript)
- **Application Feature**: Orchestrates asynchronous HTTP communication, error interception, cancellation, and response pipelining.
- **Why we need it**: Handles asynchronous event streams cleanly without messy callback pyramids or unhandled promise rejections.
- **How it works**: Uses the Publisher/Subscriber pattern (`Observable` emits values, `Observer` subscribes) and functional operators (`pipe`, `tap`, `catchError`, `finalize`).
- **Code Snippet**:
  ```typescript
  this.authService.login({ email, password })
    .pipe(
      tap((res) => console.log('Login response:', res)),
      finalize(() => this.busy.set(false))
    )
    .subscribe({
      next: (res) => this.handleSuccess(res),
      error: (err) => this.errorMessage.set(err.message)
    });
  ```

#### 6. Angular Signals
- **Application Feature**: Manages fine-grained reactive state (e.g. `currentUser`, `isLoggedIn`, `busy`, `step`) in `TokenService` and `LoginComponent`.
- **Why we need it**: Replaces bulky zone-wide change detection with surgical DOM updates, improving rendering performance and eliminating state desynchronization.
- **How it works**: Signals wrap values in a reactive getter/setter tracking graph. When `.set()` is called, Angular updates only the exact DOM nodes reading that signal.
- **Code Snippet**:
  ```typescript
  // Atomic reactive signal
  readonly step = signal<'credentials' | '2fa'>('credentials');
  
  // Computed reactive signal derived from currentUser
  readonly isOfficer = computed(() => {
    return this.currentUser()?.role === 'OFFICER';
  });
  ```

#### 7. Bootstrap / Utility CSS
- **Application Feature**: Provides flexible grid layouts, spacing utilities, and responsive breakpoints across customer and officer dashboards.
- **Why we need it**: Rapid responsive layout structuring (`d-flex`, `justify-content-between`, `mb-3`, `row`, `col-md-6`) complementing Angular Material components.

---

### 9.4 API & Web Services: RESTful APIs, HTTP Interceptors, Jakarta Validation, Swagger, Postman

#### 1. RESTful APIs
- **Application Feature**: Bridges the Angular frontend with the Spring Boot backend microservice using standard endpoints (`/api/v1/auth/**`, `/api/v1/tickets/**`).
- **Why we need it**: Universal, decoupled communication between client and server using stateless HTTP verbs and standard JSON data structures.
- **How it works**: Uses standard HTTP verbs:
  - `POST /api/v1/tickets` $\rightarrow$ Create resource
  - `GET /api/v1/tickets/{id}` $\rightarrow$ Retrieve resource
  - `PUT /api/v1/tickets/{id}` $\rightarrow$ Complete update
  - `PATCH /api/v1/tickets/{id}/status` $\rightarrow$ Partial status update
  - `DELETE /api/v1/tickets/{id}` $\rightarrow$ Remove resource
- **Code Snippet**:
  ```java
  @RestController
  @RequestMapping("/api/v1/tickets")
  public class SupportTicketController {
      @PostMapping
      public ResponseEntity<ApiResponse<SupportTicketResponse>> createTicket(@Valid @RequestBody SupportTicketRequest request) {
          return new ResponseEntity<>(new ApiResponse<>(true, "Created", service.createTicket(request)), HttpStatus.CREATED);
      }
  }
  ```

#### 2. HTTP Interceptors
- **Application Feature**: `jwtInterceptor` attaches Bearer tokens; `errorInterceptor` catches global 401/403/500 errors and redirects expired sessions.
- **Why we need it**: Centralizes cross-cutting concerns (token injection, error logging, session expiry) in one place instead of repeating logic across every API call.
- **How it works**: Chain-of-responsibility function wrapping Angular's `HttpHandler`. Intercepts outgoing `HttpRequest` and incoming `HttpResponse`.
- **Code Snippet**:
  ```typescript
  export const jwtInterceptor: HttpInterceptorFn = (req, next) => {
    const token = inject(TokenService).getAccessToken();
    if (token && !req.url.includes('/auth/')) {
      req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
    }
    return next(req);
  };
  ```

#### 3. Jakarta Validation
- **Application Feature**: Validates incoming request payloads on the backend (`RegisterRequest`, `LoginRequest`, `SupportTicketRequest`).
- **Why we need it**: Defends against malformed, malicious, or incomplete requests before executing business logic or hitting the database.
- **How it works**: Annotations placed on DTO fields are inspected by Hibernate Validator when `@Valid` is present on the `@RequestBody` in the controller. If invalid, Spring throws `MethodArgumentNotValidException`.
- **Code Snippet**:
  ```java
  public class SupportTicketRequest {
      @NotNull(message = "Customer ID is required")
      private Long customerId;

      @NotBlank(message = "Subject must not be blank")
      @Size(min = 5, max = 150, message = "Subject must be between 5 and 150 characters")
      private String subject;

      @NotBlank(message = "Description must not be blank")
      private String description;
  }
  ```

#### 4. Swagger / OpenAPI
- **Application Feature**: Generates live, interactive API documentation for all endpoints in `loan-service`.
- **Why we need it**: Enables seamless collaboration across backend and frontend developers to inspect schemas, request parameters, and response contracts without reading code.
- **How it works**: Annotations (`@Operation`, `@Tag`) are scanned at runtime by `springdoc-openapi` to generate a standard JSON specification exposed at `/swagger-ui/index.html`.

#### 5. Postman
- **Application Feature**: Used during development to test REST endpoints, verify 2FA OTP codes, test refresh token rotation, and simulate edge cases.
- **Why we need it**: Allows testing backend controllers independently before frontend screens are created, with automated collection runs and environment variable switching (`{{baseUrl}}`, `{{bearerToken}}`).

---

### 9.5 Database: MySQL

#### MySQL
- **Application Feature**: The relational database engine storing all persistent tables (`customers`, `users`, `support_tickets`, `loans`).
- **Why we need it**: Reliable, open-source relational DBMS with InnoDB storage engine supporting ACID transactions, row-level locking, and referential integrity.
- **How it works**: Handles SQL parsing, query execution planning, InnoDB buffer pool caching, redo/undo logging for durability, and indexing for $O(\log N)$ lookups on primary keys and foreign keys.

---

### 9.6 Build & Dev Tools: Git, Maven, Docker

#### 1. Git
- **Application Feature**: Version control managing team development across branches (`feature/auth`, `feature/tickets`, `main`).
- **Why we need it**: Distributed revision tracking, code review via pull requests, conflict resolution, and safe rollback of changes.
- **Commands used**: `git status`, `git add .`, `git commit -m "feat: add 2FA verification flow"`, `git push origin main`.

#### 2. Maven
- **Application Feature**: Dependency management and build tool defined in `pom.xml`.
- **Why we need it**: Resolves transitive dependencies automatically from Maven Central, enforces consistent build lifecycles, and packages the application into an executable JAR.
- **How it works**: Executes standard lifecycle phases: `validate` $\rightarrow$ `compile` $\rightarrow$ `test` $\rightarrow$ `package` $\rightarrow$ `verify` $\rightarrow$ `install` $\rightarrow$ `deploy`.
- **Code Snippet**:
  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt-api</artifactId>
      <version>0.13.0</version>
  </dependency>
  ```

#### 3. Docker (Basic)
- **Application Feature**: Containerizes the application and runs MySQL in an isolated environment.
- **Why we need it**: Solves the classic "it works on my machine" problem by bundling the application, JVM runtime, and configuration into a lightweight, portable container image.
- **How it works**: Packages Linux kernel namespaces and cgroups to isolate process execution, memory, and filesystem without full VM overhead.
- **Code Snippet**:
  ```dockerfile
  FROM eclipse-temurin:21-jre-alpine
  WORKDIR /app
  COPY target/loan-service-0.0.1-SNAPSHOT.jar app.jar
  EXPOSE 8080
  ENTRYPOINT ["java", "-jar", "app.jar"]
  ```

---

### 9.7 Core Concepts: OOP, MVC, Dependency Injection, Session Management, Exception Handling, Clean Architecture

#### 1. OOP (Object-Oriented Programming)
- **Application Feature**: Underpins all Java and TypeScript code in the application.
  - **Encapsulation**: Private fields in entities and services exposed via getters/setters or methods.
  - **Inheritance**: `BaseEntity` providing `createdAt` and `updatedAt` inherited by `SupportTicket` and `Customer`.
  - **Polymorphism**: `AuthService` interface implemented by `AuthServiceImpl`.
  - **Abstraction**: Hiding low-level database operations behind `JpaRepository` interfaces.

#### 2. MVC (Model-View-Controller)
- **Application Feature**: Architectural separation of concerns across the stack:
  - **Model**: Java Entities (`SupportTicket`, `Customer`) and Angular interfaces (`SupportTicketResponse`).
  - **View**: Angular HTML templates (`support-dashboard.component.html`).
  - **Controller**: Spring Boot `@RestController` handling HTTP requests and responses.

#### 3. Dependency Injection (IoC - Inversion of Control)
- **Application Feature**: Managed objects are injected where needed rather than created manually with `new`.
- **Why we need it**: Decouples classes, facilitates unit testing with mocks, and manages singleton lifecycles.
- **How it works**:
  - In Spring Boot: Handled by the Spring ApplicationContext container (`@RequiredArgsConstructor` / `@Autowired`).
  - In Angular: Handled by the hierarchical Injector system using the modern `inject()` function.

#### 4. Session Management
- **Application Feature**: Stateless session handling using JWT access tokens and refresh tokens.
- **Why we need it**: Allows horizontally scalable microservices where any instance can validate user requests without reading server-side session stores.
- **How it works**:
  - Server sets `SessionCreationPolicy.STATELESS`.
  - Client stores tokens in `localStorage` via `TokenService`.
  - Client attaches tokens automatically via `jwtInterceptor`.
  - Expired tokens trigger refresh token exchange or redirect to login via `errorInterceptor`.

#### 5. Exception Handling
- **Application Feature**: Centralized exception handling on both backend and frontend.
- **Why we need it**: Prevents raw stack traces from leaking to the user and guarantees consistent JSON error structures.
- **How it works**:
  - Backend: `@RestControllerAdvice` intercepts `ResourceNotFoundException`, `ResourceAlreadyExistsException`, and validation errors, returning structured `ApiResponse<T>`.
  - Frontend: `errorInterceptor` intercepts HTTP error status codes and triggers user-friendly `MatSnackBar` notifications.
- **Code Snippet**:
  ```java
  @RestControllerAdvice
  public class GlobalExceptionHandler {
      @ExceptionHandler(ResourceNotFoundException.class)
      public ResponseEntity<ApiResponse<Void>> handleNotFound(ResourceNotFoundException ex) {
          return ResponseEntity.status(HttpStatus.NOT_FOUND)
              .body(new ApiResponse<>(false, ex.getMessage(), null));
      }
  }
  ```

#### 6. Clean Architecture / Layered Architecture
- **Application Feature**: Strict separation into distinct functional layers:
  $$\text{Client (Angular)} \longrightarrow \text{Controller Layer} \longrightarrow \text{Service Layer} \longrightarrow \text{Repository Layer} \longrightarrow \text{Database (MySQL)}$$
- **Why we need it**: Each layer has a single responsibility. Business logic lives strictly in services, never in controllers or repositories. Changes to the database or UI do not break core business rules.
