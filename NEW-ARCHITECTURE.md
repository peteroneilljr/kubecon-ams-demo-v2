# Updated Demo Architecture: Per-User Access Control

## What Changed

The demo has been upgraded from role-based access (admin vs user) to **per-user identity-based access control**, making it much more compelling for your talk!

## New Architecture

### Services

| Service | Port | Who Can Access | Purpose |
|---------|------|----------------|---------|
| **public-app** | 3000 | Anyone authenticated | Shared resource |
| **alice-app** | 3002 | Only Alice | Alice's private workspace |
| **bob-app** | 3001 | Only Bob | Bob's private workspace |

### Access Control Matrix

```
┌─────────────┬──────────┬────────────┬──────────┐
│   User      │ /public  │  /alice    │  /bob    │
├─────────────┼──────────┼────────────┼──────────┤
│ No Auth     │    401   │    401     │   401    │
│ Alice       │    200   │    200 ✓   │   403 ✗  │
│ Bob         │    200   │    403 ✗   │   200 ✓  │
└─────────────┴──────────┴────────────┴──────────┘
```

## Why This Is Better for Your Talk

### Old Demo (Role-Based)
- Alice (user role) → can access public, NOT internal
- Bob (admin role) → can access public AND internal
- Message: "Admins get more access"

### New Demo (Identity-Based) ⭐
- Alice → can access public + HER app, NOT Bob's app
- Bob → can access public + HIS app, NOT Alice's app
- Message: **"Each user gets only THEIR resources"**

## Key Demo Moments

### 🎯 The "Aha" Moment #1
**Alice tries to access Bob's app:**
```bash
$ curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/bob
403 Forbidden - RBAC: access denied
```
- She's **authenticated** (has valid token)
- She's **not authorized** (not Bob!)
- This shows **per-user access control**

### 🎯 The "Aha" Moment #2
**Bob tries to access Alice's app:**
```bash
$ curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/alice
403 Forbidden - RBAC: access denied
```
- He's **authenticated** (has valid token)
- He's **not authorized** (not Alice!)
- Even though Bob is an "admin", he **can't access Alice's resources**

This demonstrates true **least-privilege access** and **zero trust**!

## Envoy Configuration

### Authentication (JWT Validation)
```yaml
# Same as before - validates JWT signature and extracts claims
jwt_authn:
  issuer: "http://localhost:8180/realms/demo"
  payload_in_metadata: "jwt_payload"
```

### Authorization (Per-User RBAC)
```yaml
# NEW: Username-based authorization
rbac:
  policies:
    "allow-alice-only":
      permissions:
        - path: "/alice"
      principals:
        - metadata:
            filter: "envoy.filters.http.jwt_authn"
            path:
            - key: "jwt_payload"
            - key: "preferred_username"
            value:
              string_match:
                exact: "alice"

    "allow-bob-only":
      permissions:
        - path: "/bob"
      principals:
        - metadata:
            filter: "envoy.filters.http.jwt_authn"
            path:
            - key: "jwt_payload"
            - key: "preferred_username"
            value:
              string_match:
                exact: "bob"
```

## Demo Flow (Enhanced)

### Part 1: Setup (2 min)
- Show VPN problem (slides)
- Explain the demo architecture

### Part 2: Alice's Journey (3 min)
1. Alice authenticates → gets JWT
2. Alice accesses public app ✓
3. Alice accesses HER app (/alice) ✓
4. **Alice tries Bob's app (/bob) → DENIED ✗** ⭐ KEY MOMENT

### Part 3: Bob's Journey (3 min)
1. Bob authenticates → gets JWT
2. Bob accesses public app ✓
3. **Bob tries Alice's app (/alice) → DENIED ✗** ⭐ KEY MOMENT
4. Bob accesses HIS app (/bob) ✓

### Part 4: Audit Trail (1 min)
- Show logs with user identity and decisions
- Every 403 is logged

