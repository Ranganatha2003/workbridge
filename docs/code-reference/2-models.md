# WorkBridge — Models (JPA Entities)

## Tech: Spring Data JPA + Hibernate + MySQL

---

## 1. Role Enum

```java
public enum Role {
    ROLE_WORKER,
    ROLE_CONTRACTOR,
    ROLE_ADMIN
}
```

---

## 2. User

```java
@Entity
@Table(name = "users")
@EntityListeners(AuditingEntityListener.class)
public class User {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String passwordHash;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;

    private boolean verified = true;
    private boolean active   = true;

    @CreatedDate  @Column(updatable = false) private LocalDateTime createdAt;
    @LastModifiedDate                        private LocalDateTime updatedAt;

    @OneToOne(mappedBy = "user")
    private WorkerProfile workerProfile;

    @OneToOne(mappedBy = "user")
    private ContractorProfile contractorProfile;

    @OneToMany(mappedBy = "user")
    private List<Notification> notifications = new ArrayList<>();
}
```

---

## 3. WorkerProfile

```java
@Entity
@Table(name = "worker_profiles")
@EntityListeners(AuditingEntityListener.class)
public class WorkerProfile {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id", unique = true, nullable = false)
    private User user;

    @Column(nullable = false) private String fullName;
    private String phoneNumber;
    private String skills;
    private String preferredCity;

    @CreatedDate  @Column(updatable = false) private LocalDateTime createdAt;
    @LastModifiedDate                        private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "worker")
    private List<Application> applications = new ArrayList<>();
}
```

---

## 4. ContractorProfile

```java
public enum ContractorStatus { ACTIVE, BLOCKED }

@Entity
@Table(name = "contractor_profiles")
@EntityListeners(AuditingEntityListener.class)
public class ContractorProfile {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id", unique = true, nullable = false)
    private User user;

    @Column(nullable = false) private String businessName;
    private String phoneNumber;
    private String address;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ContractorStatus status = ContractorStatus.ACTIVE;

    @CreatedDate  @Column(updatable = false) private LocalDateTime createdAt;
    @LastModifiedDate                        private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "contractor")
    private List<ServiceListing> serviceListings = new ArrayList<>();
}
```

---

## 5. ServiceCategory

```java
@Entity
@Table(name = "service_categories")
public class ServiceCategory {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    @OneToMany(mappedBy = "category")
    private List<ServiceListing> serviceListings = new ArrayList<>();
}
```

---

## 6. ServiceListing

```java
public enum ServiceStatus {
    DRAFT, OPEN, FILLED, IN_PROGRESS, COMPLETED, CANCELLED, EXPIRED
}

@Entity
@Table(name = "services")
@EntityListeners(AuditingEntityListener.class)
public class ServiceListing {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "contractor_id", nullable = false)
    private ContractorProfile contractor;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private ServiceCategory category;

    @Column(nullable = false) private String title;
    @Column(nullable = false) private String description;
    @Column(nullable = false) private String address;

    private Double  latitude;
    private Double  longitude;

    @Column(nullable = false) private Integer requiredWorkers;
    private Integer acceptedWorkers = 0;

    private LocalDate serviceDate;
    private LocalTime startTime;
    private LocalTime endTime;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ServiceStatus status = ServiceStatus.OPEN;

    @Version private Integer version; // optimistic lock

    @CreatedDate  @Column(updatable = false) private LocalDateTime createdAt;
    @LastModifiedDate                        private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "serviceListing")
    private List<Application> applications = new ArrayList<>();
}
```

---

## 7. Application

```java
public enum ApplicationStatus { APPLIED, ACCEPTED, REJECTED, CANCELLED }

@Entity
@Table(
    name = "applications",
    uniqueConstraints = @UniqueConstraint(columnNames = {"service_id", "worker_id"})
)
@EntityListeners(AuditingEntityListener.class)
public class Application {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "service_id", nullable = false)
    private ServiceListing serviceListing;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "worker_id", nullable = false)
    private WorkerProfile worker;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ApplicationStatus status = ApplicationStatus.APPLIED;

    @CreatedDate  @Column(updatable = false) private LocalDateTime appliedAt;
    @LastModifiedDate                        private LocalDateTime updatedAt;
}
```

---

## 8. Notification

```java
@Entity
@Table(name = "notifications")
@EntityListeners(AuditingEntityListener.class)
public class Notification {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    @JsonIgnore
    private User user;

    @Column(nullable = false, length = 200) private String title;
    @Column(nullable = false, columnDefinition = "TEXT") private String message;

    @Column(name = "is_read", nullable = false)
    private boolean read = false;

    @CreatedDate @Column(updatable = false, nullable = false)
    private LocalDateTime createdAt;
}
```

---

## 9. Payment

```java
public enum PaymentStatus { INITIATED, PENDING, SUCCESS, FAILED, REFUNDED }

@Entity
@Table(name = "payments")
@EntityListeners(AuditingEntityListener.class)
public class Payment {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "service_id", nullable = false)
    @JsonIgnore
    private ServiceListing service;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "contractor_id", nullable = false)
    @JsonIgnore
    private ContractorProfile contractor;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal amount;

    @Column(unique = true, length = 100)
    private String gatewayOrderId;  // e.g. "pay_" + UUID

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private PaymentStatus status = PaymentStatus.INITIATED;

    @CreatedDate  @Column(updatable = false) private LocalDateTime createdAt;
    @LastModifiedDate                        private LocalDateTime updatedAt;
}
```

---

## 10. Payout

```java
public enum PayoutStatus { PENDING, PROCESSING, PAID, FAILED }

@Entity
@Table(name = "payouts")
@EntityListeners(AuditingEntityListener.class)
public class Payout {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "application_id", unique = true, nullable = false)
    @JsonIgnore
    private Application application;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "worker_id", nullable = false)
    @JsonIgnore
    private WorkerProfile worker;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal amount;

    @Column(unique = true, length = 100)
    private String gatewayPayoutId;  // e.g. "pout_" + UUID

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private PayoutStatus status = PayoutStatus.PAID;

    @CreatedDate  @Column(updatable = false) private LocalDateTime createdAt;
    @LastModifiedDate                        private LocalDateTime updatedAt;
}
```

---

## Enable JPA Auditing
In main class or config:
```java
@EnableJpaAuditing
```
