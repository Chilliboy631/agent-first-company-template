# agent-first company template

A forkable scaffolding for setting up an agent-first company — the
companion artifact to the blog post [How to set up an AI-first company
like ours](https://aweb.ai/blog/ai-first-company-howto/).

## What this is

A starter directory structure with the operating documents we use at
[aweb.ai](https://aweb.ai) to run a company where most of the work is
done by named AI agents with persistent contexts, durable handoffs, and
direct AI-to-AI messaging.

This template gives you the **structure**. The principles in the blog
post are what make any structure work — the templates here are tools,
not prescriptions.

## What's in here

```
.
├── README.md               You are here
├── CLAUDE.md               Top-level company instructions for every agent
├── AGENTS.md               How agents work in this team
├── docs/
│   ├── agent-first-company.md   The operating model
│   ├── invariants.md            Guiding principles that hold across decisions
│   ├── team.md                  Your team's role-and-responsibility map
│   ├── decisions.md             Append-only log of direction changes (template)
│   └── voice.md                 Voice norms for external-facing copy (template)
├── status/
│   ├── lead.md             Per-surface status file (template)
│   ├── builder.md          Per-surface status file (template)
│   ├── runner.md           Per-surface status file (template)
│   └── weekly.md           Cross-surface weekly roll-up (template)
└── agents/
    ├── lead/               Direction / priorities / final calls
    ├── builder/            The work itself (code, content, whatever you build)
    └── runner/             Ops, ship, verify, dashboards
```

Three example agents — `lead`, `builder`, `runner` — to show the
per-agent file shape. The blog post explains the principles. Add more
agents (named for your work surfaces) when the work needs them; don't
impose six surfaces on a five-person team.

## How to use it

### Stage 1 — Two voices (3-5 person team)

You don't need to fork this template to start. The first stage is just
discipline:

1. Pick the most-used AI in your team (probably ChatGPT or Claude).
2. For substantial decisions, switch into a "now critique what we just
   decided" prompt before you act. That's your reviewer voice.
3. Write the decision down somewhere durable — a Notion page, a Google
   doc, a markdown file in your repo.

Stage 1 needs no new tools. Build the habit first.

### Stage 2 — Named surfaces (5-15 person team)

Fork this template and start adapting:

1. Edit `docs/team.md` with your actual role mapping (which work areas
   are explicit surfaces in your team).
2. Rename `agents/lead/`, `agents/builder/`, `agents/runner/` to match
   your roles. Add more if needed. Drop ones that don't apply.
3. Per agent, edit `CLAUDE.md` to describe that role's responsibilities
   and on-wake-up routine.
4. Each agent's persistent context lives in its `agents/<name>/`
   directory — `CLAUDE.md` for the role definition, `handoff.md` for
   in-session state.

Stage 2 still doesn't need cross-AI messaging. Humans relay between
agent personas.

### Stage 3 — Cross-AI coordination

When the relay overhead between agent personas starts to hurt, give
each AI an address.

The `aw` CLI ([github.com/awebai/aweb](https://github.com/awebai/aweb))
is how we do this. Install it; each agent gets a `.aw/workspace.yaml`
in its directory; agents can mail and chat each other directly.

For consumer-AI teams (using ChatGPT, Claude Desktop, claude.ai), the
hosted MCP at [app.aweb.ai/connect](https://app.aweb.ai/connect) is the
easiest start.

This template doesn't auto-set-up the `aw` workspace files — those are
specific to your team's namespace and identities. See the aweb docs for
the setup walkthrough.

### Stage 4 — Joint responsibility

The operating discipline (decision records, status files per surface,
written handoffs, build/ship boundary, two-voice review on substantial
work) is what makes the team-as-system work. The infrastructure from
Stage 3 helps; the habits are harder.

## What to change in your fork

- `CLAUDE.md` — replace REPLACE_ME tokens with your company's name,
  mission, and operating norms
- `docs/team.md` — replace with your team's role-and-responsibility map
- `docs/agent-first-company.md` — read it; keep what applies; rewrite
  what doesn't
- `docs/invariants.md` — same; the principles in the blog post are the
  durable claims, the specific invariants are yours to author
- `agents/*/CLAUDE.md` — per-agent role definition
- Remove or rename agents you don't need; add new ones with the same
  file shape

## What this template explicitly DOESN'T include

- A specific tech stack — the operating model is independent of whether
  you ship code, content, designs, or sales calls
- A specific AI provider — works with ChatGPT, Claude, Gemini, or any
  AI that supports a system-prompt / project / persistent-context shape
- A specific scale — for a 3-person team most of this is overkill; for
  a 30-person team you probably need more

## License

MIT. Fork freely. Adapt freely. Tell us how it goes:
[hello@aweb.ai](mailto:hello@aweb.ai).
