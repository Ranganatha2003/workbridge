# Service Management API Documentation

## 1. Purpose

This document describes the service management APIs implemented in the original WorkBridge backend.

The service module manages work opportunities published by contractors.

It covers:

- Creating service listings.
- Retrieving all services.
- Retrieving an individual service.
- Retrieving services belonging to the authenticated contractor.
- Searching services by geographical radius.
- Updating service listings.
- Cancelling services.
- Completing services.
- Workforce requirement validation and service status transitions.

**Source of truth:** Original WorkBridge backend source code.

## 2. Controller and Base URL

**Controller:** `ServiceController`

**Service class:** `ServiceListingService`

**Repository:** `ServiceListingRepository`

The controller supports two base paths:

```text
/api/services
/services
```

The endpoint examples below use `/api/services`.

All request and response bodies use JSON.

## 3. Service Listing Data Model

The service listing is represented by the `ServiceListing` entity and stored in the `services` table.

| Field | Type | Description |
|---|---|---|
| `id` | Long | Service primary key |
| `contractor` | ContractorProfile | Contractor who created the service |
| `category` | ServiceCategory | Service category |
| `title` | String | Service title |
| `description` | String | Service description |
| `address` | String | Service location address |
| `latitude` | Double | Geographic latitude |
| `longitude` | Double | Geographic longitude |
| `requiredWorkers` | Integer | Total workers required |
| `acceptedWorkers` | Integer | Number of accepted workers |
| `serviceDate` | LocalDate | Date of the service |
| `startTime` | LocalTime | Service starting time |
| `endTime` | LocalTime | Service ending time |
| `status` | ServiceStatus | Current service status |
| `version` | Integer | JPA optimistic version field |
| `createdAt` | LocalDateTime | Creation timestamp |
| `updatedAt` | LocalDateTime | Last update timestamp |

A service belongs to one contractor profile and one service category.

## 4. Service Status Values

The original backend defines the following statuses in `ServiceStatus.java`:

| Status | Description |
|---|---|
| `DRAFT` | Draft service state |
| `OPEN` | Service is available for applications |
| `FILLED` | Required workforce has been allocated |
| `IN_PROGRESS` | Service is in progress |
| `COMPLETED` | Service has been completed |
| `CANCELLED` | Service has been cancelled |
| `EXPIRED` | Service has expired |

The service creation method initially sets the status to `OPEN`.

## 5. Create Service API

### Endpoint

`POST /api/services`

**Authorization:** Contractor role required.

**Success status:** `201 Created`

**Request DTO:** `ServiceCreateRequest`

### Request fields

| Field | Type | Validation |
|---|---|---|
| `categoryId` | Long | Required |
| `title` | String | Required, not blank |
| `description` | String | Required, not blank |
| `address` | String | Required, not blank |
| `latitude` | Double | Required |
| `longitude` | Double | Required |
| `requiredWorkers` | Integer | Required, minimum 1 |
| `serviceDate` | LocalDate | Required, present or future |
| `startTime` | LocalTime | Required |
| `endTime` | LocalTime | Required |

### Example request

```json
{
  "categoryId": 1,
  "title": "Event Catering Staff",
  "description": "Workers required for event catering.",
  "address": "Bengaluru, Karnataka",
  "latitude": 12.9716,
  "longitude": 77.5946,
  "requiredWorkers": 10,
  "serviceDate": "2026-12-15",
  "startTime": "09:00:00",
  "endTime": "18:00:00"
}
```

### Business logic

The `createService()` method performs these operations:

1. Validates that the start time is before the end time.
2. Retrieves the contractor profile using the authenticated user's email.
3. Retrieves the requested service category.
4. Creates a new service listing.
5. Sets `acceptedWorkers` to `0`.
6. Sets the initial status to `OPEN`.
7. Saves the service listing.
8. Maps the saved entity to `ServiceResponse`.

The contractor identity is obtained from the authenticated principal. The request does not allow the client to choose the contractor ID.

## 6. Retrieve All Services API

### Endpoint

`GET /api/services`

**Authorization:** No role restriction is declared on this controller method.

**Success status:** `200 OK`

Returns a list of all service listings.

### Response structure

