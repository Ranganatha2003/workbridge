# WorkBridge — Controllers

Base path for all: `/api`

---

## 1. AuthController  →  `/api/auth`

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    // POST /api/auth/register → 201 Created
    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(@Valid @RequestBody RegisterRequest req) {
        return ResponseEntity.status(201).body(authService.register(req));
    }

    // POST /api/auth/login → 200 OK
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest req) {
        return ResponseEntity.ok(authService.login(req));
    }
}
```

---

## 2. ServiceController  →  `/api/services`

```java
@RestController
@RequestMapping({"/api/services", "/services"})
public class ServiceController {

    // POST /api/services → 201 (CONTRACTOR)
    @PostMapping
    @PreAuthorize("hasRole('CONTRACTOR')")
    public ResponseEntity<ServiceResponse> create(
            @Valid @RequestBody ServiceCreateRequest req,
            Authentication auth) {
        return ResponseEntity.status(201).body(serviceListingService.createService(req, auth.getName()));
    }

    // GET /api/services → 200 (public)
    @GetMapping
    public ResponseEntity<List<ServiceResponse>> getAll() {
        return ResponseEntity.ok(serviceListingService.getAllServices());
    }

    // GET /api/services/{id} → 200 (public)
    @GetMapping("/{id}")
    public ResponseEntity<ServiceResponse> getById(@PathVariable Long id) {
        return ResponseEntity.ok(serviceListingService.getById(id));
    }

    // GET /api/services/my-services → 200 (CONTRACTOR)
    @GetMapping("/my-services")
    @PreAuthorize("hasRole('CONTRACTOR')")
    public ResponseEntity<List<ServiceResponse>> getMyServices(Authentication auth) {
        return ResponseEntity.ok(serviceListingService.getMyServices(auth.getName()));
    }

    // GET /api/services/search/radius?lat=&lon=&radiusKm= → 200 (public)
    @GetMapping("/search/radius")
    public ResponseEntity<List<ServiceResponse>> searchByRadius(
            @RequestParam double lat,
            @RequestParam double lon,
            @RequestParam double radiusKm) {
        return ResponseEntity.ok(serviceListingService.searchByRadius(lat, lon, radiusKm));
    }

    // PUT /api/services/{id} → 200 (ADMIN or owner CONTRACTOR)
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or @serviceSecurity.isServiceOwner(#id, authentication.name)")
    public ResponseEntity<ServiceResponse> update(
            @PathVariable Long id,
            @Valid @RequestBody ServiceUpdateRequest req) {
        return ResponseEntity.ok(serviceListingService.updateService(id, req));
    }

    // PUT /api/services/{id}/cancel → 200 (ADMIN or owner CONTRACTOR)
    @PutMapping("/{id}/cancel")
    @PreAuthorize("hasRole('ADMIN') or @serviceSecurity.isServiceOwner(#id, authentication.name)")
    public ResponseEntity<ServiceResponse> cancel(@PathVariable Long id) {
        return ResponseEntity.ok(serviceListingService.cancelService(id));
    }

    // PUT /api/services/{id}/complete → 200 (ADMIN or owner CONTRACTOR)
    @PutMapping("/{id}/complete")
    @PreAuthorize("hasRole('ADMIN') or @serviceSecurity.isServiceOwner(#id, authentication.name)")
    public ResponseEntity<ServiceResponse> complete(@PathVariable Long id) {
        return ResponseEntity.ok(serviceListingService.completeService(id));
    }
}
```

---

## 3. ApplicationController  →  `/api`

```java
@RestController
@RequestMapping("/api")
public class ApplicationController {

    // POST /api/services/{serviceId}/applications → 201 (WORKER)
    @PostMapping("/services/{serviceId}/applications")
    @PreAuthorize("hasRole('WORKER')")
    public ResponseEntity<ApplicationResponse> apply(
            @PathVariable Long serviceId, Authentication auth) {
        return ResponseEntity.status(201).body(applicationService.apply(serviceId, auth.getName()));
    }

    // GET /api/applications/my-applications → 200 (WORKER)
    @GetMapping("/applications/my-applications")
    @PreAuthorize("hasRole('WORKER')")
    public ResponseEntity<List<ApplicationResponse>> myApplications(Authentication auth) {
        return ResponseEntity.ok(applicationService.getMyApplications(auth.getName()));
    }

    // GET /api/services/{serviceId}/applications → 200 (ADMIN or owner CONTRACTOR)
    @GetMapping("/services/{serviceId}/applications")
    @PreAuthorize("hasRole('ADMIN') or @serviceSecurity.isServiceOwner(#serviceId, authentication.name)")
    public ResponseEntity<List<ApplicationResponse>> getForService(@PathVariable Long serviceId) {
        return ResponseEntity.ok(applicationService.getApplicationsForService(serviceId));
    }

