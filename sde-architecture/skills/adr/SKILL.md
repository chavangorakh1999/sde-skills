---
name: adr
description: "Architecture Decision Record: context, decision, consequences, alternatives considered, status lifecycle. Use when making a significant architectural decision that the team needs to understand and revisit."
---

## Architecture Decision Record (ADR)

An ADR is a document that captures an important architectural decision, its context, and its consequences. Written decisions don't rot — they let future engineers understand why the code is the way it is.

### Context

Decision to document: **$ARGUMENTS**

---

### When to Write an ADR

Write an ADR when:
- Choosing a technology, framework, or tool (database, auth library, message queue)
- Choosing between two architecturally significant approaches
- Making a decision that will be hard or expensive to reverse
- Making a decision that will confuse future engineers if not explained
- Changing a previous decision (supersede the old ADR)

Don't write an ADR for:
- Routine implementation details
- Decisions that are obviously reversible
- Code style preferences (use ESLint/Prettier for that)

---

### ADR Format

```markdown
# ADR-[number]: [Short descriptive title]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXX
**Deciders:** [who was involved in this decision]
**Tags:** [database, auth, infrastructure, api, etc.]

## Context

[Describe the problem or question that led to this decision.
What forces are at play? What constraints exist?
This is the SITUATION, not the decision.]

## Decision

[State the decision clearly. Active voice: "We will use X." not "X was chosen."
Brief rationale in 1-3 sentences.]

## Consequences

### Positive
- [Benefit 1]
- [Benefit 2]

### Negative (Tradeoffs)
- [Cost or limitation 1]
- [Cost or limitation 2]

### Risks
- [Risk 1 — include mitigation if known]

## Alternatives Considered

### [Alternative 1]
[What it is, why we considered it, why we rejected it]

### [Alternative 2]
...

## Implementation Notes

[Optional: specific implementation constraints, migration steps, or caveats
that engineers need to know when working with this decision]

## Review Trigger

[When should this decision be reconsidered?
"Revisit if: data volume exceeds 1TB", "Revisit at 6-month checkpoint",
"Supersede if: team grows past 10 engineers"]
```

---

### Worked Example

```markdown
# ADR-004: Use JWT Bearer Tokens for API Authentication

**Date:** 2024-03-15
**Status:** Accepted
**Deciders:** Backend team, Security lead
**Tags:** auth, api, security

## Context

Our API needs an authentication mechanism for both the web frontend
and a future mobile app. We're using Node.js/Express with a stateless
API design goal. The team has prior JWT experience.

Current state: session cookies work for the web app but won't work
for mobile (cross-origin, cookie limitations). We need to serve
both clients.

## Decision

We will use short-lived JWT access tokens (15-minute expiry) paired
with long-lived refresh tokens (30-day expiry) stored in httpOnly cookies.

## Consequences

### Positive
- Stateless: API servers don't need shared session storage
- Works for both web and mobile clients
- Claims embedded in token reduce DB lookups on every request
- Standard: tooling and documentation abundant

### Negative
- Access tokens cannot be revoked before expiry (15-min window)
- Refresh token rotation adds complexity to the frontend
- Secret rotation requires all users to re-authenticate

### Risks
- If JWT_SECRET leaks, all tokens are compromised — mitigate with
  key rotation procedure and secret manager (not env var on disk)

## Alternatives Considered

### Opaque tokens stored in Redis
More revocable (delete from Redis = immediately invalidated), but
requires Redis on every request path. Adds infrastructure dependency
and latency. Rejected: the stateless benefit of JWT outweighs the
revocation concern at our current scale.

### Session cookies only
Works for web but breaks mobile clients. Requires sticky sessions
or shared session store. Rejected: doesn't meet the mobile requirement.

## Implementation Notes

- Use RS256 (RSA) not HS256 (HMAC) if multiple services verify tokens
- Always verify: signature, expiry, audience, issuer
- Store refresh tokens hashed in DB (if leaked, can't be replayed)
- Implement token rotation on every refresh (detect refresh token reuse)

## Review Trigger

Revisit if: we need immediate token revocation (e.g., account compromise
response SLA < 15 min), or if we add services that need cross-service auth.
```

---

### ADR Status Lifecycle

```
Proposed  -> team proposes, open for comment
    |
    v
Accepted  -> team agrees, decision is in effect
    |
    +-> Deprecated  -> decision no longer relevant (removed feature)
    |
    +-> Superseded by ADR-XXX  -> replaced by a newer decision
```

When superseding an ADR:
- Update the old ADR's status: `Superseded by ADR-XXX`
- Reference the old ADR in the new one: `Supersedes ADR-004`
- Explain in the context why the old decision no longer holds

---

### ADR Index

Maintain an index file listing all ADRs:

```markdown
# Architecture Decision Records

| # | Title | Status | Date | Tags |
|---|-------|--------|------|------|
| 001 | Use PostgreSQL for primary datastore | Accepted | 2023-11-01 | database |
| 002 | Adopt monorepo with npm workspaces | Accepted | 2023-11-15 | build |
| 003 | Use Bull for background job processing | Accepted | 2024-01-10 | queue |
| 004 | JWT Bearer + refresh tokens for auth | Accepted | 2024-03-15 | auth |
| 005 | Migrate to Kafka for event streaming | Proposed | 2024-06-01 | queue |
```

---

### Output Format

Produce the full ADR document in the format above, then:

```
## Review Questions
[3-5 questions for the team to answer before accepting the ADR]

## Open Issues
[Any unresolved questions or dependencies]
```
