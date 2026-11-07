# WorkBridge — Entity Relationships and JPA Mapping

## 1. Purpose

This document defines the entity relationships required by the WorkBridge backend and explains how those relationships will be represented using Jakarta Persistence (JPA).

It covers:

- Entity relationship types.
- Relationship ownership.
- Foreign key placement.
- JPA mapping annotations.
- Cascade operations.
- Fetch strategies.
- Bidirectional relationship considerations.
- Database integrity.

The final mappings must be verified against the previous WorkBridge implementation before writing entity classes.

## 2. Identified Entity Relationships

The following relationships have been identified at the conceptual level.

| Parent / Entity A | Related Entity B | Relationship to investigate |
|---|---|---|
| User | WorkerProfile | One-to-one |
| User | ContractorProfile | One-to-one |
| ContractorProfile | ServiceListing | One-to-many |
| ServiceCategory | ServiceListing | One-to-many |
| WorkerProfile | Application | One-to-many |
| ServiceListing | Application | One-to-many |
| User | Notification | One-to-many |
| Payment | Application | To be verified |
| Payout | User | To be verified |

**Important:** These are conceptual relationships. The actual foreign key columns, relationship ownership, optionality, and cardinalities must be confirmed from the old source code.

## 3. User and Profile Relationships

### 3.1 User — WorkerProfile

A worker account is associated with worker-specific profile information.

Potential JPA mapping:

```java
// User.java
@OneToOne(mappedBy = "user")
private WorkerProfile workerProfile;
```

```java
// WorkerProfile.java
@OneToOne
@JoinColumn(name = "user_id")
private User user;
```

Points to verify:

- Whether the relationship is actually one-to-one.
- Whether `WorkerProfile` owns the relationship.
- Whether `user_id` is unique.
- Whether a worker profile is mandatory for every worker account.
- Whether cascade operations are configured.

### 3.2 User — ContractorProfile

A contractor account is associated with contractor-specific profile information.

Potential JPA mapping:

```java
// User.java
@OneToOne(mappedBy = "user")
private ContractorProfile contractorProfile;
```

```java
// ContractorProfile.java
@OneToOne
@JoinColumn(name = "user_id")
private User user;
```

Points to verify:

- Whether the contractor profile owns the relationship.
- Whether the foreign key is stored in the contractor profile table.
- Whether the relationship is optional.
- Whether contractor account status is stored in User or ContractorProfile.

## 4. ContractorProfile and ServiceListing

A contractor publishes service listings.

Conceptual relationship:

- One contractor can publish multiple service listings.
- Each listing belongs to its publishing contractor.

Potential JPA mapping:

```java
// ContractorProfile.java
@OneToMany(mappedBy = "contractor")
private List<ServiceListing> serviceListings;
```

```java
// ServiceListing.java
@ManyToOne
@JoinColumn(name = "contractor_id")
private ContractorProfile contractor;
```

Points to verify:

- Actual foreign key column name.
- Whether the relationship is bidirectional.
- Whether the contractor reference is mandatory.
- Cascade configuration.
- Whether deleting a contractor is restricted while listings exist.

## 5. ServiceCategory and ServiceListing

A service listing is associated with a category.

Conceptual relationship:

- One category can be associated with multiple listings.
- Each listing belongs to a category, if the existing implementation enforces this.

Potential JPA mapping:

```java
// ServiceCategory.java
@OneToMany(mappedBy = "category")
private List<ServiceListing> serviceListings;
```

```java
// ServiceListing.java
@ManyToOne
@JoinColumn(name = "category_id")
private ServiceCategory category;
```

Points to verify:

- Whether the relationship is mandatory.
- Whether a listing can have multiple categories.
- Whether categories can be removed when listings reference them.
- Actual foreign key column name.

## 6. WorkerProfile and Application

Workers submit applications for service listings.

Conceptual relationship:

- One worker can submit multiple applications.
- Each application references a worker.

Potential JPA mapping:

```java
// WorkerProfile.java
@OneToMany(mappedBy = "worker")
private List<Application> applications;
```

```java
// Application.java
@ManyToOne
@JoinColumn(name = "worker_id")
private WorkerProfile worker;
```

Points to verify:

- Actual relationship ownership.
- Whether duplicate applications for the same listing are prevented.
- Whether the worker reference is mandatory.
- Whether application records are retained after a worker account is blocked or deactivated.

## 7. ServiceListing and Application

A service listing receives applications from workers.

Conceptual relationship:

- One listing can receive multiple applications.
- Each application references a listing.

Potential JPA mapping:

```java
// ServiceListing.java
@OneToMany(mappedBy = "serviceListing")
private List<Application> applications;
```

```java
// Application.java
@ManyToOne
@JoinColumn(name = "service_listing_id")
private ServiceListing serviceListing;
```

Points to verify:

- Actual foreign key name.
- Whether the relationship is bidirectional.
- Application status values.
- Whether applications remain after a listing is cancelled or completed.
- Whether application deletion is allowed.

## 8. User and Notification

Notifications are associated with users.

Potential relationship:

- One user can receive multiple notifications.
- Each notification belongs to a recipient.

Potential JPA mapping:

