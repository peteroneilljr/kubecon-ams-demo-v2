# Follow Along Guide: Reverse Proxy Demo

This guide allows you to follow along with the demo step-by-step. Each command can be copy-pasted into your terminal.

## Prerequisites

Before starting, ensure you have:
- Docker and Docker Compose installed
- `curl` command-line tool
- `jq` for JSON parsing (`brew install jq` on Mac, `apt-get install jq` on Ubuntu)
- Terminal access

## Setup (5 minutes)

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd kubecon-ams-demo-v2
```

### Step 2: Start All Services

```bash
# Start services in detached mode
docker-compose up -d

# This will start:
# - Keycloak (identity provider) on port 8180
# - Envoy (reverse proxy) on port 8080
# - public-app on port 3000 (internal)
# - alice-app on port 3002 (internal)
# - bob-app on port 3001 (internal)
```

### Step 3: Wait for Services to Be Ready

```bash
# Check service status
docker-compose ps

# Wait until all services show "healthy" or "running"
# This typically takes 30-60 seconds
```

### Step 4: Verify Keycloak is Ready

```bash
# Test Keycloak endpoint
curl -s http://localhost:8180/realms/demo/.well-known/openid-configuration | jq -r '.issuer'

# Expected output: http://localhost:8180/realms/demo
```

---

## Part 1: Unauthenticated Access (FAILS)

### Step 5: Try Accessing Without Authentication

```bash
# Try to access public app without token
curl -i http://localhost:8080/public

# Expected: 401 Unauthorized
# Message: "Jwt is missing"
```

```bash
# Try to access Alice's app without token
curl -i http://localhost:8080/alice

# Expected: 401 Unauthorized
```

```bash
# Try to access Bob's app without token
curl -i http://localhost:8080/bob

# Expected: 401 Unauthorized
```

**Key Point**: Envoy's JWT authentication filter blocks all unauthenticated requests.

---

## Part 2: Alice's Journey

### Step 6: Authenticate as Alice

```bash
# Get JWT token for Alice
TOKEN_ALICE=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=alice" \
  -d "password=password" \
  -d "grant_type=password" \
  -d "client_id=demo-client" \
  | jq -r '.access_token')

# Verify token was obtained
echo "Alice's token (first 50 chars): ${TOKEN_ALICE:0:50}..."
```

### Step 7: Decode Alice's JWT (Optional)

```bash
# Extract and decode JWT payload
PAYLOAD=$(echo "$TOKEN_ALICE" | cut -d'.' -f2)

# Add padding for base64 decode
case $((${#PAYLOAD} % 4)) in
  2) PAYLOAD="${PAYLOAD}==" ;;
  3) PAYLOAD="${PAYLOAD}=" ;;
esac

# View JWT claims
echo "$PAYLOAD" | base64 -d 2>/dev/null | jq '{
  username: .preferred_username,
  email: .email,
  roles: .realm_access.roles,
  expires: .exp
}'
```

**Expected Output**:
```json
{
  "username": "alice",
  "email": "alice@demo.local",
  "roles": ["user"],
  "expires": 1234567890
}
```

### Step 8: Alice Accesses Public App ✓

```bash
# Alice can access public app
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/public | jq '.'

# Expected: 200 OK
# Shows: "Welcome to the Public Service!"
```

### Step 9: Alice Accesses Her Own App ✓

```bash
# Alice can access her own app
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/alice | jq '.'

# Expected: 200 OK
# Shows: "Welcome to Alice's personal service!"
```

**Key Point**: Envoy's RBAC filter checks that `preferred_username == "alice"` for `/alice` route.

### Step 10: Alice Tries to Access Bob's App ✗

```bash
# Alice CANNOT access Bob's app
curl -i -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/bob