```json
[
  {
    "id": 1,
    "contractorId": 10,
    "contractorBusinessName": "Ravi Event Services",
    "categoryId": 1,
    "categoryName": "Catering",
    "title": "Event Catering Staff",
    "description": "Workers required for event catering.",
    "address": "Bengaluru, Karnataka",
    "latitude": 12.9716,
    "longitude": 77.5946,
    "requiredWorkers": 10,
    "acceptedWorkers": 0,
    "serviceDate": "2026-12-15",
    "startTime": "09:00:00",
    "endTime": "18:00:00",
    "status": "OPEN",
    "createdAt": "2026-11-05T10:00:00",
    "updatedAt": "2026-11-05T10:00:00"
  }
]
```

The example values are illustrative. Actual values depend on the database.

## 7. Retrieve Service By ID API

### Endpoint

`GET /api/services/{id}`

**Authorization:** No role restriction is declared on this controller method.

**Success status:** `200 OK`

### Example

```text
GET /api/services/1
```

### Business logic

1. Retrieves the service using its ID.
2. Throws `ResourceNotFoundException` if the service does not exist.
3. Converts the entity into `ServiceResponse`.
4. Returns the service details.

## 8. Retrieve Contractor's Own Services API

### Endpoint

`GET /api/services/my-services`

**Authorization:** Contractor role required.

**Success status:** `200 OK`

### Example

```text
GET /api/services/my-services
```

### Business logic

1. Reads the authenticated contractor's email.
2. Retrieves the contractor profile associated with that email.
3. Retrieves services using the contractor profile ID.
4. Converts the results into response DTOs.
5. Returns the contractor's service listings.

The contractor ID is derived from authentication rather than supplied by the frontend.

## 9. Search Services By Radius API

### Endpoint

`GET /api/services/search/radius`

**Authorization:** No role restriction is declared on this controller method.

**Success status:** `200 OK`

### Query parameters

| Parameter | Type | Description |
|---|---|---|
| `lat` | Double | Search centre latitude |
| `lon` | Double | Search centre longitude |
| `radiusKm` | Double | Search radius in kilometres |

### Example request

```text
GET /api/services/search/radius?lat=12.9716&lon=77.5946&radiusKm=10
```

### Business logic

The repository uses a native SQL query with the Haversine distance calculation.

The query:

1. Calculates the geographical distance between the supplied coordinates and each service location.
2. Filters services whose status is `OPEN`.
3. Includes services within the requested radius.
4. Orders the results by ascending distance.

The API returns the matching services as a list of `ServiceResponse` objects.

## 10. Update Service API

### Endpoint

`PUT /api/services/{id}`

**Authorization:** Admin or owning contractor, according to the controller's `@PreAuthorize` expression.

**Success status:** `200 OK`

**Request DTO:** `ServiceUpdateRequest`

The update request uses the same fields and validations as `ServiceCreateRequest`.

### Example request

```json
{
  "categoryId": 1,
  "title": "Updated Event Catering Staff",
  "description": "Updated workforce requirement for catering.",
  "address": "Bengaluru, Karnataka",
  "latitude": 12.9716,
  "longitude": 77.5946,
  "requiredWorkers": 15,
  "serviceDate": "2026-12-15",
  "startTime": "09:00:00",
  "endTime": "18:00:00"
}
```

### Business logic

The `updateService()` method performs the following operations:

1. Validates that the start time is before the end time.
2. Retrieves the service by ID.
3. Rejects updates when the service status is `CANCELLED`, `COMPLETED`, or `EXPIRED`.
4. Retrieves the requested category.
5. Checks the new workforce requirement.
6. Rejects a workforce requirement smaller than the already accepted worker count.
7. If the required workforce increases and the previous status was `FILLED`, changes the status back to `OPEN`.
8. Updates the service information.
9. Saves the service.
10. Returns the updated response.

### Workforce capacity rule

The new required worker count cannot be smaller than the already accepted worker count.

For example:

| Existing accepted workers | New required workers | Result |
|---:|---:|---|
| 5 | 10 | Allowed |
| 5 | 5 | Allowed |
| 5 | 4 | Rejected |

When the workforce requirement increases for a `FILLED` service, the service is reopened as `OPEN`.

## 11. Cancel Service API

### Endpoint

`PUT /api/services/{id}/cancel`

**Authorization:** Admin or owning contractor, according to the controller's `@PreAuthorize` expression.

