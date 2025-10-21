# Sports Management System REST API - Design Solution

## Overview

Professional REST API design for sports management system using content negotiation via Accept headers and following industry-standard best practices.

**Note**: This specification demonstrates the approach using the Athletes module as an example, but the principles and patterns apply to all system resources (Coaches, etc.).

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
- Query parameter `?view=summary` - weak type binding
- Separate endpoints `/athletes/{id}/summary` - violates "one resource = one URL" principle
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
application/vnd.api.{resource}.detail+json    # extended without readonly fields (for single resource only)
application/vnd.api.{resource}.summary+json   # minimal representation (for single resource only)
```

### Examples for different resources:
```
application/json                              # base representation (any resource)
application/vnd.api.athlete.list+json         # Athletes - lists/tables
application/vnd.api.athlete.lookup+json       # Athletes - dropdown
application/vnd.api.athlete.detail+json       # Athletes - detail
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

**Example for Athlete**:

```typescript
interface Athlete {
  id: string;              // UUID, readonly, server-generated
  coachId: string;         // UUID, FK to Coach
  name: string;            // 1-255 characters
  email: string;           // unique email
  phone: string;           // E.164 format (+1234567890)
  telegram: string | null; // @username or null
  createdAt: string;       // ISO 8601, readonly
  updatedAt: string;       // ISO 8601, readonly
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

**Example for AthleteListItem**:

```typescript
interface AthleteListItem {
  id: string;              // UUID
  name: string;            // athlete name
  email: string;           // athlete email
  phone: string;           // phone number
  coachId: string;         // coach UUID
  coachName: string;       // coach name (denormalized!)
  createdAt: string;       // ISO 8601
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

**Example for AthleteLookup**:

```typescript
interface AthleteLookup {
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

#### 4. `{Resource}Detail` - Extended without system fields
**Media Type**: `application/vnd.api.{resource}.detail+json`

**Example for AthleteDetail**:

```typescript
interface AthleteDetail {
  id: string;
  name: string;
  email: string;
  phone: string;
  telegram: string | null;
  // Excluded: coachId, createdAt, updatedAt
}
```

**Use cases**:
- Profile cards
- Edit forms
- User-facing interfaces where system fields are not needed

---

#### 5. `{Resource}Summary` - Brief Representation
**Media Type**: `application/vnd.api.{resource}.summary+json`

**Example for AthleteSummary**:

```typescript
interface AthleteSummary {
  id: string;
  name: string;
  email: string;
}
```

**Use cases**:
- Search results preview
- Compact cards
- Lists with many elements (traffic optimization)
- When you need slightly more than lookup, but less than list

---

### Request Models (what client sends)

#### 6. `{Resource}Create` - Resource creation (POST)

**Example for AthleteCreate**:

```typescript
interface AthleteCreate {
  coachId: string;         // required
  name: string;            // required
  email: string;           // required
  phone: string;           // required
  telegram?: string | null; // optional
  // Excluded: id, createdAt, updatedAt (server-generated)
}
```

**Rationale for separate schema**:
- 🔒 Security: client cannot forge `id`
- ✅ Logic: timestamps are server-generated only
- ✅ Validation: clear separation of input and output data

---

#### 7. `{Resource}Update` - Full replacement (PUT)

**Example for AthleteUpdate**:

```typescript
interface AthleteUpdate {
  coachId: string;         // required
  name: string;            // required
  email: string;           // required
  phone: string;           // required
  telegram: string | null; // required, must be explicitly specified (even null)
  // Excluded: id, createdAt, updatedAt
}
```

**PUT semantics**:
- Replaces entire resource
- All fields are required
- Missing fields will cause validation error
- Idempotent operation

---

#### 8. `{Resource}Patch` - Partial update (PATCH)

**Example for AthletePatch**:

```typescript
interface AthletePatch {
  coachId?: string;         // optional
  name?: string;            // optional
  email?: string;           // optional
  phone?: string;           // optional
  telegram?: string | null; // optional
  // At least 1 field must be sent
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
| `{Resource}Detail` | GET | Extended reading | ✅ Yes |
| `{Resource}Summary` | GET | Brief reading | ✅ Yes |
| `{Resource}Create` | POST | Creation | ✅ Yes (except optional) |
| `{Resource}Update` | PUT | Full replacement | ✅ Yes |
| `{Resource}Patch` | PATCH | Partial update | ❌ All optional |

**Examples for different resources**:
- `Athlete`, `AthleteListItem`, `AthleteLookup`, `AthleteDetail`, `AthleteSummary`, `AthleteCreate`, `AthleteUpdate`, `AthletePatch`
- `Coach`, `CoachListItem`, `CoachLookup`, `CoachDetail`, `CoachSummary`, `CoachCreate`, `CoachUpdate`, `CoachPatch`

**This is NOT accidental** - this is industry standard, used in:
- REST API Best Practices (Microsoft, Google)
- OpenAPI Generator defaults
- NestJS, Spring Boot, ASP.NET Core
- Major public APIs (Stripe, GitHub, Twilio)

---

## 🛣️ API Endpoints

**Note**: Endpoints for the Athletes module are shown as an example. Similar endpoints should be implemented for all system resources.

### 1. `GET /{resource}` - Resource list
**Example**: `GET /athletes` - List of athletes
- Pagination: `page`, `limit`
- Filtering: `coachId`, `search`
- Sorting: `sort` (name_asc, name_desc, created_asc, created_desc)
- Accept header determines representation:
  - `application/vnd.api.{resource}.list+json` → `AthleteListItem[]` (for tables, with denormalization)
  - `application/vnd.api.{resource}.lookup+json` → `AthleteLookup[]` (for dropdown)
  - `application/json` → `Athlete[]` (base, default)
- **Response**: 200 OK with array + pagination metadata

---

### 2. `POST /{resource}` - Resource creation
**Example**: `POST /athletes` - Create athlete
- **Body**: `AthleteCreate` schema
- **Response**: 201 Created + Location header
- **Returns**: Full `Athlete` object with generated fields
- **Errors**: 409 (email conflict), 422 (validation)

---

### 3. `GET /{resource}/{id}` - Get by ID
**Example**: `GET /athletes/{athleteId}` - Get athlete by ID
- Accept header determines representation:
  - `application/json` → `Athlete` (default, base)
  - `application/vnd.api.{resource}.detail+json` → `AthleteDetail`
  - `application/vnd.api.{resource}.summary+json` → `AthleteSummary` (optional, for compact view)
- **Response**: 200 OK
- **Errors**: 404 (not found), 406 (unsupported Accept)

---

### 4. `PUT /{resource}/{id}` - Full replacement
**Example**: `PUT /athletes/{athleteId}` - Full athlete replacement
- **Body**: `AthleteUpdate` schema (all fields required)
- **Response**: 200 OK with full `Athlete`
- **Semantics**: Idempotent operation
- **Errors**: 404, 422 (validation)

---

### 5. `PATCH /{resource}/{id}` - Partial update
**Example**: `PATCH /athletes/{athleteId}` - Partial athlete update
- **Body**: `AthletePatch` schema (minimum 1 field)
- **Response**: 200 OK with full `Athlete`
- **Semantics**: NOT idempotent
- **Errors**: 400 (empty body), 404, 422

---

### 6. `DELETE /{resource}/{id}` - Deletion
**Example**: `DELETE /athletes/{athleteId}` - Delete athlete
- **Response**: 204 No Content (success, empty body)
- **Semantics**: Idempotent
- **Errors**: 
  - 404 (not found)
  - 409 (has dependencies, e.g., related records)

---

### 7. `GET /{parent-resource}/{parentId}/{child-resource}` - Nested resources
**Example**: `GET /coaches/{coachId}/athletes` - Coach's athletes
- Nested resource endpoint
- Same pagination parameters and Accept header
- **Response**: 200 OK with array + pagination
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
  "error": {
    "code": "VALIDATION_ERROR",           // machine-readable code
    "message": "Request validation failed", // human-readable message
    "details": {                          // additional context
      "field": "email",
      "reason": "Invalid format"
    },
    "path": "/v1/athletes",               // request path
    "timestamp": "2024-01-20T14:25:00Z"   // error time
  }
}
```

### Special format for validation (422)

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "validationErrors": [
      {
        "field": "email",
        "message": "Invalid email format",
        "code": "INVALID_FORMAT"
      },
      {
        "field": "phone",
        "message": "Phone number must be in E.164 format",
        "code": "INVALID_FORMAT"
      }
    ]
  }
}
```

### HTTP Status Codes

| Code | Purpose | When used |
|-----|---------|-----------|
| 200 | OK | GET, PUT, PATCH successful |
| 201 | Created | POST successful + Location header |
| 204 | No Content | DELETE successful (empty body) |
| 400 | Bad Request | Invalid JSON, wrong parameters |
| 401 | Unauthorized | Missing/invalid JWT token |
| 404 | Not Found | Resource not found |
| 406 | Not Acceptable | Unsupported Accept header |
| 409 | Conflict | Email duplicate, cannot delete (dependencies) |
| 422 | Unprocessable Entity | Data validation error |
| 500 | Internal Server Error | Server error |

---

## 🔐 Authentication

**Method**: JWT Bearer token

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

- ✅ All endpoints protected
- ❌ Without token → 401 Unauthorized
- Token validated by middleware on every request

---

## 📈 Pagination

### Query parameters
- `page` - page number (1-indexed, default: 1)
- `limit` - items per page (1-100, default: 20)

### Response metadata
```json
{
  "data": [...],
  "pagination": {
    "page": 1,           // current page
    "limit": 20,         // items per page
    "total": 45,         // total items
    "totalPages": 3      // total pages (ceil(total / limit))
  }
}
```

---

## 💡 Best Practices implemented in API

1. **Versioning**: `/v1/` in URL for future breaking changes
2. **Pagination**: mandatory for all list endpoints
3. **Filtering**: via query parameters (REST standard)
4. **Validation**: at schema level (formats, patterns, length, regex)
5. **Errors**: uniform format with machine-readable codes
6. **Security**: JWT for all requests, readonly fields protected
7. **REST semantics**: proper HTTP method usage
8. **Readonly fields**: `id`, `createdAt`, `updatedAt` cannot be changed by client
9. **Location header**: when creating resource (201) points to new resource
10. **Nested resources**: `/coaches/{id}/athletes` for related data
11. **Content negotiation**: Accept header for different representations
12. **Default representation**: `application/json` → base normalized representation

---

## 🎯 Typical Use Cases

### Scenario 1: Dropdown list of athletes for select
```http
GET /athletes?limit=100
Accept: application/vnd.api.athlete.lookup+json
```
Returns only `id`, `name` - minimum data for selection.

---

### Scenario 2: Athletes table with coach name
```http
GET /athletes?page=1&limit=20
Accept: application/vnd.api.athlete.list+json
```
Returns denormalized data including `coachName` - without additional client requests.

---

### Scenario 3: Athlete edit form
```http
GET /athletes/550e8400...
Accept: application/vnd.api.athlete.detail+json
```
Returns data without `coachId`, `createdAt`, `updatedAt` - clean UI.

---

### Scenario 4: Admin panel with base data
```http
GET /athletes/550e8400...
Accept: application/json
```
Returns base normalized representation with all DB fields including metadata for auditing.

---

### Scenario 5: Create new athlete
```http
POST /athletes
Content-Type: application/json

{
  "coachId": "660e8400...",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+1234567890",
  "telegram": "@jane"
}

→ 201 Created
Location: /v1/athletes/550e8400...
{
  "id": "550e8400...",  // generated
  "coachId": "660e8400...",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+1234567890",
  "telegram": "@jane",
  "createdAt": "2024-01-20T14:25:00Z",  // generated
  "updatedAt": "2024-01-20T14:25:00Z"   // generated
}
```

---

### Scenario 6: Update only phone number
```http
PATCH /athletes/550e8400...
Content-Type: application/json

{
  "phone": "+9876543210"
}

→ 200 OK (returns full Athlete with updated phone)
```

---

## 🔧 Implementation Recommendations

### Client-side (React + TypeScript)

#### Media Type constants

```typescript
// src/api/mediaTypes.ts
export const MediaTypes = {
  JSON: 'application/json', // base representation (default)
  ATHLETE_LIST: 'application/vnd.api.athlete.list+json',
  ATHLETE_LOOKUP: 'application/vnd.api.athlete.lookup+json',
  ATHLETE_DETAIL: 'application/vnd.api.athlete.detail+json',
  ATHLETE_SUMMARY: 'application/vnd.api.athlete.summary+json',
  
  COACH_LIST: 'application/vnd.api.coach.list+json',
  COACH_LOOKUP: 'application/vnd.api.coach.lookup+json',
  COACH_DETAIL: 'application/vnd.api.coach.detail+json',
} as const;

export type MediaType = typeof MediaTypes[keyof typeof MediaTypes];
```

#### API client with Accept header

```typescript
// src/api/client.ts
import { MediaTypes, MediaType } from './mediaTypes';

export class ApiClient {
  private baseUrl: string;
  private token: string | null = null;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  setToken(token: string) {
    this.token = token;
  }

  private async request<T>(
    url: string, 
    options: RequestInit = {},
    acceptType: MediaType = MediaTypes.JSON
  ): Promise<T> {
    const headers: HeadersInit = {
      'Accept': acceptType,
      'Content-Type': 'application/json',
      ...(this.token && { 'Authorization': `Bearer ${this.token}` }),
      ...options.headers,
    };

    const response = await fetch(`${this.baseUrl}${url}`, {
      ...options,
      headers,
    });

    if (!response.ok) {
      const error = await response.json();
      throw new ApiError(response.status, error);
    }

    return response.json();
  }

  // Example methods
  async getAthletes(params: { page?: number; limit?: number }, acceptType: MediaType = MediaTypes.ATHLETE_LIST) {
    const query = new URLSearchParams(params as any).toString();
    return this.request<PaginatedResponse<AthleteListItem>>(`/v1/athletes?${query}`, {}, acceptType);
  }

  async getAthletesForDropdown() {
    return this.request<PaginatedResponse<AthleteLookup>>('/v1/athletes?limit=100', {}, MediaTypes.ATHLETE_LOOKUP);
  }

  async getAthleteById(id: string, acceptType: MediaType = MediaTypes.JSON) {
    return this.request<Athlete>(`/v1/athletes/${id}`, {}, acceptType);
  }

  async createAthlete(data: AthleteCreate) {
    return this.request<Athlete>('/v1/athletes', {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }
}

class ApiError extends Error {
  constructor(public status: number, public error: any) {
    super(error.error?.message || 'API Error');
  }
}
```

#### React Hook example

```typescript
// src/hooks/useAthletes.ts
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '@/api/client';
import { MediaTypes } from '@/api/mediaTypes';

export function useAthletesList(page: number = 1, limit: number = 20) {
  return useQuery({
    queryKey: ['athletes', 'list', page, limit],
    queryFn: () => apiClient.getAthletes({ page, limit }, MediaTypes.ATHLETE_LIST),
  });
}

export function useAthletesLookup() {
  return useQuery({
    queryKey: ['athletes', 'lookup'],
    queryFn: () => apiClient.getAthletesForDropdown(),
  });
}

export function useAthleteDetail(id: string) {
  return useQuery({
    queryKey: ['athletes', 'detail', id],
    queryFn: () => apiClient.getAthleteById(id, MediaTypes.ATHLETE_DETAIL),
  });
}
```

---

### Server-side (Spring Boot + Java)

#### Media Type constants

```java
// src/main/java/com/example/api/constants/MediaTypes.java
package com.example.api.constants;

import org.springframework.http.MediaType;

public final class MediaTypes {
    private MediaTypes() {}

    // Base
    public static final String APPLICATION_JSON_VALUE = MediaType.APPLICATION_JSON_VALUE;
    public static final MediaType APPLICATION_JSON = MediaType.APPLICATION_JSON;

    // Athletes
    public static final String ATHLETE_LIST_VALUE = "application/vnd.api.athlete.list+json";
    public static final MediaType ATHLETE_LIST = MediaType.parseMediaType(ATHLETE_LIST_VALUE);
    
    public static final String ATHLETE_LOOKUP_VALUE = "application/vnd.api.athlete.lookup+json";
    public static final MediaType ATHLETE_LOOKUP = MediaType.parseMediaType(ATHLETE_LOOKUP_VALUE);
    
    public static final String ATHLETE_DETAIL_VALUE = "application/vnd.api.athlete.detail+json";
    public static final MediaType ATHLETE_DETAIL = MediaType.parseMediaType(ATHLETE_DETAIL_VALUE);
    
    public static final String ATHLETE_SUMMARY_VALUE = "application/vnd.api.athlete.summary+json";
    public static final MediaType ATHLETE_SUMMARY = MediaType.parseMediaType(ATHLETE_SUMMARY_VALUE);

    // Coaches
    public static final String COACH_LIST_VALUE = "application/vnd.api.coach.list+json";
    public static final MediaType COACH_LIST = MediaType.parseMediaType(COACH_LIST_VALUE);
    
    public static final String COACH_LOOKUP_VALUE = "application/vnd.api.coach.lookup+json";
    public static final MediaType COACH_LOOKUP = MediaType.parseMediaType(COACH_LOOKUP_VALUE);
}
```

#### Content Negotiation Helper (Optional)

```java
// src/main/java/com/example/api/util/ContentNegotiationHelper.java
// This helper is optional - Spring Boot handles content negotiation automatically
// Use this only if you need custom logic beyond Spring's built-in support

package com.example.api.util;

import com.example.api.constants.MediaTypes;
import com.example.api.exception.NotAcceptableException;
import org.springframework.http.MediaType;

import java.util.List;

public class ContentNegotiationHelper {
    
    public enum RepresentationType {
        BASE, LIST, LOOKUP, DETAIL, SUMMARY
    }
    
    public static RepresentationType determineRepresentation(String acceptHeader) {
        if (acceptHeader == null || acceptHeader.isEmpty()) {
            return RepresentationType.BASE;
        }
        
        List<MediaType> mediaTypes = MediaType.parseMediaTypes(acceptHeader);
        
        for (MediaType mediaType : mediaTypes) {
            String value = mediaType.toString();
            
            if (value.contains(".list+json")) {
                return RepresentationType.LIST;
            }
            if (value.contains(".lookup+json")) {
                return RepresentationType.LOOKUP;
            }
            if (value.contains(".detail+json")) {
                return RepresentationType.DETAIL;
            }
            if (value.contains(".summary+json")) {
                return RepresentationType.SUMMARY;
            }
            if (value.contains("application/json")) {
                return RepresentationType.BASE;
            }
        }
        
        throw new NotAcceptableException("Unsupported Accept header: " + acceptHeader);
    }
}
```

#### Controller with Content Negotiation

```java
// src/main/java/com/example/api/controller/AthleteController.java
package com.example.api.controller;

import com.example.api.constants.MediaTypes;
import com.example.api.dto.*;
import com.example.api.service.AthleteService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.net.URI;

@RestController
@RequestMapping("/v1/athletes")
@RequiredArgsConstructor
public class AthleteController {
    
    private final AthleteService athleteService;
    
    // GET /athletes - Spring dispatches by Accept header
    @GetMapping(produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<PaginatedResponse<Athlete>> getAthletesBase(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(required = false) String coachId,
            @RequestParam(required = false) String search
    ) {
        PageRequest pageRequest = PageRequest.of(page - 1, limit);
        return ResponseEntity.ok(athleteService.getAthletes(pageRequest, coachId, search));
    }
    
    @GetMapping(produces = MediaTypes.ATHLETE_LIST_VALUE)
    public ResponseEntity<PaginatedResponse<AthleteListItem>> getAthletesList(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(required = false) String coachId,
            @RequestParam(required = false) String search
    ) {
        PageRequest pageRequest = PageRequest.of(page - 1, limit);
        return ResponseEntity.ok(athleteService.getAthletesList(pageRequest, coachId, search));
    }
    
    @GetMapping(produces = MediaTypes.ATHLETE_LOOKUP_VALUE)
    public ResponseEntity<PaginatedResponse<AthleteLookup>> getAthletesLookup(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(required = false) String search
    ) {
        PageRequest pageRequest = PageRequest.of(page - 1, limit);
        return ResponseEntity.ok(athleteService.getAthletesLookup(pageRequest, search));
    }
    
    // GET /athletes/{id} - Spring dispatches by Accept header
    @GetMapping(value = "/{id}", produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<Athlete> getAthleteBase(@PathVariable String id) {
        return ResponseEntity.ok(athleteService.getAthleteById(id));
    }
    
    @GetMapping(value = "/{id}", produces = MediaTypes.ATHLETE_DETAIL_VALUE)
    public ResponseEntity<AthleteDetail> getAthleteDetail(@PathVariable String id) {
        return ResponseEntity.ok(athleteService.getAthleteDetail(id));
    }
    
    @PostMapping(consumes = MediaTypes.APPLICATION_JSON_VALUE, produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<Athlete> createAthlete(@Valid @RequestBody AthleteCreate dto) {
        Athlete created = athleteService.createAthlete(dto);
        return ResponseEntity
                .created(URI.create("/v1/athletes/" + created.getId()))
                .body(created);
    }
    
    @PutMapping(value = "/{id}", consumes = MediaTypes.APPLICATION_JSON_VALUE, produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<Athlete> updateAthlete(
            @PathVariable String id,
            @Valid @RequestBody AthleteUpdate dto
    ) {
        Athlete updated = athleteService.updateAthlete(id, dto);
        return ResponseEntity.ok(updated);
    }
    
    @PatchMapping(value = "/{id}", consumes = MediaTypes.APPLICATION_JSON_VALUE, produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<Athlete> patchAthlete(
            @PathVariable String id,
            @Valid @RequestBody AthletePatch dto
    ) {
        Athlete patched = athleteService.patchAthlete(id, dto);
        return ResponseEntity.ok(patched);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteAthlete(@PathVariable String id) {
        athleteService.deleteAthlete(id);
    }
}
```

#### Exception Handler

```java
// src/main/java/com/example/api/exception/NotAcceptableException.java
package com.example.api.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_ACCEPTABLE)
public class NotAcceptableException extends RuntimeException {
    public NotAcceptableException(String message) {
        super(message);
    }
}
```

#### Global Exception Handler

```java
// src/main/java/com/example/api/exception/GlobalExceptionHandler.java
package com.example.api.exception;

import com.example.api.constants.MediaTypes;
import com.example.api.dto.ErrorResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.Instant;
import java.util.List;

@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(NotAcceptableException.class)
    public ResponseEntity<ErrorResponse> handleNotAcceptable(NotAcceptableException ex) {
        ErrorResponse error = ErrorResponse.builder()
                .code("NOT_ACCEPTABLE")
                .message(ex.getMessage())
                .timestamp(Instant.now().toString())
                .details(new ErrorResponse.ErrorDetails(
                    "supported", 
                    List.of(
                        MediaTypes.APPLICATION_JSON_VALUE,
                        MediaTypes.ATHLETE_LIST_VALUE,
                        MediaTypes.ATHLETE_LOOKUP_VALUE,
                        MediaTypes.ATHLETE_DETAIL_VALUE,
                        MediaTypes.ATHLETE_SUMMARY_VALUE
                    )
                ))
                .build();
        
        return ResponseEntity.status(HttpStatus.NOT_ACCEPTABLE).body(error);
    }
}
```

---

## 📚 Future Extensions

### 1. Embedded Relationships (expand parameter)
```http
GET /athletes/550e8400...?expand=coach
Accept: application/vnd.api.athlete.detail+json

Response:
{
  "id": "550e8400...",
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+1234567890",
  "telegram": "@johndoe",
  "coach": {  // embedded
    "id": "...",
    "name": "Coach Smith",
    "email": "coach@example.com"
  }
}
```

### 2. Sparse Fieldsets (additional flexibility)
```http
GET /athletes?fields=id,name,email,phone
```
For clients that need custom field sets.

### 3. HATEOAS Links
```json
{
  "id": "550e8400...",
  "name": "John Doe",
  "_links": {
    "self": "/v1/athletes/550e8400...",
    "coach": "/v1/coaches/660e8400..."
  }
}
```

---

## ✅ Final Solutions Checklist

- ✅ Content negotiation via **Accept header**
- ✅ Media Type: `application/json` (base) and `application/vnd.api.{resource}.{representation}+json` (special)
- ✅ Five representations: **base** (application/json), **list**, **lookup**, **detail** (single resource), **summary** (single resource)
- ✅ Eight schemas: **Athlete**, **AthleteListItem**, **AthleteLookup**, **AthleteDetail**, **AthleteSummary**, **AthleteCreate**, **AthleteUpdate**, **AthletePatch**
- ✅ Naming convention: `{Resource}{Operation}` (industry standard)
- ✅ REST semantics: proper usage of GET/POST/PUT/PATCH/DELETE
- ✅ Pagination with metadata for all lists
- ✅ Filtering and sorting
- ✅ Uniform errors with codes
- ✅ JWT authentication on all endpoints
- ✅ Nested resources for related data
- ✅ Location header when creating resource
- ✅ Readonly fields protected from modification
- ✅ API versioning (/v1/)

---

**This is a professional REST API, ready for production deployment! 🚀**

---

## 📋 Extension to Other Resources

This specification demonstrates the approach using the **Athletes** module as an example. For a complete sports management system, similar endpoints should be implemented for:

### Main resources:
- **Coaches** (`/coaches`, `/coaches/{id}`, etc.)
- **Users** (`/users`, `/users/{id}`, etc.)
- **Gyms** (`/gyms`, `/gyms/{id}`, etc.)

### Nested resources:
- `/coaches/{coachId}/athletes` ✅ (already implemented)
- `/gyms/{gymId}/coaches`

### Pattern application:
Each new resource should follow the same principles:
- ✅ Content negotiation via Accept header
- ✅ Five representations: base (application/json), list, lookup, detail (single resource), summary (single resource)
- ✅ Eight schemas: {Resource}, {Resource}ListItem, {Resource}Lookup, {Resource}Detail, {Resource}Summary, {Resource}Create, {Resource}Update, {Resource}Patch
- ✅ Standard HTTP methods and response codes
- ✅ Pagination, filtering, sorting
- ✅ Uniform errors
- ✅ JWT authentication

