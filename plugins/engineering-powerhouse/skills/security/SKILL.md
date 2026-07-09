---
name: security
description: >-
  Application security for web apps — OWASP Top 10, input validation, injection,
  XSS, CSRF, authentication, authorization, secrets, and data protection. Use
  when reviewing code for vulnerabilities, implementing auth, handling secrets or
  API keys, preventing injection/XSS/CSRF, preparing for a security audit, or
  responding to a vulnerability. Deny by default, never trust input, defense in
  depth. Triggers: "security review", "is this safe", "OWASP", "SQL injection",
  "XSS", "CSRF", "authentication", "authorization", "secrets", "vulnerability",
  "IDOR".
---

# Security

Security is not a feature you add; it is a property you preserve at every
boundary. Three principles carry most of the weight: **never trust input, deny by
default, and defend in depth** (no single control is the whole defense). The
attacker only needs one gap, so security is reviewed at every layer, not bolted on
at the end.

## Input and injection

- **Validate and sanitize all input at the boundary** — allowlist what is
  permitted, do not blocklist what is forbidden. Treat everything from clients,
  APIs, files, and headers as hostile.
- **Injection is always parameterization** — never concatenate user data into SQL,
  DQL, shell commands, or templates. Use bound parameters, every time.

```php
// Never — user data concatenated into the query
$conn->executeQuery("SELECT * FROM users WHERE email = '" . $email . "'");

// Always — bound parameters; the driver escapes
$conn->executeQuery('SELECT * FROM users WHERE email = :email', ['email' => $email]);
// Doctrine QueryBuilder: ->where('u.email = :email')->setParameter('email', $email)
```

The ORM helps, but raw DQL/SQL, `LIKE` fragments, and dynamic `ORDER BY` are still
injectable — parameterize or allowlist them.

## Output and XSS

- Escape output for its context. Twig auto-escapes HTML by default — **keep it on**
  and treat every `|raw` as a reviewed decision, never a convenience.
- Set a **Content-Security-Policy** to contain any XSS that slips through; it is
  the defense-in-depth layer behind escaping.

## CSRF and sessions

- Protect state-changing form/browser requests with **CSRF tokens** (Symfony's
  form CSRF / `IsCsrfTokenValid`). Stateless token APIs are not CSRF-prone the same
  way but must not accept ambient cookies for auth.
- Cookies: `HttpOnly`, `Secure`, `SameSite=Lax/Strict`. Regenerate the session id
  on login (prevent fixation).

## Authentication

- **Never roll your own.** Hash passwords with a strong adaptive algorithm
  (argon2id / bcrypt via Symfony's `PasswordHasher`), never MD5/SHA-1/plaintext.
- Support MFA for sensitive accounts; rate-limit and lock out brute force; make
  login errors generic ("invalid credentials", not "no such user").

## Authorization

- **Check authorization on every request, server-side, deny by default.** The UI
  hiding a button is not access control.
- **IDOR is the most common real breach** — always verify the current user owns or
  may access the specific record, not just that they are logged in. `GET
  /orders/42` must confirm order 42 belongs to the caller.

```php
// Enforce ownership/permission explicitly — Symfony Voter
if (!$this->isGranted('ORDER_VIEW', $order)) {
    throw $this->createAccessDeniedException();
}
```

## Secrets and data protection

- **Never commit, log, or hardcode secrets** — no keys, passwords, or tokens in
  code, VCS, or logs. Load from environment / a secrets vault (Symfony secrets);
  rotate on exposure.
- Encrypt sensitive data in transit (TLS everywhere) and at rest where required.
  Use vetted crypto (libsodium) — never invent your own. Hashing ≠ encryption.
- **Minimize and protect PII** — collect the least you need, do not log it, and
  scrub it from error reports.

## Dependencies and configuration

- Run `composer audit` in CI; patch known-vulnerable packages promptly
  (`composer` skill). Vulnerable dependencies are a top-10 category on their own.
- Disable debug in production (no stack traces to clients), set security headers,
  fail closed on misconfiguration.

## OWASP Top 10 — threat to defense

| Risk | Primary defense |
|---|---|
| Broken Access Control (incl. IDOR) | Server-side checks, deny by default, verify ownership. |
| Cryptographic Failures | TLS, vetted crypto, hash passwords, encrypt sensitive data at rest. |
| Injection | Parameterized queries; allowlist input. |
| Insecure Design | Threat-model early; secure defaults (`architecture`). |
| Security Misconfiguration | Debug off in prod, headers set, least privilege. |
| Vulnerable Components | `composer audit`, patch, remove unused deps. |
| Identification/Auth Failures | Strong hashing, MFA, rate limiting, session hygiene. |
| Software/Data Integrity | Verify sources, sign artifacts, guard deserialization. |
| Logging/Monitoring Failures | Log security events (not secrets); alert on anomalies. |
| SSRF | Allowlist outbound targets; validate URLs; block internal ranges. |

## Anti-patterns

- **Trusting the client** — validation only in the frontend.
- **String-built queries** — any user data concatenated into SQL/DQL/shell.
- **Auth by obscurity** — hiding UI instead of enforcing server-side.
- **Secrets in code/logs** — committed keys, credentials printed to logs.
- **Home-grown crypto** — inventing hashing/encryption instead of using libsodium.
- **Leaky errors** — stack traces and internals returned to users in production.

## Where this fits

`security` is the depth behind the security baseline in `engineering-standards`,
the auth layer for `api-design`, the boundary-hardening in `integration-build`,
and a lens in the self-review of `feature-delivery` and `finish-branch`.
Dependency auditing runs through `composer`; threat-aware structure through
`architecture`.
