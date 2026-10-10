# ADR 2: JWT-Based Stateless Authentication

## Status
**Accepted**

## Date
2025-10-18

## Context

The Hospital Bed Manager system serves 4 distinct user roles (ER Staff, ICU Manager, Ward Staff, Hospital Administrator) with different permission levels. The authentication mechanism must:

1. **Support role-based access control (RBAC)**: Each API endpoint must verify not just that a user is authenticated, but that they have the correct role (e.g., only ICU Managers can approve bed requests, only Admins can access the chatbot).

2. **Work with real-time WebSocket connections**: Socket.IO connections need to be authenticated. The mechanism must allow passing credentials during WebSocket handshake.

3. **Be stateless for horizontal scalability**: If the system needs to scale to multiple server instances (behind a load balancer), session-based auth would require a shared session store (Redis).

4. **Support mobile access**: Hospital staff access the system from mobile devices via the hospital WiFi network. The solution must work without cookies (for cross-origin requests from different devices).

**Alternatives considered**:
- **Session-based authentication (express-session + Redis)**: Server maintains session state. Pros: easy revocation, familiar pattern. Cons: requires Redis for multi-instance deployment, doesn't travel well with WebSocket handshakes, stateful.
- **OAuth 2.0 / OpenID Connect**: Industry standard for SSO. Pros: standardized, supports multiple identity providers. Cons: significant complexity overhead for an internal hospital tool with 4 user accounts, requires an identity provider setup.
- **API Keys**: Simple key-based auth. Pros: very simple. Cons: no user identity, no expiration, no role information embedded.

## Decision

We will use **JSON Web Tokens (JWT)** via the `jsonwebtoken` library (v9.0.2) for stateless authentication, combined with **bcrypt** (v6.0.0) for password hashing.

### Token Structure
```json
{
  "id": "<MongoDB ObjectId>",
  "username": "anuradha",
  "role": "icu_manager",
  "ward": "ICU",
  "name": "Anuradha",
  "iat": 1698000000,
  "exp": 1698086400
}
```

### Implementation
- **Login flow**: `POST /api/auth/login` → validate credentials with `bcrypt.compare()` → issue JWT with `jwt.sign()` (24h expiry)
- **Token storage**: Client stores JWT in `localStorage` and attaches via Axios interceptor (`Authorization: Bearer <token>`)
- **Verification middleware**: `authenticateToken` middleware on all `/api/*` routes extracts token from Authorization header, verifies with `jwt.verify()`, attaches decoded payload to `req.user`
- **Authorization middleware**: Higher-order function `authorize(...roles)` checks `req.user.role` against an allowed list
- **Auto-logout**: Axios response interceptor detects 401/403 responses and triggers automatic logout with token cleanup

## Consequences

### Positive
- **Stateless**: No server-side session storage. Token self-contains user identity and role. Any server instance can verify any token.
- **Role information embedded**: `req.user.role` and `req.user.ward` are available in every route handler without an additional database lookup.
- **WebSocket compatible**: Token passed in Socket.IO `auth` option during handshake (`auth: { token }`).
- **Cross-device support**: Works via Authorization header, no cookie domain restrictions. Staff can access from any device on the hospital network.
- **Simple implementation**: Two middleware functions (`authenticateToken` + `authorize`) compose cleanly in Express route definitions.

### Negative
- **No immediate revocation**: Once issued, a JWT is valid until expiry (24h). If a user is compromised, we cannot invalidate their token without maintaining a blacklist (defeating the stateless benefit). Mitigated by the 24h expiry window.
- **Token size**: JWT payload contains user details, making each HTTP request slightly larger than a session cookie. Negligible for this application.
- **localStorage vulnerability**: Storing JWT in localStorage exposes it to XSS attacks. Mitigated by the application being an internal hospital tool (not public-facing) and Content Security Policy headers.
- **No refresh token mechanism**: When the 24h token expires, the user must re-login. For a hospital application with shift-based access, this aligns with security requirements.
