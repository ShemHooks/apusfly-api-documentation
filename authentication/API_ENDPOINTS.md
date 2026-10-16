# Authentication API Endpoint Reference

**Project:** ApusFly  
**API Version:** v1  
**Base URL:** `{API_ORIGIN}/api/v1`  
**Content-Type:** `application/json`  
**Authentication:** Laravel Sanctum Bearer tokens  
**Supported Clients:** Next.js Web and React Native Mobile  
**Status:** Development implementation; Google authentication requires integration testing.

---

## Endpoint Summary

| Method | Path             | Authentication               | Purpose                                                 |
| ------ | ---------------- | ---------------------------- | ------------------------------------------------------- |
| POST   | `/auth/register` | Public                       | Register customer or pilot                              |
| POST   | `/auth/login`    | Public                       | Unified email/password login with Bearer-token issuance |
| POST   | `/auth/google`   | Public                       | Google authentication with Bearer-token issuance        |
| GET    | `/auth/me`       | Bearer token, active account | Retrieve authenticated user and platform permissions    |
| POST   | `/auth/logout`   | Bearer token, active account | Revoke current access token                             |

**Rate limiting:**

- Registration: `throttle:5,1`
- Email/password login: `throttle:5,1`
- Google login: `throttle:10,1`

Rate-limit partitioning, trusted proxy configuration, and abuse protection should be reviewed before production.

## 1. Register

**Endpoint:** `POST /api/v1/auth/register`

Registers a new ApusFly customer or pilot. Registration does not automatically issue an access token.

### Request

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

### Validation

| Field          | Rules                                                              |
| -------------- | ------------------------------------------------------------------ |
| `first_name`   | Required, string, maximum 255 characters                           |
| `middle_name`  | Nullable, string, maximum 255 characters                           |
| `last_name`    | Required, string, maximum 255 characters                           |
| `email`        | Required, valid email, unique, maximum 255 characters              |
| `phone`        | Nullable, string, unique, maximum 30 characters                    |
| `password`     | Required, confirmed, minimum 8 characters with letters and numbers |
| `account_type` | Required, `customer` or `pilot`                                    |

### Response — 201 Created

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

This is an abbreviated response. The actual `UserResource` also includes other profile fields.

**Possible errors:**

- `422` — Invalid input, duplicate email or phone.
- `429` — Rate limit exceeded.
- `500` — Unexpected server error, including missing required role seeds.

## 2. Unified Email/Password Login

**Endpoint:** `POST /api/v1/auth/login`

Authenticates users from either the Next.js web platform or the React Native mobile application.

Both clients receive a Laravel Sanctum personal access token.

**Laravel Controller:** `AuthController@login`

### Request

```json
{
  "email": "juan@example.com",
  "password": "Password123"
}
```

### Response — 200 OK

```json
{
  "success": true,
  "message": "Login successful.",
  "data": {
    "user": {
      "id": "<uuid>",
      "first_name": "Juan",
      "last_name": "Dela Cruz",
      "email": "juan@example.com",
      "status": "active",
      "roles": ["customer"],
      "permissions": []
    },
    "token": "<sanctum-plain-text-token>",
    "token_type": "Bearer"
  },
  "errors": null
}
```

### Token configuration

The current implementation issues tokens using:

```php
$user->createToken(
    'apusfly-client',
    ['access']
)->plainTextToken;
```

The token is returned in plaintext only when created.

Subsequent API requests must include:

```http
Authorization: Bearer <sanctum-plain-text-token>
Accept: application/json
```

**Possible errors:**

- `422` — Invalid credentials or authentication validation failure under the current `AuthService` behavior.
- `429` — Rate limit exceeded.
- `500` — Unexpected server error.

## 3. Google Social Login

**Endpoint:** `POST /api/v1/auth/google`

Authenticates a user using a Google ID token.

