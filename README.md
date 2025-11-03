# Sports Management System REST API - Design Solution

## Overview

REST API design for sports management system using content negotiation via Accept headers and following industry-standard best practices.

**Note**: This specification demonstrates the approach using the Runners module as an example, but the principles and patterns apply to all system resources (Coaches, etc.).

---

## 🎯 Key Architectural Decisions

### 1. Content Negotiation via Accept Header

**Solution**: Using Accept header instead of query parameter `?view=...`

**Rationale**:
- ✅ Strict typing at OpenAPI specification level
- ✅ Each Content-Type is tightly bound to its schema
- ✅ Automatic generation of typed clients
- ✅ Compliance with REST principles (content negotiation)
- ✅ Support for standard 406 Not Acceptable response code

**Rejected alternatives**:
- Query parameter `?view=detail` - weak type binding
- Separate endpoints `/runners/{id}/detail` - violates "one resource = one URL" principle
- Accept header was chosen as the more correct approach

---

## 📐 Media Types Convention

### Adopted pattern:
```
application/vnd.api.{resource}.{representation}+json
```

### For any system resource:
```
application/json                              # base normalized representation (default)
application/vnd.api.{resource}.list+json      # list representation (tables, denormalization)
application/vnd.api.{resource}.lookup+json    # for dropdown/select (id + name)
application/vnd.api.{resource}.detail+json    # enriched with denormalized relations (for single resource only)
```

### Examples for different resources:
```
application/json                              # base representation (any resource)
application/vnd.api.runner.list+json         # Runners - lists/tables
application/vnd.api.runner.lookup+json       # Runners - dropdown
application/vnd.api.runner.detail+json       # Runners - detail
application/vnd.api.coach.list+json           # Coaches - lists/tables
application/vnd.api.coach.lookup+json         # Coaches - dropdown
```

### Media Type anatomy:
```
application/vnd.api.{resource}.{representation}+json
│           │   │   │         │              │
│           │   │   │         │              └─ suffix: data format
│           │   │   │         └───────────────── representation: view
│           │   │   └─────────────────────────── resource: resource type
│           │   └─────────────────────────────── vendor: project/company
│           └─────────────────────────────────── tree: vendor-specific
└─────────────────────────────────────────────── type: application
```

**Rationale for `vnd.` prefix choice**:
- Standard for vendor-specific APIs (used by GitHub, Stripe, Heroku)
- Suitable for proprietary API products
- IANA registerable (optional)

---

## 🗂️ Data Schemas and Naming Conventions

### Response Models (what server returns)

#### 1. `{Resource}` - Base Normalized Representation
**Media Type**: `application/json`

**Example for Runner**:

```typescript
interface Runner {
  id: string;              // UUID, readonly, server-generated
  coachId: string;         // UUID, FK to Coach
  name: string;            // 1-255 characters
  email: string;           // unique email
  phone: string | null;    // E.164 format, min 8 digits (+12025551234)
  languageId: string;      // language UUID
}
```

**Use cases**:
- Default standard representation
- Normalized data "as is" from database
- Admin panels
- Debugging and auditing
- Data export
- When no specific Accept header is provided

---

#### 2. `{Resource}ListItem` - List Representation
**Media Type**: `application/vnd.api.{resource}.list+json`

**Example for RunnerListItem**:

```typescript
interface RunnerListItem {
  id: string;              // UUID
  name: string;            // runner name
  email: string;           // runner email
  phone: string;           // phone number
  coachId: string;         // coach UUID
  coachName: string;       // coach name (denormalized!)
}
```

**Use cases**:
- Lists in tables (grid/table views)
- Denormalized data for performance
- When related data is needed without client-side JOIN
- List endpoints with pagination

---

#### 3. `{Resource}Lookup` - For dropdown/select
**Media Type**: `application/vnd.api.{resource}.lookup+json`

**Example for RunnerLookup**:

```typescript
interface RunnerLookup {
  id: string;              // UUID
  name: string;            // display name
}
```