# Expected: 403 Forbidden
# Message: "RBAC: access denied"
```

**🎯 KEY MOMENT**: Alice is authenticated (has valid token) but NOT authorized to access Bob's resources!

---

## Part 3: Bob's Journey

### Step 11: Authenticate as Bob

```bash
# Get JWT token for Bob
TOKEN_BOB=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=bob" \
  -d "password=password" \
  -d "grant_type=password" \
  -d "client_id=demo-client" \
  | jq -r '.access_token')

# Verify token was obtained
echo "Bob's token (first 50 chars): ${TOKEN_BOB:0:50}..."
```

### Step 12: Decode Bob's JWT (Optional)

```bash
# Extract and decode JWT payload
PAYLOAD=$(echo "$TOKEN_BOB" | cut -d'.' -f2)

# Add padding for base64 decode
case $((${#PAYLOAD} % 4)) in
  2) PAYLOAD="${PAYLOAD}==" ;;
  3) PAYLOAD="${PAYLOAD}=" ;;
esac

# View JWT claims
echo "$PAYLOAD" | base64 -d 2>/dev/null | jq '{
  username: .preferred_username,
  email: .email,
  roles: .realm_access.roles,
  expires: .exp
}'
```

**Expected Output**:
```json
{
  "username": "bob",
  "email": "bob@demo.local",
  "roles": ["user", "admin"],
  "expires": 1234567890
}
```

**Note**: Bob has "admin" role, but this won't help him access Alice's app!

### Step 13: Bob Accesses Public App ✓

```bash
# Bob can access public app
curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/public | jq '.'

# Expected: 200 OK
# Shows: "Welcome to the Public Service!"
```

### Step 14: Bob Tries to Access Alice's App ✗

```bash
# Bob CANNOT access Alice's app (even though he's an admin!)
curl -i -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/alice

# Expected: 403 Forbidden
# Message: "RBAC: access denied"
```

**🎯 KEY MOMENT**: Even though Bob has "admin" role, he's denied access to Alice's resources. This demonstrates true zero-trust per-user authorization!

### Step 15: Bob Accesses His Own App ✓

```bash
# Bob can access his own app
curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/bob | jq '.'

# Expected: 200 OK
# Shows: "Welcome to Bob's personal service!"
```

---

## Part 4: Audit Trail

### Step 16: View Access Logs

```bash
# View Envoy access logs
docker-compose logs envoy | tail -20

# Look for JSON log entries showing:
# - user: "alice" or "bob"
# - path: "/public", "/alice", "/bob"
# - status: 200 (allowed) or 403 (denied)
```

**Example Log Entry**:
```json
{
  "timestamp": "2024-01-20T10:30:45.123Z",
  "method": "GET",
  "path": "/alice",
  "status": 403,
  "duration_ms": 5,
  "user": "bob",
  "roles": ["user", "admin"],
  "response_flags": "RBAC_ACCESS_DENIED"
}
```

### Step 17: Search for Denied Access Attempts

```bash
# Find all 403 Forbidden responses
docker-compose logs envoy | grep '"status":403'

# This shows all unauthorized access attempts
# Great for security monitoring!
```

---

## Part 5: Understanding the Architecture

### Step 18: Inspect JWT Token Structure

```bash
# View full JWT structure
echo "$TOKEN_ALICE" | cut -d'.' -f1 | base64 -d 2>/dev/null | jq '.'  # Header
echo "$TOKEN_ALICE" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq '.'  # Payload
# Signature is the third part (binary, not human-readable)
```

**JWT Structure**:
- **Header**: Algorithm (RS256), token type (JWT)
- **Payload**: User claims (username, email, roles, expiration)
- **Signature**: Cryptographic signature (prevents tampering)

### Step 19: View Keycloak's Public Keys

```bash
# See the public keys Envoy uses to verify JWT signatures
curl -s http://localhost:8180/realms/demo/protocol/openid-connect/certs | jq '.'

# These keys prove the token was issued by Keycloak
```

### Step 20: Check Envoy Admin Interface

```bash
# View Envoy's runtime configuration
curl -s http://localhost:9901/config_dump | jq '.configs[] | select(.["@type"] | contains("jwt_authn"))'