```java
// User.java
@OneToMany(mappedBy = "user")
private List<Notification> notifications;
```

```java
// Notification.java
@ManyToOne
@JoinColumn(name = "user_id")
private User user;
```

Points to verify:

- Whether notifications reference User or a profile entity.
- Whether notifications support read/unread status.
- Whether notifications are persisted before or after asynchronous delivery.
- Whether notification records are retained or deleted under any account lifecycle operation.

## 9. Payment and Payout Relationships

Payment and payout relationships require additional investigation.

The existing project includes payment and payout functionality, but their exact ownership and foreign key relationships must not be assumed.

Verify:

- Which entity owns a payment record.
- Whether a payment is associated with an application, service listing, contractor, worker, or another transaction.
- Whether one application can have multiple payment records.
- Which user or profile receives a payout.
- Whether payout records reference payment records.
- Payment and payout status values.
- Whether payment and payout records are retained for audit purposes.

No final JPA annotations should be selected until these details are verified.

## 10. Relationship Ownership and Foreign Keys

In JPA, the owning side of a relationship controls the database relationship and usually contains the foreign key mapping.

Common annotations:

| Annotation | Purpose |
|---|---|
| `@OneToOne` | One-to-one relationship |
| `@OneToMany` | One-to-many relationship |
| `@ManyToOne` | Many-to-one relationship |
| `@ManyToMany` | Many-to-many relationship |
| `@JoinColumn` | Defines a foreign key column |
| `mappedBy` | Identifies the inverse side of a bidirectional relationship |
| `@JoinTable` | Defines an intermediate join table |

The owning side must be selected based on the actual database design.

## 11. Cascade Operations

Cascade operations determine whether persistence operations on one entity propagate to related entities.

| Cascade type | Behavior |
|---|---|
| `PERSIST` | Propagates persist operations |
| `MERGE` | Propagates merge operations |
| `REMOVE` | Propagates remove operations |
| `REFRESH` | Propagates refresh operations |
| `DETACH` | Propagates detach operations |
| `ALL` | Includes all cascade operations |

Cascade configuration must be selected according to the actual lifecycle of each entity.

Important considerations:

- Do not automatically apply `CascadeType.ALL` to every relationship.
- Do not cascade deletion from a contractor to historical service listings without confirming the business rules.
- Do not delete application history merely because a related profile is removed.
- Payment and payout records may require retention.
- Use `orphanRemoval = true` only when a child entity should genuinely be deleted after removal from its parent relationship.

All cascade settings are to be verified against the old implementation and business requirements.

## 12. Fetch Strategies

JPA supports two common fetch strategies.

| Strategy | Description |
|---|---|
| LAZY | Related data is loaded when accessed |
| EAGER | Related data is loaded immediately with the entity |

Important considerations:

- Avoid unnecessary eager loading of large collections.
- Prefer explicit fetching when an API needs related data.
- Be aware of N+1 query problems.
- Use repository queries, entity graphs, or fetch joins when appropriate.
- Avoid exposing complete entity relationships directly through API responses.

Fetch strategies must be selected according to the actual API access patterns.

## 13. Bidirectional Relationship Considerations

When both entities maintain references to each other, the relationship is bidirectional.

For example:

- ContractorProfile contains a collection of ServiceListing objects.
- ServiceListing contains a reference to ContractorProfile.

The owning side must maintain the relationship correctly.

If helper methods are used, they should update both sides consistently.

Example:

```java
public void addServiceListing(ServiceListing listing) {
    serviceListings.add(listing);
    listing.setContractor(this);
}
```

This is an illustrative helper method, not a confirmed method from the existing project.

## 14. JSON Serialization Considerations

Bidirectional JPA relationships can create recursive serialization problems.

For example:

`ContractorProfile → ServiceListing → ContractorProfile`

Returning these entities directly from REST controllers can result in infinite recursion or oversized responses.

WorkBridge should use DTOs for API request and response data.

Serialization annotations such as `@JsonManagedReference` and `@JsonBackReference` may be considered where appropriate, but DTO-based responses should be the primary API design approach.

## 15. Verification Checklist

Before implementing the entity classes, verify the following against the previous WorkBridge source code:

- [ ] User and WorkerProfile mapping.
- [ ] User and ContractorProfile mapping.
- [ ] ContractorProfile and ServiceListing mapping.
- [ ] ServiceCategory and ServiceListing mapping.
- [ ] WorkerProfile and Application mapping.
- [ ] ServiceListing and Application mapping.
- [ ] User and Notification mapping.
- [ ] Payment relationships.
- [ ] Payout relationships.
- [ ] Foreign key names and unique constraints.
- [ ] Owning and inverse sides.
- [ ] Cascade settings.
- [ ] Fetch settings.
- [ ] Nullability and optional relationships.
- [ ] Delete and retention behavior.

## 16. Completion Criteria

The relationship design is ready for implementation when:

1. Every entity relationship is verified.
2. Foreign key placement is confirmed.
3. Ownership is documented.
4. Cascade and fetch strategies are justified.
5. Database constraints match the business rules.
6. Payment and payout relationships are confirmed.
7. The ER diagram matches the actual entity mappings.

**Next document:**