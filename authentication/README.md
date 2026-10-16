# Authentication Overview

**Version:** 1.1 — Unified Authentication and Social Login  
**API base path:** `/api/v1`  
**Framework:** Laravel  
**Authentication:** Laravel Sanctum Personal Access Tokens  
**Authorization:** Role-Based Access Control (RBAC) and Laravel Policies  
**Clients:** Next.js Web Platform and React Native + Expo Mobile Application  
**Status:** Development implementation

## 1. Architecture and Design

ApusFly uses a centralized Laravel REST API for authentication and authorization across web and mobile applications.

Both clients authenticate using Laravel Sanctum Bearer tokens.

The authentication module follows a thin-controller, service-oriented architecture.

**Email/password authentication:**

`Form Request → AuthController → AuthService → User / Role Models → UserResource → BaseController Response`

**Google social authentication:**

`Request Validation → SocialAuthController → SocialAuthService → Google Token Verification → SocialAccount / User Models → UserResource → BaseController Response`

### Component Responsibilities

| Component                  | Responsibility                                                                                   |
| -------------------------- | ------------------------------------------------------------------------------------------------ |
| Form Requests              | Validate registration and email/password login payloads                                          |
| `AuthController`           | Registration, unified login, current-user retrieval, and logout                                  |
| `AuthService`              | Account creation, credential verification, role assignment, and login tracking                   |
| `SocialAuthController`     | Handle Google authentication requests and return standardized API responses                      |
| `SocialAuthService`        | Verify Google identity, resolve linked accounts, create eligible users, and issue Sanctum tokens |
| `UserResource`             | Serialize user identity, platform roles, and permissions                                         |
| `BaseController`           | Standardize success and expected error responses                                                 |
| Exception Rendering        | Normalize validation, authentication, authorization, and HTTP errors                             |
| `EnsureAccountIsActive`    | Block protected requests from suspended or deactivated accounts                                  |
| `EnsurePlatformPermission` | Enforce platform-scoped RBAC permissions                                                         |
| Laravel Policies           | Enforce resource ownership, membership, and business authorization rules                         |

Password hashes, stored Sanctum token hashes, and other sensitive credentials must never appear in API responses.

## 2. Client Authentication Modes

Both supported clients use the same Laravel authentication endpoints.

| Client              | Authentication Method | Credential Handling                                                        |
| ------------------- | --------------------- | -------------------------------------------------------------------------- |
| Next.js Web         | Sanctum Bearer token  | Prefer server-side token handling with Secure, HttpOnly browser cookies    |
| React Native + Expo | Sanctum Bearer token  | Store token using Expo SecureStore and send it in the Authorization header |

**Shared login endpoint:**

`POST /api/v1/auth/login`

**Shared Google authentication endpoint:**

`POST /api/v1/auth/google`

**Authenticated requests:**

```http
Authorization: Bearer <sanctum-access-token>
Accept: application/json
```

The Laravel API no longer intentionally uses separate web-session and mobile-token login endpoints.

The former `webLogin()` and `mobileLogin()` controller methods have been replaced by a unified `login()` method.

### Web Security

The recommended Next.js implementation handles Laravel Bearer tokens through a trusted server-side layer.

The browser maintains its Next.js session using a Secure, HttpOnly cookie, while the Next.js server attaches the Bearer token when communicating with Laravel.

This approach avoids unnecessarily exposing Laravel access tokens to browser JavaScript.

CSRF protection remains necessary for browser-facing operations that rely on cookies.

### Mobile Security

React Native stores Sanctum tokens in Expo SecureStore.

Tokens must not be stored in AsyncStorage, hardcoded in source code, or included in application logs.

## 3. Account Creation

Public registration accepts only the following account types:

- `customer`
- `pilot`

The server resolves the matching platform-scoped role and attaches it through the `user_roles` pivot table.

Public registration requests must never accept arbitrary role IDs, permission lists, verification status, or administrative flags.

Registration returns HTTP **201 Created** with the new user's information.

It does not automatically authenticate the user or issue an access token.

**Important:** A pilot role does not automatically grant operational eligibility. Pilot KYC, licensing, provider affiliations, and job acceptance permissions require additional business verification.

## 4. Google Social Authentication

ApusFly supports an initial Google authentication implementation through:

`POST /api/v1/auth/google`

The client obtains a Google ID token and submits it to Laravel.

`SocialAuthService` verifies the token and identifies the Google account using the stable Google `sub` claim.

### Account Resolution

If a matching Google identity exists in `social_accounts`, Laravel retrieves the associated user.

If the Google identity is new and its verified email is not registered, Laravel creates a customer account and links the Google identity.

If the email belongs to an existing unlinked ApusFly account, automatic linking is rejected. A separate authenticated account-linking process is required.

