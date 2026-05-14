# Decisions

Append-only log of decisions that change direction. Newest first.
Every entry has a date, who decided, what changed, why, and what
surfaces it affects. Reference the relevant commit if applicable.

This log is read on wake-up: any agent reads entries newer than their
last handoff so they're current on direction changes before acting.

## Template

When adding a new entry, copy this shape:

```markdown
## YYYY-MM-DD — Short summary

**Decided by:** Name(s)
**Commit:** abc1234 (if applicable)
**Status:** active / superseded / rolled back

**What changed:**
One or two sentences on the substantive change.

**Why:**
The reasoning — what evidence prompted this, what alternatives were
weighed, why this call.

**Affects:**
Which surfaces / agents / files this changes for.
```

---

(Add your first decision entry above this line when one arises. Do not
delete this template section; it's the reference for how new entries
should look.)