**Use cases**:
- Dropdown lists (select boxes)
- Autocomplete fields
- Reference fields in forms
- Minimal data volume for selection

---

#### 4. `{Resource}Detail` - Enriched with denormalized relations
**Media Type**: `application/vnd.api.{resource}.detail+json`

**Example for RunnerDetail**:

```typescript
interface RunnerDetail {
  id: string;
  name: string;
  email: string;
  phone: string | null;
  coachId: string;         // coach UUID
  coachName: string;       // coach name (denormalized!)
  languageId: string;      // language UUID
  languageName: string;    // language name (denormalized!)
}
```

**Use cases**:
- Profile cards and detail views
- Edit forms with related entity names pre-loaded
- Rich UI displays without additional API calls
- Combines base entity with denormalized related data

---


---

### Request Models (what client sends)

#### 6. `{Resource}Create` - Resource creation (POST)

**Example for RunnerCreate**:

```typescript
interface RunnerCreate {
  coachId: string;         // required
  name: string;            // required
  email: string;           // required
  phone?: string | null;   // optional
  languageId: string;      // required
}
```

**Rationale for separate schema**:
- 🔒 Security: client cannot forge `id`
- ✅ Logic: id is server-generated only
- ✅ Validation: clear separation of input and output data

---

#### 7. `{Resource}Update` - Full replacement (PUT)

**Example for RunnerUpdate**:

```typescript
interface RunnerUpdate {
  coachId: string;         // required
  name: string;            // required
  email: string;           // required
  phone: string | null;    // required, must be explicitly specified (even null)
  languageId: string;      // required
}
```

**PUT semantics**:
- Replaces entire resource
- All fields are required
- Missing fields will cause error
- Idempotent operation

---

#### 8. `{Resource}Patch` - Partial update (PATCH)

**Example for RunnerPatch**:

```typescript
interface RunnerPatch {
  coachId?: string;         // optional
  name?: string;            // optional
  email?: string;           // optional
  phone?: string | null;    // optional
  languageId?: string;      // optional
}
```

**PATCH semantics**:
- Updates only specified fields
- All fields are optional
- Other fields remain unchanged
- NOT idempotent operation with concurrent updates

---

## 🏗️ Naming Convention

### Adopted standard: `{Resource}{Operation}`

| Schema | HTTP Method | Purpose | All fields required? |
|-------|-------------|---------|---------------------|
| `{Resource}` | GET | Base reading | ✅ Yes |
| `{Resource}ListItem` | GET | Lists/tables | ✅ Yes |
| `{Resource}Lookup` | GET | Dropdown (id+name) | ✅ Yes |
| `{Resource}Detail` | GET | Enriched with relations | ✅ Yes |
| `{Resource}Create` | POST | Creation | ✅ Yes (except optional) |
| `{Resource}Update` | PUT | Full replacement | ✅ Yes |
| `{Resource}Patch` | PATCH | Partial update | ❌ All optional |

**Examples for different resources**:
- `Runner`, `RunnerListItem`, `RunnerLookup`, `RunnerDetail`, `RunnerCreate`, `RunnerUpdate`, `RunnerPatch`
- `Coach`, `CoachListItem`, `CoachLookup`, `CoachDetail`, `CoachCreate`, `CoachUpdate`, `CoachPatch`

**This is NOT accidental** - this is industry standard, used in:
- REST API Best Practices (Microsoft, Google)
- OpenAPI Generator defaults
- NestJS, Spring Boot, ASP.NET Core
- Major public APIs (Stripe, GitHub, Twilio)

---

## 🛣️ API Endpoints

**Note**: Endpoints for the Runners module are shown as an example. Similar endpoints should be implemented for all system resources.

### 1. `GET /{resource}` - Resource list
**Example**: `GET /runners` - List of runners
- Pagination: `page`, `limit`
- Filtering: `coachId`, `search`
- Sorting: `sort` - comma-separated fields, prefix with `-` for descending
  - Format: `sort=field1,-field2,field3`
  - Examples:
    - `sort=name` (ascending)
    - `sort=-createdAt` (descending)
    - `sort=name,-createdAt` (multiple fields)
  - Default: `-createdAt` (newest first)
