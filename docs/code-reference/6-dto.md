# WorkBridge — DTOs

---

## Auth DTOs

### RegisterRequest

```java
public class RegisterRequest {
    @Email @NotBlank   private String email;
    @NotBlank @Size(min = 6) private String password;
    @NotNull           private Role   role;         // ROLE_WORKER | ROLE_CONTRACTOR
    @NotBlank          private String fullName;
    private String phoneNumber;

    // Worker-only
    private String skills;
    private String preferredCity;

    // Contractor-only
    private String businessName;
    private String address;
}
```

### LoginRequest

```java
public class LoginRequest {
    @Email @NotBlank  private String email;
    @NotBlank         private String password;
}
```

### AuthResponse

```java
public class AuthResponse {
    private String accessToken;
    private String refreshToken;
    private String email;
    private Role   role;
    private Long   userId;
    private Long   profileId; // null for ADMIN
}
```

---

## Service DTOs

### ServiceCreateRequest  (also used as ServiceUpdateRequest)

```java
public class ServiceCreateRequest {
    @NotNull           private Long      categoryId;
    @NotBlank          private String    title;
    @NotBlank          private String    description;
    @NotBlank          private String    address;
    @NotNull           private Double    latitude;
    @NotNull           private Double    longitude;
    @NotNull @Min(1)   private Integer   requiredWorkers;
    @NotNull @FutureOrPresent private LocalDate serviceDate;
    @NotNull           private LocalTime startTime;
    @NotNull           private LocalTime endTime;
}
```

### ServiceResponse

```java
public class ServiceResponse {
    private Long          id;
    private Long          contractorId;
    private String        contractorBusinessName;
    private Long          categoryId;
    private String        categoryName;
    private String        title;
    private String        description;
    private String        address;
    private Double        latitude;
    private Double        longitude;
    private Integer       requiredWorkers;
    private Integer       acceptedWorkers;
    private LocalDate     serviceDate;
    private LocalTime     startTime;
    private LocalTime     endTime;
    private String        status;          // ServiceStatus.name()
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

---

## Application DTOs

### ApplicationResponse

```java
public class ApplicationResponse {
    private Long          id;
    private Long          serviceId;
    private String        serviceTitle;
    private Long          workerId;
    private String        workerName;
    private String        status;          // ApplicationStatus.name()
    private LocalDateTime appliedAt;
    private LocalDateTime updatedAt;
}
```

*(No request body for apply — worker identity comes from JWT)*

---

## Notification DTOs

### NotificationResponse

```java
public class NotificationResponse {
    private Long          id;
    private String        title;
    private String        message;
    private boolean       read;
    private LocalDateTime createdAt;
}
```

---

## Notification Event Classes

```java
// Abstract base
public abstract class NotificationEvent extends ApplicationEvent {
    private final Long   recipientId;
    private final String title;
    private final String message;
    // constructor + getters
}

// Concrete implementation
public class InAppNotificationEvent extends NotificationEvent {
    public InAppNotificationEvent(Object source, Long recipientId, String title, String message) {
        super(source, recipientId, title, message);
    }
}
```

---

## Common Error Response (for GlobalExceptionHandler)

```java
public class ErrorResponse {
    private boolean success = false;
    private String  message;
    private int     status;
    private LocalDateTime timestamp = LocalDateTime.now();
}
```

### GlobalExceptionHandler skeleton

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(new ErrorResponse(ex.getMessage(), 404));
    }

    @ExceptionHandler(ConflictException.class)
    public ResponseEntity<ErrorResponse> handleConflict(ConflictException ex) {
        return ResponseEntity.status(409).body(new ErrorResponse(ex.getMessage(), 409));
    }

    @ExceptionHandler(InvalidStateTransitionException.class)
    public ResponseEntity<ErrorResponse> handleInvalidState(InvalidStateTransitionException ex) {
        return ResponseEntity.status(400).body(new ErrorResponse(ex.getMessage(), 400));
    }

    @ExceptionHandler(ForbiddenException.class)
    public ResponseEntity<ErrorResponse> handleForbidden(ForbiddenException ex) {
        return ResponseEntity.status(403).body(new ErrorResponse(ex.getMessage(), 403));
    }

    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<ErrorResponse> handleBadRequest(BadRequestException ex) {
        return ResponseEntity.status(400).body(new ErrorResponse(ex.getMessage(), 400));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String msg = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.status(400).body(new ErrorResponse(msg, 400));
    }
}
```

---

## Custom Exceptions to Create

```java
public class ResourceNotFoundException    extends RuntimeException { ... }
public class ConflictException            extends RuntimeException { ... }
public class ForbiddenException           extends RuntimeException { ... }
public class BadRequestException          extends RuntimeException { ... }
public class InvalidStateTransitionException extends RuntimeException { ... }
```
