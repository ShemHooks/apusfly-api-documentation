# ApusFly API Documentation

Version: 1.0 (development baseline)  
Backend: Laravel REST API (`/api/v1`)  
Authentication: Laravel Sanctum

## Contents

- [Authentication overview](authentication/README.md)
- [Endpoint reference](authentication/API_ENDPOINTS.md)
- [Authentication flows](authentication/AUTHENTICATION_FLOW.md)
- [Security and deployment checklist](authentication/SECURITY.md)

## Implementation status

The developer reported **12 passing tests** in the full Laravel suite, including **7 authentication tests** and **3 RBAC authorization tests**. These results are from the developer's local environment and are not an independent audit.

This documentation reflects the code discussed and tested in the development conversation. Verify against the current repository before treating it as an authoritative deployed API specification.

## Response convention

Successful controller responses generally use:

```json
{"success":true,"message":"Request successful.","data":{},"errors":null}
```

Expected API errors use:

```json
{"success":false,"message":"Validation failed.","data":null,"errors":{"email":["Invalid email or password."]}}
```

A `204 No Content` response intentionally has no body.
