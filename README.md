# Signed, Sealed, Delivered: Reverse Proxy Demo

This repository demonstrates **per-user identity-based access control** using a reverse proxy architecture. It showcases how to implement zero-trust security principles where each user gets access only to their own resources, contrasting with traditional VPN network-level access.

## Architecture Overview

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│    Keycloak      │ ← Identity Provider (OAuth2/OIDC)
│  Port: 8180      │   Issues JWT tokens with user identity
└──────┬───────────┘
       │ JWT Token (signed, contains username + roles)
       ▼
┌──────────────────────────────────────────┐
│            Envoy Proxy                   │
│            Port: 8080                    │
│  ┌─────────────────────────────────────┐ │
│  │ 1. JWT Authentication Filter        │ │ ← Validates token signature
│  │    - Verifies JWT signature (JWKS)  │ │ ← Extracts claims (username, roles)
│  │    - Extracts user claims           │ │
│  └─────────────────────────────────────┘ │
│  ┌─────────────────────────────────────┐ │
│  │ 2. RBAC Authorization Filter        │ │ ← Per-user access control
│  │    - Checks preferred_username      │ │ ← Route-specific policies
│  │    - Enforces per-route policies    │ │
│  └─────────────────────────────────────┘ │
│  ┌─────────────────────────────────────┐ │
│  │ 3. Access Logger                    │ │ ← Audit trail with identity
│  │    - Logs user, path, decision      │ │
│  └─────────────────────────────────────┘ │
└────┬──────────────┬──────────────┬───────┘
     │              │              │
     ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Public   │  │ Alice's  │  │  Bob's   │
│   App    │  │   App    │  │   App    │
│ Port:3000│  │ Port:3002│  │ Port:3001│
│          │  │          │  │          │
│ Anyone   │  │ Alice    │  │ Bob      │
│ authed   │  │ ONLY     │  │ ONLY     │
└──────────┘  └──────────┘  └──────────┘
```

## How Each Technology is Used

### 1. Keycloak (Identity Provider)

**Role**: Centralized authentication and identity management

**What it does**:
- Authenticates users (alice, bob) with username/password
- Issues JWT tokens containing user identity and roles
- Manages user credentials and realm configuration
- Provides JWKS (JSON Web Key Set) endpoint for public key distribution
- Implements OAuth2/OpenID Connect protocols

**Security features**:
- Secure password storage with bcrypt hashing
- Token-based authentication (no session state)
- Automatic token expiration (configurable TTL)
- Public/private key cryptography for JWT signing
- Realm isolation (multi-tenancy support)

**Configuration in this demo**:
- Realm: `demo`
- Client: `demo-client` (public client for password grant)
- Users:
  - `alice` / `password` → roles: [user]
  - `bob` / `password` → roles: [user, admin]
- Token format: JWT with RS256 signature
- Claims included: `preferred_username`, `email`, `realm_access.roles`

**Integration point**:
- Envoy fetches public keys from `http://keycloak:8180/realms/demo/protocol/openid-connect/certs`
- Tokens issued with issuer claim `http://localhost:8180/realms/demo`

### 2. Envoy Proxy (Reverse Proxy & Security Gateway)

**Role**: Request routing, authentication validation, and authorization enforcement

**What it does**:
- Acts as single entry point for all backend services
- Validates JWT signatures using Keycloak's public keys (JWKS)
- Extracts user identity from validated tokens
- Enforces per-user, per-route authorization policies
- Routes requests to appropriate backend services
- Logs every request with full context (user, path, decision)
- Handles HTTPS/TLS termination (not enabled in demo for simplicity)

**Security architecture**:

#### Layer 1: JWT Authentication Filter
```yaml
envoy.filters.http.jwt_authn
```
- **Validates** JWT signature against Keycloak's public key
- **Verifies** issuer, expiration (exp), issued-at (iat) claims
- **Extracts** JWT payload into Envoy metadata for downstream filters
- **Rejects** invalid, expired, or malformed tokens (401)
- **Caches** JWKS keys (5-minute TTL) to reduce Keycloak load

