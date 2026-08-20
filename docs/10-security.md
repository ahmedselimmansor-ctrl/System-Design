# 10 — Security

*Identity, authorisation, secrets, zero trust, the OWASP classes, and multi-tenancy.*

[← back to the handbook](../README.md)

---

## 1. The threat model comes first

Security work without a threat model is guesswork. Answer four questions:

| Question | Example |
|---|---|
| **What are we protecting?** | User PII, payment data, proprietary content, availability itself |
| **From whom?** | Opportunistic scanners, credential stuffers, competitors, insiders, nation states |
| **What can they reach?** | Public API, an authenticated tenant boundary, the internal network, the database |
| **What is the cost if they succeed?** | Regulatory fine, customer loss, fraud, life safety |

The answers determine how much of what follows is proportionate. A hobby project
and a payments platform should not have the same controls, and pretending otherwise
produces security theatre that nobody maintains.

---

## 2. Authentication

### 2.1 Passwords, if you must

```mermaid
flowchart TD
    P["Password received"] --> H["Hash with a MEMORY-HARD KDF"]
    H --> A["Argon2id — 19 MiB, t=2, p=1 (OWASP minimum)"]
    H --> B["scrypt — N=2^17, r=8, p=1"]
    H --> C["bcrypt — cost ≥ 10 (legacy, 72-byte input limit)"]
    P --> N["NEVER: MD5, SHA-1, SHA-256, or any<br/>fast hash — GPUs test billions/second"]
    H --> S["Salt is per-password and stored with the hash<br/>(all three formats do this for you)"]
    H --> PP["Add a PEPPER in a KMS —<br/>a secret NOT in the database, so a<br/>database dump alone is not crackable"]
    style N fill:#7d1128,color:#fff
    style PP fill:#14532d,color:#fff
```

Other password essentials:

- **Check against known-breached password lists** (k-anonymity APIs let you do this
  without sending the password). This blocks credential stuffing more effectively
  than any complexity rule.
- **Drop composition rules.** "One uppercase, one symbol" produces `Password1!` and
  measurably weakens security. Enforce length (12+) and a breach check instead.
- **Rate limit and lock out carefully.** Per-account lockout enables a denial of
  service against a known user; prefer progressive delays plus per-IP and
  per-account limits, with CAPTCHA or step-up as escalation.
- **Constant-time comparison** everywhere a secret is compared.

### 2.2 OAuth 2.0 / OIDC, correctly

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client (SPA or mobile)
    participant A as Authorisation server
    participant R as Resource server

    C->>C: generate code_verifier (random 43-128 chars)<br/>code_challenge = SHA256(verifier)
    C->>A: /authorize?response_type=code&code_challenge=...<br/>&state=<csrf>&redirect_uri=<exact>
    A->>U: authenticate + consent
    U->>A: credentials + MFA
    A-->>C: redirect with ?code=...&state=...
    C->>C: verify state matches
    C->>A: POST /token {code, code_verifier}
    A->>A: SHA256(verifier) == stored challenge?
    A-->>C: {access_token (15 min), refresh_token, id_token}
    C->>R: GET /resource  Authorization: Bearer <access_token>
    R->>R: validate signature, iss, aud, exp, scope
    R-->>C: 200
```

The non-negotiables:

| Control | Why |
|---|---|
| **PKCE, always** — including for confidential clients | Prevents authorisation-code interception |
| **`state` parameter, verified** | CSRF protection on the callback |
| **Exact redirect URI matching** | Open redirects lead directly to token theft |
| **Never use the implicit flow** | Tokens in the URL fragment leak via history, referrers and logs. Deprecated |
| **Validate `aud` on the resource server** | Otherwise a token minted for another service is accepted by yours |
| **Rotate refresh tokens on use, detect reuse** | Reuse of a rotated refresh token means it was stolen — revoke the whole family |

### 2.3 Session and token storage in browsers

| Storage | XSS risk | CSRF risk | Verdict |
|---|---|---|---|
| `localStorage` | **Readable by any script** | None | Avoid for tokens |
| `sessionStorage` | Readable by any script | None | Same problem |
| Cookie, `HttpOnly` + `Secure` + `SameSite=Lax`/`Strict` | Not readable by script | Mitigated by SameSite + a CSRF token | **Preferred** |
| In-memory (JS variable) | Lost on refresh; still XSS-reachable while live | None | Good for access tokens with a refresh flow |

The honest summary: **if you have an XSS vulnerability, you have lost regardless of
storage.** But `HttpOnly` cookies remove the easiest exfiltration path and are
strictly better than `localStorage`. Combine `HttpOnly` cookies for the refresh
token with an in-memory access token for the best available posture.

---

## 3. Authorisation

### 3.1 The models

```mermaid
flowchart TD
    subgraph rbac["RBAC — role-based"]
        R1["User → Role → Permissions"]
        R2["Simple, auditable, familiar.<br/>Role explosion when rules get specific:<br/>'editor-for-org-7-except-billing'"]
    end
    subgraph abac["ABAC — attribute-based"]
        A1["Decision = f(subject, resource,<br/>action, environment)"]
        A2["Expressive: 'managers may approve<br/>expenses under $5k in their own<br/>department during business hours'.<br/>Harder to audit and reason about."]
    end
    subgraph rebac["ReBAC — relationship-based"]
        B1["Zanzibar-style: document:42#viewer@user:alice<br/>Permissions derive from a graph"]
        B2["Ideal for sharing hierarchies:<br/>folders, teams, nested groups.<br/>Needs a dedicated service to be fast."]
    end
    style rbac fill:#0b2545,color:#fff
    style rebac fill:#134e4a,color:#fff
