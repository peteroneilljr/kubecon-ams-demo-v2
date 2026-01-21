# Quick Start Guide

Get the demo running in 5 minutes.

## Prerequisites

Install these tools if you don't have them:

```bash
# macOS
brew install docker docker-compose jq

# Verify installation
docker --version
docker-compose --version
jq --version
```

## Step 1: Start Services

```bash
# Start all services
docker-compose up -d

# This will:
# - Start Keycloak (takes ~60 seconds to be ready)
# - Start Envoy proxy
# - Start public and internal apps
```

## Step 2: Wait for Services

```bash
# Check status (wait until all show "Up")
docker-compose ps

# Check Keycloak is ready
curl http://localhost:8180/realms/demo

# If you get a response, you're ready!
```

## Step 3: Run the Demo

### Option A: Automated Demo Script

```bash
# Run the full demo with nice formatting
./demo-script.sh
```

This will walk through all the scenarios with colored output.

### Option B: Manual Testing

```bash
# 1. Try unauthenticated (should fail with 401)
curl -i http://localhost:8080/public

# 2. Get token for Alice (regular user)
TOKEN_ALICE=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=alice&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')

# 3. Alice can access public
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/public | jq

# 4. Alice CANNOT access internal (should get 403)
curl -i -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/internal

# 5. Get token for Bob (admin user)
TOKEN_BOB=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=bob&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')

# 6. Bob CAN access internal
curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/internal | jq

# 7. View access logs
docker-compose logs envoy --tail=20
```

## Step 4: Run Integration Tests

```bash
# Verify everything works
./tests/test-demo.sh
```

Should show all tests passing.

## Quick Demo (10 minutes)

For your KubeCon presentation, follow these steps:

1. **Start services** 10 minutes before your talk
2. **Open terminal** with large font (24pt+)
3. **Run demo script**: `./demo-script.sh`
4. **Key points to emphasize**:
   - Unauthenticated requests blocked
   - Alice can access public but not internal
   - Bob (admin) can access both
   - All access logged with identity

## Troubleshooting

### Services won't start
```bash
docker-compose down
docker-compose up -d
```

### Keycloak taking too long
```bash
# Check logs
docker-compose logs keycloak

# Keycloak needs 30-60 seconds to start
# Wait and retry
```

### Port conflicts
```bash
# Check what's using ports
lsof -i :8080 -i :8180

# Kill conflicting processes
lsof -ti :8080,:8180 | xargs kill -9
```

### Token issues
```bash
# Regenerate token
TOKEN_ALICE=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -d "username=alice&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')
```

## Cleanup

```bash
# Stop all services
docker-compose down

# Remove all data (reset everything)
docker-compose down -v
```

## Architecture Summary

```
User → Envoy (validates JWT + checks roles) → Backend Apps
       ↑
       Keycloak (issues JWT tokens)
```

**Key Principle**: Every request is authenticated and authorized at the edge, before reaching backend services.

## Users & Passwords

| Username | Password | Roles | Access |
|----------|----------|-------|--------|
| alice | password | user | Public only |
| bob | password | user, admin | Public + Internal |

## Endpoints

- Keycloak: http://localhost:8180
- Envoy Proxy: http://localhost:8080
- Public App (via Envoy): http://localhost:8080/public
- Internal App (via Envoy): http://localhost:8080/internal

## Next Steps

- Review [DESIGN.md](DESIGN.md) for technical architecture
- Review [DEMO-SCRIPT.md](DEMO-SCRIPT.md) for presentation flow
- Practice the demo multiple times
- Test on the actual presentation machine

Good luck with your talk!
