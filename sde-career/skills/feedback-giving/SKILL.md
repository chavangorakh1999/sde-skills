---
name: feedback-giving
description: "Give effective feedback: SBI model (Situation-Behavior-Impact), written peer review feedback, handling difficult conversations, and giving upward feedback. Use when writing peer feedback or preparing to give feedback."
---

## Feedback Giving

### Context

Feedback situation or person to give feedback to: **$ARGUMENTS**

---

### The SBI Model

```
Situation:  When and where — specific instance, not a pattern accusation
Behavior:   What they DID — observable, not interpretive
Impact:     The effect it had — on you, the team, the project

This model:
  - Makes feedback specific (not "you always...")
  - Focuses on behavior (not personality)
  - Explains the impact (gives context for why it matters)
```

---

### SBI Examples

```
WEAK feedback:
  "You're not collaborative enough."
  "Your code quality needs improvement."
  "You interrupted me in the meeting."

STRONG feedback (SBI):

(Positive)
"In the incident on Thursday [S], you stayed calm under pressure and walked the team
through the rollback steps methodically [B]. That kept everyone focused and we resolved
it 40% faster than our usual P1 time [I]."

(Constructive)
"In yesterday's architecture review [S], you interrupted Alice mid-sentence twice while
she was explaining her proposal [B]. She visibly disengaged after the second time, and
two design options she raised weren't discussed [I]."

(Constructive — technical)
"In the pull request for the auth service refactor [S], the error handling in the token
refresh endpoint wasn't tested [B]. This caused the silent token expiry bug we hit in
staging, which blocked the QA team for half a day [I]."
```

---

### Written Peer Review (Performance Cycle)

```
Format for peer review responses:

## Strengths
[2-3 specific examples using SBI — not just "great communicator"]

## Areas for Growth
[1-2 specific observations — critical but constructive]

## Summary
[1-2 sentences for the reviewer to use in the promo/perf narrative]

---

Example (for a Senior SWE):

## Strengths

**Technical leadership:** During the Q3 migration to PostgreSQL, Alice owned the
schema design and wrote an RFC that prevented three data consistency bugs we would
have discovered in production. Her habit of writing thorough tech specs has measurably
raised the quality of our design reviews.

**Cross-team influence:** When the payments team was about to implement their own
auth layer, Alice proactively reached out, explained our existing library, and
convinced them to adopt it instead. This saved ~2 sprint weeks of duplicated work.

## Areas for Growth

**Scoping ambiguity early:** On the notification service project, Alice dove into
implementation before requirements from the PM were finalized. When requirements
changed in week 3, we had to undo significant work. Earlier alignment conversations
would have prevented this.

## Summary
Alice consistently operates at the Senior level through strong technical ownership
and proactive cross-team collaboration. Her most impactful growth opportunity is
getting explicit alignment on requirements before building.
```

---

### Difficult Feedback Conversations

```
The COIN model for harder conversations:
  Context:   Set the stage — "I want to talk about something I observed..."
  Observation: Specific behavior — "I noticed that in the last 3 PRs..."
  Impact:    Effect — "The effect was..."
  Next Steps: What you're asking for — "Going forward, could you..."

Before the conversation, ask yourself:
  - Is this a pattern or a one-time thing? (One-time → address it but lightly)
  - What behavior change am I actually asking for?
  - Is there context I might be missing? (Assume positive intent first)

During:
  1. Share the observation — pause and let them respond
  2. Listen — they may have context that changes the picture
  3. Explore impact together — "Does that make sense from your perspective?"
  4. Agree on next steps — don't just vent; what changes?

Common pitfalls:
  - Feedback sandwich (positive-negative-positive) → the negative gets lost
  - "No offense but..." → it IS an offense
  - "I feel like you don't care" → this is an interpretation, not an observation
  - Waiting until performance review to give critical feedback → too late
```

---

### Upward Feedback (To Your Manager)

```
Giving feedback upward is valuable but requires care:

When to give upward feedback:
  - During 1:1 when asked "Is there anything I could do better?"
  - In a dedicated skip-level or team health survey
  - When the issue affects your ability to do your job (blockers, not preferences)

How to frame it:
  - SBI still applies — specific situation, behavior, impact
  - Lead with intent: "I'm sharing this because I want us to work well together"
  - Frame as your experience, not a character judgment

Example:
  "In the sprint planning meeting on Monday [S], the scope of the API project expanded
  mid-meeting without a decision point [B]. I wasn't sure what to implement next, and
  the team had 3 different understandings by EOD [I]. Would it help to have a brief
  written summary after planning meetings? I'd be happy to draft those."

  (Note: ends with a solution offer — not just a complaint)
```

---

### Feedback Timing

```
Give positive feedback: immediately — don't wait for the quarterly cycle
Give constructive feedback: within 48 hours of the incident while it's fresh
Give feedback in perf cycles: amplify what you've already said in person

Don't save up feedback for the review cycle.
People can't change behavior they don't know about.
The review is for calibration, not for delivering surprises.
```