    // PUT /api/applications/{id}/accept → 200 (ADMIN or owner CONTRACTOR)
    @PutMapping("/applications/{id}/accept")
    @PreAuthorize("hasRole('ADMIN') or @applicationSecurity.isServiceOwnerOfApplication(#id, authentication.name)")
    public ResponseEntity<ApplicationResponse> accept(@PathVariable Long id) {
        return ResponseEntity.ok(applicationService.accept(id));
    }

    // PUT /api/applications/{id}/reject → 200 (ADMIN or owner CONTRACTOR)
    @PutMapping("/applications/{id}/reject")
    @PreAuthorize("hasRole('ADMIN') or @applicationSecurity.isServiceOwnerOfApplication(#id, authentication.name)")
    public ResponseEntity<ApplicationResponse> reject(@PathVariable Long id) {
        return ResponseEntity.ok(applicationService.reject(id));
    }

    // PUT /api/applications/{id}/cancel → 200 (owning WORKER)
    @PutMapping("/applications/{id}/cancel")
    @PreAuthorize("@applicationSecurity.isApplicationOwner(#id, authentication.name)")
    public ResponseEntity<ApplicationResponse> cancel(@PathVariable Long id) {
        return ResponseEntity.ok(applicationService.cancel(id));
    }
}
```

---

## 4. NotificationController  →  `/api/notifications`

```java
@RestController
@RequestMapping("/api/notifications")
public class NotificationController {

    // GET /api/notifications → 200 (any authenticated)
    @GetMapping
    public ResponseEntity<List<NotificationResponse>> getAll(Authentication auth) {
        return ResponseEntity.ok(notificationService.getMyNotifications(auth.getName()));
    }

    // GET /api/notifications/unread → 200 (any authenticated)
    @GetMapping("/unread")
    public ResponseEntity<List<NotificationResponse>> getUnread(Authentication auth) {
        return ResponseEntity.ok(notificationService.getMyUnreadNotifications(auth.getName()));
    }

    // PUT /api/notifications/{id}/read → 200 (owner only)
    @PutMapping("/{id}/read")
    public ResponseEntity<Void> markAsRead(@PathVariable Long id, Authentication auth) {
        notificationService.markAsRead(id, auth.getName());
        return ResponseEntity.ok().build();
    }
}
```

---

## 5. PaymentController  →  `/api/payments`

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    // POST /api/payments/initiate?serviceId=&amount= → 200 (owner CONTRACTOR)
    @PostMapping("/initiate")
    @PreAuthorize("@serviceSecurity.isServiceOwner(#serviceId, authentication.name)")
    public ResponseEntity<Payment> initiate(
            @RequestParam Long serviceId,
            @RequestParam BigDecimal amount,
            Authentication auth) {
        return ResponseEntity.ok(paymentService.initiatePayment(serviceId, amount, auth.getName()));
    }

    // POST /api/payments/confirm?gatewayOrderId= → 200 (CONTRACTOR)
    @PostMapping("/confirm")
    @PreAuthorize("hasRole('CONTRACTOR')")
    public ResponseEntity<Payment> confirm(@RequestParam String gatewayOrderId) {
        return ResponseEntity.ok(paymentService.confirmPayment(gatewayOrderId));
    }

    // POST /api/payments/fail?gatewayOrderId= → 200 (CONTRACTOR)
    @PostMapping("/fail")
    @PreAuthorize("hasRole('CONTRACTOR')")
    public ResponseEntity<Payment> fail(@RequestParam String gatewayOrderId) {
        return ResponseEntity.ok(paymentService.failPayment(gatewayOrderId));
    }

    // POST /api/payments/payout?applicationId=&amount= → 200 (ADMIN or owner CONTRACTOR)
    @PostMapping("/payout")
    @PreAuthorize("hasRole('ADMIN') or @applicationSecurity.isServiceOwnerOfApplication(#applicationId, authentication.name)")
    public ResponseEntity<Payout> payout(
            @RequestParam Long applicationId,
            @RequestParam BigDecimal amount) {
        return ResponseEntity.ok(paymentService.processPayout(applicationId, amount));
    }
}
```

---

## Security Evaluator Beans (referenced in @PreAuthorize)

```java
@Component("serviceSecurity")
public class ServiceSecurityEvaluator {
    public boolean isServiceOwner(Long serviceId, String email) {
        // load service → check contractor.user.email == email
    }
}

@Component("applicationSecurity")
public class ApplicationSecurityEvaluator {
    public boolean isApplicationOwner(Long applicationId, String email) {
        // load application → check worker.user.email == email
    }
    public boolean isServiceOwnerOfApplication(Long applicationId, String email) {
        // load application → check service.contractor.user.email == email
    }
}
```