- Accept header determines representation:
  - `application/vnd.api.{resource}.list+json` → `RunnerListItem[]` (for tables, with denormalization)
  - `application/vnd.api.{resource}.lookup+json` → `RunnerLookup[]` (for dropdown)
  - `application/json` → `Runner[]` (base, default)
- **Response**: 200 OK with array directly

---

### 2. `POST /{resource}` - Resource creation
**Example**: `POST /runners` - Create runner
- **Body**: `RunnerCreate` schema
- **Response**: 201 Created + Location header
- **Returns**: Full `Runner` object with generated fields
- **Errors**: 409 (email conflict), 422 (validation)

---

### 3. `GET /{resource}/{id}` - Get by ID
**Example**: `GET /runners/{runnerId}` - Get runner by ID
- Accept header determines representation:
  - `application/json` → `Runner` (default, base)
  - `application/vnd.api.{resource}.detail+json` → `RunnerDetail`
- **Response**: 200 OK
- **Errors**: 404 (not found), 406 (unsupported Accept)

---

### 4. `PUT /{resource}/{id}` - Full replacement
**Example**: `PUT /runners/{runnerId}` - Full runner replacement
- **Body**: `RunnerUpdate` schema (all fields required)
- **Response**: 200 OK with full `Runner`
- **Semantics**: Idempotent operation
- **Errors**: 404, 422 (validation)

---

### 5. `PATCH /{resource}/{id}` - Partial update
**Example**: `PATCH /runners/{runnerId}` - Partial runner update
- **Body**: `RunnerPatch` schema (minimum 1 field)
- **Response**: 200 OK with full `Runner`
- **Semantics**: NOT idempotent
- **Errors**: 400 (empty body), 404, 422

---

### 6. `DELETE /{resource}/{id}` - Deletion
**Example**: `DELETE /runners/{runnerId}` - Delete runner
- **Authorization**: Requires `admin` role
- **Response**: 204 No Content (success, empty body)
- **Semantics**: Idempotent
- **Errors**:
  - 401 (missing/invalid token)
  - 403 (valid token but insufficient permissions - user role)
  - 404 (not found)
  - 409 (has dependencies, e.g., related records)

---

### 7. `GET /{parent-resource}/{parentId}/{child-resource}` - Nested resources
**Example**: `GET /coaches/{coachId}/runners` - Coach's runners
- Nested resource endpoint
- Same pagination parameters and Accept header
- **Response**: 200 OK with array directly
- **Errors**: 404 (coach not found)

---

## 📊 HTTP Methods and Semantics

| Method | Idempotent? | Safe? | Purpose | Success Code |
|-------|-------------|-------|---------|-------------|
| GET | ✅ Yes | ✅ Yes | Data retrieval | 200 |
| POST | ❌ No | ❌ No | Resource creation | 201 |
| PUT | ✅ Yes | ❌ No | Full replacement | 200 |
| PATCH | ❌ No | ❌ No | Partial update | 200 |
| DELETE | ✅ Yes | ❌ No | Deletion | 204 |

**Idempotency**: repeated request gives same result  
**Safety**: does not change server state

---

## 🚨 Error Structure

### Uniform format

```json
{
  "message": "Request validation failed"
}
```

**Rationale for simple structure**:
- ✅ Simplicity: Single field, easy to parse
- ✅ Sufficient for most use cases
- ✅ HTTP status code already provides error categorization
- ✅ Message contains all necessary information for debugging and user display

### HTTP Status Codes