```

Most systems should start with **RBAC plus resource ownership** (`role` for what
kind of action, `owner_id`/`tenant_id` for which specific objects) and only adopt
ABAC or ReBAC when the rules genuinely need them.

### 3.2 Broken object-level authorisation — the number one API risk

```mermaid
flowchart TD
    subgraph bad["Vulnerable"]
        B1["GET /api/invoices/9182"] --> B2["verify JWT is valid ✓"] --> B3["SELECT * FROM invoices WHERE id = 9182"] --> B4["return it — to ANY authenticated user"]
    end
    subgraph good["Correct"]
        G1["GET /api/invoices/9182"] --> G2["verify JWT ✓ → subject = user 42"] --> G3["SELECT * FROM invoices<br/>WHERE id = 9182<br/>AND tenant_id = :caller_tenant"] --> G4["0 rows → 404 (not 403 —<br/>do not confirm existence)"]
    end
    style bad fill:#7d1128,color:#fff
    style good fill:#14532d,color:#fff
```

The structural defences, in order of reliability:

1. **Scope every query by the caller's tenant/owner** — put it in the `WHERE`
   clause, not in an `if` statement after fetching. The database enforces it.
2. **Row-level security in the database**, so even a forgotten predicate is safe.
3. **A repository layer that cannot construct an unscoped query.** Make the unsafe
   thing impossible to express rather than merely discouraged.
4. **Automated tests that attempt cross-tenant access** on every endpoint. This is
   the only defence that catches regressions.

---

## 4. Secrets

```mermaid
flowchart TD
    S["A secret"] --> W{"Where does it live?"}
    W -->|"❌ in code / git"| B1["Permanent. Git history keeps it forever.<br/>Rotate immediately if it ever happened."]
    W -->|"❌ in a .env committed"| B2["Same problem, one step removed"]
    W -->|"⚠ environment variables"| B3["Visible in /proc, crash dumps, child<br/>processes and many logging setups.<br/>Acceptable with care, not ideal"]
    W -->|"✓ secrets manager"| G1["Vault, AWS Secrets Manager, GCP SM<br/>+ audit log, rotation, per-workload access"]
    W -->|"✓ workload identity"| G2["No stored secret at all — the platform<br/>attests the workload and issues<br/>short-lived credentials. BEST."]
    style B1 fill:#7d1128,color:#fff
    style G2 fill:#14532d,color:#fff
```

**Workload identity is the endgame.** If a service can prove what it is to the
platform (IAM roles for service accounts, SPIFFE identities, cloud instance
metadata) and receive a short-lived credential, there is no long-lived secret to
leak, rotate, or find in a repository.

Rotation guidance: rotate **automatically** on a schedule, and rotate
**immediately** on any suspicion. A rotation procedure that has never been executed
does not work — practise it. And support **two valid credentials at once** during
rotation, or every rotation is an outage.

---

## 5. The OWASP classes that matter most

| Risk | Concrete failure | Defence |
|---|---|---|
| **Broken access control** | Cross-tenant data via an id in the URL | Scope every query; RLS; cross-tenant tests |
| **Injection** | `"SELECT * FROM u WHERE n='" + name + "'"` | Parameterised queries; never build SQL/shell/LDAP by concatenation |
| **Cryptographic failures** | Fast password hashes, plaintext PII, TLS 1.0 | Argon2id, TLS 1.3, field-level encryption via KMS |
| **Insecure design** | No rate limit on password reset; no re-auth for changing email | Threat model the flow, not just the endpoint |
| **Security misconfiguration** | Public S3 bucket, debug mode on, default credentials | Infrastructure as code, scanned; secure defaults |
| **Vulnerable components** | An unpatched library with a known RCE | SBOM, automated dependency scanning, an actual patch SLA |
| **Auth failures** | No MFA, no breach check, session fixation | Rotate session id on login; MFA; breached-password checks |
| **Integrity failures** | Unsigned CI artefacts; unpinned build dependencies | Signed builds, provenance, pinned and verified dependencies |
| **Logging failures** | No record of who accessed what | Immutable audit log of security-relevant events |
| **SSRF** | User-supplied URL fetched by the server, reaching the metadata endpoint | Allowlist destinations; block link-local (169.254.169.254) and RFC1918; resolve then validate the IP |

**SSRF deserves a specific note** because cloud metadata endpoints turn it into full
credential theft. Validating the URL string is insufficient — DNS can resolve a
benign hostname to an internal IP, and a redirect can move an allowed URL to a
forbidden one. Resolve the hostname, check the resulting IP against a denylist,
then connect to **that IP** directly, and re-validate on every redirect.

---

## 6. Zero trust and the internal network

```mermaid
flowchart TD
    subgraph old["Perimeter model"]
        O1["Hard shell, soft centre"] --> O2["Inside the VPC = trusted"] --> O3["One compromised pod<br/>reaches everything"]
    end
    subgraph zt["Zero trust"]
        Z1["No implicit trust from network location"]
        Z2["mTLS: every service proves identity"]
        Z3["Authorisation policy per service pair"]
        Z4["Short-lived, automatically rotated certs"]
        Z5["Least privilege on every credential"]
    end
    old --> zt
    style old fill:#7d1128,color:#fff
    style zt fill:#14532d,color:#fff
