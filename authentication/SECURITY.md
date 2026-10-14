# Authentication Security & Production Checklist

**Document status:** Development guidance; not a security certification.

## Verified by reported local tests

- [x] Customer registration and platform role attachment
- [x] Mobile login returns a Sanctum token
- [x] Invalid password is rejected
- [x] Unauthenticated `/auth/me` is rejected
- [x] Suspended user cannot log in
- [x] Logout deletes current mobile token
- [x] Suspended token cannot access `/auth/me`

## Required before production

- [ ] Remove broad `Gate::before()` bypasses; use platform-scoped middleware and ownership-aware policies.
- [ ] Test Next.js stateful login, CSRF enforcement, session renewal, and logout in a browser across actual subdomains.
- [ ] Restrict browser login to intended stateful clients; do not rely on a caller-provided `client_type` to distinguish browser and mobile.
- [ ] Configure HTTPS, secure/HttpOnly/SameSite cookies, trusted origins, CORS credentials, and stateful domains.
- [ ] Verify API error envelope for authentication, validation, 403/404/429, and unexpected exceptions; do not expose stack traces in production.
- [ ] Implement email verification and appropriate gating for sensitive actions.
- [ ] Implement password reset with expiring single-use tokens and rate limits.
- [ ] Define Sanctum token expiration and device/session revocation policy; consider token rotation and per-device names.
- [ ] Revoke outstanding mobile tokens on suspension/password reset as appropriate; review web session invalidation.
- [ ] Protect login/registration with rate limits keyed appropriately for IP and account; review account enumeration.
- [ ] Confirm user passwords are hashed by User model cast or explicitly with `Hash::make()`; never store plaintext.
- [ ] Store mobile token in Expo SecureStore; clear on logout; avoid token logging and analytics exposure.
- [ ] Use server-side policy checks for booking ownership, provider membership, pilot credentials, and KYC.
- [ ] Implement Didit KYC with verified webhook signatures and replay resistance; identity checks do not substitute for pilot credential verification.
- [ ] Validate UUID Sanctum tokenable morphs and schema migrations in the deployment database.
- [ ] Add tests for token reuse after logout, suspended user access, role escalation, cross-user resource access, and web CSRF.

## Known implementation caveats

1. `UserResource` uses `full_name`; verify the accessor exists on `User`.
2. If `roles.permissions` is not eager-loaded, serialization may trigger extra database queries.
3. Current registration creates active users without requiring verified email; production access controls must distinguish account existence from verified/eligible status.
4. `AuthService::authenticate` reports inactive status via a validation exception; consider a consistent non-enumerating public login response.
5. Authentication tests alone do not establish production readiness.

## Configuration review

Example environment keys to review (values intentionally omitted):

```dotenv
APP_URL=
APP_DEBUG=false
SESSION_DRIVER=
SESSION_DOMAIN=
SESSION_SECURE_COOKIE=true
SANCTUM_STATEFUL_DOMAINS=
FRONTEND_URL=
```

Ensure the configured CORS origins and credentials policy match the deployed Next.js domain and API domain. Never commit `.env`, secret keys, or bearer tokens.
