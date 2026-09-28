# Coding Hub: architecture proposal

Status: **draft for review** (2026-09-28). Based on [research/landscape.md](../research/landscape.md) and
[research/context-skills-slack.md](../research/context-skills-slack.md).

## Goal

A desktop and web app where many AI coding agents (Claude Code, Codex, OpenCode, Gemini CLI and others) work
together on one large project across several connected VPSes. The hub gives every agent:

- the right **context**: a code graph plus a project knowledge graph (graph RAG),
- the right **skills**, measured and improved over time,
- a shared **task board** so their work fits together,
- and it can all be driven from **Slack** as well as the app.

## Decisions so far

| Topic | Decision | Source |
|---|---|---|
| Languages | **Rust** for the backend, runners and indexing. **TypeScript** for the UI and the Slack app. | Your answer |
| Client | **Desktop app (Tauri 2)**. The same React UI is also served as a web app. | Your answer |
| How we reach agents | Mainly **ACP** (Agent Client Protocol). Native SDKs (Claude Agent SDK, Codex app-server) where they give more detail. Terminal (PTY) control only as a fallback. | Research |
| How machines connect | Each VPS runs a **runner** that opens an outbound connection to the hub. No inbound ports needed and no VPN required; Tailscale is optional. | Research |
| Base project | **Build our own**, reusing permissively licensed pieces: the ACP Rust crate, codegraph, and patterns from cc-switch, OpenTag and Paseo. This replaces the earlier "start from Paseo" suggestion, because Paseo and Orca are Node/Electron. | Research + your stack choice |
| Dependency licenses | **MIT or Apache-2.0 only.** This keeps personal use and a future product both possible. | Default until you decide |

## System overview

```
 Clients         Desktop app (Tauri 2)   Web UI (same React)   Slack app   CLI
                        \                     |                 /          /
                         \______ HTTPS + WebSocket (hub API) ______________/
                                              |
 Hub (Rust)      +----------------------------------------------------------+
 runs on one     | Projects and task graph      Scheduler                   |
 VPS, always on  | Event log (every agent step) Policies, secrets, budgets  |
                 | Context service              Skills registry + evals     |
                 | MCP server for agents        Slack gateway               |
                 +----------------------------------------------------------+
                                              |
                          outbound WebSocket from each runner (token + TLS)
                     _________________________|_________________________
                    |                         |                         |
 Runners (Rust)  VPS 1 runner              VPS 2 runner              VPS N runner
                 - git worktrees per task
                 - ACP client -> claude-agent-acp | codex | opencode | gemini | goose
                 - code graph indexer (codegraph)
                 - skills synced into each agent's folder
                 - PTY fallback for agents without ACP
```

### Hub (Rust: axum + tokio + sqlx)

- **Projects and task graph.** Epics split into tasks with dependencies (a DAG). Each task has a spec, an
  owner agent, a runner, a branch and a status.
- **Scheduler.** Assigns ready tasks to an agent on a runner, based on agent strengths, runner capacity,
  where the repo is already checked out, and the cost budget.
- **Event log.** Every agent step (messages, tool calls, diffs, permission requests, tokens, cost) is stored
  as a structured event. The UI, Slack, cost tracking and skill evaluation all read from this one log.
- **Policies, secrets and budgets.** Which agents may run which commands, API keys stored once and handed to
  runners, spending limits per project and per task.
