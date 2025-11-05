# TODO

## Add register operation

### Operation semantics
- `register` - authentication operation that creates a new user AND immediately authorizes them
- Returns `AuthResponse` (accessToken, tokenType, etc.) - this is the key point!
- User is logged in immediately after registration - this is the auth flow

### REST API Convention
- `POST /auth/register` ← Auth context (creation + auto-login)

### Request body structure
```typescript
// User data sent during registration:
{
  name: string,
  email: string,
  role?: string,  // optional, defaults to user role
  // ... other fields
}
```

### Return types
```typescript
// Auth operations return tokens:
authService.register() → AuthResponse { accessToken, tokenType, ... }
```

### Use cases
| Scenario                              | Endpoint            | Service      | Return                  |
|---------------------------------------|---------------------|--------------|-------------------------|
| Standalone registration (sign up)     | POST /auth/register | authService  | AuthResponse + auto-login |

### Concept from ports-management-api.yaml
```yaml
/auth/register:
  post:
    tags: [Authentication]  ← Auth context!
    responses:
      201:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/AuthResponse'  ← Returns tokens!
```
