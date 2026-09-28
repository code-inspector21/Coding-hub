# Agentic Coding Hub: GitHub Landscape Research

Snapshot date: **2026-09-28**. Star counts are rounded from GitHub search on that date.
Licenses marked *verify* were not confirmed during this pass.

---

## TL;DR

1. **Most of this already exists in pieces.** Tools that run many CLI coding agents in parallel
   (Claude Code, Codex, OpenCode, Gemini CLI, and so on) now have a category name:
   **ADE (Agentic Development Environment)**. A typical ADE runs each task in its own
   git worktree, on your machine or a remote one, with desktop and mobile clients.
2. **Closest existing matches to what we want:** Orca (MIT, ~80k★), Paseo (Apache-2.0, ~19k★),
   Agent Orchestrator (Apache-2.0, ~12k★) and Omnigent (Apache-2.0, ~10k★).
   AgentsMesh has the best multi-machine design, but its license is BSL, which bars
   production use without a commercial license until 2030. Superset uses Elastic-2.0,
   which is source-available, not open source.
3. **We can't merge Claude Code's source into our repo.** Its license reads
   *"© Anthropic PBC. All rights reserved."* We can run it through its CLI, the Claude Agent SDK
   or an ACP adapter, but we can't fork it. Codex (Apache-2.0), OpenCode (MIT), Gemini CLI, Goose, Qwen Code and pi can all be forked.
4. **Shared protocols are how the agents connect, not merged source code.** ACP connects editors to agents, MCP
   connects agents to tools, and the new UHP (from HarnessRouter) is a server-side API for agent
   harnesses. If our hub speaks these protocols, every agent plugs in without us owning its code.
5. **The gap we can fill:** a self-hosted **fleet** control plane under a permissive license. It
   would have one runner daemon per VPS, a private network between them, and one UI for web,
   desktop and phone. Today the options are BSL (AgentsMesh), desktop apps that treat remote
   machines as SSH targets (Orca, Emdash, Superset), or one daemon per machine without a
   scheduler across machines (Paseo).
