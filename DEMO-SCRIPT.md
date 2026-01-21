# KubeCon Demo Script: Reverse Proxy vs VPN

This is your presentation script for the live demo. Practice this flow to deliver a smooth, impactful demonstration.

## Pre-Demo Setup (Before Your Talk)

```bash
# Start all services 10 minutes before your talk
docker-compose up -d

# Verify everything is running
docker-compose ps

# Pre-load tokens to avoid typing during talk (optional)
export TOKEN_ALICE=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=alice&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')

export TOKEN_BOB=$(curl -s -X POST "http://localhost:8180/realms/demo/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=bob&password=password&grant_type=password&client_id=demo-client" \
  | jq -r '.access_token')

# Open your terminal with large font (24pt+)
# Have browser tab ready: http://localhost:8180/admin (Keycloak admin console)
```

---

## Demo Flow (8-10 minutes)

### Part 1: The VPN Problem (2 minutes - Slides Only)

**[SLIDE: VPN Architecture Diagram]**

> "Let me show you why VPNs are like that shared storage room I mentioned. Here's a typical VPN setup."

**[Point to diagram showing user inside network perimeter]**

> "Once Alice connects to the VPN, she's inside the network perimeter. She can reach the public website, the internal admin API, the database, monitoring tools—everything. There's no per-service authorization. The VPN just asks: 'Are you authenticated?' After that, you're trusted everywhere."

**[SLIDE: VPN Problems List]**

> "This creates three major problems:
> 1. Excessive privilege - Alice might only need the public app, but she gets access to everything
> 2. Lateral movement - If Alice's laptop is compromised, attackers can reach internal systems
> 3. Limited visibility - VPN logs show 'Alice connected' but not what she accessed or did"

> "Now let me show you how a reverse proxy solves these problems."

---

### Part 2: Live Demo - Architecture (1 minute)

**[SLIDE: Reverse Proxy Architecture Diagram]**

> "In this demo, we have:
> - Envoy as our reverse proxy
> - Keycloak for identity management
> - Two sample apps: a public service and an internal admin service
>
> The key difference: instead of network access, users get identity tokens. Envoy validates every request and enforces per-route authorization."

**[Switch to Terminal]**

> "Let me show you this in action."

---

### Part 3: Unauthenticated Access (1 minute)

**[TERMINAL - Large font, clear commands]**

```bash
# Show that services are running
echo "First, our services are running..."
docker-compose ps
```

**[Wait for output]**

```bash
# Try to access without authentication
echo "Let's try to access the public app without authentication..."
curl -i http://localhost:8080/public
```

> "See? 401 Unauthorized. Envoy requires a valid JWT token for every request. No token, no access—even to 'public' services."

```bash
echo "Same for the internal app..."
curl -i http://localhost:8080/internal
```

> "Also blocked. This is the first layer: authentication. Every request must have a valid identity."

---

### Part 4: Regular User Access (2 minutes)

```bash
# Authenticate as Alice
echo "Now let's authenticate as Alice, a regular user..."
```

**[If pre-loaded tokens, show the token]**
```bash
echo $TOKEN_ALICE | cut -d'.' -f2 | base64 -d 2>/dev/null | jq .
```

> "This is Alice's JWT token. Notice the claims: her username is 'alice' and she has the role 'user'. This token is signed by Keycloak, so Envoy trusts it."

```bash
# Access public app
echo "Alice can access the public app..."
curl -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/public
```

**[Point to response]**

> "Success! The public app sees Alice's request. Notice it knows who she is—identity-aware access."

```bash
# Try to access internal app
echo "But watch what happens when Alice tries to access the internal admin API..."
curl -i -H "Authorization: Bearer $TOKEN_ALICE" http://localhost:8080/internal
```

**[Pause for effect]**

> "403 Forbidden. Alice is authenticated—she has a valid token—but she's not authorized for this route. Envoy checked her role claims and enforced the policy: only admins can access /internal."

> "This is least-privilege access. Alice gets exactly what she needs, nothing more."

---

### Part 5: Admin User Access (2 minutes)

```bash
# Authenticate as Bob
echo "Now let's see Bob, an admin user..."
echo $TOKEN_BOB | cut -d'.' -f2 | base64 -d 2>/dev/null | jq .
```

> "Bob's token includes the 'admin' role. Let's see the difference."

