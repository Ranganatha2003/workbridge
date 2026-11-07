# WorkBridge — Services

---

## 1. AuthService

```java
@Service
public class AuthService {

    public AuthResponse register(RegisterRequest req) {
        if (userRepository.existsByEmail(req.getEmail()))
            throw new ConflictException("Email already in use");

        User user = new User();
        user.setEmail(req.getEmail());
        user.setPasswordHash(passwordEncoder.encode(req.getPassword()));
        user.setRole(req.getRole());
        user.setVerified(true);
        user.setActive(true);
        user = userRepository.save(user);

        Long profileId = null;
        if (req.getRole() == Role.ROLE_WORKER) {
            WorkerProfile wp = new WorkerProfile();
            wp.setUser(user);
            wp.setFullName(req.getFullName());
            wp.setPhoneNumber(req.getPhoneNumber());
            wp.setSkills(req.getSkills());
            wp.setPreferredCity(req.getPreferredCity());
            profileId = workerProfileRepository.save(wp).getId();
        } else if (req.getRole() == Role.ROLE_CONTRACTOR) {
            ContractorProfile cp = new ContractorProfile();
            cp.setUser(user);
            cp.setBusinessName(req.getBusinessName() != null ? req.getBusinessName() : req.getFullName() + " Services");
            cp.setPhoneNumber(req.getPhoneNumber());
            cp.setAddress(req.getAddress());
            profileId = contractorProfileRepository.save(cp).getId();
        }

        String accessToken  = jwtService.generateAccessToken(...);
        String refreshToken = jwtService.generateRefreshToken(...);
        return new AuthResponse(accessToken, refreshToken, user.getEmail(), user.getRole(), user.getId(), profileId);
    }

    public AuthResponse login(LoginRequest req) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(req.getEmail(), req.getPassword())
        );
        User user = userRepository.findByEmail(req.getEmail()).orElseThrow();
        Long profileId = resolveProfileId(user);
        String accessToken  = jwtService.generateAccessToken(...);
        String refreshToken = jwtService.generateRefreshToken(...);
        return new AuthResponse(accessToken, refreshToken, user.getEmail(), user.getRole(), user.getId(), profileId);
    }
}
```

---

## 2. ServiceListingService

```java
@Service
public class ServiceListingService {

    @Transactional
    public ServiceResponse createService(ServiceCreateRequest req, String email) {
        if (req.getStartTime().isAfter(req.getEndTime()))
            throw new BadRequestException("Start time must be before end time");

        ContractorProfile contractor = contractorProfileRepository.findByUserEmail(email).orElseThrow();
        ServiceCategory   category   = serviceCategoryRepository.findById(req.getCategoryId()).orElseThrow();

        ServiceListing sl = new ServiceListing();
        // map all fields from req
        sl.setContractor(contractor);
        sl.setCategory(category);
        sl.setAcceptedWorkers(0);
        sl.setStatus(ServiceStatus.OPEN);
        return toResponse(serviceListingRepository.save(sl));
    }

    @Transactional(readOnly = true)
    public List<ServiceResponse> getAllServices() {
        return serviceListingRepository.findAll().stream().map(this::toResponse).toList();
    }

    @Transactional(readOnly = true)
    public ServiceResponse getById(Long id) {
        return toResponse(serviceListingRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Service not found")));
    }

    @Transactional(readOnly = true)
    public List<ServiceResponse> getMyServices(String email) {
        ContractorProfile cp = contractorProfileRepository.findByUserEmail(email).orElseThrow();
        return serviceListingRepository.findByContractorId(cp.getId()).stream().map(this::toResponse).toList();
    }

    @Transactional(readOnly = true)
    public List<ServiceResponse> searchByRadius(double lat, double lon, double radiusKm) {
        return serviceListingRepository.findOpenServicesWithinRadius(lat, lon, radiusKm)
            .stream().map(this::toResponse).toList();
    }

    @Transactional
    public ServiceResponse updateService(Long id, ServiceUpdateRequest req) {
        ServiceListing sl = serviceListingRepository.findById(id).orElseThrow();
        if (Set.of(CANCELLED, COMPLETED, EXPIRED).contains(sl.getStatus()))
            throw new InvalidStateTransitionException("Cannot update service in current status");
        if (req.getRequiredWorkers() < sl.getAcceptedWorkers())
            throw new BadRequestException("Required workers cannot be less than accepted workers");
        if (req.getRequiredWorkers() > sl.getRequiredWorkers() && sl.getStatus() == FILLED)
            sl.setStatus(ServiceStatus.OPEN);
        // map fields, save, return
    }

    @Transactional
    public ServiceResponse cancelService(Long id) {
        ServiceListing sl = serviceListingRepository.findById(id).orElseThrow();
        if (sl.getStatus() == COMPLETED || sl.getStatus() == CANCELLED)
            throw new InvalidStateTransitionException("Cannot cancel");
        sl.setStatus(ServiceStatus.CANCELLED);
        return toResponse(serviceListingRepository.save(sl));
    }

    @Transactional
    public ServiceResponse completeService(Long id) {
        ServiceListing sl = serviceListingRepository.findById(id).orElseThrow();
        if (sl.getStatus() != OPEN && sl.getStatus() != FILLED)
            throw new InvalidStateTransitionException("Can complete only OPEN or FILLED services");
        sl.setStatus(ServiceStatus.COMPLETED);
        return toResponse(serviceListingRepository.save(sl));
    }
}
```