### Part 5: Wrap-up (1 min)
- Compare to VPN (network-level access = see everything)
- Reverse proxy = identity-aware, per-resource authorization

## Talk Points to Emphasize

### 1. Identity Matters
> "Notice Alice and Bob both have valid tokens - they're both authenticated. But Alice can't access Bob's resources, and Bob can't access Alice's resources. The proxy enforces **identity-based access** at every request."

### 2. Zero Trust in Action
> "Even though Bob is marked as an 'admin' in the system, he's **still blocked** from Alice's app. There's no implicit trust - every request is validated against that specific resource."

### 3. Least Privilege
> "Each user gets exactly what they need - nothing more, nothing less. Alice doesn't need Bob's resources, so she doesn't get them. This is **least-privilege access** in practice."

### 4. Compare to VPN
> "With a VPN, once Alice is 'inside' the network, she could potentially reach Bob's services. With a reverse proxy, every single request is checked against the user's identity."

## Access Logs Show Everything

```json
// Alice accessing her own app - ALLOWED
{"user":"alice","path":"/alice","status":200}

// Alice trying Bob's app - DENIED
{"user":"alice","path":"/bob","status":403}

// Bob trying Alice's app - DENIED
{"user":"bob","path":"/alice","status":403}

// Bob accessing his own app - ALLOWED
{"user":"bob","path":"/bob","status":200}
```

Every authorization decision is logged with full context!

## Real-World Analogies

### VPN Model
"It's like giving everyone who enters the office building a master key. Once inside, you can open any door."

### Reverse Proxy Model
"It's like a smart lock system where each person's badge only opens their assigned rooms, even though everyone can enter the building."

## Technical Benefits Shown

1. ✅ **Fine-grained authorization**: Per-user, per-resource
2. ✅ **Identity-aware**: System knows exactly who is accessing what
3. ✅ **Zero trust**: Never trust, always verify (even for "admins")
4. ✅ **Complete audit**: Every decision logged
5. ✅ **Scalable**: Add new users/resources without changing network topology

## Running the Demo

```bash
# Start services
docker-compose up -d

# Run automated demo
./demo-script.sh

# Or manual testing
TOKEN_ALICE=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=alice&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')

# Alice can access her app
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/alice

# Alice CANNOT access Bob's app (403)
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/bob
```

## What Makes This Demo Compelling

1. **Clear Access Denial**: Seeing "403 Forbidden" for cross-user access makes the point visceral
2. **Symmetry**: Both users have the same restriction pattern
3. **Admin Not Special**: Even Bob (admin) can't access Alice's resources
4. **Real-World Relatable**: Everyone understands "my files vs your files"
5. **Visual Impact**: The demo shows actual requests being blocked

## Diagram for Slides

```
VPN Model:
┌──────────────────────────────────┐
│       Inside VPN Network         │
│  Alice & Bob can access:         │
│  • All services                  │
│  • Each other's data             │
│  • Internal systems              │
└──────────────────────────────────┘
                ❌ TOO MUCH ACCESS

Reverse Proxy Model:
┌─────────────────────────────────┐
│      Envoy validates every      │
│      request by identity        │
├─────────────────────────────────┤
│ Alice → /alice     ✓ Allowed    │
│ Alice → /bob       ✗ Denied     │
│ Bob → /bob         ✓ Allowed    │
│ Bob → /alice       ✗ Denied     │
└─────────────────────────────────┘
          ✓ LEAST PRIVILEGE
```

## Success Metrics

After your demo, the audience will understand:
1. Why network-level access (VPN) is too coarse
2. How identity-based access works at the proxy layer
3. What zero trust looks like in practice
4. How to implement this with Envoy + JWT

---

**This is a much more powerful demo!** Instead of "users vs admins", you're showing "this user vs that user" - which is more relatable and clearer for demonstrating least-privilege access.

Good luck with your talk! 🎤🚀
