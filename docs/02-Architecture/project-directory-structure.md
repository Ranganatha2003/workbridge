# WorkBridge — Best-Practice Project Directory Structure

## Stack: Spring Boot · Spring Security · JWT · Spring Data JPA · MySQL

---

## Backend Directory Tree

```text
workbridge-backend/
│
├── pom.xml
├── Dockerfile
│
└── src/
    ├── main/
    │   ├── java/
    │   │   └── com/workbridge/
    │   │       │
    │   │       ├── WorkBridgeApplication.java           ← @SpringBootApplication + @EnableJpaAuditing + @EnableAsync
    │   │       │
    │   │       ├── config/                              ← App-level beans & configs
    │   │       │   ├── AsyncConfig.java                 ← @EnableAsync thread-pool setup
    │   │       │   └── CorsConfig.java                  ← Global CORS bean
    │   │       │
    │   │       ├── security/                            ← Spring Security + JWT plumbing
    │   │       │   ├── SecurityConfig.java              ← Filter chain, CSRF off, session stateless
    │   │       │   ├── JwtAuthenticationFilter.java     ← OncePerRequestFilter – validates Bearer token
    │   │       │   ├── JwtService.java                  ← generate / validate access & refresh tokens
    │   │       │   ├── CustomUserDetailsService.java    ← UserDetailsService – loads by email
    │   │       │   ├── evaluator/
    │   │       │   │   ├── ServiceSecurityEvaluator.java     ← @Component("serviceSecurity")
    │   │       │   │   └── ApplicationSecurityEvaluator.java ← @Component("applicationSecurity")
    │   │       │
    │   │       ├── entity/                              ← JPA entities (domain model)
    │   │       │   ├── enums/
    │   │       │   │   ├── Role.java                    ← ROLE_WORKER | ROLE_CONTRACTOR | ROLE_ADMIN
    │   │       │   │   ├── ContractorStatus.java        ← ACTIVE | BLOCKED
    │   │       │   │   ├── ServiceStatus.java           ← DRAFT | OPEN | FILLED | IN_PROGRESS | COMPLETED | CANCELLED | EXPIRED
    │   │       │   │   ├── ApplicationStatus.java       ← APPLIED | ACCEPTED | REJECTED | CANCELLED
    │   │       │   │   ├── PaymentStatus.java           ← INITIATED | PENDING | SUCCESS | FAILED | REFUNDED
    │   │       │   │   └── PayoutStatus.java            ← PENDING | PROCESSING | PAID | FAILED
    │   │       │   │
    │   │       │   ├── User.java                        ← users table
    │   │       │   ├── WorkerProfile.java               ← worker_profiles table
    │   │       │   ├── ContractorProfile.java           ← contractor_profiles table
    │   │       │   ├── ServiceCategory.java             ← service_categories table
    │   │       │   ├── ServiceListing.java              ← services table (with @Version for optimistic lock)
    │   │       │   ├── Application.java                 ← applications table (unique: service_id + worker_id)
    │   │       │   ├── Notification.java                ← notifications table
    │   │       │   ├── Payment.java                     ← payments table
    │   │       │   └── Payout.java                      ← payouts table
    │   │       │
    │   │       ├── repository/                          ← Spring Data JPA interfaces
    │   │       │   ├── UserRepository.java
    │   │       │   ├── WorkerProfileRepository.java
    │   │       │   ├── ContractorProfileRepository.java
    │   │       │   ├── ServiceCategoryRepository.java
    │   │       │   ├── ServiceListingRepository.java    ← custom: findByIdWithLock, findOpenServicesWithinRadius
    │   │       │   ├── ApplicationRepository.java       ← custom: existsByWorkerIdAndServiceListingId
    │   │       │   ├── NotificationRepository.java      ← custom: findByUserIdOrderByCreatedAtDesc
    │   │       │   ├── PaymentRepository.java           ← custom: findByGatewayOrderId
    │   │       │   └── PayoutRepository.java
    │   │       │
    │   │       ├── service/                             ← Business logic layer (@Service beans)
    │   │       │   ├── AuthService.java                 ← register, login → issues JWT
    │   │       │   ├── ServiceListingService.java       ← CRUD + search/radius + status transitions
    │   │       │   ├── ApplicationService.java          ← apply, accept (pessimistic lock), reject, cancel
    │   │       │   ├── NotificationService.java         ← createNotification, getMyNotifications, markAsRead
    │   │       │   ├── PaymentService.java              ← initiatePayment, confirmPayment, failPayment, processPayout
    │   │       │   └── listener/
    │   │       │       └── NotificationEventListener.java  ← @Async @EventListener(InAppNotificationEvent)
    │   │       │
    │   │       ├── controller/                          ← REST API layer (@RestController)
    │   │       │   ├── AuthController.java              ← POST /api/auth/register  POST /api/auth/login
    │   │       │   ├── ServiceController.java           ← CRUD /api/services  + radius search
    │   │       │   ├── ApplicationController.java       ← /api/services/{id}/applications  + accept/reject/cancel
    │   │       │   ├── NotificationController.java      ← GET /api/notifications  + mark-read
    │   │       │   └── PaymentController.java           ← initiate / confirm / fail / payout
    │   │       │
    │   │       ├── dto/                                 ← Request & Response objects (no JPA annotations)
    │   │       │   ├── request/
    │   │       │   │   ├── RegisterRequest.java
    │   │       │   │   ├── LoginRequest.java
    │   │       │   │   ├── ServiceCreateRequest.java
    │   │       │   │   └── ServiceUpdateRequest.java
    │   │       │   └── response/
    │   │       │       ├── AuthResponse.java            ← accessToken, refreshToken, email, role, userId, profileId
    │   │       │       ├── ServiceResponse.java
    │   │       │       ├── ApplicationResponse.java
    │   │       │       └── NotificationResponse.java
    │   │       │
    │   │       ├── event/                               ← Spring ApplicationEvents for async notifications
    │   │       │   └── InAppNotificationEvent.java      ← recipientId, title, message
    │   │       │
    │   │       └── exception/                           ← Custom exceptions + global handler
    │   │           ├── BadRequestException.java         ← 400
    │   │           ├── ResourceNotFoundException.java   ← 404
    │   │           ├── ConflictException.java           ← 409
    │   │           ├── ForbiddenException.java          ← 403
    │   │           ├── InvalidStateTransitionException.java ← 422 (business rule violation)
    │   │           └── GlobalExceptionHandler.java      ← @RestControllerAdvice
    │   │
    │   └── resources/
    │       ├── application.properties                   ← DB, JWT secrets, Hibernate DDL
    │       └── application-prod.properties              ← Production overrides
    │
    └── test/
        └── java/
            └── com/workbridge/
                ├── service/
                │   ├── AuthServiceTest.java
                │   ├── ServiceListingServiceTest.java
                │   └── ApplicationServiceTest.java
                └── controller/
                    ├── AuthControllerTest.java
                    └── ServiceControllerTest.java
```