---

## 3. ApplicationService

```java
@Service
public class ApplicationService {

    @Transactional
    public ApplicationResponse apply(Long serviceId, String email) {
        WorkerProfile worker = workerProfileRepository.findByUserEmail(email).orElseThrow();

        // Pessimistic lock on service
        ServiceListing sl = serviceListingRepository.findByIdWithLock(serviceId)
            .orElseThrow(() -> new ResourceNotFoundException("Service not found"));

        if (sl.getStatus() != ServiceStatus.OPEN)
            throw new InvalidStateTransitionException("Service is not OPEN");
        if (sl.getServiceDate().isBefore(LocalDate.now()))
            throw new BadRequestException("Service date is in the past");
        if (applicationRepository.existsByWorkerIdAndServiceListingId(worker.getId(), serviceId))
            throw new ConflictException("Already applied");
        if (sl.getAcceptedWorkers() >= sl.getRequiredWorkers())
            throw new BadRequestException("Service is at full capacity");

        Application app = new Application();
        app.setWorker(worker);
        app.setServiceListing(sl);
        app.setStatus(ApplicationStatus.APPLIED);
        app = applicationRepository.save(app);

        // publish: New Application Received → contractor
        eventPublisher.publishEvent(new InAppNotificationEvent(...));
        return toResponse(app);
    }

    @Transactional(readOnly = true)
    public List<ApplicationResponse> getMyApplications(String email) {
        WorkerProfile wp = workerProfileRepository.findByUserEmail(email).orElseThrow();
        return applicationRepository.findByWorkerId(wp.getId()).stream().map(this::toResponse).toList();
    }

    @Transactional(readOnly = true)
    public List<ApplicationResponse> getApplicationsForService(Long serviceId) {
        return applicationRepository.findByServiceListingId(serviceId).stream().map(this::toResponse).toList();
    }

    @Transactional
    public ApplicationResponse accept(Long id) {
        Application app = applicationRepository.findById(id).orElseThrow();
        if (app.getStatus() == ACCEPTED) return toResponse(app);
        if (app.getStatus() == CANCELLED || app.getStatus() == REJECTED)
            throw new InvalidStateTransitionException("Cannot accept");

        // Pessimistic lock on service
        ServiceListing sl = serviceListingRepository.findByIdWithLock(app.getServiceListing().getId()).orElseThrow();
        if (sl.getAcceptedWorkers() >= sl.getRequiredWorkers())
            throw new BadRequestException("Service is at full capacity");

        app.setStatus(ApplicationStatus.ACCEPTED);
        sl.setAcceptedWorkers(sl.getAcceptedWorkers() + 1);
        if (sl.getAcceptedWorkers().equals(sl.getRequiredWorkers()))
            sl.setStatus(ServiceStatus.FILLED);

        serviceListingRepository.save(sl);
        applicationRepository.save(app);

        // publish: Application Accepted → worker; Service Filled → contractor (if filled)
        return toResponse(app);
    }

    @Transactional
    public ApplicationResponse reject(Long id) {
        Application app = applicationRepository.findById(id).orElseThrow();
        if (app.getStatus() == REJECTED)  return toResponse(app);
        if (app.getStatus() == ACCEPTED)  throw new InvalidStateTransitionException("Cannot reject accepted application");
        app.setStatus(ApplicationStatus.REJECTED);
        applicationRepository.save(app);
        // publish: Application Screened → worker
        return toResponse(app);
    }

    @Transactional
    public ApplicationResponse cancel(Long id) {
        Application app = applicationRepository.findById(id).orElseThrow();
        if (app.getStatus() == CANCELLED) return toResponse(app);

        if (app.getStatus() == ACCEPTED) {
            ServiceListing sl = serviceListingRepository.findByIdWithLock(app.getServiceListing().getId()).orElseThrow();
            sl.setAcceptedWorkers(sl.getAcceptedWorkers() - 1);
            if (sl.getStatus() == ServiceStatus.FILLED)
                sl.setStatus(ServiceStatus.OPEN);
            serviceListingRepository.save(sl);
        }

        app.setStatus(ApplicationStatus.CANCELLED);
        return toResponse(applicationRepository.save(app));
        // NOTE: no notification event on cancel
    }
}
```

