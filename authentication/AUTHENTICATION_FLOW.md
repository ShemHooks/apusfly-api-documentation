# Authentication Flows

**Project:** ApusFly  
**API Version:** v1  
**Authentication:** Laravel Sanctum Bearer Tokens  
**Clients:** Next.js Web and React Native + Expo  
**Status:** Development — Google integration and refactored authentication tests pending verification.

---

## A. Customer/Pilot Registration

1. Client sends `POST /api/v1/auth/register` with profile fields, password confirmation, and `account_type` (`customer` or `pilot`).
2. `RegisterRequest` validates the payload.
3. `AuthService` opens a database transaction, creates the user, looks up the matching platform role, and attaches it through `user_roles`.
4. API responds **201 Created** with `UserResource` in the standardized response envelope.
5. The user signs in separately through email/password or a supported social authentication flow.

**Trust boundary:** Ignore client-supplied role IDs, permissions, verification status, and administrative flags. Role assignment must be controlled by the backend.

**Future implementation:** Email verification, identity verification, and KYC-based eligibility restrictions.

## B. Unified Email/Password Authentication

Both Next.js and React Native authenticate through the same endpoint:

`POST /api/v1/auth/login`

1. Client submits email and password over HTTPS.
2. `LoginRequest` validates the submitted credentials.
3. `AuthService` verifies the password hash and confirms the user account is active.
4. Laravel issues a Sanctum personal access token using the `apusfly-client` name and `access` ability.
5. `AuthService` updates the user's `last_login_at`.
6. API returns `UserResource`, access token, and token type.
7. Client securely stores the token according to its platform.
8. Subsequent authenticated API requests include `Authorization: Bearer <token>`.

**Example response:**

```json
{
  "success": true,
  "message": "Login successful.",
  "data": {
    "user": {
      "id": "<uuid>",
      "email": "juan@example.com",
      "roles": ["customer"],
      "permissions": []
    },
    "token": "<sanctum-plain-text-token>",
    "token_type": "Bearer"
  },
  "errors": null
}
```

The user object is abbreviated for documentation purposes.

**Important:** Sanctum token abilities are not substitutes for RBAC. Laravel must independently enforce role permissions and resource ownership policies.

## C. React Native + Expo Mobile Authentication

1. The mobile application submits credentials to `POST /api/v1/auth/login`.
2. Laravel validates the credentials and returns a Sanctum Bearer token.
3. The application stores the token using Expo SecureStore, not AsyncStorage.
4. The application sends the token in the `Authorization` header for protected API requests.
5. The application calls `GET /api/v1/auth/me` to retrieve the authenticated user.
6. Roles and permissions determine which navigation options are visible.
7. Backend middleware and policies independently authorize every protected operation.
8. On logout, the application calls `POST /api/v1/auth/logout` and deletes its locally stored token.

**Security considerations:**

- Use HTTPS for production API communication.
- Do not log or expose access tokens.
- Do not store tokens in application source code.
- Handle expired, revoked, and invalid tokens by clearing the local authenticated state.
- Do not treat hidden mobile UI elements as an authorization boundary.

## D. Next.js Web Authentication

The Next.js web platform also uses Laravel Sanctum Bearer tokens.

The recommended implementation uses a Next.js server-side authentication layer so that the browser does not directly manage Laravel access tokens.

1. User submits credentials through the Next.js login interface.
2. A trusted Next.js server-side handler forwards the credentials to `POST /api/v1/auth/login`.
3. Laravel verifies the credentials and returns a Sanctum Bearer token.
4. The Next.js server securely maintains the token and associates it with the user's browser session, using a Secure, HttpOnly cookie-based approach.
5. When the browser requests protected information, the Next.js server forwards the request to Laravel with the appropriate Bearer token.
6. Laravel authenticates the token and applies RBAC and resource policies.
7. During logout, the Next.js server requests `POST /api/v1/auth/logout`, then clears the browser session cookie.

**Important architectural distinction:** The browser may still use a cookie to maintain its Next.js session. This does not mean Laravel is using its `web` session guard. Laravel receives a Bearer token from the trusted Next.js server.

The old Laravel session login flow using `/sanctum/csrf-cookie`, `Auth::guard('web')->login()`, and session regeneration is no longer the intended authentication flow.

**Configuration considerations:** Secure cookies, CSRF protection for Next.js browser-facing session operations, HTTPS, session expiration, and server-side token protection remain necessary.

## E. Google Social Authentication

**Endpoint:** `POST /api/v1/auth/google`

**Controller:** `SocialAuthController@googleLogin`  
**Service:** `SocialAuthService@authenticateWithGoogle`