Inactive accounts are not permitted to authenticate.

### Social Account Model

The `social_accounts` table supports Google and Apple providers.

It records:

- Provider and provider-specific user identifier
- Associated ApusFly user
- Provider email and verification status
- Account linking timestamp
- Last-used timestamp

Unique constraints prevent duplicate provider identities and multiple links to the same provider for one user.

**Current scope:** Google authentication has an initial backend implementation. Apple authentication and authenticated account linking are planned.

## 5. Sanctum Token Lifecycle

Successful email/password and Google authentication issue a Sanctum personal access token.

The current implementation uses:

```php
$user->createToken(
    'apusfly-client',
    ['access']
)->plainTextToken;
```

Tokens are opaque credentials backed by Laravel's personal access token storage.

The plaintext token is returned only when created.

### Current Operations

| Operation                 | Behavior                                  |
| ------------------------- | ----------------------------------------- |
| Login                     | Creates a new access token                |
| Google login              | Creates a new access token                |
| Authenticated request     | Validates token and active-account status |
| Logout                    | Revokes the current personal access token |
| Account suspension        | Protected requests are rejected           |
| Logout all devices        | Not yet implemented                       |
| Token expiration policy   | Requires production configuration         |
| Device/session management | Planned                                   |

Sanctum token abilities must not be treated as the primary source of business permissions.

RBAC and resource policies remain authoritative.

## 6. RBAC Integration

The `user_roles` pivot represents global platform role assignments.

Provider-scoped roles will be assigned through provider memberships.

Organization-scoped roles will be assigned through organization memberships.

Platform permissions and resource-specific authorization must remain separate.

### Authorization Sequence

1. `auth:sanctum` authenticates the request.
2. `account.active` verifies account status.
3. `platform.permission` checks applicable platform permissions.
4. Laravel Policies and business services enforce resource ownership, membership, and eligibility requirements.

The previous broad `Gate::before()` authorization approach must remain removed or appropriately restricted so that global permissions cannot bypass booking ownership, provider boundaries, or tenant isolation.

Super Admin privileges must not automatically override resource-specific security rules.

## 7. API Endpoints

| Method | Endpoint         | Purpose                               |
| ------ | ---------------- | ------------------------------------- |
| POST   | `/auth/register` | Register customer or pilot            |
| POST   | `/auth/login`    | Unified email/password authentication |
| POST   | `/auth/google`   | Google social authentication          |
| GET    | `/auth/me`       | Retrieve authenticated user           |
| POST   | `/auth/logout`   | Revoke current access token           |

All paths are relative to `/api/v1`.

The `/auth/me` and `/auth/logout` endpoints require `auth:sanctum` and `account.active` middleware.

## 8. Testing and Implementation Status

### Previously Reported Local Tests

The original authentication implementation was tested for:

- Customer registration and role assignment
- Mobile Bearer-token issuance
- Invalid credentials
- Unauthenticated `/auth/me` requests
- Suspended account login rejection
- Bearer-token logout and revocation
- Suspended account access rejection

These results apply to the earlier authentication implementation and do not establish that the refactored shared endpoints have passed testing.

### Current Development Status

| Component                               | Status                 |
| --------------------------------------- | ---------------------- |
| Registration                            | Implemented            |
| Unified email/password login            | Refactored             |
| Sanctum Bearer-token issuance           | Implemented            |
| Current-user endpoint                   | Implemented            |
| Current-token logout                    | Implemented            |
| Platform RBAC                           | Implemented            |
| Dedicated SocialAuthService             | Initial implementation |
| Google Social Login endpoint            | Implemented            |
| Google authentication integration tests | Pending                |
| React Native Google Sign-In             | Pending                |
| Next.js authentication integration      | Pending                |
| Email verification                      | Planned                |
| Password reset                          | Planned                |
| Pilot KYC                               | Planned                |
| Device/session management               | Planned                |
| Production token lifecycle policy       | Pending                |
| Comprehensive ownership policies        | Pending                |
| Production observability                | Pending                |

### Production Readiness Requirements

Before production deployment, verify the unified authentication tests, Google OAuth audience validation, token revocation behavior, account-linking security, rate limiting, secure client-side session handling, and authorization boundaries.

Also confirm that Laravel's Sanctum configuration enforces the intended Bearer-token authentication behavior.

## 9. Related Documentation

- [API Endpoint Reference](API_ENDPOINTS.md)
- [Authentication Flows](AUTHENTICATION_FLOW.md)
- [Authentication Security](SECURITY.md)

---

**Documentation version:** 1.1  
**Project:** ApusFly  
**Module:** Authentication and Social Login  
**Last updated:** October 17, 2026
