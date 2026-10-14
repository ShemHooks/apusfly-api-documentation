# Authentication Overview

**Version:** 1.0 — development baseline  
**API base path:** `/api/v1`  
**Framework:** Laravel  
**Authentication:** Laravel Sanctum  
**Clients:** Next.js web platform and React Native + Expo mobile app

## Design

The API uses a thin-controller architecture:

`Form Request → AuthController → AuthService → User / Role models → UserResource → BaseController response`

- **Form Requests:** validate registration and login payloads.
- **AuthController:** exposes registration, web login, mobile login, current-user, and logout endpoints.
- **AuthService:** creates users and attaches an allowed platform role in a database transaction; checks credentials and active status; updates `last_login_at`.
- **UserResource:** serializes user identity, platform roles, and permission names; never return the password hash or stored token hash.
- **BaseController:** standardizes success and expected error JSON envelopes.
- **Exception rendering:** normalizes API validation (422), authentication (401), authorization (403), and common HTTP errors.
- **EnsureAccountIsActive:** blocks protected requests for suspended/deactivated accounts.

## Client authentication modes

| Client | Method | Credential handling |
|---|---|---|
| Next.js web | Laravel web session through Sanctum's stateful SPA support | HttpOnly session cookie, CSRF protection; use credentials-enabled requests |
| React Native + Expo | Sanctum personal access token | `Authorization: Bearer <token>`; store securely using Expo SecureStore |

Do not use browser localStorage for mobile-style bearer tokens. The web login path must be limited to trusted stateful SPA requests and requires correct session, CORS, cookie, and CSRF configuration.

## Account creation

Public registration accepts `account_type` of `customer` or `pilot` only. The server resolves the matching **platform-scoped** role and attaches it to the user. Public requests must never accept arbitrary role IDs or permission lists.

**Pilot role does not equal operational eligibility.** Pilot KYC, licensing/credentials, provider affiliations, and job acceptance permissions require separate business checks.

Registration returns HTTP 201 and user data; it does **not** automatically log the user in or issue a token in the current design.

## RBAC integration

- The `user_roles` pivot represents global platform role assignments.
- Provider-scoped roles belong to provider memberships; organization-scoped roles belong to organization memberships when implemented.
- Global platform permission checks and resource-specific ownership checks must remain separate.
- The earlier broad `Gate::before()` callback must be removed/restricted so it cannot bypass booking ownership or tenant isolation.

## Status

**Tested locally (reported):** registration with customer role assignment; mobile login/token issuance; invalid credentials; unauthenticated `/me`; suspended login rejection; mobile logout/token deletion; suspended bearer-token rejection.

**Not yet confirmed production-ready:** browser cookie/CSRF integration, verified email flows, password reset, KYC, device/session management, token lifecycle policy, comprehensive permission/ownership policies, production observability.

See [API endpoints](API_ENDPOINTS.md), [flows](AUTHENTICATION_FLOW.md), and [security](SECURITY.md).
