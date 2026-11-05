# WorkBridge — Database Constraints and Indexes

## 1. Purpose

This document defines the database constraints and indexing strategy for the WorkBridge application.

The objectives are to:

- Maintain data consistency and integrity.
- Prevent invalid or duplicate records.
- Enforce relationships between entities.
- Improve database query performance.
- Support reliable workforce allocation.
- Preserve payment and application records correctly.

The final constraints and indexes must be verified against the previous WorkBridge implementation and actual query patterns.

## 2. Primary Key Constraints

Every database entity must have a primary key that uniquely identifies each record.

Identified entities:

| Entity | Primary key | Verification |
|---|---|---|
| User | To verify | Old implementation |
| WorkerProfile | To verify | Old implementation |
| ContractorProfile | To verify | Old implementation |
| ServiceListing | To verify | Old implementation |
| ServiceCategory | To verify | Old implementation |
| Application | To verify | Old implementation |
| Notification | To verify | Old implementation |
| Payment | To verify | Old implementation |
| Payout | To verify | Old implementation |

### Requirements

- Every persisted entity must have a primary key.
- Primary keys must not contain null values.
- Primary key values must be unique.
- The ID generation strategy must be consistent with the existing application.

Potential JPA annotation:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

This is an example only. The actual ID type and generation strategy must be verified.

## 3. Foreign Key Constraints

Foreign keys maintain referential integrity between related tables.

Potential foreign key relationships:

| Child table | Referenced table | Relationship |
|---|---|---|
| WorkerProfile | User | User reference |
| ContractorProfile | User | User reference |
| ServiceListing | ContractorProfile | Contractor reference |
| ServiceListing | ServiceCategory | Category reference |
| Application | WorkerProfile | Worker reference |
| Application | ServiceListing | Listing reference |
| Notification | User | Recipient reference |
| Payment | To verify | Business association |
| Payout | To verify | Recipient association |

### Requirements

1. A foreign key must reference an existing record.
2. Mandatory relationships must not accept null foreign keys.
3. Deletion behavior must be explicitly defined.
4. Historical records must not be accidentally deleted through cascading operations.

The actual foreign key column names must match the entity relationship design.

## 4. Unique Constraints

Unique constraints prevent duplicate values in fields that must identify a record or relationship.

Potential unique constraints:

| Entity | Candidate field or combination | Purpose |
|---|---|---|
| User | Login identifier | Prevent duplicate accounts |
| WorkerProfile | User reference | Enforce one profile per user, if applicable |
| ContractorProfile | User reference | Enforce one profile per user, if applicable |
| Application | Worker + ServiceListing | Prevent duplicate applications, if required |

These are proposed constraints, not confirmed database rules.

Before applying them, verify:

- Whether the login identifier is an email, phone number, username, or another value.
- Whether the existing implementation permits multiple profiles.
- Whether a worker can submit another application after withdrawing or receiving a rejection.
- Whether duplicate applications are prevented at the service layer or database layer.

Example:

```java
@Table(
    uniqueConstraints = {
        @UniqueConstraint(
            columnNames = {"worker_id", "service_listing_id"}
        )
    }
)
```

Do not add this constraint until the application's reapplication rules are confirmed.

## 5. NOT NULL Constraints

Fields that are mandatory for a valid record should use NOT NULL constraints.

Potential examples:

| Entity | Field category | Requirement |
|---|---|---|
| User | Login and authentication data | Verify required fields |
| WorkerProfile | Profile information | Verify required fields |
| ContractorProfile | Contractor information | Verify required fields |
| ServiceListing | Listing information | Verify required fields |
| Application | Worker and listing references | Verify mandatory references |
| Notification | Recipient reference | Verify mandatory reference |
| Payment | Transaction information | Verify required fields |
| Payout | Payout information | Verify required fields |

The exact field names and nullability must be taken from the existing implementation.

## 6. Check Constraints and Enum Values

Business states should contain only supported values.

### 6.1 Service Listing Status

Known statuses:

| Status | Description |
|---|---|
| OPEN | Listing is accepting applications |
| FILLED | Required workforce has been allocated |
| CANCELLED | Listing has been cancelled |
| COMPLETED | Work has been completed |

The database representation may use a string enum or another existing strategy.

Example:

```java
@Enumerated(EnumType.STRING)
private ServiceStatus status;
```

The final status values and transitions must match the old implementation.

### 6.2 Contractor Account Status

Known administrative states:

| Status | Description |
|---|---|
| ACTIVE | Contractor account is active |
| BLOCKED | Contractor account is blocked by administration |

The exact storage location and allowed transitions must be verified.

### 6.3 Application Status

Application statuses are not finalized in this document.

They must be extracted from the previous source code, including:

- All supported status values.
- Allowed status transitions.
- Approval and rejection behavior.
- Withdrawal behavior, if supported.
- Relationship between approval and workforce allocation.

### 6.4 Payment and Payout Status

