# API Contract: Habit Tracking App

**Created**: 2026-03-09  
**Purpose**: Define REST API endpoints for habit management and sync

**Base URL**: `https://api.habitseed.app/v1`  
**Authentication**: JWT Bearer token in `Authorization` header  
**Content-Type**: `application/json`  
**Protocol**: HTTPS only (TLS 1.3+)

---

## Authentication

### Register / Login (Future)

```
POST /auth/register
POST /auth/login
POST /auth/refresh
```

**Status**: Out of scope for MVP (single user per device for now)

---

## Habits Endpoints

### 1. Create Habit

```http
POST /habits
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "name": "Morning jog",
  "description": "Run 30 minutes before breakfast",
  "frequency": "DAILY"
}
```

**Success Response (201)**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "660e8400-e29b-41d4-a716-446655440000",
  "name": "Morning jog",
  "description": "Run 30 minutes before breakfast",
  "frequency": "DAILY",
  "createdAt": "2026-03-09T08:00:00Z",
  "updatedAt": "2026-03-09T08:00:00Z",
  "isDeleted": false
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input (missing name, frequency not recognized, etc.)
- `401 Unauthorized`: Missing or invalid JWT token
- `422 Unprocessable Entity`: Name too long, blank, etc.

---

### 2. Get All Habits

```http
GET /habits
Authorization: Bearer <JWT>
```

**Query Parameters**:
- `includeDeleted` (optional): `true` to include soft-deleted habits (default: false)

**Success Response (200)**:
```json
{
  "habits": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "userId": "660e8400-e29b-41d4-a716-446655440000",
      "name": "Morning jog",
      "description": "Run 30 minutes before breakfast",
      "frequency": "DAILY",
      "createdAt": "2026-03-09T08:00:00Z",
      "updatedAt": "2026-03-09T08:00:00Z",
      "isDeleted": false
    }
  ]
}
```

**Error Responses**:
- `401 Unauthorized`: Missing or invalid JWT token

---

### 3. Get Habit by ID

```http
GET /habits/:habitId
Authorization: Bearer <JWT>
```

**Success Response (200)**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "660e8400-e29b-41d4-a716-446655440000",
  "name": "Morning jog",
  "description": "Run 30 minutes before breakfast",
  "frequency": "DAILY",
  "createdAt": "2026-03-09T08:00:00Z",
  "updatedAt": "2026-03-09T08:00:00Z",
  "isDeleted": false
}
```

**Error Responses**:
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Habit doesn't exist or belongs to different user

---

### 4. Update Habit

```http
PUT /habits/:habitId
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "name": "Morning run (updated)",
  "description": "Run 45 minutes",
  "frequency": "DAILY"
}
```

**Success Response (200)**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "660e8400-e29b-41d4-a716-446655440000",
  "name": "Morning run (updated)",
  "description": "Run 45 minutes",
  "frequency": "DAILY",
  "createdAt": "2026-03-09T08:00:00Z",
  "updatedAt": "2026-03-09T10:30:00Z",
  "isDeleted": false
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Habit doesn't exist

---

### 5. Delete Habit (Soft Delete)

```http
DELETE /habits/:habitId
Authorization: Bearer <JWT>
```

**Success Response (200)**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "isDeleted": true,
  "deletedAt": "2026-03-09T10:30:00Z"
}
```

**Error Responses**:
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Habit doesn't exist

---

## Completions Endpoints

### 1. Record Completion

```http
POST /completions
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "habitId": "550e8400-e29b-41d4-a716-446655440000",
  "completedAt": "2026-03-09",
  "notes": "Ran 5 km at pace"
}
```

**Parameters**:
- `habitId` (string, required): UUID of the habit
- `completedAt` (string, required): ISO date (YYYY-MM-DD) or ISO timestamp
- `notes` (string, optional): Max 255 characters

**Success Response (201)**:
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440000",
  "habitId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "660e8400-e29b-41d4-a716-446655440000",
  "completedAt": "2026-03-09T00:00:00Z",
  "recordedAt": "2026-03-09T10:35:00Z",
  "notes": "Ran 5 km at pace",
  "isSyncPending": false
}
```

**Error Responses**:
- `400 Bad Request`: Invalid date format, future date, etc.
- `401 Unauthorized`: Missing or invalid JWT token
- `409 Conflict`: Completion already exists for this habit on this date (respond with existing record)

---

### 2. Get Completions for Habit

```http
GET /habits/:habitId/completions
Authorization: Bearer <JWT>
```

**Query Parameters**:
- `from` (optional): ISO date to filter from (inclusive)
- `to` (optional): ISO date to filter to (inclusive)
- `limit` (optional): Max results (default: 100)

**Success Response (200)**:
```json
{
  "habitId": "550e8400-e29b-41d4-a716-446655440000",
  "completions": [
    {
      "id": "770e8400-e29b-41d4-a716-446655440000",
      "habitId": "550e8400-e29b-41d4-a716-446655440000",
      "userId": "660e8400-e29b-41d4-a716-446655440000",
      "completedAt": "2026-03-09T00:00:00Z",
      "recordedAt": "2026-03-09T10:35:00Z",
      "notes": "Ran 5 km at pace",
      "isSyncPending": false
    }
  ]
}
```

**Error Responses**:
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Habit doesn't exist

---

### 3. Delete Completion

```http
DELETE /completions/:completionId
Authorization: Bearer <JWT>
```

**Success Response (200)**:
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440000",
  "deleted": true
}
```

**Error Responses**:
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Completion doesn't exist

---

## Sync / Batch Endpoints (Future)

```http
POST /sync
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "clientVersion": "1.0.0",
  "lastSyncAt": "2026-03-08T10:30:00Z",
  "changes": {
    "habits": [
      { "operation": "CREATE", "data": {...} },
      { "operation": "UPDATE", "data": {...} },
      { "operation": "DELETE", "id": "..." }
    ],
    "completions": [...]
  }
}
```

**Status**: Out of scope for MVP (sync implemented via individual endpoints)

---

## Error Response Format

All errors return consistent JSON:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Name is required",
  "details": {
    "field": "name",
    "reason": "empty"
  }
}
```

**Common Error Codes**:
- `VALIDATION_ERROR` (400): Input validation failed
- `UNAUTHORIZED` (401): JWT missing or invalid
- `FORBIDDEN` (403): User doesn't have permission
- `NOT_FOUND` (404): Resource not found
- `CONFLICT` (409): Duplicate resource (e.g., completion already exists)
- `INTERNAL_SERVER_ERROR` (500): Server error

---

## Rate Limiting

- **Limit**: 100 requests per minute per user
- **Headers**: 
  - `X-RateLimit-Limit: 100`
  - `X-RateLimit-Remaining: 95`
  - `X-RateLimit-Reset: 1678440000`

---

## Versioning

- Current version: **v1**
- Authentication: Legacy (single device user)
- Future: v2 with multi-user accounts, OAuth, advanced sync

---

## Notes for Implementation

1. **JWT Validation**: Verify token signature and expiration on every request
2. **User Isolation**: Always filter queries by authenticated userId (from JWT)
3. **Timestamps**: All timestamps in response must be UTC ISO 8601 format
4. **Idempotency**: Batch operations should be idempotent (safe to retry)
5. **HTTPS Only**: Reject any HTTP requests with 400 Bad Request
