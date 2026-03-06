# Contributing to SDE Skills Marketplace

## How to contribute

### Adding a new skill

1. Choose the right plugin for the skill domain
2. Create a directory under `<plugin-name>/skills/<skill-name>/`
3. Add a `SKILL.md` file using the format below
4. Register the skill in the plugin's `README.md`

### Adding a new command

1. Create a `.md` file under `<plugin-name>/commands/<command-name>.md`
2. Follow the command format below
3. Document it in the plugin's `README.md`

### Adding a new plugin

1. Create a directory `sde-<domain>/`
2. Add `.claude-plugin/plugin.json`, `README.md`, `skills/`, and `commands/`
3. Register the plugin in `.claude-plugin/marketplace.json`

---

## SKILL.md format

```markdown
---
name: skill-name
description: "One sentence: what this skill does and when to load it."
---

## Skill Title

Brief intro paragraph.

### Context

$ARGUMENTS placeholder if the skill accepts input.

### Framework / Instructions

Step-by-step methodology...

### Output Format

What the skill produces.

### Further Reading

- Links to authoritative sources
```

## Command format

```markdown
---
description: What this command does end-to-end
argument-hint: "<what to pass as argument>"
---

# /command-name -- Title

## Invocation

Examples of how to call the command.

## Workflow

### Step 1: ...
### Step N: ...

## Output

What gets produced.

## Next Steps

Suggested follow-on commands.
```

## Quality bar

- Every skill must encode a real engineering framework, not generic advice
- JavaScript/Node.js examples take priority; note where patterns differ for other languages
- Every recommendation must include tradeoffs (why this, what it costs, alternatives)
- Quantify where possible: O(n) not "slow", "hits connection pool at 1K QPS" not "won't scale"
- Security implications must be flagged explicitly
- No hallucinated APIs — use pseudocode or note "verify syntax" when uncertain
