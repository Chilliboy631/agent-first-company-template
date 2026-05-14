# Operating model — agent-first company

Adapted from the operating model we use at [aweb.ai](https://aweb.ai).
The version here is generic; rewrite to match your actual work.

## What "agent-first" means

The work is done by AI agents with named responsibilities, persistent
context, and durable handoffs between them. Humans set direction, hold
founding judgment, and carry the parts that need human presence.

This is different from "AI-assisted." AI-assisted is a human using
ChatGPT to write emails faster — useful, but the AI is still serving an
individual workflow. Agent-first is the work itself being structured
around AI as the primary doer.

## Operating principles

(These are six principles we lean on. Adapt to your actual work.)

### 1. Work needs artifacts

If work matters, it needs a durable artifact: a task, a claim, a
handoff, a decision record, a release-notes draft, a verified-live
mail. Conversation is not enough. Conversations evaporate at the end
of the session; artifacts survive.

### 2. Substantial work needs two voices

A builder and a reviewer. The voices don't have to be different agents
(a code-reviewer subagent counts; a "now switch and critique what we
just decided" prompt counts), but they have to be different
perspectives.

### 3. Surfaces are owned, not walled

Within a role, you decide; across roles, you collaborate. When peers
see something differently, they work it out together. The goal is the
right call for the company, not the win.

### 4. Shared state beats status routing

The company should be queryable through artifacts. Tasks show active
work; status files publish current state; handoffs preserve area-
specific memory; decision records explain how state changed. Any fresh
agent (or human) should be able to inspect the artifacts and
understand what's happening.

### 5. Look for feedback, grade its strength

Some feedback is closing-quality ("the test passes; the customer
confirmed"). Some is weak signal ("traffic increased after the post;
attribution unclear"). Capture both. Don't claim causality the evidence
doesn't support.

### 6. Distribution over features

Zero users means nothing else matters. Once the product works, every
hour spent on more engineering instead of getting it in front of
people is wasted.

## Operating rhythms

### Wake-up

Every agent reads, in order:
1. `CLAUDE.md` (top-level)
2. `docs/team.md`, `docs/agent-first-company.md`, `docs/invariants.md`
3. Status file for their surface (`status/<surface>.md`)
4. New entries in `docs/decisions.md` since their last handoff
5. Their own `handoff.md`
6. Any in-flight messages addressed to them (if cross-AI messaging is
   set up)

### Work cycle

A typical work cycle:

1. Priority surfaces (from `lead`, a customer signal, an architectural
   read, etc.)
2. The owning agent picks up the priority — either writes the change
   themselves or scopes a brief to dispatch
3. Work happens; artifact lands on a branch or in a draft
4. Reviewer voice: a different perspective on the same work before it
   commits
5. Shipping path (runner): ops gates, deploy, verified-live
6. Customer-facing followups if any (status posts, release notes,
   support runbook updates)
7. Signal capture (what did this work produce; what's the feedback)

### Handoffs

When an agent goes idle (end of session, context-clear, etc.), update
`handoff.md` with what's in-flight. Commit and push. The next agent
session picks up from there.

## Adapting this model

The principles travel. The specific rhythms (six surfaces, the precise
rollout order, etc.) are downstream of what your company actually
does. Start with the principles; build the specific rhythms when the
work needs them.
