# Authentication API Endpoint Reference

**Base URL:** `{API_ORIGIN}/api/v1`  
**Content-Type:** `application/json`  
**Status:** Development implementation; validate against current routes and controllers.

## Endpoint summary

| Method | Path | Authentication | Purpose |
|---|---|---|---|
| POST | `/auth/register` | Public | Register customer or pilot |
| POST | `/auth/login` | Stateful web session | Web login for Next.js |
| POST | `/auth/mobile/login` | Public | Mobile login and token issuance |
| GET | `/auth/me` | `auth:sanctum`, active account | Current user profile and global roles |
| POST | `/auth/logout` | `auth:sanctum`, active account | End session or revoke current bearer token |

Registration and login routes were configured with `throttle:5,1` (five requests per minute per Laravel's throttle key). Review rate-limit partitioning and proxy/IP handling for production.

## 1. Register

`POST /api/v1/auth/register`

Example request:

```json
{
  "first_name": "Juan",
  "middle_name": null,
  "last_name": "Dela Cruz",
  "email": "juan@example.com",
  "phone": null,
  "password": "Password123",
  "password_confirmation": "Password123",
  "account_type": "customer"
}
```

Validation rules in the discussed `RegisterRequest`:

- `first_name`, `last_name`: required strings, max 255.
- `middle_name`: optional string, max 255.
- `email`: required, valid email, unique in users, max 255.
- `phone`: optional string, max 30, unique in users.
- `password`: required, confirmed, at least 8 characters with letters and numbers.
- `account_type`: `customer` or `pilot`.

**201 Created**, example abbreviated response:

```json
{
  "success": true,
  "message": "Account registered successfully.",
  "data": {
    "id": "<uuid>",
    "first_name": "Juan",
    "middle_name": null,
    "last_name": "Dela Cruz",
    "email": "juan@example.com",
    "status": "active",
    "roles": ["customer"],
    "permissions": []
  },
  "errors": null
}
```

The `permissions` array depends on seeded role-permission relationships. Other fields defined in `UserResource` may also appear. Duplicate/invalid inputs return **422**. A missing role seed may cause a server error until the database is initialized.

## 2. Web login

`POST /api/v1/auth/login`

```json
{"email":"juan@example.com","password":"Password123"}
```

Uses Laravel `web` guard login and session regeneration. Returns **200** with the normalized user resource, **not** a bearer token. This requires a stateful session-capable request. Before logging in from a browser, request `/sanctum/csrf-cookie` with credentials enabled and submit the CSRF token as required by Sanctum.

**200 OK** example:

```json
{"success":true,"message":"Login successful.","data":{"id":"<uuid>","email":"juan@example.com","roles":["customer"]},"errors":null}
```

Example is abbreviated; `UserResource` defines the complete user payload.

## 3. Mobile login

`POST /api/v1/auth/mobile/login`

```json
{"email":"juan@example.com","password":"Password123"}
```

**200 OK** example:

```json
{
  "success": true,
  "message": "Login successful.",
  "data": {
    "user": {"id":"<uuid>","email":"juan@example.com","roles":["customer"]},
    "token": "<sanctum-plain-text-token>",
    "token_type": "Bearer"
  },
  "errors": null
}
```

The plaintext token is returned **once**. Never log it, email it, or place it in URLs. Subsequent requests:

```http
Authorization: Bearer <sanctum-plain-text-token>
Accept: application/json
```

Incorrect credentials and inactive accounts currently return **422** validation-style errors. Consider changing inactive-account error semantics and avoiding account-status disclosure before production.

## 4. Current user

`GET /api/v1/auth/me`

Requires a valid Sanctum session/token and active account.

**200 OK**:

```json
{
  "success": true,
  "message": "User retrieved successfully.",
  "data": {
    "id": "<uuid>",
    "first_name": "Juan",
    "middle_name": null,
    "last_name": "Dela Cruz",
    "full_name": "Juan Dela Cruz",
    "email": "juan@example.com",
    "phone": null,
    "status": "active",
    "email_verified_at": null,
    "profile_photo_path": null,
    "roles": ["customer"],
    "permissions": ["profile.manage_own", "bookings.create"]
  },
  "errors": null
}
```

The `full_name` field assumes the User model has a corresponding accessor; verify this before shipping. Permission values are examples, not a guaranteed seeded set.

Missing authentication returns **401**; suspended/deactivated account returns **403**.

## 5. Logout

`POST /api/v1/auth/logout`

- Mobile bearer token: deletes the **current** Sanctum personal access token.
- Web session: logs out the web guard, invalidates the session, and regenerates the CSRF token.

**200 OK**:

```json
{"success":true,"message":"Logged out successfully.","data":null,"errors":null}
```

The current route applies active-account middleware. A suspended user may not be able to call logout; the backend should revoke tokens administratively when suspending an account.

## Standard errors

```json
{
  "success": false,
  "message": "Validation failed.",
  "data": null,
  "errors": {"email":["Invalid email or password."]}
}
```

| HTTP code | Meaning |
|---|---|
| 401 | Not authenticated / token invalid |
| 403 | Authenticated but inactive or unauthorized |
| 404 | API route or resource not found |
| 405 | Method not allowed |
| 422 | Validation failed / invalid credentials |
| 429 | Rate limit exceeded |
| 500 | Unexpected server error; avoid sensitive details |

For unexpected errors, debug mode may return Laravel's default error shape. Consumers should gracefully handle non-envelope errors until production error handling is finalized.