**Success status:** `200 OK`

### Example request

```text
PUT /api/services/1/cancel
```

No request body is required.

### Business logic

1. Retrieves the service by ID.
2. Checks the current status.
3. Rejects cancellation if the service is already `COMPLETED` or `CANCELLED`.
4. Sets the service status to `CANCELLED`.
5. Saves the service.
6. Returns the updated service response.

The current implementation does not explicitly reject cancellation from `EXPIRED` or `IN_PROGRESS` states.

## 12. Complete Service API

### Endpoint

`PUT /api/services/{id}/complete`

**Authorization:** Admin or owning contractor, according to the controller's `@PreAuthorize` expression.

**Success status:** `200 OK`

### Example request

```text
PUT /api/services/1/complete
```

No request body is required.

### Business logic

1. Retrieves the service by ID.
2. Checks the current service status.
3. Allows completion only when the current status is `OPEN` or `FILLED`.
4. Rejects completion from other statuses.
5. Changes the status to `COMPLETED`.
6. Saves the service.
7. Returns the updated response.

## 13. Service Response DTO

**DTO:** `ServiceResponse`

All service retrieval and mutation operations return the following response fields:

| Field | Type |
|---|---|
| `id` | Long |
| `contractorId` | Long |
| `contractorBusinessName` | String |
| `categoryId` | Long |
| `categoryName` | String |
| `title` | String |
| `description` | String |
| `address` | String |
| `latitude` | Double |
| `longitude` | Double |
| `requiredWorkers` | Integer |
| `acceptedWorkers` | Integer |
| `serviceDate` | LocalDate |
| `startTime` | LocalTime |
| `endTime` | LocalTime |
| `status` | String |
| `createdAt` | LocalDateTime |
| `updatedAt` | LocalDateTime |

The response DTO does not expose the complete contractor profile or the persistence entity.

## 14. Authorization Summary

| Endpoint | Controller-level authorization |
|---|---|
| `POST /api/services` | Contractor |
| `GET /api/services` | No role restriction declared |
| `GET /api/services/{id}` | No role restriction declared |
| `GET /api/services/my-services` | Contractor |
| `GET /api/services/search/radius` | No role restriction declared |
| `PUT /api/services/{id}` | Admin or owning contractor |
| `PUT /api/services/{id}/cancel` | Admin or owning contractor |
| `PUT /api/services/{id}/complete` | Admin or owning contractor |

The security configuration permits requests to `/api/services/**` and `/services/**` at the URL authorization layer. Method-level `@PreAuthorize` rules provide additional restrictions on selected operations.

The authenticated user's identity is used to identify the contractor for service creation and the contractor's own-service retrieval.

## 15. Current Implementation Gaps

The following points are important when rebuilding the module.

### 15.1 Delete endpoint is missing

The original functional requirements mention deleting service listings.

However, the inspected `ServiceController` does not contain a `@DeleteMapping` endpoint.

Therefore, service deletion is a documented functional requirement but is not an implemented API in the original controller.

### 15.2 Category management endpoints are missing

The service module contains:

- `ServiceCategory` entity.
- `ServiceCategoryRepository`.
- Category lookup by ID and name.

However, no category controller or category CRUD API was found in the inspected backend source.

The service creation and update operations require an existing category ID.

### 15.3 No pagination is implemented

The current service retrieval methods return lists without pagination parameters.

### 15.4 Radius search only returns OPEN services

The repository query explicitly filters using:

```sql
WHERE s.status = 'OPEN'
```

Other service statuses are excluded from radius search.

### 15.5 Service status rules are implemented in the service layer

The update, cancel, and complete methods enforce different status conditions. These conditions must be preserved when rebuilding the service module.

---

## 16. Rebuild Requirements

During the Spring Boot rebuild:

1. Preserve the existing service listing fields.
2. Preserve the contractor and category relationships.
3. Preserve the service creation behavior.
4. Preserve the workforce capacity validation.
5. Preserve the radius-based geographical search.
6. Preserve contractor ownership checks.
7. Preserve cancellation and completion rules.
8. Preserve the service status enum and its existing values.
9. Reproduce the actual response DTO structure.
10. Treat the missing delete endpoint and category APIs as gaps requiring an explicit decision before implementation.