```bash
# Access public app
echo "Bob can access the public app..."
curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/public
```

> "That works, as expected."

```bash
# Access internal app
echo "And Bob CAN access the internal app..."
curl -H "Authorization: Bearer $TOKEN_BOB" http://localhost:8080/internal
```

> "Success! Bob's admin role satisfies the authorization policy for this route. Same proxy, same service, different outcome based on identity."

---

### Part 6: Audit Trail (1-2 minutes)

```bash
# Show access logs
echo "Finally, let's look at the audit trail..."
docker-compose logs envoy --tail=20 | grep "access_log"
```

**[Point to log entries]**

> "Every single request is logged with:
> - Timestamp
> - User identity (from the JWT)
> - Route accessed
> - Authorization decision
> - Response code
>
> Compare this to VPN logs that just say 'Alice connected.' Here we know exactly what Alice accessed, when, and whether she was authorized."

**[Optional: Show in Keycloak admin console]**

> "You can also see active sessions, token claims, and audit events in Keycloak."

---

### Part 7: Wrap-Up (1 minute)

**[Switch back to slides]**

**[SLIDE: Comparison Table - VPN vs Reverse Proxy]**

> "So let's recap what we just saw:
>
> **VPNs**: Network-level trust, everything accessible, limited audit
>
> **Reverse Proxies**: Identity-aware, per-route authorization, complete audit trail
>
> This is why reverse proxies are becoming the standard for cloud-native security. They align with zero-trust principles: never trust, always verify."

---

## Demo Variations & Backup Plans

### If Token Generation Fails
- Have pre-generated tokens in a file
- Show token validation directly with jwt.io

### If Envoy Crashes
- Have screenshots/video backup
- Walk through the configuration files instead

### If Time is Short
- Skip showing both users
- Show one successful access and one failed authorization

### If You Have Extra Time
- Decode JWT token step-by-step to show claims
- Demonstrate token expiration (if tokens are set to expire quickly)
- Show Envoy configuration file and explain filters

---

## Talking Points to Emphasize

1. **Identity at Every Request**: Unlike VPNs where identity is checked once at connection, reverse proxies validate identity on every single request

2. **Per-Route Policies**: You can have different authorization rules for different endpoints, even within the same service

3. **No Trust Inside the Perimeter**: Even after authentication, users must be authorized for each specific resource

4. **Observable Security**: Every access decision is logged and auditable

5. **Cloud-Native Fit**: This model works across clusters, clouds, and hybrid environments without complex network tunnels

---

## Audience Questions to Anticipate

**Q: How is this different from API Gateway?**
> "Great question! This IS an API gateway pattern. Envoy acts as an API gateway with security enforcement. The key is that it's identity-aware and enforces authorization at the edge, before requests reach your services."

**Q: What about performance?**
> "JWT validation is fast—microseconds. The overhead is minimal compared to network latency. Plus, you can cache validations in Envoy for even better performance."

**Q: Can you use this with Kubernetes?**
> "Absolutely! Envoy is the foundation of many Kubernetes ingress controllers and service meshes like Istio. This same pattern scales from local development to production clusters."

**Q: What about VPN for remote work?**
> "Good clarification—I'm not saying VPNs have no place. For general network connectivity, they're fine. But for application access, especially internal tools and APIs, reverse proxies give you much better security and visibility."

**Q: How do you handle token refresh?**
> "Keycloak supports refresh tokens. When the access token expires, your client can use the refresh token to get a new one without re-authenticating. This balances security (short-lived tokens) with user experience."

---

## Post-Demo

```bash
# Clean shutdown (optional, if reusing laptop)
docker-compose down
```

**[SLIDE: Resources]**

> "All the code for this demo is on GitHub. You can run it yourself and extend it. Thank you!"

---

## Timing Breakdown

- VPN Problem (slides): 2 min
- Architecture overview: 1 min
- Unauthenticated access: 1 min
- Regular user demo: 2 min
- Admin user demo: 2 min
- Audit trail: 1 min
- Wrap-up: 1 min

**Total: 10 minutes** (leaves buffer for Q&A or issues)

---

## Confidence Boosters

- The demo is simple and forgiving
- If something fails, you can explain the concept with slides
- The core message is clear even without the demo
- You've tested this multiple times
- The audience is rooting for you

**You've got this!** 🎤
