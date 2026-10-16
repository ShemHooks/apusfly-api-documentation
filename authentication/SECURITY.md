# Authentication Security & Production Checklist

**Project:** ApusFly  
**Module:** Authentication, Social Login & Authorization  
**Version:** 1.1  
**Authentication:** Laravel Sanctum Bearer Tokens  
**Clients:** Next.js Web and React Native + Expo  
**Document status:** Development guidance; not a security certification.

---

## 1. Previously Reported Local Tests

The following behaviors were reported as passing before the unified authentication refactor:

- [x] Customer registration and platform role attachment
- [x] Mobile login returns a Sanctum personal access token
- [x] Invalid password is rejected
- [x] Unauthenticated `/auth/me` is rejected
- [x] Suspended user cannot log in using email/password
- [x] Logout deletes the current mobile token
- [x] Suspended token cannot access `/auth/me`

These results do not confirm that the refactored shared endpoints or Google authentication have passed testing.

## 2. Authentication Architecture

Both supported clients use the Laravel REST API and Sanctum personal access tokens.

| Client              | Authentication | Token handling                                                            |
| ------------------- | -------------- | ------------------------------------------------------------------------- |
| Next.js             | Bearer token   | Prefer server-side handling with Secure, HttpOnly browser session cookies |
| React Native + Expo | Bearer token   | Expo SecureStore                                                          |
| Laravel API         | Sanctum        | Validates tokens and enforces authorization                               |

