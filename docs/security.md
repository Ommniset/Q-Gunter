# Security Model — Q-Gunter

Q-Gunter runs offensive tooling. This makes its own security doubly critical: a compromised platform could be weaponized to attack third parties. Security is implemented in layers across every component.

---

## Layers

### 1. Perimeter — Cloudflare WAF

Custom ruleset covering:
- **Rate limiting** on `/login`, `/contact`, `/api/*`: max 10 req/10s per IP, managed challenge on breach
- **Offensive tool blocking**: requests with User-Agent signatures of sqlmap, nmap, nikto, dirb, w3af are blocked at the perimeter
- **Tor exit node blocking**: `T1` country code blocked to mitigate anonymized attack vectors
- **Sensitive path protection**: `.env`, `.git`, `/wp-admin` and similar paths blocked perimeter-side
- **HTTP method sanitization**: only standard methods allowed
- **WebSocket bypass**: Bot Challenge skipped for `/ws/*` to allow CLI WebSocket upgrades

### 2. Network segmentation

Three networks with no cross-contamination:
- Database nodes have zero internet access (CEC2 only)
- Backend is the sole CEC1↔CEC2 bridge
- All SSH restricted to Tailscale VPN range (`100.64.0.0/10`)
- Default-deny (`iptables -P INPUT DROP`) on all nodes

### 3. Remote access — Tailscale + SSH hardening

- Password authentication disabled on all nodes (`PasswordAuthentication no`)
- Key-only authentication (RSA 4096-bit or Ed25519)
- Root login disabled (`PermitRootLogin no`)
- SSH port blocked on physical interfaces; open only on Tailscale virtual interface
- Fail2Ban active on monitoring node: 5 failed attempts within 10 minutes → 1-hour IP ban

### 4. Application security

- **Authentication:** JWT (asymmetrically signed) for frontend; `SERVICE_TOKEN` header for internal service-to-service communication
- **Input validation:** Pydantic schemas on all POST/PUT endpoints; parametrized queries (no string concatenation in SQL)
- **Scope validation:** engagement targets validated via regex in Pydantic models before reaching the orchestrator
- **Token expiry:** frontend JWT tokens have a strict TTL; expired tokens return HTTP 401 immediately

### 5. Client isolation

Each client's AI agent runs in a dedicated Docker container. The orchestrator WebSocket router validates the API key on the initial handshake and establishes a proxy exclusively to that client's container. No container can interact with another client's container or directly access the database or MinIO.

---

## Authorization model

Q-Gunter is an offensive tool. The authorization chain is:

```
1. Manual review of access request (use_case field evaluated by team)
2. Client signs Rules of Engagement (ROE) before account activation
3. CLI wizard re-confirms ROE on every engagement launch
4. Scope (in/out) injected directly into AI agent system prompt
5. Agent has hard-coded ethical limits that cannot be overridden by scope config
```

The client is legally responsible for ensuring all targets are authorized. The ROE, combined with the documented use_case at registration, forms the contractual and ethical basis for each engagement.

---

## Known technical debt (documented for transparency)

| Issue | Current state | Planned fix |
|-------|--------------|-------------|
| Anthropic API keys in DB | Stored in plaintext | AES-256 encryption at rest |
| Internal traffic | Plaintext on CEC2 | mTLS with internal CA |
| Secrets management | `.env` files | HashiCorp Vault or equivalent |
| SAST/DAST pipeline | Not implemented | SonarQube / Snyk integration in CI/CD |
| API key format validation | No format check | Validation + log masking |

These are documented in the project memory and are prioritized for a future commercial release.