---

## 4. NotificationService

```java
@Service
public class NotificationService {

    @Transactional
    public void createNotification(Long userId, String title, String message) {
        User user = userRepository.findById(userId).orElseThrow();
        Notification n = new Notification();
        n.setUser(user);
        n.setTitle(title);
        n.setMessage(message);
        n.setRead(false);
        notificationRepository.save(n);
    }

    @Transactional(readOnly = true)
    public List<NotificationResponse> getMyNotifications(String email) {
        User user = userRepository.findByEmail(email).orElseThrow();
        return notificationRepository.findByUserIdOrderByCreatedAtDesc(user.getId())
            .stream().map(this::toResponse).toList();
    }

    @Transactional(readOnly = true)
    public List<NotificationResponse> getMyUnreadNotifications(String email) {
        User user = userRepository.findByEmail(email).orElseThrow();
        return notificationRepository.findByUserIdAndReadFalseOrderByCreatedAtDesc(user.getId())
            .stream().map(this::toResponse).toList();
    }

    @Transactional
    public void markAsRead(Long id, String email) {
        Notification n = notificationRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Notification not found"));
        if (!n.getUser().getEmail().equals(email))
            throw new ForbiddenException("Not your notification");
        n.setRead(true);
        notificationRepository.save(n);
    }
}
```

### NotificationEventListener

```java
@Component
public class NotificationEventListener {

    @Async
    @EventListener
    public void handleInAppNotification(InAppNotificationEvent event) {
        try {
            notificationService.createNotification(event.getRecipientId(), event.getTitle(), event.getMessage());
        } catch (Exception e) {
            log.error("Notification creation failed", e);
        }
    }
}
```

---

## 5. PaymentService

```java
@Service
public class PaymentService {

    @Transactional
    public Payment initiatePayment(Long serviceId, BigDecimal amount, String email) {
        ServiceListing sl = serviceListingRepository.findById(serviceId).orElseThrow();
        ContractorProfile cp = contractorProfileRepository.findByUserEmail(email).orElseThrow();

        Payment p = new Payment();
        p.setService(sl);
        p.setContractor(cp);
        p.setAmount(amount);
        p.setGatewayOrderId("pay_" + UUID.randomUUID().toString().replace("-", ""));
        p.setStatus(PaymentStatus.INITIATED);
        return paymentRepository.save(p);
    }

    @Transactional
    public Payment confirmPayment(String gatewayOrderId) {
        Payment p = paymentRepository.findByGatewayOrderId(gatewayOrderId)
            .orElseThrow(() -> new ResourceNotFoundException("Payment not found"));
        if (p.getStatus() == PaymentStatus.SUCCESS) return p;
        p.setStatus(PaymentStatus.SUCCESS);
        return paymentRepository.save(p);
    }

    @Transactional
    public Payment failPayment(String gatewayOrderId) {
        Payment p = paymentRepository.findByGatewayOrderId(gatewayOrderId)
            .orElseThrow(() -> new ResourceNotFoundException("Payment not found"));
        p.setStatus(PaymentStatus.FAILED);
        return paymentRepository.save(p);
    }

    @Transactional
    public Payout processPayout(Long applicationId, BigDecimal amount) {
        Application app = applicationRepository.findById(applicationId)
            .orElseThrow(() -> new ResourceNotFoundException("Application not found"));
        if (app.getStatus() != ApplicationStatus.ACCEPTED)
            throw new InvalidStateTransitionException("Application must be ACCEPTED for payout");

        Payout payout = new Payout();
        payout.setApplication(app);
        payout.setWorker(app.getWorker());
        payout.setAmount(amount);
        payout.setGatewayPayoutId("pout_" + UUID.randomUUID().toString().replace("-", ""));
        payout.setStatus(PayoutStatus.PAID);
        return payoutRepository.save(payout);
    }
}
```