The shared authentication endpoints are:

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/google`
- `GET /api/v1/auth/me`
- `POST /api/v1/auth/logout`

The previous separate web-session and mobile login endpoints are no longer part of the intended API design.

## 3. Required Before Production

### A. Authentication and Token Security

- [ ] Re-run and update automated tests for unified `/auth/login` and `/auth/logout`.
- [ ] Verify both Next.js and React Native authenticate successfully using Sanctum Bearer tokens.
- [ ] Enforce Bearer-only authentication where required; review Laravel Sanctum's `statefulApi()` configuration.
- [ ] Define Sanctum token expiration, revocation, and device/session management policies.
- [ ] Consider per-device token names, token rotation, and a logout-all-devices endpoint.
- [ ] Revoke outstanding tokens after account suspension and security-sensitive credential changes where appropriate.
- [ ] Confirm token reuse fails after logout.
- [ ] Ensure passwords are hashed using Laravel's `hashed` cast or `Hash::make()`.
- [ ] Never store plaintext passwords, access tokens, or Google ID tokens in application logs.
- [ ] Ensure authentication credentials are transmitted only over HTTPS in production.

### B. Google Social Authentication

- [ ] Verify Google ID-token signatures, issuer, expiration, and allowed OAuth audience.
- [ ] Confirm the backend accepts only explicitly configured Google OAuth client IDs.
- [ ] Validate the Google `sub` claim and verified-email status.
- [ ] Ensure an existing Google identity resolves only to its linked ApusFly account.
- [ ] Prevent automatic linking of existing accounts based solely on matching email.
- [ ] Implement authenticated account linking with proof of ownership.
- [ ] Reject Google authentication for suspended or deactivated users.
- [ ] Test new Google account creation and assignment of the customer role.
- [ ] Test duplicate provider identities and concurrent first-time sign-ins.
- [ ] Replace generic `RuntimeException` handling with appropriate authentication and domain exceptions.
- [ ] Return appropriate status codes for invalid credentials, inactive accounts, and account-linking conflicts.
- [ ] Avoid exposing internal configuration errors or sensitive identity details.
- [ ] Validate Android package name and SHA-1 certificate configuration for EAS development and production builds.
- [ ] Test Google Sign-In on an actual Android development build.
- [ ] Validate the separate Next.js Google authentication flow before enabling it.

**Important:** The current Google login implementation is an initial backend implementation, not a production-verified integration.

### C. Next.js Web Security

- [ ] Implement a trusted server-side authentication layer for Laravel API communication.
- [ ] Store browser session credentials in Secure, HttpOnly cookies.
- [ ] Configure an appropriate SameSite policy.
- [ ] Protect cookie-authenticated Next.js operations against CSRF.
- [ ] Ensure Laravel Bearer tokens are not exposed unnecessarily to browser JavaScript.
- [ ] Avoid storing long-lived access tokens in `localStorage`.
- [ ] Verify server-side token forwarding to Laravel.
- [ ] Test logout, browser session expiration, and revoked-token behavior.
- [ ] Validate cookie domain, HTTPS, and reverse-proxy configuration across actual production subdomains.
- [ ] Configure CORS only for browser origins that directly access the Laravel API.

The browser may use a Next.js session cookie without Laravel using its own stateful web guard.

Do not reintroduce Laravel session authentication unless explicitly required by a future architectural decision.

### D. React Native + Expo Security

- [ ] Store Sanctum tokens using Expo SecureStore.
- [ ] Remove stored tokens after logout.
- [ ] Clear local authenticated state when a token is invalid or revoked.
- [ ] Avoid token exposure in debugging tools, analytics, and crash reports.
- [ ] Confirm Google Sign-In works in an Expo development build.
- [ ] Verify the Android OAuth configuration matches the actual signing certificate.
- [ ] Use HTTPS for deployed API communication.
- [ ] Prevent secrets from being embedded in `EXPO_PUBLIC_` environment variables.

**Note:** Google OAuth Client IDs are public identifiers. OAuth client secrets, private keys, and authentication tokens are not.

### E. Authorization and RBAC

- [ ] Confirm the broad `Gate::before()` authorization bypass has been removed.
- [ ] Keep platform permission checks separate from resource ownership policies.
- [ ] Enforce booking ownership through server-side authorization.
- [ ] Verify provider membership before provider-scoped operations.
- [ ] Verify organization membership before organization-scoped operations.
- [ ] Enforce pilot eligibility, licensing, and KYC requirements independently of platform role assignment.
- [ ] Prevent privilege escalation through registration and profile-update requests.
- [ ] Test cross-user and cross-provider resource access.
- [ ] Ensure Super Admin permissions cannot silently bypass tenant isolation or transaction integrity.

Client-side roles and permissions may control navigation visibility, but they must never be the sole authorization mechanism.

### F. Account Security

- [ ] Implement email verification and appropriate access restrictions.
- [ ] Implement password reset using expiring, single-use tokens.
- [ ] Rate-limit login, registration, password reset, and social authentication.
- [ ] Review rate-limit keys for account identifiers, IP addresses, and trusted proxies.
- [ ] Avoid unnecessary account enumeration through public authentication errors.
- [ ] Implement administrative account suspension and token revocation.
- [ ] Review account reactivation and recovery procedures.
- [ ] Define appropriate handling for users with nullable passwords who registered through social authentication.

### G. KYC and Identity Verification

- [ ] Integrate Didit KYC using server-side API credentials.
- [ ] Verify webhook authenticity according to the provider's documented signing mechanism.
- [ ] Implement replay protection and idempotent webhook processing.
- [ ] Store sensitive identity documents securely with restricted access.
- [ ] Apply appropriate retention and deletion policies.
- [ ] Keep KYC status separate from platform RBAC permissions.
- [ ] Independently verify pilot licensing and operational credentials.

Identity verification alone does not establish eligibility to operate a drone commercially.

### H. API and Infrastructure Security

- [ ] Verify consistent error envelopes for 401, 403, 404, 405, 422, 429, and 500 responses.
- [ ] Disable production debug output using `APP_DEBUG=false`.
- [ ] Avoid exposing stack traces, SQL queries, tokens, and internal configuration details.
- [ ] Validate UUID-based Sanctum token relationships and deployment migrations.
- [ ] Configure trusted proxies and correct client IP handling.
- [ ] Apply HTTPS and secure transport settings.
- [ ] Review API CORS configuration.
- [ ] Configure production logging, monitoring, and suspicious authentication activity alerts.
- [ ] Implement database backups and secure credential management.
- [ ] Review dependencies for security updates before deployment.

## 4. Known Implementation Caveats

1. `UserResource` references `full_name`; verify that the corresponding accessor exists in the `User` model.
2. Load `roles.permissions` before serializing users to avoid unnecessary database queries.
3. Public registration currently creates active accounts without requiring verified email.
4. `AuthService::authenticate` currently uses validation-style errors for invalid credentials and inactive accounts.
5. `SocialAuthService` currently uses generic runtime exceptions for several distinct authentication conditions.
6. The Google authentication controller currently treats those runtime exceptions as HTTP 401, including cases that should eventually return 403, 409, or 500.
7. The initial Google social account creation flow requires concurrency testing and appropriate database-conflict handling.
8. Sanctum's `auth:sanctum` middleware can accept stateful session authentication when configured; confirm Bearer-only enforcement.
9. Existing authentication tests may still reference removed endpoints or methods.
10. Passing authentication tests does not establish production security or authorization correctness.

## 5. Environment Configuration Review

### Laravel API

Example configuration keys:

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_URL=

FRONTEND_URL=

GOOGLE_WEB_CLIENT_ID=

SANCTUM_STATEFUL_DOMAINS=
SESSION_DRIVER=
SESSION_DOMAIN=
SESSION_SECURE_COOKIE=true
```

