# Agent employees: Kobold and Goblin

Decision record for the always-on agent employees. The living plan (richer,
continuously updated, RPG-flavored) is a shared Claude artifact; this file is
the engineering summary the repo can hold us to. Status: pre-implementation —
the PostHog Prologue (analytics + error tracking the agents will watch) ships
first, the agents follow.

## Decisions

**Platform: Pi Durable on Rivet Actors.** Each agent instance is one Rivet
Actor running Pi through [Pi Durable](https://rivet.dev/agents/docs/pi/)
(`piDurable()` from `@rivet-dev/pi/durable`, over
`@earendil-works/pi-durable`). Every step — model call, tool call, task
phase — is checkpointed to the actor's SQLite before the next begins, so a
crash, an OOM or a deploy resumes the run instead of losing it. Idle actors
sleep. Coding tools (read, write, edit, bash) run in a sandbox attached to
the actor; Rivet Containers stay disposable tools, never residences. Chosen
over Cloudflare's agent stack and over managed agents for ownership: we own
the harness, the state, and the exit path, and we coast on free tiers while
Zagrajmy makes no money.

**Two agents.** *Kobold* does everything except coding and is the one that
may someday be exposed to event organizers (through organizer-scoped MCP
tokens). *Goblin* codes, is team-only, and is effectively a superset of
Kobold. Neither acts as a human: Goblin acts on GitHub as a GitHub App, not
a user account (a previous bot user got banned; Apps are the sanctioned
identity), and on Zagrajmy as its own service superuser, not as a person's
account.

**Harness: Pi, both agents.** The only Zagrajmy tool surface is the Zagrajmy
MCP server
([mcp.md](https://github.com/zagrajmy/ludamus/blob/main/docs/agents/mcp.md))
consumed through a thin discovery-based client — one Pi tool per MCP tool,
registered at startup (`extensions/goblin-zagrajmy/`). No native-tool
mirrors, no codemode layer: one surface, growth happens server-side. What
Pi Durable on Rivet changes about that:

- Extensions are `defineExtension` registries of `defineTool`s, not Pi CLI
  extension files. The bridge scaffolding targets the CLI API today and has
  to be retargeted; every community extension must be ported or replaced.
- Every tool declares whether it is safe to replay after a crash. A
  Zagrajmy MCP tool is replay-unsafe unless the server marks it read-only or
  idempotent, so a crash mid-write reports "interrupted" instead of writing
  twice.
- Pi Durable loads no skills and no `AGENTS.md`, so house context —
  `CLAUDE.md`, the glimpse rules, the Polish glossary — is supplied to the
  agent explicitly.
- It is beta and roll-forward only: versions are pinned, and an actor's
  SQLite is backed up before any upgrade.

**Gateway doctrine.** Two things are never outsourced: the auditor (an
interceptor on every tool call — policy, approvals, audit log) and the
secrets broker (an actor injecting short-lived credentials per call;
nothing secret at rest with the agent). Pi Durable has no built-in approval
flow, so approvals stay ours.

**Humans as MCP.** *Divine Intervention* (Fleshling): a real Discord DM to
Piotr or Radek, reserved for blocking-and-important calls, with GitHub
@mention as fallback. *Simulacrum*: distilled personas demoted to
disagreement skills (`/where-would-piotr-disagree`) with an apprenticeship
path — a persona graduates per-domain when its predicted disagreements match
the human's actual ones ≥95% over 20+ samples.

**Verification tiers.**

- *Trial I* — in-sandbox fail-fast checks (not load-bearing).
- *Trial II* — the pre-review gauntlet: `mise run verify` (hk lint, pytest
  on SQLite, Playwright e2e), runnable anywhere via ludamus's
  `docker/verification.Dockerfile`.
- *Trial III* — the real CI on GitHub.
- *Torment Nexus* — repeated `/thermo-nuclear-code-quality-review` rounds by
  a fresh-context reviewer until approval; round cap 5; frontier models
  only (a weak reviewer's approval is worse than none); final round adds an
  outsider lens; at spend cap it stops rather than degrades. The review
  lands as one PR comment with expandable `<details>` sections per round.
- CodeRabbit stays installed as the outer gate; PostHog watches after
  deploy.

**Merge tiers.** Routine PRs (Nexus-approved, CI green, migration-free, no
guarded paths) auto-merge with a Discord notice, a post-deploy PostHog
watch, and auto-revert. *Important* PRs — migrations, auth/authz (MCP
tokens included), consent/privacy, settings/CSP, CI/deploy, dependency
changes, or anything that took the Nexus >2 rounds — wait for a human.
Default is important.

**Staging remote.** Agents iterate against a staging git remote; GitHub
sees conclusions, not process.

**Battlemap: the operator surface.** A successor to Zagrajmy's Django Admin,
where operators run the platform and its agents. A Bun and Vite app built
with the Cloudflare Vite plugin, explicitly not Django.

**Memory: not OptChat.** OptChat's compressed-tree chat memory was
considered and is not used for either agent.

**Auth & spend.** Pi is not Claude Code, so whether a Claude subscription
token may drive it must be confirmed before relying on it. If it may, the
subscription carries the agents while pre-revenue, and the flip to API keys
happens when either revenue exists or agent usage rate-limits a human twice
in one week. If it may not, model spend is a capped API key from the start.

## Where things live

This repository, `zagrajmy/goblin`, holds the agents' own code and policy:

- `extensions/goblin-zagrajmy/` — Pi extension scaffolding (MCP bridge).

[zagrajmy/ludamus](https://github.com/zagrajmy/ludamus) keeps everything
that describes or verifies ludamus itself:

- `docker/verification.Dockerfile` + `mise run verify` — Trial II.
- PostHog Prologue: consent-gated analytics (`src/prologue.ts`), masked
  session replay (`data-ph-mask`), guarded source-map upload in the deploy.
- The Zagrajmy MCP server and its docs.

## Still open

- **pgbot.** A direct read-only database connection goes around the
  only-through-MCP rule. Either ratify a diagnostics exception, or expose
  its findings through a maintainer MCP tool.
- **Simulacrum graduation.** Graduating a persona needs an auditor
  exception, because the auditor refuses any action justified by simulated
  opinion alone.
- **Subscription or API key.** See *Auth & spend*.
- **Effect and Alchemy.** They arrived with the Cloudflare leaning for the
  agents, which Rivet now supersedes. Whether they stay for the agents'
  TypeScript, or only for the Battlemap, is undecided.