#### Layer 2: RBAC Authorization Filter
```yaml
envoy.filters.http.rbac
```
- **Reads** `preferred_username` from JWT metadata
- **Enforces** policies based on username and path
- **Blocks** unauthorized access (403 Forbidden)
- **Allows** only exact username matches for personal resources

**Authorization Policies**:
```yaml
policies:
  "allow-public":
    # Anyone authenticated can access /public and /health
    permissions: [path prefix: /public OR /health]
    principals: [any: true]

  "allow-alice-only":
    # ONLY alice can access /alice
    permissions: [path prefix: /alice]
    principals: [jwt_payload.preferred_username == "alice"]

  "allow-bob-only":
    # ONLY bob can access /bob
    permissions: [path prefix: /bob]
    principals: [jwt_payload.preferred_username == "bob"]
```

**Security implications**:
- Zero trust: Every request validated, even from "trusted" users
- Least privilege: Users get exactly their resources, nothing more
- Defense in depth: Multiple layers (authentication → authorization → routing)
- Fail-secure: Default deny (if no policy matches, request blocked)
- Immutable enforcement: Backend services can't bypass authorization

#### Layer 3: Access Logging
```yaml
envoy.access_loggers.stdout
```
- Logs JSON-formatted access records
- Captures: timestamp, method, path, status, user, roles, duration
- Enables audit trails and security monitoring
- Supports compliance requirements (who accessed what, when)

**Integration points**:
- Consumes JWTs issued by Keycloak
- Validates tokens using Keycloak's public keys
- Routes to backend services (public-app, alice-app, bob-app)
- Enriches requests with `x-jwt-payload` header for backends

### 3. Backend Services (Node.js/Express)

**Three services**:

#### public-app (Port 3000)
- **Access**: Any authenticated user
- **Purpose**: Shared resource accessible to all users
- **Security**: Relies on Envoy for authentication enforcement
- **Response**: Echoes user identity from JWT header

#### alice-app (Port 3002)
- **Access**: Only user "alice"
- **Purpose**: Alice's personal workspace/data
- **Security**: Protected by Envoy's username-based RBAC
- **Response**: Alice-specific content

#### bob-app (Port 3001)
- **Access**: Only user "bob"
- **Purpose**: Bob's personal workspace/data
- **Security**: Protected by Envoy's username-based RBAC
- **Response**: Bob-specific content
- **Note**: Even though Bob has "admin" role, this doesn't grant him access to Alice's app

**Security model**:
- Services trust Envoy's authorization decisions
- No authentication logic in application code
- Can optionally read `x-jwt-payload` header for user context
- Isolated via Docker network (not directly accessible)
- Run as non-root users in containers

