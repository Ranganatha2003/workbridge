# WorkBridge — Repositories

All repositories extend `JpaRepository<Entity, Long>`.

---

## 1. UserRepository

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}
```

---

## 2. WorkerProfileRepository

```java
public interface WorkerProfileRepository extends JpaRepository<WorkerProfile, Long> {
    Optional<WorkerProfile> findByUserId(Long userId);
    Optional<WorkerProfile> findByUserEmail(String email);
}
```

---

## 3. ContractorProfileRepository

```java
public interface ContractorProfileRepository extends JpaRepository<ContractorProfile, Long> {
    Optional<ContractorProfile> findByUserId(Long userId);
    Optional<ContractorProfile> findByUserEmail(String email);
}
```

---

## 4. ServiceCategoryRepository

```java
public interface ServiceCategoryRepository extends JpaRepository<ServiceCategory, Long> {
    Optional<ServiceCategory> findByName(String name);
}
```

---

## 5. ServiceListingRepository

```java
public interface ServiceListingRepository extends JpaRepository<ServiceListing, Long> {

    List<ServiceListing> findByContractorId(Long contractorId);

    // Haversine radius search — OPEN services only
    @Query(value = """
        SELECT * FROM services s
        WHERE s.status = 'OPEN'
          AND (6371 * ACOS(
                COS(RADIANS(:lat)) * COS(RADIANS(s.latitude)) *
                COS(RADIANS(s.longitude) - RADIANS(:lon)) +
                SIN(RADIANS(:lat)) * SIN(RADIANS(s.latitude))
              )) <= :radiusKm
        ORDER BY (6371 * ACOS(
                COS(RADIANS(:lat)) * COS(RADIANS(s.latitude)) *
                COS(RADIANS(s.longitude) - RADIANS(:lon)) +
                SIN(RADIANS(:lat)) * SIN(RADIANS(s.latitude))
              ))
        """, nativeQuery = true)
    List<ServiceListing> findOpenServicesWithinRadius(
        @Param("lat") double lat,
        @Param("lon") double lon,
        @Param("radiusKm") double radiusKm
    );

    // Pessimistic lock — used when accepting/cancelling applications
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT s FROM ServiceListing s WHERE s.id = :id")
    Optional<ServiceListing> findByIdWithLock(@Param("id") Long id);
}
```

---

## 6. ApplicationRepository

```java
public interface ApplicationRepository extends JpaRepository<Application, Long> {

    List<Application> findByWorkerId(Long workerId);
    List<Application> findByServiceListingId(Long serviceId);
    List<Application> findByServiceListingContractorId(Long contractorId);

    Optional<Application> findByWorkerIdAndServiceListingId(Long workerId, Long serviceId);
    boolean existsByWorkerIdAndServiceListingId(Long workerId, Long serviceId);
}
```

---

## 7. NotificationRepository

```java
public interface NotificationRepository extends JpaRepository<Notification, Long> {

    List<Notification> findByUserIdOrderByCreatedAtDesc(Long userId);
    List<Notification> findByUserIdAndReadFalseOrderByCreatedAtDesc(Long userId);
}
```

---

## 8. PaymentRepository

```java
public interface PaymentRepository extends JpaRepository<Payment, Long> {

    List<Payment>     findByContractorId(Long contractorId);
    Optional<Payment> findByGatewayOrderId(String gatewayOrderId);
}
```

---

## 9. PayoutRepository

```java
public interface PayoutRepository extends JpaRepository<Payout, Long> {

    List<Payout>     findByWorkerId(Long workerId);
    Optional<Payout> findByGatewayPayoutId(String gatewayPayoutId);
}
```
