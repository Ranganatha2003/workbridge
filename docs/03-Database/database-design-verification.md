# WorkBridge — Database Design Verification

## 1. Purpose

This document provides the final verification checklist for the WorkBridge database design.

Before implementing the database layer in the rebuilt Spring Boot application, the existing WorkBridge implementation must be inspected.

The objective is to ensure that the rebuilt application preserves the existing database structure, relationships, business rules, and transaction behavior.

No new database features or business rules should be introduced during this verification.

## 2. Database Documentation Review

The following documents have been prepared:

| Document | Purpose |
|---|---|
| `database-design.md` | Identifies the domain entities and conceptual ER diagram |
| `entity-relationships.md` | Documents entity relationships and JPA mapping considerations |
| `database-constraints-and-indexes.md` | Documents integrity constraints and indexing |
| `transaction-management-and-concurrency.md` | Documents transactions and workforce allocation concurrency |

All four documents must be reviewed together before implementation.

## 3. Entity Verification

Verify each entity against the old WorkBridge source code.

| Entity | Fields verified | Relationships verified | Status |
|---|---|---|---|
| User | Pending | Pending | Not verified |
| WorkerProfile | Pending | Pending | Not verified |
| ContractorProfile | Pending | Pending | Not verified |
| ServiceListing | Pending | Pending | Not verified |
| ServiceCategory | Pending | Pending | Not verified |
| Application | Pending | Pending | Not verified |
| Notification | Pending | Pending | Not verified |
| Payment | Pending | Pending | Not verified |
| Payout | Pending | Pending | Not verified |

For every entity, verify:

- Class name and table name.
- Field names and Java data types.
- Primary key and ID generation strategy.
- Nullable and non-nullable fields.
- Unique constraints.
- Default values.
- Enum fields.
- Relationship annotations.
- Foreign key column names.

## 4. Relationship Verification

Verify the actual JPA relationships.

| Relationship | Expected investigation |
|---|---|
| User — WorkerProfile | Cardinality and ownership |
| User — ContractorProfile | Cardinality and ownership |
| ContractorProfile — ServiceListing | One-to-many mapping |
| ServiceCategory — ServiceListing | Category mapping |
| WorkerProfile — Application | Application ownership |
| ServiceListing — Application | Application ownership |
| User — Notification | Notification recipient |
| Payment relationships | Actual transaction association |
| Payout relationships | Actual recipient association |

For every relationship, verify:

1. Whether it is unidirectional or bidirectional.
2. Which entity owns the relationship.
3. Where the foreign key is stored.
4. Whether the relationship is optional.
5. Cascade configuration.
6. Fetch strategy.
7. Delete behavior.

## 5. Business Rule Verification

Database structures must support the existing WorkBridge business rules.

### 5.1 Service Listing

Verify:

- Required workforce capacity.
- Current workforce allocation.
- Service listing status.
- Contractor ownership.
- Service category association.
- Location information.
- Radius-based discovery fields.

Known service listing statuses:

- OPEN
- FILLED
- CANCELLED
- COMPLETED

### 5.2 Application

Verify:

- Worker and service listing references.
- Application status values.
- Approval and rejection behavior.
- Duplicate application prevention.
- Withdrawal behavior, if supported.
- Workforce allocation updates.

### 5.3 Contractor Administration

Verify:

- How contractor account status is stored.
- How the Super Admin blocks a contractor.
- How the Super Admin unblocks a contractor.
- Whether blocking affects existing listings.
- Whether blocking affects pending applications.
- Whether blocking affects already-approved workers.
- Whether blocking affects payment and payout operations.

The intended administrative states are ACTIVE and BLOCKED. Their actual representation must be confirmed.

### 5.4 Payment and Payout

Verify:

- Payment entity fields.
- Payout entity fields.
- Payment ownership and associations.
- Payout recipient associations.
- Payment and payout status values.
- Transaction history retention.
- Existing payment and payout simulation behavior.

## 6. Transaction and Concurrency Verification

Workforce allocation is a critical operation.

Verify the existing implementation of:

- `@Transactional`.
- `@Lock`.
- `LockModeType.PESSIMISTIC_WRITE`, if used.
- Repository methods that acquire locks.
- Transaction boundaries.
- Workforce capacity validation.
- Application approval updates.
- Duplicate approval prevention.
- Rollback behavior.
- Lock timeout configuration, if present.

The rebuilt application must preserve the existing concurrency protection.

## 7. Index and Constraint Verification

Inspect the existing database schema and verify:

- Primary key constraints.
- Foreign key constraints.
- Unique constraints.
- NOT NULL constraints.
- Existing indexes.
- Composite indexes.
- Location-related indexes, if any.
- Database-level check constraints.

Indexes must be justified by actual queries and access patterns.

Do not add speculative indexes simply because a column exists.

## 8. Database Schema Verification

Compare the old database schema with the proposed design.

| Verification item | Expected result |
|---|---|
| Table names | Match existing implementation |
| Column names | Match existing implementation |
| Column data types | Match existing implementation |
| Primary keys | Match existing implementation |
| Foreign keys | Match existing relationships |
| Constraints | Preserve business rules |
| Indexes | Preserve useful existing indexes |
| Enum representation | Match existing implementation |
| Defaults | Preserve existing behavior |

Any intentional difference must be documented before implementation.

## 9. Implementation Readiness Checklist

### Entities

- [ ] All existing entities identified.
- [ ] All entity fields verified.
- [ ] Primary keys verified.
- [ ] ID generation strategies verified.
- [ ] Enum values verified.

### Relationships

- [ ] All JPA relationships verified.
- [ ] Relationship ownership verified.
- [ ] Foreign key columns verified.
- [ ] Cascade configurations verified.
- [ ] Fetch strategies verified.

### Constraints

- [ ] Unique constraints verified.
- [ ] Nullability verified.
- [ ] Foreign key constraints verified.
- [ ] Indexes reviewed.
- [ ] Deletion and retention behavior verified.

### Business Logic

- [ ] Service listing lifecycle verified.
- [ ] Application lifecycle verified.
- [ ] Workforce capacity rules verified.
- [ ] Contractor blocking behavior verified.
- [ ] Payment and payout behavior verified.

### Transactions

- [ ] Transaction boundaries verified.
- [ ] Pessimistic locking implementation verified.
- [ ] Concurrent allocation behavior verified.
- [ ] Rollback behavior verified.

## 10. Database Implementation Sign-off

The database implementation phase can begin only after the important verification items are completed.

| Area | Status |
|---|---|
| Entity definitions | Pending old source code review |
| Relationships | Pending old source code review |
| Constraints and indexes | Pending old source code review |
| Workforce allocation | Pending old source code review |
| Payment and payout | Pending old source code review |
| Database schema | Pending comparison |

**Completion status:** Not yet verified.

This document is a checklist, not confirmation that the old database schema has already been inspected.

## 11. Next Phase

After completing the database verification, proceed to:

**Phase 4 — API Documentation**

Planned documentation:

1. REST API overview.
2. Authentication and authorization endpoints.
3. User and profile endpoints.
4. Service listing endpoints.
5. Application and workforce allocation endpoints.
6. Notification endpoints.
7. Payment and payout endpoints.
8. Super Admin endpoints.

All endpoint paths, HTTP methods, request bodies, response structures, and permissions must be verified against the existing WorkBridge implementation before being documented.

No backend implementation should begin until the documentation review is complete.