6. **Recommendation:** don't start from zero. Main candidate for the base: **Paseo**. Runner-up:
   **Orca**. Use **AgentsMesh** as a design reference only; don't copy its BSL code. Drive agents over
   **ACP** and add the multi-VPS runner and scheduler as our own contribution. Before committing to a base, try both candidates on 2 real VPSes (see [Next steps](#next-steps)).

---

## The system we want, broken into layers

| # | Layer | What it does | Best open-source options found |
|---|-------|--------------|------------------------------|
| 1 | Agent harnesses | The coding agent itself: agent loop, tools, model calls | OpenCode, Codex, Claude Code (CLI/SDK only), Gemini CLI, Goose, pi, Cline, Aider |
| 2 | Protocols | Standard way to drive any agent or tool | ACP, MCP, UHP, Codex app-server, Claude Agent SDK, AGENTS.md / Agent Skills |
| 3 | Orchestration / ADE | Runs parallel agents: worktrees, task board, review/merge | Orca, Paseo, Agent Orchestrator, Omnigent, herdr, Emdash |
| 4 | Editor surface | The Cursor-like UI where you open files and review diffs | Zed (ACP host), code-server, Theia; VS Code extensions (Cline, Kilo, Continue) |
| 5 | Multi-machine | Connects N VPSes: runners, relay, network mesh | Paseo daemon + relay, AgentsMesh (reference only), Coder, Tailscale/Headscale |
| 6 | Isolation | Sandboxes and microVMs per agent/task | E2B, container-use, Daytona (*verify license*), arcbox, forkd, clawk |
| 7 | Model routing | Any model behind any agent, plus cost tracking | LiteLLM, claude-code-router, opencodex |
| 8 | Memory, context, skills | Cross-session memory, code search, skill packs | claude-mem, serena, headroom, superpowers, agent-skills |

---

## 1. Agent harnesses

| Project | ★ | License | Notes |
|---|---|---|---|
| [anomalyco/opencode](https://github.com/anomalyco/opencode) | 211k | MIT | Most complete agent we can fork. Monorepo has `server`, `sdk`, `client`, `tui`, `desktop`, `web`, `protocol` and `containers` packages, so it is built client/server. Works with any provider. |
| [anthropics/claude-code](https://github.com/anthropics/claude-code) | 148k | **Proprietary** | The repo holds plugins, docs and issues, not source you can fork. Integrate via CLI, [claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python) / [-typescript](https://github.com/anthropics/claude-agent-sdk-typescript), or an ACP adapter. |
| [openai/codex](https://github.com/openai/codex) | 127k | Apache-2.0 | Written in Rust. Crates include `app-server`, `app-server-daemon`, `exec`, `exec-server`, `cloud-tasks`, `linux-sandbox`, `ollama`, `lmstudio` and `model-provider`, so it can run headless, run remotely, and use local models. |
| [earendil-works/pi](https://github.com/earendil-works/pi) | 110k | MIT | Toolkit: unified LLM API (`pi-ai`), `pi-agent-core`, TUI and a coding agent. Good as a library if we write our own harness. |
| [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli) | 107k | Apache-2.0 | Terminal agent. MCP client and server. |
| [OpenHands/OpenHands](https://github.com/OpenHands/OpenHands) | 89k | MIT | Self-hosted "control center for coding agents", with local, remote and cloud runtimes. Also ships [software-agent-sdk](https://github.com/OpenHands/software-agent-sdk). |
| [cline/cline](https://github.com/cline/cline) | 69k | Apache-2.0 | Available as an SDK, an IDE extension or a CLI. |
| [aaif-goose/goose](https://github.com/aaif-goose/goose) | 55k | Apache-2.0 | Written in Rust, with native MCP and ACP support. |
| [Aider-AI/aider](https://github.com/Aider-AI/aider) | 49k | Apache-2.0 | Git-first pair programmer in the terminal. |
| [continuedev/continue](https://github.com/continuedev/continue) | 36k | Apache-2.0 | IDE and CLI agent. |
| [QwenLM/qwen-code](https://github.com/QwenLM/qwen-code) | 28k | Apache-2.0 | Terminal agent tuned for Qwen models. |
| [charmbracelet/crush](https://github.com/charmbracelet/crush) | 28k | *verify* | Go TUI agent. |
| [Kilo-Org/kilocode](https://github.com/Kilo-Org/kilocode) | 27k | MIT | VS Code and JetBrains extension plus CLI. |
| [plandex-ai/plandex](https://github.com/plandex-ai/plandex) | 16k | *verify* | Go. Built for large tasks. |

General-purpose agents that most orchestrators also support: [openclaw/openclaw](https://github.com/openclaw/openclaw)
(391k★) and [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) (250k★). Neither is
built mainly for coding, but users will expect them to plug in.

> **Claude Code caution:** several repos created around 2026-03-31 say they analyse or
> re-implement Claude Code's source. The official license is all-rights-reserved, so **don't
> copy code from those repos into ours.** Integrate the official CLI/SDK instead.

---

## 2. Protocols

| Protocol | Connects | Why it matters for us |
|---|---|---|
| **MCP** | agent ↔ tools/data | Every harness supports it. Our hub can expose its own tools (tasks, other machines, memory) as an MCP server. |
| **ACP** (Agent Client Protocol, started by Zed) | client/editor ↔ agent, JSON-RPC over stdio | The [ACP registry](https://github.com/agentclientprotocol/registry) lists 50+ agents, including Claude (via adapter), Copilot, Cursor, Cline, Devin, Gemini, Mistral Vibe and Qwen Code. **This is the best single way to drive every agent from one hub.** |
| **UHP** (Unified Harness Protocol) | backend ↔ harness, REST + SSE | From [HarnessRouter](https://github.com/HarnessRouter/harnessrouter) (Apache-2.0, 2.7k★, Aug 2026). Serves an OpenAI Responses-compatible `/v1/responses` API in front of Codex, Claude Code, Hermes, PI and DSH. |
| **Codex app-server** | client ↔ Codex | Codex's own JSON-RPC protocol, used by its IDE and desktop clients. |
| **Claude Agent SDK** | code ↔ Claude Code | Official way to run Claude Code from code. |
| **AGENTS.md / Agent Skills** | repo ↔ any agent | Portable instructions and skill packs that work across agents. |

Useful ACP building blocks:
- [openclaw/acpx](https://github.com/openclaw/acpx) (3.3k★): headless CLI client for stateful ACP sessions.
- [formulahendry/acp-ui](https://github.com/formulahendry/acp-ui) (486★): ACP client for desktop, mobile and web.
- [formulahendry/vscode-acp](https://github.com/formulahendry/vscode-acp): any ACP agent inside VS Code.
- [coder/acp-go-sdk](https://github.com/coder/acp-go-sdk), [marimo-team/use-acp](https://github.com/marimo-team/use-acp) (React hooks), [zvzuola/acp-components](https://github.com/zvzuola/acp-components) (UI kit).
- [langgenius/mosoo-agent-driver](https://github.com/langgenius/mosoo-agent-driver): one bridge over the Claude Agent SDK, the Codex app-server and ACP.
- [BrokkAi/mjolnir](https://github.com/BrokkAi/mjolnir) (64★): Rust control plane for ACP agents, with durable sessions, isolated environments, quotas and remote control. Tiny, but its architecture is the closest to our idea.

**Recommendation:** drive agents with ACP first. Use the Codex app-server or Claude Agent SDK only where they give
more than ACP. Use MCP for tools. Expose a UHP-compatible HTTP API from our hub so other tools can drive it.

---

## 3. Orchestrators / ADEs (the "Cursor for agentic coding" layer)

| Project | ★ | License | Surfaces | Remote / multi-machine | Notes |
|---|---|---|---|---|---|
| [stablyai/orca](https://github.com/stablyai/orca) | 80k | MIT | Desktop (Electron), iOS, Android | SSH worktrees with auto-reconnect and port forwarding. A cloud relay pairs the mobile app with the desktop. | Supports 40+ agents. Uses worktrees and GPU-rendered terminals; has a design mode with embedded Chromium. About 7k open issues, so it changes fast and a fork would be hard to keep in sync. |
| [herdrdev/herdr](https://github.com/herdrdev/herdr) | 41k | Apache-2.0 | TUI (Rust) | Saved SSH machines, one combined agent list, reconnects each machine separately | Terminal multiplexer built for agents. Sessions survive disconnects and restarts. |
| [iOfficeAI/AionUi](https://github.com/iOfficeAI/AionUi) | 33k | *verify* | Desktop and web UI | – | "Cowork" app for 20+ CLI agents. |
| [BloopAI/vibe-kanban](https://github.com/BloopAI/vibe-kanban) | 28k | Apache-2.0 | Web | Docker self-host, SSH | **Being shut down.** Borrow ideas, but don't build on it. |
| [manaflow-ai/cmux](https://github.com/manaflow-ai/cmux) | 27k | *verify* | macOS terminal | – | macOS only. |
| [getpaseo/paseo](https://github.com/getpaseo/paseo) | 19k | Apache-2.0 | Desktop, iOS, Android, web, CLI | **A daemon on every machine** with a WebSocket API. Devices pair through an end-to-end encrypted relay, TCP or Tailscale. | Node daemon with MCP support, plugins and skills. **Closest open-source match to "multiple VPS, all connected".** |
| [superset-sh/superset](https://github.com/superset-sh/superset) | 15k | Elastic-2.0 | Desktop, CLI, iPhone | "Connect another machine" | Source-available only: the license bars offering it as a hosted service. Not a good base. |
| [Untrivial-ai/agent-orchestrator](https://github.com/Untrivial-ai/agent-orchestrator) | 12k | Apache-2.0 | Desktop, web, mobile | Cloud agents | Covers planning through merge, with a kanban board and 28 agents. Built with Go and Electron. |
| [omnigent-ai/omnigent](https://github.com/omnigent-ai/omnigent) | 10k | Apache-2.0 | Terminal, web, phone, desktop | Cloud sandboxes (Modal, E2B, Kubernetes) | A layer that runs any of several agent harnesses, with policies, spend limits and multi-user co-driving. Python. |
| [smtg-ai/claude-squad](https://github.com/smtg-ai/claude-squad) | 8.5k | *verify* | TUI (tmux) | – | Simple and well known. |
| [generalaction/emdash](https://github.com/generalaction/emdash) | 5.9k | Apache-2.0 | Desktop | SSH/SFTP | Linear and Jira integration. Local-first. |
| [AgentsMesh/AgentsMesh](https://github.com/AgentsMesh/AgentsMesh) | 2.4k | **BSL-1.1** (becomes GPL-2.0+ in 2030) | Web, Electron, iOS | **Control plane (Go, gRPC with mTLS), a stateless relay for terminal traffic (WebSocket), and self-hosted runner daemons** that start isolated PTY pods | Exactly the multi-VPS fleet design, but it can't be used in production without a commercial license. **Study the design; don't copy the code.** |
| [coder/xum](https://github.com/coder/xum) | 2k | *verify* | Desktop | – | From Coder: isolated, parallel agent work. |
| [kdlbs/kandev](https://github.com/kdlbs/kandev) | 858 | *verify* | Web, TUI | Self-hostable | Kanban built on ACP. |
| [Ark0N/Codeman](https://github.com/Ark0N/Codeman) | 773 | *verify* | Web | Self-hosted, runs 24/7 | tmux-based. |
| [vicoa-ai/vicoa](https://github.com/vicoa-ai/vicoa) | 383 | *verify* | Desktop, mobile | "…from desktop, mobile, VPS" | New (Aug 2026), with nearly the same pitch as ours. |

Remote clients for agents that are already running:
- [slopus/happy](https://github.com/slopus/happy) (24k★): mobile and web client for Claude Code and Codex, with end-to-end encryption and voice.
- [siteboon/claudecodeui](https://github.com/siteboon/claudecodeui) (CloudCLI, 14k★, AGPL-3.0).
- [K9i-0/ccpocket](https://github.com/K9i-0/ccpocket).

Running index of new orchestrators: [andyrewlee/awesome-agent-orchestrators](https://github.com/andyrewlee/awesome-agent-orchestrators).

---

## 4. Editor surface (the "Cursor" part)

- **VS Code forks are a trap.** [voideditor/void](https://github.com/voideditor/void)
  (29k★) was the best open-source Cursor clone and is now **archived**. [Roo Code](https://github.com/RooCodeInc/Roo-Code) is
  archived too. Keeping a VS Code fork up to date is expensive.
- **[zed-industries/zed](https://github.com/zed-industries/zed)** (91k★, mixed per-crate licenses): created ACP and is
  the open editor best prepared for agents. Treat it as a client that connects to our hub over ACP, not as something to fork.
- **[coder/code-server](https://github.com/coder/code-server)** (80k★, MIT): VS Code in the browser. Install it on
  each VPS so you can open and edit files remotely.
- **[eclipse-theia/theia](https://github.com/eclipse-theia/theia)** (22k★, EPL-2.0): a framework for building
  your own cloud or desktop IDE, if we ever need one.
- **Extensions** (Cline, Kilo, Continue, vscode-acp) bring agents into VS Code and JetBrains, which people already use.

**Recommendation:** don't fork VS Code. The hub is a purpose-built agent UI with a task board, sessions, diffs and
approvals. It opens files through code-server on the relevant VPS. Zed, JetBrains and VS Code users attach over ACP.

---

## 5. Multi-machine and isolation infrastructure

- [coder/coder](https://github.com/coder/coder) (17k★, AGPL-3.0): "secure environments for developers and their agents".
  It creates workspaces on any cloud or VPS from Terraform. Useful if we want the hub to create VPSes itself.
- [coder/agentapi](https://github.com/coder/agentapi) (HTTP API over Claude Code, Goose, Aider, Gemini, Amp and Codex) is
  **archived**. It's another sign that the industry is moving to ACP.
- [daytonaio/daytona](https://github.com/daytonaio/daytona) (72k★): sandbox infrastructure for AI-generated code.
  GitHub detects **no license** on it now, so check before depending on it.
- [e2b-dev/E2B](https://github.com/e2b-dev/E2B) (14k★, Apache-2.0) and [e2b-dev/runtime](https://github.com/e2b-dev/runtime) (Firecracker).
  [BitMiracle-AI/Dormice](https://github.com/BitMiracle-AI/Dormice) is a self-hosted, E2B-compatible alternative.
- [dagger/container-use](https://github.com/dagger/container-use) (4k★, Apache-2.0): a container environment per agent, exposed over MCP.
- MicroVMs: [arcbox](https://github.com/arcboxlabs/arcbox) (4.3k★), [forkd](https://github.com/deeplethe/forkd) (2.9k★),
  [zeroboot](https://github.com/zerobootdev/zeroboot) (2.5k★), [clawk](https://github.com/clawkwork/clawk) (1k★, "a disposable Linux VM for coding agents").
- Networking between VPSes: Tailscale/Headscale or plain WireGuard for the private network (Paseo already supports Tailscale),
  or an end-to-end encrypted relay like Paseo's and Happy's so phones never need open ports.

---

## 6. Model routing

- [BerriAI/litellm](https://github.com/BerriAI/litellm) (60k★): AI gateway for 100+ providers, with cost tracking and load balancing.
- [musistudio/claude-code-router](https://github.com/musistudio/claude-code-router) (37k★): points Claude Code at any model.
- [lidge-jun/opencodex](https://github.com/lidge-jun/opencodex) (17k★): provider proxy for Codex and Claude Code.
- [tbphp/gpt-load](https://github.com/tbphp/gpt-load) (7k★) and [kittors/CliRelay](https://github.com/kittors/CliRelay) (1k★): self-hosted gateways for coding CLIs.

## 7. Memory, context, skills and observability (add-ons, not core)

- Memory: [claude-mem](https://github.com/thedotmack/claude-mem) (95k★), [agentmemory](https://github.com/rohitg00/agentmemory) (29k★), [TencentDB-Agent-Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory) (27k★).
- Code understanding: [oraios/serena](https://github.com/oraios/serena) (30k★), an MCP server that gives agents IDE-style code search and editing through language servers.
- Context compression: [headroom](https://github.com/headroomlabs-ai/headroom) (74k★), [context-mode](https://github.com/mksglu/context-mode) (24k★).
- Skills: [obra/superpowers](https://github.com/obra/superpowers) (292k★), [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) (100k★). Scan skills before installing them with [NVIDIA/SkillSpector](https://github.com/NVIDIA/SkillSpector) (18k★).
- Observability: [openlit](https://github.com/openlit/openlit), [abtop](https://github.com/graykode/abtop), [agent-console](https://github.com/LockedinLabs-AI/agent-console).
- Evidence that people want to combine agents: [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) (34k★) lets you call Codex from inside Claude Code for reviews and delegated tasks.

---

## Licensing cheat sheet

| Bucket | What it means | Projects |
|---|---|---|
| **Permissive** (MIT, Apache-2.0) | Fork, embed or relicense freely | Orca, Paseo, OpenCode, Codex, pi, Agent Orchestrator, Omnigent, herdr, Emdash, HarnessRouter, code-server, Kilo, Goose, Cline, Gemini CLI, E2B |
| **Copyleft** (AGPL-3.0, EPL-2.0, GPL) | Can be used, but modifications must be shared. AGPL also covers software offered over a network. | coder/coder, CloudCLI, Theia, Zed (mixed) |
| **Source-available** (BSL, Elastic) | Not open source, so avoid as a base | AgentsMesh (BSL-1.1), Superset (Elastic-2.0) |
| **Proprietary** | Integrate only | Claude Code CLI |

---

## Where the gaps are (what we would add)

1. **A fleet control plane under a permissive license.** Runners on N VPSes register with one hub. The hub
   schedules tasks to machines by capacity and labels, and survives disconnects. Only AgentsMesh does all of this, and it's BSL.
2. **Driving agents over protocols.** Most ADEs read and write each CLI's terminal (PTY). Driving agents over
   ACP, the Codex app-server or the Agent SDK gives structured events: tool calls, diffs, permission prompts and token counts.
   That makes better approvals, review views and cost tracking possible. Few ADEs do it today (kandev, mjolnir).
3. **Workflows across agents.** For example, one agent plans, a second implements and a third reviews, all sharing
   one task board and one memory. The demand is real (codex-plugin-cc has 34k★), but support is patchy.
4. **One set of policies, secrets and cost limits across all machines and agents.**
5. **Phone access without exposing ports,** using an end-to-end encrypted relay or a private tailnet.

---

## Options for the base

| Option | Pros | Cons |
|---|---|---|
| **A. Fork Orca** (MIT, TS/Electron) | Most complete UX and largest community. Mobile apps and SSH remotes already exist. | The desktop app is the hub and VPSes are just SSH targets. With ~7k open issues it changes fast, so a fork would drift quickly. |
| **B. Build on Paseo** (Apache-2.0, TS) | Its daemon-per-machine design already matches "multiple VPS, all connected". Has an end-to-end encrypted relay, Tailscale support, clients for desktop, mobile, web and CLI, and plugins. | Smaller project. Supports 5 agents today. No scheduler across machines yet. |
| **C. Build a thin hub from protocols** | Clean design we fully own. Runner daemon on each VPS speaks ACP to agents; code-server for files; Headscale/Tailscale network; our own web UI. We can borrow Apache-licensed code from Agent Orchestrator and Omnigent. | The most work before anything is usable. |

**Main candidate: B, then grow toward C.** Start from Paseo and send changes upstream where it makes sense. Build our
runner and scheduler as a separate package that speaks ACP and serves a UHP-compatible API. Use AgentsMesh's
control-plane/relay/runner split as the design reference, without copying its BSL code.

---

## Next steps

1. **Hands-on trial (1–2 days):** install the Paseo daemon on 2 VPSes, and separately connect Orca to the
   same 2 VPSes over SSH. On both, run Claude Code, Codex and OpenCode on one test repo. Compare setup effort,
   reconnect behaviour, phone access and how easy each would be to extend.
2. Decide the base (A, B or C) from the trial.
3. Write the architecture document: hub, runner daemon, relay/network, ACP adapters, task board, policies.
4. Build a minimal version: 2 VPSes, 3 agents, one task board and one web UI.

## Open questions for you

- Is this **for you or your team only**, or a **product you'd offer to others**? The answer decides which licenses
  (AGPL, BSL, Elastic) are acceptable.
- Should the hub be a **web app** (reachable from your phone), a **desktop app**, or both?
- Which agents must work on day one? Claude Code, Codex, OpenCode, Gemini CLI, others?
- Which VPS provider(s), how many machines, and do you already use Tailscale?
- Preferred language: **TypeScript** (Paseo, Orca, OpenCode), **Rust** (Codex, herdr) or **Go**?

---

## Method and sources

- GitHub repository search sorted by stars, across topics: `coding-agent`, `parallel-agents`, `ai-ide`,
  `sandbox`, `self-hosted` + `claude-code`, `mobile` + `claude-code`, ACP, plus direct lookups of known projects.
- READMEs and license metadata of the main candidates (Orca, Paseo, AgentsMesh, Agent Orchestrator, Omnigent,
  Superset, Emdash, Vibe Kanban, HarnessRouter, OpenCode, Codex, pi, herdr, OpenHands, Claude Code).
- Web sources on the state of open-source Cursor alternatives:
  - [Cursor Alternatives in 2026 (Builder.io)](https://www.builder.io/blog/cursor-alternatives-2026)
  - [10 Best Open Source Cursor Alternatives (OpenAlternative)](https://openalternative.co/alternatives/cursor)
  - [Best Open Source Cursor Alternatives 2026 (OSSAlt)](https://ossalt.com/guides/best-open-source-cursor-alternatives-2026)
  - [Best Cursor Alternatives in 2026 (Novita)](https://blogs.novita.ai/cursor-alternatives/)