| Code | Purpose | When used |
|-----|---------|-----------|
| 200 | OK | GET, PUT, PATCH successful |
| 201 | Created | POST successful + Location header |
| 204 | No Content | DELETE successful (empty body) |
| 400 | Bad Request | Invalid JSON, wrong parameters |
| 401 | Unauthorized | Missing/invalid JWT token |
| 403 | Forbidden | Valid token but insufficient permissions (e.g., non-admin trying to DELETE) |
| 404 | Not Found | Resource not found |
| 406 | Not Acceptable | Unsupported Accept header |
| 409 | Conflict | Email duplicate, cannot delete (dependencies) |
| 422 | Unprocessable Entity | Data validation error |
| 500 | Internal Server Error | Server error |

---

## 🔐 Authentication & Authorization

### JWT Bearer Token

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### JWT Payload Structure

```json
{
  "sub": "user-uuid",           // User ID
  "email": "user@example.com",  // User email
  "role": "user",               // Role: "user" or "admin"
  "iat": 1609459200,            // Issued at
  "exp": 1609545600             // Expiration
}
```

### Role-Based Access Control (RBAC)

**Roles**:
- `user` - Regular user with standard permissions
- `admin` - Administrator with full permissions

**Permissions by role**:

| Operation | Endpoint Example | user | admin |
|-----------|-----------------|------|-------|
| GET (list) | `GET /runners` | ✅ | ✅ |
| GET (by ID) | `GET /runners/{id}` | ✅ | ✅ |
| POST | `POST /runners` | ✅ | ✅ |
| PUT | `PUT /runners/{id}` | ✅ | ✅ |
| PATCH | `PATCH /runners/{id}` | ✅ | ✅ |
| DELETE | `DELETE /runners/{id}` | ❌ | ✅ |

**DELETE operations** require `admin` role:
- ❌ User with `role: "user"` → 403 Forbidden
- ✅ User with `role: "admin"` → 204 No Content (success)

### Security Rules

- ✅ All endpoints require authentication
- ✅ Token validated by middleware on every request
- ✅ Role extracted from JWT and checked for DELETE operations
- ❌ Without token → 401 Unauthorized
- ❌ Invalid/expired token → 401 Unauthorized
- ❌ DELETE without admin role → 403 Forbidden

---

## 📈 Pagination

### Strategy: Cursor-based without COUNT

**Adopted approach**: Pagination without total count metadata

**Rationale**:
- ✅ **Performance**: No expensive `COUNT(*)` queries on large tables
- ✅ **Scalability**: Works efficiently with millions of records
- ✅ **Consistency**: No race conditions when data changes between requests
- ✅ **Filtering**: COUNT queries become very slow with complex WHERE clauses
- ❌ Trade-off: Client cannot know total pages/records upfront

### Query parameters
- `page` - page number (1-indexed, default: 1)
- `limit` - items per page (1-100, default: 20)

### Response format: Array without metadata wrapper

List endpoints return **array directly** (no wrapper object):

```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    ...
  },
  {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "name": "Jane Smith",
    ...
  }
]
```

**Client determines "has next page"**:
```javascript
// Request with limit=20
const response = await fetch('/runners?page=1&limit=20');
const items = await response.json();

// If received exactly 20 items, likely has next page
const hasNextPage = items.length === 20;

// Request next page
if (hasNextPage) {
  const nextPage = await fetch('/runners?page=2&limit=20');
}
```

**Benefits of direct array response**:
- ✅ Simpler client code (no `response.data` unwrapping)
- ✅ Smaller payload size (no wrapper overhead)
- ✅ Standard RESTful pattern (used by GitHub, Stripe, Slack)
- ✅ Easy to work with in TypeScript: `Runner[]` instead of `{ data: Runner[] }`

---

## 💡 Best Practices implemented in API

1. **Versioning**: `/v1/` in URL for future breaking changes
2. **Pagination**: mandatory for all list endpoints, cursor-based without COUNT for performance
3. **Filtering**: via query parameters (REST standard)
4. **Sorting**: flexible multi-field sorting with `sort=field1,-field2` format (JSON:API standard)
5. **Validation**: at schema level (formats, patterns, length, regex)
6. **Errors**: simple uniform format with single message field
7. **Security**: JWT for all requests, readonly fields protected
8. **REST semantics**: proper HTTP method usage
9. **Readonly fields**: `id` cannot be changed by client
10. **Location header**: when creating resource (201) points to new resource
11. **Nested resources**: `/coaches/{id}/runners` for related data
12. **Content negotiation**: Accept header for different representations
13. **Default representation**: `application/json` → base normalized representation
14. **Performance-first**: no expensive COUNT queries on large datasets