Payment and payout status values must be extracted from the old implementation.

No new status values should be introduced during the rebuild without explicit agreement.

## 7. Indexing Strategy

Indexes improve query performance by reducing the amount of data the database needs to scan.

Indexes should be selected based on actual WorkBridge queries rather than added indiscriminately.

### 7.1 Candidate Indexes

| Table | Candidate column(s) | Reason |
|---|---|---|
| User | Login identifier | Authentication lookup |
| ServiceListing | Contractor foreign key | Contractor listing retrieval |
| ServiceListing | Category foreign key | Category filtering |
| ServiceListing | Status | Retrieve open listings |
| Application | Worker foreign key | Worker application history |
| Application | ServiceListing foreign key | Listing application retrieval |
| Notification | User foreign key | Retrieve recipient notifications |
| Payment | Related transaction reference | Payment lookup, after verification |
| Payout | Recipient reference | Payout lookup, after verification |

These are candidate indexes. Existing primary key and foreign key indexes must also be inspected.

### 7.2 Composite Indexes

A composite index contains multiple columns.

For example, if the application frequently retrieves listings by category and status, a composite index may be considered:

```sql
CREATE INDEX idx_listing_category_status
ON service_listing(category_id, status);
```

This is an illustrative query optimization.

Before creating a composite index:

1. Inspect the actual SQL generated by Hibernate.
2. Identify frequently executed queries.
3. Check existing indexes.
4. Examine query execution plans.
5. Measure performance before and after the index.

Column order matters in a composite index. It should reflect the application's filtering and sorting patterns.

## 8. Location-Based Search Considerations

WorkBridge supports radius-based worker or service discovery.

The old implementation must be inspected to identify:

- Which entity stores location information.
- Whether latitude and longitude are stored separately.
- Whether location is represented through an address or coordinates.
- How the search radius is calculated.
- Whether the database performs the distance calculation.
- Whether a spatial index or ordinary indexes are used.

Do not introduce spatial columns or spatial indexes until the existing location-search implementation has been examined.

## 9. Workforce Allocation and Data Integrity

Workforce allocation is a critical business operation.

Multiple contractors or administrators may trigger operations that affect the same listing, so concurrent requests must not allocate more workers than the listing requires.

The existing implementation uses transactional processing and pessimistic locking.

Required integrity behavior:

1. Begin a transaction for the allocation operation.
2. Lock the relevant service listing using the existing pessimistic locking strategy.
3. Read the current allocation state.
4. Verify that sufficient capacity remains.
5. Approve the application and update allocation information.
6. Commit the transaction only if the operation succeeds.
7. Roll back changes if an operation fails.

The exact columns used to track capacity and allocated workers must be verified from the old implementation.

## 10. Deletion and Retention Rules

Deleting records can affect related business information.

The following behavior must be investigated:

| Record | Consideration |
|---|---|
| User | Check account deletion and profile lifecycle |
| WorkerProfile | Preserve application history where required |
| ContractorProfile | Check existing listings and account blocking |
| ServiceListing | Preserve relevant application and transaction history |
| Application | Check approval and allocation history |
| Notification | Verify notification retention policy |
| Payment | Preserve transaction history where required |
| Payout | Preserve payout history where required |

Contractor blocking is an administrative state change and should not automatically be treated as deletion.

The actual deletion and retention policies must be verified before configuring `CascadeType.REMOVE`, `orphanRemoval`, or database `ON DELETE` behavior.

## 11. Database Migration and Schema Management

The database schema must evolve in a controlled way as WorkBridge is developed.

The project should use one consistent schema management approach.

Before implementation, decide whether the existing project uses:

- Hibernate schema generation.
- SQL scripts.
- Flyway or Liquibase migrations.

The rebuild should not mix multiple schema management strategies without a clear reason.

Production schema changes should be version-controlled and repeatable.

## 12. Verification Checklist

Before implementing the final database constraints:

- [ ] Verify all primary key types.
- [ ] Verify ID generation strategies.
- [ ] Verify foreign key column names.
- [ ] Verify mandatory and optional relationships.
- [ ] Verify unique constraints.
- [ ] Verify NOT NULL constraints.
- [ ] Verify service listing statuses.
- [ ] Extract application status values.
- [ ] Extract payment and payout status values.
- [ ] Inspect existing indexes.
- [ ] Inspect location-based queries.
- [ ] Verify workforce allocation locking.
- [ ] Verify deletion and retention rules.
- [ ] Select the schema migration strategy.

## 13. Completion Criteria

This document is finalized when:

1. All primary keys are verified.
2. All foreign key relationships are confirmed.
3. Unique and nullability constraints match business rules.
4. Indexes are justified by actual query patterns.
5. Workforce allocation integrity is documented.
6. Deletion and retention rules are confirmed.
7. Schema management is agreed upon.

**Next document:** `docs/03-Database/transaction-management-and-concurrency.md`