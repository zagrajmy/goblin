# goblin

Kobold and Goblin are Zagrajmy's always-on agent employees. This repository
holds the agents' own code and the policy that governs them. Zagrajmy itself,
the Django app the agents work on, lives in
[zagrajmy/ludamus](https://github.com/zagrajmy/ludamus).

## What lives where

Here:

- `extensions/goblin-zagrajmy/` — a Pi extension: a thin MCP client that
  discovers the Zagrajmy MCP server's tools at startup and registers one
  agent tool per MCP tool.
- `docs/employees.md` — the decision record.
- Still to come: actor and infrastructure code, the auditor and merge-tier
  policy, routine definitions, and the Battlemap.

In ludamus: everything that describes or verifies ludamus itself — the MCP
server, `.mcp.json`, the troubleshooting runbook, the verification image, and
Parley.

## The leash

The policy that decides whether a Goblin pull request may auto-merge lives
here, not in ludamus. The Goblin GitHub App gets write access to ludamus and
none to this repository, so an agent can never open a pull request that
changes the rules judging its own pull requests. Changes to the guarded-path
list and the auditor policy are made by humans.

## Licence

AGPL-3.0, carried over from ludamus, where this code was first written.