```

The practical entry point is **mTLS via a service mesh**, which gives you
cryptographic service identity, encryption in transit, and per-pair authorisation
policy without changing application code. Add network policies so that pods can
only reach the services they actually need — most breaches escalate through lateral
movement that a default-deny policy would have stopped.

---

## 7. Multi-tenancy

```mermaid
flowchart TD
    Q["Isolation model"] --> S["Shared everything<br/>tenant_id column"]
    Q --> D["Schema per tenant"]
    Q --> B["Database per tenant"]
    Q --> C["Cluster/cell per tenant"]

    S --> S1["Cheapest, most efficient.<br/>One missing WHERE clause = breach.<br/>Noisy neighbours affect everyone."]
    D --> D1["Better isolation, same cluster.<br/>Migrations must run N times."]
    B --> B1["Strong isolation, per-tenant backup<br/>and restore, residency control.<br/>Expensive above ~hundreds of tenants."]
    C --> C1["Maximum isolation, compliance-friendly.<br/>Highest cost and operational load."]

    style S fill:#3b0d0d,color:#fff
    style B fill:#14532d,color:#fff
```

For the shared model — which most SaaS uses — **row-level security is the control
that actually holds**, because it moves enforcement from every developer's memory
into the database:

```sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON invoices
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- the application sets this per connection/transaction:
SET LOCAL app.tenant_id = '...';
```

Get the connection-pooling interaction right: with a transaction-scoped pool,
`SET LOCAL` inside the transaction is correct; a session-level `SET` can leak a
tenant context to the next borrower of that connection, which is precisely the
breach you were preventing.

Also isolate **beyond the database**: per-tenant rate limits and quotas (so one
tenant cannot starve the rest), per-tenant cache key prefixes, and per-tenant
encryption keys where regulation requires crypto-shredding on deletion.

---

## 8. Privacy and data lifecycle

```mermaid
flowchart LR
    C["Collect<br/>minimise — only what you need"] --> S["Store<br/>encrypted, classified, located per residency rules"] --> U["Use<br/>purpose-limited, access-logged"] --> R["Retain<br/>defined period, enforced automatically"] --> D["Delete<br/>everywhere"]
    D --> N["'Everywhere' = primary DB, replicas,<br/>backups, caches, search indexes,<br/>warehouse, logs, third parties"]
    style N fill:#7d1128,color:#fff
```

Deletion is the requirement that retrofits worst. Two designs make it tractable:

- **Key every piece of personal data by `user_id`** so a deletion job can find all
  of it. Data that is only reachable by a join through three tables will be missed.
- **Crypto-shredding.** Encrypt each user's data with a per-user key held in a KMS.
  Deleting the key renders every copy — including immutable backups you cannot
  edit — permanently unreadable. This is the only practical answer to "delete this
  user from a backup taken last March".

---

## 9. Checklist

```
□ Threat model written: assets, adversaries, entry points, impact
□ Passwords hashed with Argon2id/scrypt/bcrypt + KMS pepper; breach-list checked
□ MFA available, enforced for privileged accounts
□ OAuth: PKCE, state verified, exact redirect URIs, no implicit flow
□ Tokens short-lived; refresh tokens rotate with reuse detection
□ Every request authorises against the SPECIFIC object, server-side
□ Queries scoped by tenant in the WHERE clause; RLS enabled as a backstop
□ Automated cross-tenant access tests on every endpoint
□ Secrets in a manager or workload identity — never in git, ideally not in env
□ Rotation automated AND practised; two credentials valid during rotation
□ Parameterised queries everywhere
□ SSRF defence resolves the hostname and validates the IP, on every redirect
□ mTLS between services; default-deny network policies
□ TLS 1.2+ only, HSTS, secure cookie flags
□ Dependency scanning with a patch SLA
□ Immutable audit log of security-relevant events
□ Deletion pipeline reaches backups, caches, indexes and third parties
□ Incident response plan includes breach notification obligations and timelines
```

---

[← previous: Observability and operations](09-observability-and-operations.md) · [back to the handbook](../README.md) · [next: Case studies →](11-case-studies.md)
