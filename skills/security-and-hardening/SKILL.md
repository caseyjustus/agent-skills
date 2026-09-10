---
name: security-and-hardening
description: Hardens code against vulnerabilities. Use when auditing an input handler for vulnerabilities, when handling user input, authentication, data storage, or external integrations, or when checking a login flow is safe against the OWASP Top Ten. Use when building any feature that accepts untrusted data, manages user sessions, or interacts with third-party services. Use when auditing dependencies for known vulnerabilities, triaging package-manager audit findings, or assessing supply-chain risk in a new package. Use when personal data or privacy compliance (GDPR, CCPA) is involved.
---

# Security and Hardening

## Overview

Treat every external input as hostile, every secret as sacred, every authorization check as mandatory. Security isn't a phase — it's a constraint on every line that touches user data, auth, or external systems. Code patterns and pre-commit steps live in `../../references/security-checklist.md`.

## When to Use

Building anything that accepts user input; implementing authentication or authorization; storing or transmitting sensitive data; integrating external APIs; adding file uploads, webhooks, or callbacks; handling payment or PII data.

## Threat Model First

Controls bolted on without a threat model are guesses. Before hardening, spend five minutes thinking like an attacker:

1. **Map the trust boundaries.** Where does untrusted data cross in? HTTP requests, form fields, uploads, webhooks, third-party APIs, queues, and **LLM output** — plus local values that look internal because the OS handed them to you: another process's command line or environment, filenames on a shared volume, a path in a job payload. Trust follows who *wrote* a value, not which channel delivered it.
2. **Name the assets.** What's worth stealing or breaking? Credentials, PII, payment data, admin actions, money movement.
3. **Run STRIDE over each boundary** — a quick lens, not a ceremony.
4. **Write abuse cases next to use cases.** For each feature ask "how would I misuse this?" — then make that your first test.

| Threat | Ask | Typical mitigation |
|---|---|---|
| **S**poofing | Can someone impersonate a user/service? | Authentication, signature verification |
| **T**ampering | Can data be altered in transit or at rest? | Integrity checks, parameterized queries, HTTPS |
| **R**epudiation | Can an action be denied later? | Audit logging of security events |
| **I**nformation disclosure | Can data leak? | Encryption, field allowlists, generic errors |
| **D**enial of service | Can it be overwhelmed? | Rate limiting, input size caps, timeouts |
| **E**levation of privilege | Can a user gain rights they shouldn't? | Authorization checks, least privilege |

If you can't name the trust boundaries for a feature, you're not ready to secure it — OWASP **A04: Insecure Design**. Most breaches begin in design, not code.

## The Three-Tier Boundary System

**Always, no exceptions:** validate all external input at the system boundary; parameterize every database query; encode output to prevent XSS (use the framework's auto-escaping, don't bypass it); HTTPS for all external communication; hash passwords with bcrypt/scrypt/argon2 (salt rounds ≥ 12); set security headers (CSP, HSTS, X-Frame-Options, X-Content-Type-Options); use httpOnly + secure + sameSite session cookies; run the package manager's native audit against the committed lockfile before every release.

**Ask first (human approval):** new or changed authentication flows; storing a new category of sensitive data; new external integrations; CORS changes; new file-upload handlers; rate-limit or throttling changes; granting elevated permissions or roles.

**Never:** commit secrets to version control; log sensitive data (passwords, tokens, full card numbers); treat client-side validation as a security boundary; disable security headers for convenience; pass user data to `eval()` or `innerHTML`; store auth tokens in client-accessible storage; expose stack traces or internal errors to users.

## OWASP Prevention Patterns

- **Injection.** Parameterized queries or an ORM's typed API only — never string-concatenate user input into SQL, NoSQL, or a shell command.
- **Broken authentication.** Hash with bcrypt/scrypt/argon2; keep the session secret in the environment; set `httpOnly`, `secure`, `sameSite`, and a bounded `maxAge`; expire password-reset tokens.
- **XSS.** Rely on framework escaping; if you must render HTML, sanitize with DOMPurify first. Never assign user or model text to `innerHTML`.
- **Broken access control.** Authentication is not authorization — after loading a resource, verify *this* user may act on it (ownership or role) and return 403 otherwise.
- **Security misconfiguration.** Helmet-style headers, an explicit CSP, and CORS restricted to known origins from config, never `*` with credentials.
- **Sensitive data exposure.** Strip secret fields (`passwordHash`, `resetToken`) in a serializer rather than per-route; read secrets from the environment and fail fast when one is missing.

