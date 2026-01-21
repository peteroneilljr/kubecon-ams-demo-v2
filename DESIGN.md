# Design Document: Reverse Proxy Demo Architecture

## Executive Summary

This document details the technical architecture and implementation requirements for a live demo showcasing reverse proxy security patterns using Envoy, Keycloak, and sample applications.

**Goal**: Demonstrate identity-aware, least-privilege access control that outperforms traditional VPN models.

**Target Duration**: 10-minute live demo
**Complexity**: Medium (containerized services, minimal configuration)
**Reliability**: High (no external dependencies, runs locally)

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Component Specifications](#component-specifications)
3. [Network Flow](#network-flow)
4. [Security Model](#security-model)
5. [Implementation Requirements](#implementation-requirements)
6. [Configuration Details](#configuration-details)
7. [Testing Strategy](#testing-strategy)
8. [Failure Modes & Mitigations](#failure-modes--mitigations)

---

## Architecture Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                         Host Machine                          │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                    Docker Network                       │  │
│  │                   (demo-network)                        │  │
│  │                                                         │  │
│  │  ┌─────────────┐       ┌──────────────┐               │  │
│  │  │  Keycloak   │       │    Envoy     │               │  │
│  │  │   :8180     │◄──────┤    :8080     │               │  │
│  │  │             │       │              │               │  │
│  │  │  - Issues   │       │  - Validates │               │  │
│  │  │    JWT      │       │    JWT       │               │  │
│  │  │  - Manages  │       │  - Enforces  │               │  │
│  │  │    users    │       │    RBAC      │               │  │
│  │  └─────────────┘       │  - Routes    │               │  │
│  │                        └──────┬───────┘               │  │
│  │                               │                        │  │
│  │                    ┌──────────┴──────────┐            │  │
│  │                    │                     │            │  │
│  │             ┌──────▼──────┐      ┌──────▼──────┐     │  │
│  │             │ Public App  │      │ Internal App│     │  │
│  │             │   :3000     │      │   :3001     │     │  │
│  │             │             │      │             │     │  │
│  │             │ - Echo      │      │ - Echo      │     │  │
│  │             │   service   │      │   service   │     │  │
│  │             │ - No auth   │      │ - No auth   │     │  │
│  │             │   logic     │      │   logic     │     │  │
│  │             └─────────────┘      └─────────────┘     │  │
│  │                                                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  Exposed Ports:                                              │
│  - 8080  → Envoy (main entry point)                         │
│  - 8180  → Keycloak (for token generation & admin)          │
│                                                              │
└──────────────────────────────────────────────────────────────┘

External Access:
- Demo commands: curl http://localhost:8080/*
- Token generation: curl http://localhost:8180/realms/demo/protocol/openid-connect/token
- Keycloak admin: http://localhost:8180/admin
```

### Components Summary

| Component | Purpose | Technology | Port |
|-----------|---------|------------|------|
| Envoy | Reverse proxy with JWT validation & RBAC | Envoy 1.28+ | 8080 |
| Keycloak | Identity provider & token issuer | Keycloak 23+ | 8180 |
| Public App | Sample service (any user) | Node.js/Python/Go | 3000 |
| Internal App | Sample service (admin only) | Node.js/Python/Go | 3001 |

---

## Component Specifications

### 1. Envoy Proxy

**Role**: API Gateway with security enforcement

**Responsibilities**:
- Terminate incoming HTTP requests
- Validate JWT tokens (signature + expiration)
- Extract claims from JWT (username, roles)
- Enforce RBAC policies per route
- Route authorized requests to backend services
- Log all access attempts with identity context

**Key Configuration Elements**:
- **Listeners**: Single HTTP listener on port 8080
- **Routes**:
  - `/public` → public-app:3000
  - `/internal` → internal-app:3001
- **HTTP Filters** (in order):
  1. JWT Authentication Filter
  2. RBAC Filter
  3. Router Filter
- **Access Logging**: JSON format with JWT claims

**Technology Choice**: Envoy 1.28+ (latest stable)
- Well-documented JWT filter
- Native RBAC support
- Production-grade reverse proxy
- Used in Kubernetes ecosystem (Istio, Ambassador, etc.)

**Configuration File**: `envoy.yaml` (~150-200 lines)

---

### 2. Keycloak

**Role**: Identity provider (IdP) and authorization server

**Responsibilities**:
- Authenticate users (username/password for demo)
- Issue JWT access tokens
- Embed user claims (username, roles) in tokens
- Provide JWKS endpoint for JWT signature validation
- Manage realm, clients, users, and roles

**Realm Configuration**:
- **Realm Name**: `demo`
- **Client**: `demo-client`
  - Client Protocol: `openid-connect`
  - Access Type: `public` (for demo simplicity, no client secret)
  - Direct Access Grants: `enabled` (allows password grant type)
  - Valid Redirect URIs: `*` (demo only)

**Users**:
| Username | Password | Roles | Description |
|----------|----------|-------|-------------|
| alice | password | user | Regular user, can access public only |
| bob | password | user, admin | Admin user, can access both services |

**Roles**:
- `user`: Default role, grants access to public services
- `admin`: Elevated role, grants access to internal services

**Token Configuration**:
- Access Token Lifespan: 5 minutes (short for demo)
- Include user roles in token claims: Yes
- Claim name: `realm_access.roles` (Keycloak default)

**JWKS Endpoint**: `http://keycloak:8180/realms/demo/protocol/openid-connect/certs`

**Technology Choice**: Keycloak 23+ (latest stable)
- Industry-standard OIDC provider
- Easy to configure for demos
- Provides admin UI for visibility
- Supports all necessary OIDC flows

**Configuration Method**:
- Option A: Realm import JSON (pre-configured)
- Option B: Initialization scripts (automated setup)

---

### 3. Public App

**Role**: Sample backend service (accessible to any authenticated user)

**Responsibilities**:
- Accept HTTP requests on `/` or `/health`
- Echo request information (headers, path, method)
- Show authenticated user identity (from headers added by Envoy)
- Return JSON response

**No Security Logic**: The app trusts headers from Envoy (assumes Envoy is the only entry point)

**Endpoints**:
- `GET /`: Main endpoint
  - Returns JSON with request metadata
  - Includes `x-forwarded-user` header (set by Envoy from JWT)

- `GET /health`: Health check endpoint
  - Returns `{"status": "healthy"}`

**Response Format**:
```json
{
  "service": "public-app",
  "message": "Welcome to the public service",
  "authenticated_user": "alice",
  "timestamp": "2026-01-21T10:30:00Z",
  "path": "/",
  "method": "GET"
}
```

**Technology Choice**:
- **Preferred**: Node.js (Express) - ~30 lines of code
- **Alternative**: Python (Flask) - ~25 lines of code
- **Alternative**: Go (net/http) - ~40 lines of code

Selection criteria: Simplicity, fast startup, small container image

**Container Image**: Custom (built from Dockerfile)
- Base: `node:20-alpine` or `python:3.11-slim` or `golang:1.21-alpine`
- Size target: < 50MB
- Startup time: < 2 seconds

---

### 4. Internal App

**Role**: Sample backend service (accessible to admin users only)

**Responsibilities**: Same as Public App, but represents sensitive internal service

**Key Difference**: Envoy enforces different authorization policy for routes to this service

**Endpoints**:
- `GET /`: Main endpoint
  - Returns JSON with request metadata

- `GET /health`: Health check endpoint

**Response Format**:
```json
{
  "service": "internal-app",
  "message": "Welcome to the internal admin service",
  "authenticated_user": "bob",
  "roles": ["user", "admin"],
  "timestamp": "2026-01-21T10:30:00Z",
  "path": "/",
  "method": "GET",
  "warning": "This is a sensitive internal service"
}
```

**Technology Choice**: Same as Public App (for consistency)

**Container Image**: Same base as Public App, different code/name

---

## Network Flow

### Flow 1: Unauthenticated Request (Blocked)

```
1. Client → Envoy
   GET /public HTTP/1.1
   Host: localhost:8080
   (no Authorization header)

2. Envoy: JWT Filter
   - Check for Authorization header
   - Not found → REJECT

3. Envoy → Client
   HTTP/1.1 401 Unauthorized
   {"message": "Jwt is missing"}
```

### Flow 2: Authenticated Request - Regular User to Public App (Allowed)

```
1. Client → Keycloak
   POST /realms/demo/protocol/openid-connect/token
   username=alice&password=password&grant_type=password&client_id=demo-client

2. Keycloak → Client
   {
     "access_token": "eyJhbGc...",
     "expires_in": 300,
     "token_type": "Bearer"
   }

3. Client → Envoy
   GET /public HTTP/1.1
   Host: localhost:8080
   Authorization: Bearer eyJhbGc...

4. Envoy: JWT Filter
   - Extract token from header
   - Fetch JWKS from Keycloak (cached)
   - Validate signature → VALID
   - Check expiration → VALID
   - Extract claims: {username: "alice", realm_access: {roles: ["user"]}}
   - Set header: x-forwarded-user: alice
   - PASS to next filter

5. Envoy: RBAC Filter
   - Check policy for route /public
   - Policy: "any authenticated user"
   - User is authenticated → ALLOW
   - PASS to router

6. Envoy → Public App
   GET / HTTP/1.1
   Host: public-app:3000
   x-forwarded-user: alice
   x-forwarded-roles: user

7. Public App → Envoy
   HTTP/1.1 200 OK
   {"service": "public-app", "authenticated_user": "alice", ...}

8. Envoy → Client
   HTTP/1.1 200 OK
   {"service": "public-app", "authenticated_user": "alice", ...}

9. Envoy: Access Log
   [2026-01-21T10:30:00.000Z] "GET /public HTTP/1.1" 200 - user=alice roles=user
```

### Flow 3: Authenticated Request - Regular User to Internal App (Denied)

```
1-4. [Same as Flow 2: Token acquisition and JWT validation]

5. Envoy: RBAC Filter
   - Check policy for route /internal
   - Policy: "requires role 'admin'"
   - User roles: ["user"]
   - "admin" NOT in roles → DENY
   - REJECT request

6. Envoy → Client
   HTTP/1.1 403 Forbidden
   {"message": "RBAC: access denied"}

7. Envoy: Access Log
   [2026-01-21T10:30:05.000Z] "GET /internal HTTP/1.1" 403 - user=alice roles=user DENIED
```

### Flow 4: Authenticated Request - Admin User to Internal App (Allowed)

```
1. Client → Keycloak
   [Authenticate as bob]

2-4. [JWT validation succeeds, claims extracted: {username: "bob", roles: ["user", "admin"]}]

5. Envoy: RBAC Filter
   - Check policy for route /internal
   - Policy: "requires role 'admin'"
   - User roles: ["user", "admin"]
   - "admin" IN roles → ALLOW
   - PASS to router

6-9. [Request forwarded to internal-app, response returned, access logged]
```

---

## Security Model

### Authentication Layer (JWT Validation)

**Goal**: Verify the identity of the requester

**Mechanism**:
- Client presents Bearer token in `Authorization` header
- Envoy validates JWT signature using Keycloak's JWKS
- Envoy checks token expiration (`exp` claim)
- Envoy extracts identity claims (`sub`, `preferred_username`, `realm_access`)

**Outcomes**:
- ✅ Valid token → Extract claims, proceed to authorization
- ❌ No token → 401 Unauthorized
- ❌ Invalid signature → 401 Unauthorized
- ❌ Expired token → 401 Unauthorized

**Configuration in Envoy**:
```yaml
http_filters:
- name: envoy.filters.http.jwt_authn
  typed_config:
    providers:
      keycloak:
        issuer: "http://keycloak:8180/realms/demo"
        remote_jwks:
          http_uri:
            uri: "http://keycloak:8180/realms/demo/protocol/openid-connect/certs"
            cluster: keycloak_cluster
          cache_duration: 300s
        forward: true
        payload_in_metadata: "jwt_payload"
    rules:
    - match: { prefix: "/" }
      requires: { provider_name: "keycloak" }
```

### Authorization Layer (RBAC)

**Goal**: Enforce least-privilege access based on user roles

**Mechanism**:
- Envoy extracts role claims from validated JWT
- Envoy evaluates RBAC policy for the requested route
- Policy specifies which roles are required
- Request allowed only if user has required role

**Policies**:

| Route | Policy | Required Role | Outcome |
|-------|--------|---------------|---------|
| `/public` | Any authenticated user | (any) | All authenticated users allowed |
| `/internal` | Admin users only | `admin` | Only users with "admin" role allowed |

**Outcomes**:
- ✅ User has required role → Forward to backend
- ❌ User lacks required role → 403 Forbidden

**Configuration in Envoy**:
```yaml
http_filters:
- name: envoy.filters.http.rbac
  typed_config:
    rules:
      action: ALLOW
      policies:
        "public-access":
          permissions:
          - header:
              name: ":path"
              prefix_match: "/public"
          principals:
          - any: true  # Any authenticated user (JWT filter already validated)

        "admin-access":
          permissions:
          - header:
              name: ":path"
              prefix_match: "/internal"
          principals:
          - metadata:
              filter: "envoy.filters.http.jwt_authn"
              path:
              - key: "jwt_payload"
              - key: "realm_access"
              - key: "roles"
              value:
                list_match:
                  one_of:
                    string_match:
                      exact: "admin"
```

### Trust Boundaries

```
┌─────────────────────────────────────────────────────────┐
│                    Untrusted Zone                        │
│                                                          │
│  - User's browser/curl                                  │
│  - Must present valid JWT                               │
│  - No direct access to backend services                 │
│                                                          │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ JWT Token
                     │
              ┌──────▼──────┐
              │    Envoy    │  ← Enforcement Point
              │             │
              │  Validates  │
              │  Authorizes │
              └──────┬──────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│                    Trusted Zone                          │
│                                                          │
│  - Backend services (public-app, internal-app)          │
│  - Trust headers set by Envoy                           │
│  - No authentication logic needed                       │
│  - Keycloak (trusted issuer)                            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Key Security Properties**:
1. **Single Enforcement Point**: All traffic flows through Envoy
2. **No Backend Auth**: Services don't implement auth (Envoy handles it)
3. **Defense in Depth**: Both authentication AND authorization required
4. **Audit Trail**: All decisions logged with identity context

---

## Implementation Requirements

### File Structure

```
kubecon-ams-demo-v2/
├── README.md                    # User-facing documentation
├── DEMO-SCRIPT.md              # Presentation script
├── DESIGN.md                   # This document
├── docker-compose.yml          # Service orchestration
├── demo-script.sh              # Automated demo commands
│
├── envoy/
│   ├── envoy.yaml              # Envoy configuration
│   └── Dockerfile              # Custom Envoy image (optional)
│
├── keycloak/
│   ├── realm-export.json       # Pre-configured realm
│   └── init-scripts/           # Alternative: setup scripts
│       └── setup-realm.sh
│
├── public-app/
│   ├── Dockerfile
│   ├── package.json            # If Node.js
│   ├── server.js               # Application code
│   └── .dockerignore
│
├── internal-app/
│   ├── Dockerfile
│   ├── package.json            # If Node.js
│   ├── server.js               # Application code
│   └── .dockerignore
│
└── tests/
    ├── test-demo.sh            # Integration tests
    └── expected-responses/     # Expected outputs for validation
```

### Docker Compose Services

```yaml
version: '3.8'

services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_HTTP_PORT: 8180
    command: start-dev
    volumes:
      - ./keycloak/realm-export.json:/opt/keycloak/data/import/realm.json
    command: start-dev --import-realm
    ports:
      - "8180:8180"
    networks:
      - demo-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8180/realms/demo"]
      interval: 10s
      timeout: 5s
      retries: 30

  envoy:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy/envoy.yaml:/etc/envoy/envoy.yaml
    ports:
      - "8080:8080"
      - "9901:9901"  # Admin interface (optional)
    networks:
      - demo-network
    depends_on:
      keycloak:
        condition: service_healthy

  public-app:
    build: ./public-app
    ports:
      - "3000:3000"
    networks:
      - demo-network
    environment:
      - SERVICE_NAME=public-app

  internal-app:
    build: ./internal-app
    ports:
      - "3001:3001"
    networks:
      - demo-network
    environment:
      - SERVICE_NAME=internal-app

networks:
  demo-network:
    driver: bridge
```

### Envoy Configuration Structure

**File**: `envoy/envoy.yaml`

**Sections**:
1. **Static Resources**
   - Listeners (port 8080)
   - Clusters (keycloak, public-app, internal-app)

2. **Listener Configuration**
   - HTTP connection manager
   - Route configuration
   - HTTP filters chain

3. **HTTP Filters**
   - JWT Authentication filter
   - RBAC filter
   - Router filter

4. **Route Configuration**
   - Route `/public` to public-app cluster
   - Route `/internal` to internal-app cluster

5. **Clusters**
   - `keycloak_cluster`: For JWKS fetching
   - `public_app_cluster`: Backend for public service
   - `internal_app_cluster`: Backend for internal service

6. **Access Logging**
   - Format: JSON
   - Include: timestamp, method, path, status, user, roles, decision

**Estimated Size**: 150-200 lines (well-commented)

### Keycloak Realm Configuration

**File**: `keycloak/realm-export.json`

**Contents**:
- Realm settings (name: "demo")
- Client configuration (demo-client)
- User definitions (alice, bob)
- Role definitions (user, admin)
- Role mappings (alice→user, bob→user+admin)
- Token settings (lifespan, claims)

**Generation Method**:
1. Manually configure Keycloak via admin UI
2. Export realm: Admin Console → Realm Settings → Export
3. Save JSON file for re-import

**Alternative**: Shell scripts that use Keycloak Admin API

### Application Code (Node.js Example)

**File**: `public-app/server.js`

```javascript
const express = require('express');
const app = express();
const PORT = 3000;
const SERVICE_NAME = process.env.SERVICE_NAME || 'public-app';

app.get('/', (req, res) => {
  res.json({
    service: SERVICE_NAME,
    message: `Welcome to the ${SERVICE_NAME}`,
    authenticated_user: req.headers['x-forwarded-user'] || 'anonymous',
    roles: req.headers['x-forwarded-roles'] || 'none',
    timestamp: new Date().toISOString(),
    path: req.path,
    method: req.method
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(PORT, () => {
  console.log(`${SERVICE_NAME} listening on port ${PORT}`);
});
```

**Dependencies**: `express` (package.json)

**Dockerfile**:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## Configuration Details

### Envoy JWT Filter Configuration

```yaml
http_filters:
- name: envoy.filters.http.jwt_authn
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
    providers:
      keycloak:
        issuer: "http://keycloak:8180/realms/demo"
        audiences:
        - "account"
        remote_jwks:
          http_uri:
            uri: "http://keycloak:8180/realms/demo/protocol/openid-connect/certs"
            cluster: keycloak_cluster
            timeout: 5s
          cache_duration:
            seconds: 300
        forward: true
        forward_payload_header: "x-jwt-payload"
        payload_in_metadata: "jwt_payload"
        from_headers:
        - name: "Authorization"
          value_prefix: "Bearer "
    rules:
    - match:
        prefix: "/"
      requires:
        provider_name: "keycloak"
```

**Key Settings**:
- `issuer`: Must match JWT `iss` claim
- `remote_jwks`: JWKS endpoint for signature validation
- `cache_duration`: Cache JWKS for 5 minutes (reduce Keycloak load)
- `forward`: Forward JWT to backend (optional, for debugging)
- `payload_in_metadata`: Store claims for RBAC filter access

### Envoy RBAC Filter Configuration

```yaml
http_filters:
- name: envoy.filters.http.rbac
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC
    rules:
      action: ALLOW
      policies:
        "allow-public":
          permissions:
          - and_rules:
              rules:
              - header:
                  name: ":path"
                  prefix_match: "/public"
          principals:
          - authenticated:
              principal_name:
                safe_regex:
                  google_re2: {}
                  regex: ".*"

        "allow-admin-internal":
          permissions:
          - and_rules:
              rules:
              - header:
                  name: ":path"
                  prefix_match: "/internal"
          principals:
          - and_ids:
              ids:
              - authenticated:
                  principal_name:
                    safe_regex:
                      google_re2: {}
                      regex: ".*"
              - metadata:
                  filter: "envoy.filters.http.jwt_authn"
                  path:
                  - key: "jwt_payload"
                  - key: "realm_access"
                  - key: "roles"
                  value:
                    list_match:
                      one_of:
                        string_match:
                          exact: "admin"
```

**Key Settings**:
- `action: ALLOW`: Whitelist mode (explicit allow rules)
- Policies are evaluated in order
- `authenticated`: Requires JWT validation to pass
- `metadata`: Access JWT claims stored by JWT filter
- Path matching: `realm_access.roles` contains "admin"

### Envoy Access Logging

```yaml
access_log:
- name: envoy.access_loggers.stdout
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
    log_format:
      json_format:
        timestamp: "%START_TIME%"
        method: "%REQ(:METHOD)%"
        path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
        status: "%RESPONSE_CODE%"
        duration_ms: "%DURATION%"
        user: "%DYNAMIC_METADATA(envoy.filters.http.jwt_authn:jwt_payload:preferred_username)%"
        roles: "%DYNAMIC_METADATA(envoy.filters.http.jwt_authn:jwt_payload:realm_access:roles)%"
        rbac_decision: "%DYNAMIC_METADATA(envoy.filters.http.rbac)%"
```

**Output Example**:
```json
{
  "timestamp": "2026-01-21T10:30:00.123Z",
  "method": "GET",
  "path": "/internal",
  "status": 403,
  "duration_ms": 5,
  "user": "alice",
  "roles": ["user"],
  "rbac_decision": "denied"
}
```

---

## Testing Strategy

### Unit Tests (Optional, for demo reliability)

Test individual components in isolation:

1. **Keycloak Token Generation**
   - Test: Can authenticate with alice/password
   - Test: Can authenticate with bob/password
   - Test: Invalid credentials are rejected
   - Test: Token contains expected claims

2. **Sample Apps**
   - Test: Health endpoint returns 200
   - Test: Main endpoint returns expected JSON
   - Test: Headers are properly echoed

### Integration Tests

Test the full stack:

**File**: `tests/test-demo.sh`

```bash
#!/bin/bash

# Test 1: Unauthenticated access (should fail)
echo "Test 1: Unauthenticated access"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/public)
if [ "$STATUS" == "401" ]; then
  echo "✓ PASS: Unauthenticated request blocked"
else
  echo "✗ FAIL: Expected 401, got $STATUS"
fi

# Test 2: Alice can access public
echo "Test 2: Alice accessing public app"
TOKEN_ALICE=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=alice&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $TOKEN_ALICE" \
  http://localhost:8080/public)
if [ "$STATUS" == "200" ]; then
  echo "✓ PASS: Alice can access public app"
else
  echo "✗ FAIL: Expected 200, got $STATUS"
fi

# Test 3: Alice cannot access internal
echo "Test 3: Alice accessing internal app"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $TOKEN_ALICE" \
  http://localhost:8080/internal)
if [ "$STATUS" == "403" ]; then
  echo "✓ PASS: Alice blocked from internal app"
else
  echo "✗ FAIL: Expected 403, got $STATUS"
fi

# Test 4: Bob can access internal
echo "Test 4: Bob accessing internal app"
TOKEN_BOB=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=bob&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $TOKEN_BOB" \
  http://localhost:8080/internal)
if [ "$STATUS" == "200" ]; then
  echo "✓ PASS: Bob can access internal app"
else
  echo "✗ FAIL: Expected 200, got $STATUS"
fi

# Test 5: Access logs contain identity
echo "Test 5: Checking access logs"
if docker-compose logs envoy | grep -q "alice"; then
  echo "✓ PASS: Access logs contain user identity"
else
  echo "✗ FAIL: Access logs missing user identity"
fi
```

**Run tests**:
```bash
chmod +x tests/test-demo.sh
./tests/test-demo.sh
```

### Pre-Demo Validation Checklist

Before the presentation, run this checklist:

```bash
# 1. All services start successfully
docker-compose up -d
docker-compose ps  # All should be "Up"

# 2. Keycloak is ready
curl -f http://localhost:8180/realms/demo

# 3. Envoy is ready
curl -f http://localhost:8080/

# 4. Can generate tokens
curl -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=alice&password=password&grant_type=password&client_id=demo-client"

# 5. Run integration tests
./tests/test-demo.sh

# 6. Check logs are working
docker-compose logs envoy --tail=10
```

---

## Failure Modes & Mitigations

### Failure Mode 1: Keycloak Takes Too Long to Start

**Symptom**: Envoy fails to start or JWT validation fails

**Root Cause**: Keycloak initialization can take 30-60 seconds

**Mitigation**:
1. Add healthcheck to Keycloak service in docker-compose
2. Make Envoy depend on Keycloak healthcheck
3. Start services 10 minutes before demo
4. Pre-validate with test script

**Docker Compose Fix**:
```yaml
keycloak:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8180/realms/demo"]
    interval: 10s
    timeout: 5s
    retries: 30

envoy:
  depends_on:
    keycloak:
      condition: service_healthy
```

### Failure Mode 2: Port Conflicts

**Symptom**: "Port already in use" error during docker-compose up

**Root Cause**: Ports 8080, 8180, 3000, or 3001 already bound

**Mitigation**:
1. Check for conflicts before demo: `lsof -i :8080 -i :8180 -i :3000 -i :3001`
2. Document how to change ports in docker-compose.yml
3. Use less common ports (e.g., 18080, 18180)

**Pre-Demo Check**:
```bash
# Kill any conflicting processes
lsof -ti :8080,:8180,:3000,:3001 | xargs kill -9 2>/dev/null
```

### Failure Mode 3: JWT Validation Fails

**Symptom**: All authenticated requests return 401

**Root Cause**:
- Envoy can't reach Keycloak JWKS endpoint
- Issuer mismatch in JWT vs Envoy config
- Network issues between containers

**Debug Steps**:
1. Check Envoy logs: `docker-compose logs envoy`
2. Verify JWKS endpoint: `curl http://localhost:8180/realms/demo/protocol/openid-connect/certs`
3. Decode JWT and check `iss` claim: `jwt.io`
4. Check container networking: `docker network inspect demo-network`

**Mitigation**:
- Use container hostnames (not localhost) in Envoy config
- Test JWKS fetch from Envoy container: `docker-compose exec envoy curl http://keycloak:8180/...`

### Failure Mode 4: RBAC Always Denies

**Symptom**: All requests return 403, even for Bob

**Root Cause**:
- Role claim path mismatch
- Keycloak sends roles in different claim structure
- RBAC policy syntax error

**Debug Steps**:
1. Decode JWT and inspect exact claim structure
2. Check Envoy dynamic metadata logs
3. Verify RBAC policy matches actual claim path

**Mitigation**:
- Test with actual token from Keycloak (not mock)
- Use Envoy admin interface to inspect metadata: `curl localhost:9901/config_dump`

### Failure Mode 5: Demo Network Issues (Conference WiFi)

**Symptom**: Can't reach localhost, DNS issues, firewall blocks

**Root Cause**: Conference network restrictions

**Mitigation**:
1. **Run everything locally** (no external dependencies)
2. Pre-load all Docker images
3. Use `127.0.0.1` instead of `localhost`
4. Have backup: pre-recorded video or screenshots
5. Test on conference WiFi during setup time

**Pre-Demo Preparation**:
```bash
# Pull all images ahead of time
docker-compose pull

# Save images to tar (backup)
docker save envoyproxy/envoy:v1.28-latest > envoy.tar
docker save quay.io/keycloak/keycloak:23.0 > keycloak.tar
```

### Failure Mode 6: Token Expiration During Demo

**Symptom**: Requests start failing mid-demo (401 after working earlier)

**Root Cause**: JWT tokens expire (5 minute lifespan)

**Mitigation**:
1. Generate fresh tokens immediately before demo
2. Extend token lifespan to 15 minutes in Keycloak
3. Have token generation commands ready to re-run
4. Store tokens in environment variables (easy to regenerate)

**Keycloak Setting**: Realm Settings → Tokens → Access Token Lifespan → 15 minutes

---

## Performance Considerations

### Startup Time

| Component | Startup Time | Optimization |
|-----------|--------------|--------------|
| Keycloak | 30-60s | Pre-import realm, use dev mode |
| Envoy | 2-5s | Minimal config, no LDS/CDS |
| Public App | 1-3s | Alpine base image, no deps |
| Internal App | 1-3s | Alpine base image, no deps |
| **Total** | **~60s** | Start 10 min before demo |

### Resource Usage

| Component | Memory | CPU | Notes |
|-----------|--------|-----|-------|
| Keycloak | ~500MB | Low | Java, largest consumer |
| Envoy | ~50MB | Low | C++, very efficient |
| Public App | ~30MB | Low | Node/Python, minimal |
| Internal App | ~30MB | Low | Node/Python, minimal |
| **Total** | **~600MB** | **Minimal** | Runs on any laptop |

### Request Latency

| Operation | Latency | Notes |
|-----------|---------|-------|
| JWT validation (cached JWKS) | <5ms | Signature check |
| RBAC evaluation | <1ms | In-memory policy check |
| Backend request | <10ms | Local container network |
| **Total per request** | **<20ms** | Acceptable for demo |

---

## Success Criteria

The demo is successful if it demonstrates:

1. ✅ **Authentication**: Unauthenticated requests are blocked
2. ✅ **Identity Awareness**: Service knows who the user is
3. ✅ **Authorization**: Different users have different access
4. ✅ **Least Privilege**: Regular users can't access admin services
5. ✅ **Audit Trail**: All access is logged with identity context
6. ✅ **Zero Trust**: Every request is validated, no implicit trust

**Measurable Outcomes**:
- 401 for requests without JWT
- 403 for alice accessing /internal
- 200 for bob accessing /internal
- Access logs show username and roles for all requests

---

## Next Steps

To implement this design:

1. ✅ Create directory structure
2. ✅ Write docker-compose.yml
3. ✅ Configure Keycloak (realm export)
4. ✅ Configure Envoy (envoy.yaml)
5. ✅ Build sample applications
6. ✅ Create demo automation script
7. ✅ Write integration tests
8. ✅ Test end-to-end
9. ✅ Document troubleshooting steps
10. ✅ Practice demo delivery

---

## Appendix: Alternative Implementations

### Alternative 1: Kubernetes-Native

Instead of Docker Compose, deploy to Kubernetes:
- Keycloak: StatefulSet
- Envoy: Deployment with Ingress
- Apps: Deployments with Services

**Pros**: More realistic production setup
**Cons**: Requires K8s cluster, more complex, slower startup

### Alternative 2: Service Mesh (Istio)

Use Istio instead of standalone Envoy:
- Istio control plane
- Envoy sidecars
- RequestAuthentication and AuthorizationPolicy CRDs

**Pros**: Demonstrates service mesh capabilities
**Cons**: Complex setup, harder to debug, longer startup

### Alternative 3: Cloud-Hosted Identity

Use Auth0, Okta, or Azure AD instead of Keycloak:

**Pros**: More realistic for production
**Cons**: Requires internet, external dependency, not self-contained

### Alternative 4: Different Languages

Use Go or Python instead of Node.js:

**Go**:
```go
package main

import (
    "encoding/json"
    "net/http"
    "os"
    "time"
)

type Response struct {
    Service   string    `json:"service"`
    Message   string    `json:"message"`
    User      string    `json:"authenticated_user"`
    Timestamp time.Time `json:"timestamp"`
}

func handler(w http.ResponseWriter, r *http.Request) {
    resp := Response{
        Service:   os.Getenv("SERVICE_NAME"),
        Message:   "Welcome",
        User:      r.Header.Get("x-forwarded-user"),
        Timestamp: time.Now(),
    }
    json.NewEncoder(w).Encode(resp)
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":3000", nil)
}
```

**Pros**: Single binary, fast startup, small image
**Cons**: Slightly more verbose than Node.js

---

## Conclusion

This design provides a reliable, self-contained demo that:
- Runs entirely on a laptop (no cloud dependencies)
- Starts in ~60 seconds
- Demonstrates all key concepts clearly
- Has minimal failure modes
- Is easy to debug and recover from issues

The architecture is simple enough for a 10-minute demo but sophisticated enough to illustrate production-grade security patterns.

**Estimated Implementation Time**: 4-6 hours for a complete working demo with tests and documentation.