---

## 🎯 Typical Use Cases

### Scenario 1: Dropdown list of runners for select
```http
GET /runners?limit=100
Accept: application/vnd.api.runner.lookup+json
```
Returns only `id`, `name` - minimum data for selection.

---

### Scenario 2: Runners table with coach name, sorted by name
```http
GET /runners?page=1&limit=20&sort=name
Accept: application/vnd.api.runner.list+json
```
Returns denormalized data including `coachName`, sorted alphabetically by name - without additional client requests.

---

### Scenario 3: Runner detail view
```http
GET /runners/550e8400...
Accept: application/vnd.api.runner.detail+json
```
Returns extended data with denormalized coach and language names - no additional requests needed.

---

### Scenario 4: Admin panel with base data
```http
GET /runners/550e8400...
Accept: application/json
```
Returns base normalized representation with all DB fields including metadata for auditing.

---

### Scenario 5: Create new runner
```http
POST /runners
Content-Type: application/json

{
  "coachId": "660e8400...",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+12025551235",
  "languageId": "560e8400..."
}

→ 201 Created
Location: /v1/runners/550e8400...
{
  "id": "550e8400...",  // generated
  "coachId": "660e8400...",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+12025551235",
  "languageId": "560e8400..."
}
```

---

### Scenario 6: Update only phone number
```http
PATCH /runners/550e8400...
Content-Type: application/json

{
  "phone": "+12025559999"
}

→ 200 OK (returns full Runner with updated phone)
```

---

## ✅ Final Solutions Checklist

- ✅ Content negotiation via **Accept header**
- ✅ Media Type: `application/json` (base) and `application/vnd.api.{resource}.{representation}+json` (special)
- ✅ Four representations: **base** (application/json), **list**, **lookup**, **detail** (single resource)
- ✅ Seven schemas: **Runner**, **RunnerListItem**, **RunnerLookup**, **RunnerDetail**, **RunnerCreate**, **RunnerUpdate**, **RunnerPatch**
- ✅ Naming convention: `{Resource}{Operation}` (industry standard)
- ✅ REST semantics: proper usage of GET/POST/PUT/PATCH/DELETE
- ✅ Cursor-based pagination without COUNT for performance (direct array response)
- ✅ Filtering and sorting
- ✅ Simple uniform error format (single message field)
- ✅ JWT authentication on all endpoints
- ✅ Nested resources for related data
- ✅ Location header when creating resource
- ✅ Readonly fields protected from modification
- ✅ API versioning (/v1/)
- ✅ Performance-first approach (no expensive aggregations)

---

**This is a professional REST API, ready for production deployment! 🚀**

---

## 📋 Extension to Other Resources

This specification demonstrates the approach using the **Runners** module as an example. For a complete sports management system, similar endpoints should be implemented for:

### Main resources:
- **Coaches** (`/coaches`, `/coaches/{id}`, etc.)
- **Users** (`/users`, `/users/{id}`, etc.)

### Nested resources:
- `/coaches/{coachId}/runners` ✅ (already implemented)

### Pattern application:
Each new resource should follow the same principles:
- ✅ Content negotiation via Accept header
- ✅ Four representations: base (application/json), list, lookup, detail (single resource)
- ✅ Seven schemas: {Resource}, {Resource}ListItem, {Resource}Lookup, {Resource}Detail, {Resource}Create, {Resource}Update, {Resource}Patch
- ✅ Standard HTTP methods and response codes
- ✅ Pagination, filtering, sorting
- ✅ Uniform errors
- ✅ JWT authentication