**Laravel Controller:** `SocialAuthController@googleLogin`  
**Service:** `SocialAuthService@authenticateWithGoogle`

The endpoint is shared by Next.js and React Native, provided that the Google ID token is issued for an OAuth audience accepted by the backend.

### Request

```json
{
  "id_token": "<google-id-token>"
}
```

### Authentication process

1. The client completes Google Sign-In.
2. Google issues an ID token.
3. The client sends the ID token to the Laravel API.
4. Laravel verifies the token using Google's SDK.
5. Laravel identifies the Google account using its stable `sub` claim.
6. The service checks the `social_accounts` table.
7. If the Google identity is already linked, the corresponding ApusFly account is retrieved.
8. If it is not linked, a new customer account may be created when the verified email does not already belong to an ApusFly user.
9. The service issues a Sanctum Bearer token.
10. The client stores the ApusFly token securely.

### Response — 200 OK

```json
{
  "success": true,
  "message": "Google authentication successful.",
  "data": {
    "user": {
      "id": "<uuid>",
      "first_name": "Juan",
      "last_name": "Dela Cruz",
      "email": "juan@example.com",
      "status": "active",
      "roles": ["customer"],
      "permissions": []
    },
    "token": "<sanctum-plain-text-token>",
    "token_type": "Bearer",
    "is_new_user": true
  },
  "errors": null
}
```

### Social account storage

Google identities are stored in `social_accounts` using:

- `user_id`
- `provider`
- `provider_user_id`
- `provider_email`
- `provider_email_verified`
- `linked_at`
- `last_used_at`

The database enforces uniqueness for `(provider, provider_user_id)` and `(user_id, provider)`.

### Important security behavior

- Google ID tokens must be verified server-side.
- The Google `sub` claim is the primary social identity identifier.
- A verified email is required by the current service.
- Existing ApusFly accounts are not automatically linked based solely on matching email.
- Suspended or deactivated accounts cannot authenticate.
- New social registrations receive the customer role by default.
- Google authentication does not replace Laravel Sanctum authorization.

### Current error handling

The current controller catches `RuntimeException` and returns `401`.

This is provisional and must be refined before production to distinguish:

| HTTP Code | Intended Meaning                                  |
| --------- | ------------------------------------------------- |
| 401       | Invalid or expired Google ID token                |
| 403       | Inactive or suspended account                     |
| 409       | Existing account requires authenticated linking   |
| 422       | Missing or invalid request fields                 |
| 429       | Rate limit exceeded                               |
| 500       | Server configuration or unexpected internal error |

The service also needs automated tests for token verification, duplicate identities, account creation, and concurrent login attempts.

**Implementation status:** Backend endpoint defined; native Expo integration and live Google authentication remain unverified.

## 4. Current Authenticated User

**Endpoint:** `GET /api/v1/auth/me`

**Middleware:**

- `auth:sanctum`
- `account.active`

### Request

```http
GET /api/v1/auth/me
Authorization: Bearer <sanctum-plain-text-token>
Accept: application/json
```

### Response — 200 OK

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

The permission values are illustrative and depend on the actual role-permission seeds.

The `full_name` value depends on the corresponding accessor in the `User` model.

**Possible errors:**

- `401` — Missing, expired, revoked, or invalid token.
- `403` — Inactive account.

## 5. Logout

**Endpoint:** `POST /api/v1/auth/logout`

Revokes the currently authenticated Sanctum personal access token.

**Laravel Controller:** `AuthController@logout`

### Request

```http
POST /api/v1/auth/logout
Authorization: Bearer <sanctum-plain-text-token>
Accept: application/json
```

### Response — 200 OK

```json
{
  "success": true,
  "message": "Logged out successfully.",
  "data": null,
  "errors": null
}
```

### Behavior

The controller checks whether the current access token is a `PersonalAccessToken`. If so, it deletes that token.

Other active tokens belonging to the same user remain valid.

**Important:** The current implementation does not provide a logout-all-devices endpoint.