**SSRF.** Any server-side fetch of a user-influenced URL — webhooks, "import from URL", image proxies, link previews — can be aimed at internal services. Require all of: an https-only scheme check, a host allowlist, rejection if *any* resolved IP is outside `unicast` range (this covers loopback, private, unique-local, and link-local `169.254.169.254`, the cloud-metadata endpoint that is the #1 SSRF target), and `redirect: 'error'`. **This still leaves a TOCTOU gap** — `fetch` re-resolves DNS after your check, so a short-TTL record can rebind to an internal IP. For high-risk surfaces, resolve once and connect to the pinned IP, or front it with a filtering agent (`request-filtering-agent` / `ssrf-req-filter`).

## Input Validation

- **Validate with a schema at the boundary** (Zod or equivalent) in the route handler, not deep in the service layer. Return 422 with a structured error; downstream code then receives typed, validated data.
- **File uploads:** allowlist MIME types, cap size, don't trust the extension — check magic bytes when it matters.

## Destructive Operations on Derived Paths

A delete, move, or overwrite is only as safe as the value naming its target. Reading that value from the kernel, a job payload, or a sibling service proves where it *arrived from*, not who *wrote* it. A shape check ("absolute path, at least one directory deep") proves well-formedness and gets mistaken for authorization — that is how a cleanup routine deletes the root instead of the leaf.

Before a destructive call, require all three: the resolved target sits under an **allowlisted root** (compare after resolving symlinks, never on the raw string); it is at least one level **below** that root, so a root is never itself the target; and it carries **evidence that it is yours**, read *before* the operation and before any teardown that removes it — otherwise "absent" and "not mine" are indistinguishable. On refusal, log the rejected target and stop; a cleanup that falls back to a broader default path is the failure this guards against.

Two limits, because the check reads stronger than it is. A marker inside the tree is self-attestation — anything that can write there can write the marker — so the expected owner must come from authenticated state, and the marker needs integrity protection (restrictive ownership, or a MAC) before it counts as authorization. And resolving a path then operating on the *name* is a check/use race wherever an untrusted process can swap an ancestor: hold the target by descriptor and use no-follow, beneath-the-root operations, or ensure the hierarchy can't change for the duration.

## Dependency Audits and Supply Chain

Audits report known advisories; they do not prove a package is trustworthy or that vulnerable code is reachable. Triage by reachability, not severity alone:

- **Critical/high:** reachable in runtime, build, test, or deploy paths → fix immediately. Confirmed unreachable → fix soon, not a blocker. No fix available → workaround, replace the dependency, or allowlist with a review date.
- **Moderate:** reachable in production → next release cycle; dev-only → backlog. **Low:** fix during regular dependency updates.
- Ask whether the vulnerable *function* is called, whether the dependency is runtime or dev-only, and whether it's exploitable in your deployment context. When you defer, document the reason and set a review date.

**Hygiene.** Don't assume npm or treat the nearest manifest as the install root. Find the installation boundary — the workspace root owning the lockfile — and corroborate `packageManager`, the lockfile, and CI; stop on disagreement or competing lockfiles. **Block dependency scripts before first execution:** bootstrap with scripts disabled, inspect the pending script source, approve only the minimum, commit the policy, then verify with a clean frozen install. Never blanket-approve scripts.

- **Never run forced audit remediation automatically** (`npm audit fix --force`): it may cross declared ranges. Preview, read changelogs, test each upgrade.
- **Verify registry signatures and provenance** where supported (`npm audit signatures`); treat absence as a signal to investigate, not proof of compromise.
- **Review new dependencies, lockfile diffs, and script-policy changes together** — ownership, maintenance, release age, provenance, transitive graph, and typosquats like `cross-env` vs `crossenv` (OWASP **A06**, **LLM03**).

## Rate Limiting

Rate-limit the API generally and auth endpoints strictly (e.g. 10 attempts per 15 minutes). **Count in a shared store once there is more than one process:** in-memory counters (the `express-rate-limit` default) mean the effective limit is `max × instances` behind a load balancer, and on serverless or edge runtimes each invocation starts from zero, so a strict auth limit may never fire. Use a Redis-backed store, or an HTTP-based limiter where a long-lived TCP connection isn't available.

## Secrets Management

Commit `.env.example` with placeholders; never commit `.env`, `.env.local`, `*.pem`, or `*.key` — put them in `.gitignore`. Scan staged changes before committing (`git diff --cached | grep -i "password\|secret\|api_key\|token"`). **If a secret is ever committed, rotate it.** Deleting the line or rewriting history is not enough — assume it's compromised the moment it reaches a remote. Revoke and reissue first, then purge from history.

## Data Privacy and Compliance

Securing data asks "can an attacker read it?" Privacy asks "should *we* hold it, and for how long?" — a separate question hardening doesn't answer. The cheapest data to protect, breach, and comply over is data you never collected.

| Class | Examples | Handling |
|---|---|---|
| **Non-personal** | Aggregates, anonymized counts | Normal handling |
| **Personal (PII)** | Name, email, IP, device/user IDs | Minimize, access-control, include in export/delete |
| **Sensitive** | Health, finance, location, biometrics, gov IDs, minors' data | Extra basis to collect, stricter access, often encryption + audit logging |

- **Minimize and set a purpose.** Collect a field only against a stated use — "might be useful later" is latent breach scope, not a purpose. Keep PII out of telemetry.
- **Set retention up front, then actually delete** — including backups, caches, search indexes, and analytics copies. Data with no expiry is a breach scheduled for later.
- **Support data-subject rights** (GDPR/CCPA and kin): export, correct, delete. These are engineering features — design the schema so a user's data is findable and erasable.
- **Get auditable consent before collection or third-party sharing.** Sending PII to an analytics/ad/LLM vendor is sharing, and needs a data-processing agreement.
- **Localize defaults.** Residency rules differ by user location; make the policy a configurable boundary, not an assumption.

## Securing AI / LLM Features

An app that calls an LLM inherits a new attack surface — the [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/):

- **Treat all model output as untrusted input (LLM05).** Never pass it into `eval`, SQL, a shell, `innerHTML`, or a file path. Parse defensively, validate against a schema, dispatch only allowlisted actions, and render with `textContent`.
- **Assume prompts can be hijacked (LLM01).** Untrusted text in the context window — a user message, a fetched page, a PDF — can carry instructions. The system prompt is not a security boundary; enforce permissions in code.
- **Keep secrets and other users' data out of prompts (LLM02/LLM07).** Anything in context can be echoed back.
- **Constrain tool and agent permissions (LLM06).** Scope tools to the minimum, confirm destructive or irreversible actions, validate every tool argument.
- **Bound consumption (LLM10).** Cap tokens, request rate, and loop depth so crafted input can't run up cost or hang the system.
- **Isolate retrieval data (LLM08).** Partition embeddings per tenant and validate documents before indexing so poisoned content can't steer answers.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "It's an internal tool, security doesn't matter" | Internal tools get compromised. Attackers target the weakest link. |
| "We'll add security later" | Retrofitting is 10x harder than building it in. |
| "No one would try to exploit this" | Automated scanners will find it. Obscurity is not security, and frameworks give tools, not guarantees. |
| "Threat modeling is overkill here" | Five minutes of "how would I attack this?" prevents design flaws no control can patch later. |
| "It's just LLM output, it's only text" | That text can be a SQL statement, a script tag, or a shell command. |
| "The audit passed, so the dependency is safe" | Audits match known advisories. They don't detect a newly malicious package or make install scripts safe. |
| "Collect it now, we might need it later" | Data you don't hold can't be breached, subpoenaed, or mis-deleted. |
| "Compliance is legal's problem" | Export, deletion, retention, and consent are schema and code, not paperwork. |

## Red Flags

- User input passed directly to database queries, shell commands, or HTML rendering
- A delete, move, or overwrite whose target comes from a payload, config value, or another process's command line, guarded only by a shape check on the path
- Secrets in source code or commit history; stack traces or internal errors exposed to users
- API endpoints without authentication *or* authorization checks; missing CORS config or wildcard origins
- No rate limiting on auth endpoints, or an in-memory limiter in front of more than one instance
- Known-critical dependency vulnerabilities, competing lockfiles at one installation boundary, non-reproducible installs, or blanket-approved scripts
- Server fetches user-supplied URLs without an allowlist (SSRF); LLM output passed into a query, the DOM, a shell, or `eval`; secrets, PII, or the full system prompt placed in a context window
- Personal data collected with no stated purpose, retention limit, or deletion path; PII sent to vendors with no consent or DPA
- "Delete my account" that only flips a flag while personal data lingers in stores and backups

## Verification

- [ ] Native audit has no unmitigated reachable critical/high findings; CI preserves the authoritative lockfile and blocks unreviewed dependency scripts
- [ ] No secrets in source code or git history; error responses don't expose internals
- [ ] All user input validated at system boundaries; auth *and* authorization checked on every protected endpoint
- [ ] Destructive filesystem operations resolve symlinks, then verify allowlisted root, minimum depth, and ownership before running
- [ ] Security headers present in responses; rate limiting active on auth endpoints, backed by a shared store when more than one instance serves traffic
- [ ] Server-side URL fetches validated against an allowlist (no SSRF); LLM output validated and encoded before use, with secrets and cross-tenant data kept out of prompts
- [ ] Personal data classified, minimized to a stated purpose, with a retention limit; export and deletion work end-to-end including backups, caches, and analytics copies