- **MCP server for agents.** Every agent gets hub tools:
  - `tasks.create_subtask`, `tasks.comment`, `tasks.complete`
  - `context.query` (code graph + project graph), `memory.search`, `memory.write`
  - `ask_human` (the question goes to the task's Slack thread and the desktop app)
- **Storage.** SQLite first, with Postgres as an option once there are several users.

### Runner (Rust daemon, one per VPS)

- Connects out to the hub, registers its capacity and the agents it has installed, and reconnects on its own.
- Creates a **git worktree per task**, so agents never step on each other.
- Starts agents through **ACP** (Rust `agent-client-protocol` crate) and streams their structured events to the hub.
- Runs the **code graph indexer** for each checked-out repo and serves queries locally, which is faster.
- Syncs approved **skills** into each agent's skills folder before a task starts.
- Optional: runs each task in a container for isolation.

### Clients (TypeScript)

- **Desktop (Tauri 2):** task board, live agent sessions, diff review, approvals, code graph explorer,
  skills manager and cost view. It can also run a local runner, so your laptop joins the fleet.
- **Web:** the same React app, served by the hub.
- **Slack app (Bolt, Socket Mode):**
  - Each channel maps to a project. An @mention creates a task, and that **thread becomes the task**.
  - Progress is shown by editing a single status message.
  - Buttons approve plans, permission requests and merges.
  - Commands: `/hub status`, `/hub assign`, `/hub budget`, plus a daily summary.

## How agents cooperate on a large project

1. **Plan.** A planner agent turns a goal into an epic and a task graph. It writes interfaces and contracts
   first (API shapes, module boundaries) so parallel tasks don't collide. You approve the plan in the app or
   in Slack.
2. **Assign.** The scheduler hands ready tasks to agents on runners. Different agents suit different tasks,
   for example one model for planning and review and another for bulk implementation.
3. **Prepare context.** Before a task starts, the hub builds a context pack: the task spec, the relevant slice of
   the code graph, linked decisions and past tasks, and the selected skills.
4. **Work.** The agent works in its own worktree. It can ask the hub for more context, create subtasks, or ask
   a human.
5. **Review.** A different agent (ideally a different model) reviews the diff, and CI runs. Failures go back to
   the author agent.
6. **Merge.** A merge queue rebases and merges finished tasks. Conflicts reopen the task.
7. **Learn.** The outcome (tests, review verdict, tokens, time) is stored against the task, the agent and the
   skills used. This feeds skill optimization and future scheduling.

## Context and graph RAG

| Layer | Contents | Built by |
|---|---|---|
| Code graph | Symbols, files, calls, imports, and tests linked to the code they cover, per repo and branch | **codegraph** (MIT, Rust core, SQLite, live sync) on each runner. Graphify added later for docs, SQL and infra. |
| Project graph | Tasks, decisions (ADRs), PRs, reviews and Slack threads, **linked to code nodes** | Our own work, stored in the hub. |
| Memory | Lessons from past tasks ("this module needs X", "tests here need Y"), linked to graph nodes | Our own work: distilled from the event log after each task. |

Agents query all three through one MCP tool (`context.query`). We will measure the effect on real tasks
(success rate, tokens, tool calls) against a no-graph baseline, instead of relying on the projects' own claims.

## Skills

1. **Registry.** Versioned `SKILL.md` packs (Agent Skills format), scoped globally or per project.
2. **Safety.** Every new or changed skill is scanned (SkillSpector) before a runner receives it.
3. **Sync.** Runners install the approved versions into each agent's skills folder.
4. **Evaluate.** Each skill version is scored on real task outcomes from the event log, plus small eval sets.
5. **Optimize.** A GEPA-style loop proposes better versions from failure traces. A human approves before a
   new version goes live. This follows the approach of hermes-agent-self-evolution.

## Repository layout (planned)

```
crates/
  protocol/   shared types: hub <-> runner messages and events (TS types generated with ts-rs)
  hub/        control plane server
  runner/     per-VPS daemon
  context/    graph queries, context packs, project graph
  cli/        `hub` command-line tool
apps/
  desktop/    Tauri 2 shell
  web/        React UI (used by desktop and web)
  slack/      Slack app (Bolt, TypeScript)
packages/
  sdk/        TypeScript client for the hub API
  ui/         shared UI components
skills/       first-party skills
docs/
```

## Roadmap

| Milestone | Result you can see |
|---|---|
| **M0 Scaffold** | Cargo + pnpm workspaces, CI (format, lint, tests, typecheck), shared protocol types. |
| **M1 Core loop** | One runner on one VPS. Start Claude Code (via claude-agent-acp) in a worktree from the web UI, watch it work live, approve its permission requests. |
| **M2 Fleet** | Several VPSes and agents (add Codex, OpenCode, Gemini CLI). Task board, scheduler, diff review, merge. |
| **M3 Slack** | Tasks from Slack threads, live status, approval buttons, `ask_human` routed to the thread. |
| **M4 Desktop** | Tauri app wrapping the UI, with an optional local runner. |
| **M5 Context** | codegraph on runners, `context.query` MCP tool, context packs, and measurements against a baseline. |
| **M6 Skills** | Registry, scanning, sync, evaluation, then the optimization loop. |
| **M7 Large projects** | Planner, task DAG with contracts, merge queue, per-project budgets. |

## Risks

- **Scope.** This is a big system. Each milestone must work end-to-end before the next one starts.
- **Agent terms of use.** Check each provider's terms before running subscription logins (for example
  Claude Pro/Max or ChatGPT plans) automatically on many machines. API keys avoid most of this.
- **Uneven ACP support.** Some agents expose less over ACP than their native SDK. Keep PTY and native
  adapters as fallbacks.
- **Security.** Agents have a shell on the VPSes, and Slack becomes a control channel. Map Slack users to hub
  permissions, keep secrets out of agent-visible files, and require approval for risky commands.

## Open questions

1. Is this for your own use or your team's, or a product for others? This decides our own repo license
   (proposed: Apache-2.0).
2. Which agents must work in M1 and M2? Assumed: Claude Code, Codex, OpenCode, Gemini CLI.
3. Which VPS provider, and how many machines to start with? Tailscale is optional in this design.
4. Should the hub run on one of the VPSes? That is needed for Slack to work around the clock.