1. User selects **Continue with Google** in the Next.js or React Native application.
2. The client initiates Google Sign-In using its configured OAuth client.
3. Google authenticates the user and issues an ID token.
4. The client submits the Google ID token to the Laravel API.
5. Laravel verifies the token's signature, issuer, expiration, and allowed audience.
6. `SocialAuthService` extracts Google's stable account identifier (`sub`) and verified email.
7. The service searches `social_accounts` for the matching Google identity.
8. If an existing social account is found, Laravel retrieves its linked ApusFly user.
9. If no social account exists and the email is not already registered, Laravel creates a new user, assigns the platform `customer` role, and creates the social account record.
10. If the email already belongs to an unlinked ApusFly account, the service rejects automatic linking. A separate authenticated linking process is required.
11. Laravel issues a Sanctum Bearer token.
12. The client stores the token securely and enters the authenticated application.

**Example request:**

```json
{
  "id_token": "<google-id-token>"
}
```

**Example response:**

```json
{
  "success": true,
  "message": "Google authentication successful.",
  "data": {
    "user": {
      "id": "<uuid>",
      "email": "juan@example.com",
      "roles": ["customer"]
    },
    "token": "<sanctum-plain-text-token>",
    "token_type": "Bearer",
    "is_new_user": true
  },
  "errors": null
}
```

**Trust boundary:** Never accept a Google user ID, email, or verification status from unverified client-provided profile data. The backend must derive identity information from the verified Google ID token.

**Implementation note:** Google authentication has an initial backend implementation. Automated verification tests, detailed exception handling, and native Expo integration are still pending.

## F. Logout and Token Revocation

**Endpoint:** `POST /api/v1/auth/logout`

1. Client sends its current Sanctum Bearer token.
2. `auth:sanctum` identifies the authenticated user.
3. `account.active` confirms the account is active.
4. `AuthController` retrieves the current access token.
5. If the token is a Sanctum `PersonalAccessToken`, Laravel deletes it.
6. API responds **200 OK**.
7. The client clears its local token or browser session.

**Current behavior:** Only the current personal access token is revoked. Other device tokens remain valid.

**Future enhancements:** Logout from all devices, token expiration policies, session management, and device-specific token identification.

## G. Account Suspension

1. An authorized administrator suspends a user account. The management endpoint is not yet implemented.
2. `EnsureAccountIsActive` rejects protected API requests with **403 Forbidden**.
3. `AuthService` rejects email/password authentication for inactive accounts.
4. `SocialAuthService` rejects Google authentication for inactive accounts.
5. Future suspension workflows should revoke outstanding Sanctum personal access tokens.

**Security consideration:** Account suspension must be enforced server-side, regardless of whether a client still has a previously issued token.

## H. RBAC Authorization Decision Flow

1. `auth:sanctum` authenticates the request.
2. `account.active` checks the user's account status.
3. Platform administrative routes use `platform.permission` to check the required platform permission.
4. `UserResource` exposes platform roles and permissions for client navigation.
5. Provider and organization operations must additionally verify membership, ownership, and relevant business eligibility.
6. Resource-specific authorization is enforced through Laravel policies and service-level business rules.

**Critical restriction:** Do not use a blanket `Gate::before()` implementation that grants all abilities to Super Admin or interprets every permission name as a global authorization grant. Such behavior can bypass resource-level policies.

Super Admin privileges must not automatically bypass ownership, transaction integrity, verification, or other business-specific security requirements.

## I. Authentication Architecture Summary

| Component              | Responsibility                                                 |
| ---------------------- | -------------------------------------------------------------- |
| `AuthController`       | Registration, unified login, current user, logout              |
| `AuthService`          | Account registration, credential verification, login tracking  |
| `SocialAuthController` | Google authentication HTTP handling                            |
| `SocialAuthService`    | Google verification, social account resolution, token issuance |
| `UserResource`         | Standardized authenticated user representation                 |
| `auth:sanctum`         | Sanctum authentication                                         |
| `account.active`       | Active-account enforcement                                     |
| `platform.permission`  | Platform RBAC enforcement                                      |
| Laravel Policies       | Resource ownership and contextual authorization                |
| Expo SecureStore       | Secure mobile token storage                                    |
| Next.js Server Layer   | Secure web token handling and API forwarding                   |

## J. Implementation Status

| Feature                             | Status                 |
| ----------------------------------- | ---------------------- |
| Customer/pilot registration         | Implemented            |
| Unified email/password login        | Refactored             |
| Sanctum Bearer-token authentication | Implemented            |
| Current user endpoint               | Implemented            |
| Current-token logout                | Implemented            |
| Platform RBAC middleware            | Implemented            |
| Google SocialAuthService            | Initial implementation |
| Google SocialAuthController         | Implemented            |
| Google account-linking flow         | Planned                |
| React Native Google Sign-In         | Pending                |
| Next.js Google Sign-In              | Pending                |
| Account suspension management       | Planned                |
| Automated tests after refactor      | Pending verification   |
| Production authentication hardening | Pending                |

---

**Documentation version:** 1.1  
**Module:** Authentication and Social Login  
**Last updated:** October 17, 2026
