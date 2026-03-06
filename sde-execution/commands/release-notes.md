---
description: Formatted changelog from raw commits or PR list — categorized, deduplicated, humanized
argument-hint: "<paste commit log or PR list>"
---

# /release-notes -- Generate Release Notes

Transform raw commits or PR titles into human-readable, categorized release notes. Chains release-notes skill.

## Invocation

```
/release-notes [paste git log --oneline output]
/release-notes [paste list of PR titles and numbers]
/release-notes v1.4.0 to v1.5.0 — add Redis caching, fix login bug, upgrade Node.js
```

## Workflow

### Step 1: Get the Changes

If commits/PRs are not provided, ask:
"Please paste the output of `git log v1.4.0..v1.5.0 --oneline` or a list of PR titles."

### Step 2: Categorize Each Change

Apply **release-notes** skill.

Categorize each commit/PR into:
- `feat:` — New feature
- `fix:` — Bug fix
- `security:` — Security fix
- `perf:` — Performance improvement
- `breaking:` — Breaking change
- `deprecated:` — Deprecated feature
- `chore:` — Internal (deps update, tooling) — may skip unless relevant

### Step 3: Filter and Prioritize

Remove noise:
- Skip: "fix typo", "update deps" (unless security), "WIP", "merge branch"
- Combine: related fixes to the same bug into one entry
- Promote: security fixes to the top of their category

### Step 4: Humanize Each Entry

Transform developer language into user language:
- "Refactor UserRepository" -> skip (internal)
- "Fix N+1 query in /api/users" -> "Fixed intermittent slow response on user list endpoint"
- "Add rate limiting middleware" -> "Added rate limiting on authentication endpoints to prevent brute-force attacks"
- "bump stripe to 14.2.0" -> mention only if it includes security fixes users should know about

### Step 5: Flag Breaking Changes

If any commit contains "BREAKING CHANGE", "breaking:", or obvious API changes:
- Move to the top of release notes
- Write migration instructions
- Emphasize with > blockquote

### Step 6: Verify Version and Date

Ask for or infer:
- Version number (if not provided)
- Release date (use today's date if not specified)

## Output

```markdown
## v[X.Y.Z] — [Release Date]

### Breaking Changes

> **Action required:** [instructions]

- **[Component]:** [What changed, what you need to do] — (#[PR])

---

### New Features

- **[Feature]:** [User-facing description] — (#[PR])

---

### Bug Fixes

- **[Component]:** [What was broken, now fixed] — (#[PR])

---

### Security

- **[Component]:** [Vague but honest description] — (#[PR])
  > Recommend upgrading immediately.

---

### Performance

- **[Operation]:** [Improvement with numbers if available] — (#[PR])

---

### Deprecated

- **[Feature]:** Deprecated, will be removed in v[N.0]. Use [alternative].

---

### Upgrade Notes

[If there are specific steps required to upgrade]
```

## Next Steps

- "Want a deployment plan for this release? -> `/deploy-plan [service]`"
