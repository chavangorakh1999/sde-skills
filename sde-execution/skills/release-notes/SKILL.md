---
name: release-notes
description: "Developer-facing changelog from commit list or PR titles: categorized sections (feat/fix/breaking/deprecated/security). Use when generating a changelog for a release."
---

## Release Notes

Good release notes tell users what changed and why it matters. They're not a git log dump — they're a curated, human-readable summary of significant changes.

### Context

Commits, PRs, or version changes to document: **$ARGUMENTS**

---

### Categorization

Classify every change into one of these categories (following Conventional Commits):

```
feat:     New feature or capability
fix:      Bug fix — describe the bug, not just the fix
security: Security fix — always document these clearly
perf:     Performance improvement with measurable numbers
breaking: BREAKING CHANGE — this changes existing behavior
deprecated: Feature being phased out (include migration path)
docs:     Documentation update (usually skip in user-facing notes)
chore:    Internal tooling, deps update (often skip unless security)
```

---

### Output Format

```markdown
## [Version] — [Release Date]

### Breaking Changes

> **Action required before upgrading:** [instructions]

- **[Component]:** [What changed and what you need to do] — ([PR #N](link))
  - Before: `old behavior`
  - After: `new behavior`
  - Migration: [step-by-step migration instructions]

---

### New Features

- **[Feature name]:** [What it does and why it's useful] — ([PR #N](link))
  ```javascript
  // Example usage
  ```

- **[Feature name]:** [Description] — ([PR #N](link))

---

### Bug Fixes

- **[Component]:** [What was broken and what the user experienced] — ([PR #N](link))

---

### Security

- **[Component]:** [Vague but honest description] — ([PR #N](link))
  > Recommend upgrading immediately.

---

### Performance

- **[Operation]:** [Improvement with numbers] — ([PR #N](link))
  - Before: [X ms / X MB / X QPS]
  - After: [Y ms / Y MB / Y QPS]
  - Conditions: [when you'll see this improvement]

---

### Deprecated

- **[Feature]:** Deprecated in this version, will be removed in [version].
  - Migration: Use `[new feature]` instead: `[example]`

---

### Internal Changes

[Only include if relevant to developers: tooling changes, dependency upgrades with impact]

---

### Upgrade Notes

[Any specific steps required to upgrade, database migrations, configuration changes]
```

---

### Rules for Good Release Notes

```
1. Write from the user's perspective, not the engineer's
   Bad:  "Refactored UserRepository to use repository pattern"
   Good: "Fixed intermittent login failures under high load"

2. Include numbers for performance changes
   Bad:  "Improved search performance"
   Good: "Search API now returns results in 50ms (down from 400ms on large catalogs)"

3. Breaking changes must include migration instructions
   Bad:  "Renamed getUser to findUser"
   Good: "Renamed getUser to findUser — update all callsites: s/getUser/findUser/g"

4. Security fixes: vague but honest
   Don't expose the attack vector (helps attackers who haven't upgraded)
   Do tell users to upgrade immediately
   "Fixed authentication bypass in /api/admin routes — recommend immediate upgrade"

5. Don't document internal changes unless they affect users/developers
   CI improvements, linting changes, internal refactors: skip

6. Link to migration guides for complex changes
   "See [migration guide](link) for step-by-step upgrade instructions"
```

---

### From Raw Commits

If given a list of commits:

```
# Transform this:
8a7b3d1 fix login bug
a3c92f0 add rate limiting
bc71234 update deps
d1e9031 perf: faster search queries
e2f4567 BREAKING: rename getUser to findUser
f3g5678 add 2FA support

# Into this:
### Breaking Changes
- **User API:** Renamed `getUser` to `findUser` — update all usages

### New Features
- **Authentication:** Added two-factor authentication support via TOTP

### Bug Fixes
- **Login:** Fixed intermittent login failures [describe the user-facing symptom]

### Performance
- **Search:** Query time reduced by ~X% for large datasets

### Security
- Added rate limiting on authentication endpoints to prevent brute-force attacks
```
