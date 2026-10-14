# Authentication Flows

## A. Customer/pilot registration

1. Client sends `POST /api/v1/auth/register` with validated profile fields, password confirmation, and `account_type` (`customer` or `pilot`).
2. `RegisterRequest` validates the payload.
3. `AuthService` opens a database transaction, creates the user, looks up the matching platform role, and attaches it to `user_roles`.
4. API responds **201** with `UserResource` in the normalized response envelope.
5. User signs in separately. Email verification and KYC gates are planned.

**Trust boundary:** ignore client-supplied role IDs, permissions, verification status, or admin flags.

## B. React Native + Expo mobile login

1. Send `POST /api/v1/auth/mobile/login` with email/password over HTTPS.
2. `AuthService` verifies the password hash and active status.
3. API issues a Sanctum personal access token and updates `last_login_at`.
4. App stores token in Expo SecureStore (not AsyncStorage).
5. App sends `Authorization: Bearer <token>` to `/api/v1/auth/me`.
6. App uses returned roles/permissions to choose visible navigation, but backend must independently authorize all operations.
7. On logout, call `POST /api/v1/auth/logout`, then remove token from SecureStore.

**Important:** Token issuance currently specifies ability `mobile`. Avoid relying on Sanctum token abilities as the sole permission source; enforce platform RBAC and resource policies server-side.

## C. Next.js browser login

1. Browser calls `/sanctum/csrf-cookie` with credentials enabled.
2. Browser sends `POST /api/v1/auth/login` with email/password, session cookies, and CSRF header/cookie handling.
3. Laravel logs into the `web` guard and regenerates session ID.
4. Browser calls `/api/v1/auth/me` using session cookies.
5. Browser calls `POST /api/v1/auth/logout`; Laravel invalidates session and rotates CSRF token.

**Configuration required:** `SANCTUM_STATEFUL_DOMAINS`, CORS `supports_credentials`, trusted origins, session cookie domain, HTTPS and secure cookie flags. Browser integration and SSR proxy behavior need dedicated tests.

## D. Account suspension

1. Administrator marks user as suspended (management endpoint not yet implemented).
2. `EnsureAccountIsActive` rejects protected requests with **403**.
3. Mobile login rejects the suspended account.
4. Future suspension workflow should also revoke outstanding personal access tokens and invalidate web sessions as appropriate.

## E. RBAC decision flow

1. `auth:sanctum` identifies user.
2. `account.active` checks account status.
3. Platform administrative route checks a platform-scoped permission.
4. Booking, provider, and organization resources must additionally enforce membership, ownership, and business eligibility via policies/services.

**Do not** use a blanket `Gate::before()` return-true for all Super Admin or permission names: it can bypass resource-level policies.