**Why this matters**:
- Separation of concerns: Auth logic centralized in Envoy
- Services can be written in any language
- Consistent security policy across all services
- Easier to add new services (inherit Envoy's protection)

## Technology Integration Flow

### 1. User Authentication
```
Client → Keycloak
POST /realms/demo/protocol/openid-connect/token
  username=alice&password=password&grant_type=password&client_id=demo-client

Keycloak:
  1. Validates credentials
  2. Generates JWT with claims:
     - iss: http://localhost:8180/realms/demo
     - preferred_username: alice
     - realm_access.roles: [user]
  3. Signs JWT with private key (RS256)
  4. Returns: {access_token: "eyJhbGc...", expires_in: 300}
```

### 2. Request Authorization
```
Client → Envoy
GET /alice
Authorization: Bearer eyJhbGc...

Envoy JWT Filter:
  1. Extracts token from Authorization header
  2. Fetches Keycloak's public key (cached)
  3. Verifies JWT signature
  4. Checks expiration
  5. Extracts preferred_username: "alice"
  6. Stores in metadata

Envoy RBAC Filter:
  1. Reads path: /alice
  2. Reads username from metadata: "alice"
  3. Matches policy: "allow-alice-only"
  4. Decision: ALLOW ✓

Envoy Router:
  1. Routes to alice-app cluster
  2. Rewrites path: /alice → /
  3. Adds x-jwt-payload header
  4. Forwards request

Alice-app:
  1. Receives request
  2. Responds with Alice-specific content

Envoy Access Logger:
  1. Logs: {user: "alice", path: "/alice", status: 200}
```

### 3. Authorization Denial Example
```
Client → Envoy
GET /bob
Authorization: Bearer <alice's token>

Envoy JWT Filter:
  ✓ Token valid, user: alice

Envoy RBAC Filter:
  Path: /bob
  Username: alice
  Policy check: "allow-bob-only" requires username == "bob"
  Decision: DENY ✗

Envoy Response:
  403 Forbidden
  RBAC: access denied

Envoy Access Logger:
  {user: "alice", path: "/bob", status: 403}
```

## Security Implications & Threat Model

### What This Architecture Protects Against

#### 1. Unauthorized Access (Solved ✓)
- **Threat**: Alice accessing Bob's resources
- **Protection**: Username-based RBAC policies
- **Result**: 403 Forbidden even with valid authentication

#### 2. Privilege Escalation (Solved ✓)
- **Threat**: User claiming admin role
- **Protection**: JWT signature verification prevents token tampering
- **Result**: Modified tokens rejected with 401

#### 3. Lateral Movement (Solved ✓)
- **Threat**: Compromised user accessing other services
- **Protection**: Per-service authorization checks
- **Result**: Limited blast radius (Alice can't reach Bob's app)

#### 4. Credential Theft (Mitigated ⚠️)
- **Threat**: Stolen JWT used by attacker
- **Protection**: Short token expiration (5 minutes)
- **Limitation**: Valid tokens still work until expiration
- **Mitigation**: Use short-lived tokens + refresh tokens

#### 5. Network Sniffing (Not Protected ✗)
- **Threat**: Man-in-the-middle attacks
- **Protection**: None (HTTP only in demo)
- **Production fix**: Enable HTTPS/TLS in Envoy configuration

#### 6. Audit & Compliance (Solved ✓)
- **Requirement**: Know who accessed what and when
- **Protection**: Comprehensive access logging with user identity
- **Result**: Full audit trail for compliance

### Attack Scenarios Tested

| Attack | Protected? | How |
|--------|-----------|-----|
| No authentication | ✓ Yes | JWT filter returns 401 |
| Expired token | ✓ Yes | JWT filter validates `exp` claim |
| Modified token | ✓ Yes | Signature verification fails |
| Wrong user accessing resource | ✓ Yes | RBAC filter denies (403) |
| Admin accessing user resources | ✓ Yes | Username checked, not roles |
| Direct backend access | ✓ Yes | Services isolated in Docker network |
| Replay attack | ⚠️ Partial | Limited by token expiration |

### Security Best Practices Demonstrated

1. **Defense in Depth**: Multiple security layers (authn → authz → logging)
2. **Least Privilege**: Users get only their resources
3. **Zero Trust**: Always verify, never trust
4. **Fail Secure**: Default deny policy
5. **Separation of Concerns**: Auth logic separate from business logic
6. **Cryptographic Verification**: JWT signatures prevent tampering
7. **Audit Logging**: Every decision recorded
8. **Immutable Enforcement**: Can't bypass proxy to reach backends

### Production Hardening Checklist

For production use, add:
- [ ] HTTPS/TLS termination in Envoy
- [ ] Token revocation (implement token blacklist)
- [ ] Rate limiting per user
- [ ] Request size limits
- [ ] IP allowlisting for admin endpoints
- [ ] Refresh token flow (don't use password grant)
- [ ] Security headers (HSTS, CSP, X-Frame-Options)
- [ ] Centralized log aggregation (ELK/Splunk)
- [ ] Alert on repeated 403s (potential attack)
- [ ] Regular JWKS key rotation
- [ ] Use confidential clients (not public)
- [ ] Enable Envoy access logging to SIEM

## Quick Start

### Prerequisites
- Docker and Docker Compose
- curl (for testing)
- jq (for parsing JSON)

### Start the Demo

```bash
# Build and start all services
docker-compose up -d

# Wait for services to be ready (30-60 seconds)
docker-compose ps

# Run automated demo
./demo-script.sh
```

### Manual Testing

```bash
# Get Alice's token
TOKEN_ALICE=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=alice&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')

# Get Bob's token
TOKEN_BOB=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=bob&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')

# Test access patterns
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/public  # 200 ✓
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/alice   # 200 ✓
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/bob     # 403 ✗

curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/public    # 200 ✓
curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/alice     # 403 ✗
curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/bob       # 200 ✓
```

## Access Control Matrix

```
┌─────────────┬──────────┬────────────┬──────────┐
│   User      │ /public  │  /alice    │  /bob    │
├─────────────┼──────────┼────────────┼──────────┤
│ No Auth     │    401   │    401     │   401    │
│ Alice       │    200   │    200 ✓   │   403 ✗  │
│ Bob         │    200   │    403 ✗   │   200 ✓  │
└─────────────┴──────────┴────────────┴──────────┘
```

## Key Demo Points

### VPN Problems
- **Network-level access**: Once inside, you can reach everything
- **No identity awareness**: Systems don't know who you are
- **Coarse-grained**: All or nothing access
- **Lateral movement**: Compromised user can pivot to other services
- **Limited audit**: Network logs don't show application-level actions

### Reverse Proxy Benefits (Shown in Demo)
1. **Identity-Aware**: Every request tied to a user
2. **Least Privilege**: Alice gets Alice's resources, Bob gets Bob's
3. **Zero Trust**: Even "admin" Bob can't access Alice's app
4. **Fine-Grained**: Per-user, per-resource authorization
5. **Complete Audit**: Full trail of who accessed what

### The "Aha" Moment
When Bob (admin) tries to access Alice's app and gets **403 Forbidden**:
- Authentication ≠ Authorization
- Roles don't override per-user policies
- True zero trust in action

## Comparison: VPN vs Reverse Proxy

| Aspect | VPN | Reverse Proxy |
|--------|-----|---------------|
| **Access Model** | Network-level (all or nothing) | Resource-level (per-user) |
| **Identity** | IP address | User identity (JWT) |
| **Authorization** | None (trust network) | Every request validated |
| **Audit** | Network logs (IP, port) | Application logs (user, resource) |
| **Lateral Movement** | Easy (same network) | Blocked (per-resource authz) |
| **Scalability** | VPN server bottleneck | Stateless, horizontally scalable |
| **Complexity** | VPN client required | Standard HTTP (any client) |

## Troubleshooting

### Services won't start
```bash
docker-compose logs
docker-compose down && docker-compose up -d
```

### 401 Unauthorized
- Token expired (5 min TTL)
- Invalid token format
- Check: `echo $TOKEN_ALICE | cut -d'.' -f2 | base64 -d | jq .`

### 403 Forbidden
- This is expected! Alice can't access /bob
- Check access logs: `docker-compose logs envoy`

## Cleanup

```bash
docker-compose down        # Stop services
docker-compose down -v     # Stop and remove volumes (reset Keycloak)
```

## Further Reading

### In-Depth Explanations
- [Reverse Proxy Architecture](docs/REVERSE-PROXY-EXPLAINED.md) - How reverse proxies provide security
- [OAuth2 and OIDC](docs/OAUTH-OIDC-EXPLAINED.md) - Authentication protocols explained
- [JWT Tokens](docs/JWT-EXPLAINED.md) - Token structure and security properties
- [Access Logging](docs/ACCESS-LOGGING-EXPLAINED.md) - Identity-aware audit trails

### External Resources
- [Envoy JWT Authentication](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/jwt_authn_filter)
- [Envoy RBAC](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rbac_filter)
- [NIST Zero Trust Architecture](https://www.nist.gov/publications/zero-trust-architecture)
- [OAuth2 Password Grant](https://oauth.net/2/grant-types/password/) (not for production!)