---

## Package Responsibility Summary

| Package | Contains | Rule |
|---|---|---|
| `config` | `AsyncConfig`, `CorsConfig` | App-wide beans only; no business logic |
| `security` | `SecurityConfig`, `JwtService`, `JwtAuthenticationFilter`, `CustomUserDetailsService`, `evaluator/` | Security plumbing and `@PreAuthorize` evaluators |
| `entity` | All JPA `@Entity` classes + all enums | No service or controller logic |
| `repository` | All `JpaRepository` interfaces | Database access only; no business logic |
| `service` | All `@Service` classes + `listener/` | All business logic; calls repositories; publishes events |
| `controller` | All `@RestController` classes | HTTP in/out only; delegates to services |
| `dto` | `request/` and `response/` POJOs | No persistence annotations |
| `event` | Spring `ApplicationEvent` subclasses | Decouples async notification triggering |
| `exception` | Custom exceptions + `GlobalExceptionHandler` | Centralised error responses |

---

## Controller → Service → Repository Mapping

| Controller | Calls Service | Service Uses Repository |
|---|---|---|
| `AuthController` | `AuthService` | `UserRepository`, `WorkerProfileRepository`, `ContractorProfileRepository` |
| `ServiceController` | `ServiceListingService` | `ServiceListingRepository`, `ContractorProfileRepository`, `ServiceCategoryRepository` |
| `ApplicationController` | `ApplicationService` | `ApplicationRepository`, `ServiceListingRepository`, `WorkerProfileRepository` |
| `NotificationController` | `NotificationService` | `NotificationRepository`, `UserRepository` |
| `PaymentController` | `PaymentService` | `PaymentRepository`, `PayoutRepository`, `ServiceListingRepository`, `ContractorProfileRepository`, `ApplicationRepository` |

---

## Key Design Rules

1. **Controllers are thin** — validate input, call one service method, return `ResponseEntity`.
2. **Services own all business logic** — state transitions, capacity checks, event publishing.
3. **Repositories are interfaces only** — custom queries via `@Query` or method naming; no SQL in services.
4. **DTOs never expose JPA entities** — always map entity → response DTO before returning to client.
5. **Pessimistic locking lives in the service** — `serviceListingRepository.findByIdWithLock(id)` called inside `@Transactional`.
6. **Async notifications via events** — service calls `eventPublisher.publishEvent(new InAppNotificationEvent(...))` and returns immediately; `NotificationEventListener` handles persistence in the background.
7. **Enums are in `entity/enums/`** — shared across entities, services and DTOs without circular deps.
8. **`@PreAuthorize` references beans in `security/evaluator/`** — keeps authorization logic out of controllers.

---

*End of WorkBridge Project Directory Structure*