The active-account middleware also means suspended users may be unable to invoke logout. Administrative account suspension should include an appropriate token-revocation policy.

## 6. Standard API Response Format

All API endpoints follow the ApusFly response envelope.

### Successful response

```json
{
  "success": true,
  "message": "Operation completed successfully.",
  "data": {},
  "errors": null
}
```

### Error response

```json
{
  "success": false,
  "message": "Validation failed.",
  "data": null,
  "errors": {
    "email": ["Invalid email or password."]
  }
}
```

### HTTP Status Codes

| Code | Meaning                                        |
| ---- | ---------------------------------------------- |
| 200  | Successful request                             |
| 201  | Resource created                               |
| 401  | Authentication required or credentials invalid |
| 403  | Account inactive or access forbidden           |
| 404  | Resource or endpoint not found                 |
| 405  | HTTP method not allowed                        |
| 409  | Account-linking conflict (planned)             |
| 422  | Request validation failed                      |
| 429  | Rate limit exceeded                            |
| 500  | Unexpected server error                        |

Error envelopes depend on the configured Laravel exception handlers. Debug responses may differ until production error normalization is fully tested.

## 7. Client Authentication Integration

### Next.js Web Platform

The Next.js application communicates with Laravel through the REST API.

Recommended flow:

1. Submit credentials to `/api/v1/auth/login` through a trusted Next.js server-side handler.
2. Receive the Sanctum token on the server.
3. Store the session reference or token using a Secure, HttpOnly cookie.
4. Have the Next.js server attach the Bearer token when calling Laravel.
5. Revoke the token during logout and clear the browser cookie.

Avoid storing long-lived access tokens in `localStorage` or exposing them unnecessarily to browser JavaScript.

### React Native Mobile

Recommended flow:

1. Submit credentials to `/api/v1/auth/login`, or a Google ID token to `/api/v1/auth/google`.
2. Receive the Sanctum Bearer token.
3. Store it using `expo-secure-store`.
4. Attach the token to authenticated API requests.
5. Delete the locally stored token after a successful logout request.

Never store access tokens in AsyncStorage or application source code.

## 8. Authentication Security Notes

- All authentication traffic must use HTTPS in production.
- Sanctum tokens are opaque, database-backed credentials, not JWTs.
- Token names and abilities do not replace RBAC or resource authorization policies.
- Token expiration and revocation policies must be defined before production.
- Social account linking must require authenticated proof of ownership.
- Google OAuth audience verification must support only explicitly trusted client IDs.
- Rate limits should account for reverse proxies and shared IP addresses.
- Secrets and tokens must not appear in application logs.
- If Bearer-only authentication is required, review Sanctum's stateful middleware and enforce the intended authentication mode.
- Changes to the authentication endpoints require updates to automated tests and frontend API clients.

## 9. Implementation Status

| Component                                     | Status                                            |
| --------------------------------------------- | ------------------------------------------------- |
| Customer/pilot registration                   | Implemented                                       |
| Unified email/password login                  | Refactored                                        |
| Sanctum Bearer-token issuance                 | Implemented in controller                         |
| Authenticated user endpoint                   | Implemented                                       |
| Current-token logout                          | Implemented                                       |
| Google authentication controller              | Implemented                                       |
| Dedicated SocialAuthService                   | Initial implementation                            |
| Google ID-token verification                  | Implemented in service; live verification pending |
| Google account creation/linking               | Initial implementation                            |
| Expo Google Sign-In integration               | Pending                                           |
| Apple Sign-In                                 | Planned                                           |
| Automated tests after authentication refactor | Pending verification                              |
| Production security hardening                 | Pending                                           |

---

**Documentation version:** 1.1  
**Project:** ApusFly API  
**Module:** Authentication and Social Login  
**Architecture:** Laravel REST API + Sanctum + RBAC  
**Last updated:** October 17, 2026