If the Laravel API is strictly Bearer-only, review whether stateful Sanctum domains and Laravel session-related settings are needed for API authentication. Do not enable stateful authentication merely because Sanctum is installed.

Configure production token expiration in `config/sanctum.php` and ensure expired-token cleanup is scheduled where applicable.

### React Native + Expo

```dotenv
EXPO_PUBLIC_API_URL=
EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID=
```

Only public configuration values should use the `EXPO_PUBLIC_` prefix.

### Next.js

Keep private Laravel API tokens and server-only credentials outside variables prefixed with `NEXT_PUBLIC_`.

Use server-side environment variables for sensitive authentication integration.

Never commit `.env` files, private keys, access tokens, or OAuth client secrets.

## 6. Security Test Checklist

### Authentication Tests

- [ ] Registration assigns only permitted platform roles.
- [ ] Login issues a valid Sanctum token.
- [ ] Invalid credentials are rejected.
- [ ] Missing Bearer token returns 401.
- [ ] Invalid Bearer token returns 401.
- [ ] Suspended users cannot authenticate.
- [ ] Suspended users cannot access protected endpoints.
- [ ] Logout revokes the current token.
- [ ] Revoked tokens cannot be reused.
- [ ] Other device tokens remain valid after single-device logout.

### Social Authentication Tests

- [ ] Invalid Google ID token is rejected.
- [ ] Expired Google ID token is rejected.
- [ ] Incorrect OAuth audience is rejected.
- [ ] Unverified Google email is rejected.
- [ ] New Google identity creates a customer account.
- [ ] Existing linked Google identity retrieves the correct user.
- [ ] Existing unlinked email requires authenticated account linking.
- [ ] Suspended linked account cannot authenticate.
- [ ] Duplicate social identities are prevented.
- [ ] Concurrent social login requests do not create duplicate users.

### Authorization Tests

- [ ] Customer cannot access administrative endpoints.
- [ ] User cannot assign themselves administrative roles.
- [ ] Platform permissions are enforced.
- [ ] Cross-user booking access is denied.
- [ ] Provider membership is enforced.
- [ ] Organization membership is enforced.
- [ ] Pilot eligibility is enforced.
- [ ] Super Admin cannot bypass protected resource invariants.

## 7. Production Readiness Status

| Area                                                  | Status                                            |
| ----------------------------------------------------- | ------------------------------------------------- |
| Original registration and mobile authentication tests | Previously reported passing                       |
| Unified authentication refactor                       | Implemented; tests pending verification           |
| Sanctum Bearer-token architecture                     | Implemented; security review pending              |
| Google Social Login backend                           | Initial implementation                            |
| Google Social Login integration tests                 | Pending                                           |
| Next.js authentication integration                    | Pending                                           |
| React Native Google Sign-In                           | Pending                                           |
| Account-linking security                              | Planned                                           |
| RBAC middleware                                       | Implemented; comprehensive policy testing pending |
| Token lifecycle and device management                 | Pending                                           |
| Email verification and password reset                 | Planned                                           |
| Didit KYC integration                                 | Planned                                           |
| Production security audit                             | Not completed                                     |

---

**Documentation version:** 1.1  
**Project:** ApusFly  
**Module:** Authentication Security  
**Last updated:** October 17, 2026