# This shows how Envoy is configured for JWT authentication
```

---

## Experiments to Try

### Experiment 1: Token Expiration

```bash
# Wait 5 minutes for token to expire, then try again
sleep 300

# Try with expired token
curl -i -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/alice

# Expected: 401 Unauthorized (token expired)
```

### Experiment 2: Modified Token (Attack Simulation)

```bash
# Try to modify the token (simulate attack)
FAKE_TOKEN="${TOKEN_ALICE:0:50}HACKED${TOKEN_ALICE:56}"

curl -i -H "Authorization: Bearer $FAKE_TOKEN" http://localhost:8080/alice

# Expected: 401 Unauthorized
# Reason: Signature verification fails
```

### Experiment 3: View All Access Patterns

```bash
# Create a summary of all access attempts
echo "=== Alice's Access Pattern ==="
echo "Public app:  $(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/public)"
echo "Alice's app: $(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/alice)"
echo "Bob's app:   $(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/bob)"

echo ""
echo "=== Bob's Access Pattern ==="
echo "Public app:  $(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/public)"
echo "Alice's app: $(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/alice)"
echo "Bob's app:   $(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/bob)"
```

**Expected Output**:
```
=== Alice's Access Pattern ===
Public app:  200
Alice's app: 200
Bob's app:   403

=== Bob's Access Pattern ===
Public app:  200
Alice's app: 403
Bob's app:   200
```

---

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

---

## Key Takeaways

### 1. Authentication vs Authorization
- **Authentication** (401): Proves who you are (JWT validation)
- **Authorization** (403): Proves what you can access (RBAC policies)

### 2. Zero Trust in Action
- Every request is validated
- Even "admin" users don't get universal access
- Identity checked at the proxy layer

### 3. Least Privilege
- Alice gets exactly Alice's resources
- Bob gets exactly Bob's resources
- No lateral movement possible

### 4. Compare to VPN
- **VPN**: Network-level access → Alice and Bob can reach everything
- **Reverse Proxy**: Resource-level access → Alice and Bob get only their resources

### 5. Complete Audit Trail
- Every request logged with user identity
- Security teams can see who accessed what
- Compliance requirements satisfied

---

## Cleanup

### When You're Done

```bash
# Stop all services
docker-compose down

# Remove volumes (reset Keycloak data)
docker-compose down -v

# Remove built images (optional)
docker-compose down --rmi local
```

---

## Troubleshooting

### Problem: Services won't start

```bash
# Check logs
docker-compose logs

# Restart everything
docker-compose down
docker-compose up -d
```

### Problem: "Jwt is missing" error

```bash
# Make sure you included the Authorization header
# Check token is not empty
echo "Token length: ${#TOKEN_ALICE}"

# If empty, re-authenticate
TOKEN_ALICE=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=alice&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')
```

### Problem: 403 Forbidden when you expect 200

This is expected behavior! Check:
- Alice cannot access `/bob` → 403 is correct
- Bob cannot access `/alice` → 403 is correct
- Only deny `/public` access would be unexpected (should be 200 for both)

### Problem: jq command not found

```bash
# Install jq
# macOS
brew install jq

# Ubuntu/Debian
sudo apt-get install jq

# Or view raw JSON without jq
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/public
```

---

## Next Steps

1. Read [README.md](./README.md) for detailed architecture explanation
2. Review [NEW-ARCHITECTURE.md](./NEW-ARCHITECTURE.md) for design decisions
3. Examine [envoy/envoy.yaml](./envoy/envoy.yaml) to see RBAC policies
4. Check [keycloak/realm-export.json](./keycloak/realm-export.json) for user setup

---

## Automated Demo Script

If you prefer an automated version:

```bash
# Run the full demo automatically
./demo-script.sh

# This will execute all steps with explanations
```

---

**End of Follow-Along Guide** 🎉

You now understand how reverse proxies provide identity-aware, least-privilege access control!
