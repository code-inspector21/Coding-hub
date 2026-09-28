# Research addendum: code context, skills, Slack and Rust/TS desktop

Snapshot date: **2026-09-28**. Follows [landscape.md](landscape.md). This pass covers the areas added to the
goal after the first round: graph RAG and code context, skill optimization, Slack control, and
Rust + TypeScript desktop apps. Licenses marked *verify* were not confirmed.

---

## 1. Code context and graph RAG

The goal: give every agent the relevant slice of a large codebase, instead of letting each one grep its way
around. Most tools here parse code with tree-sitter into a graph of symbols, files, calls and imports, then
expose it to agents as MCP tools.

| Project | ★ | License | How it works | Fit for us |
|---|---|---|---|---|
| [Graphify-Labs/graphify](https://github.com/Graphify-Labs/graphify) | 122k | Apache-2.0 / MIT | Parses with tree-sitter (no LLM needed) across ~37 languages plus SQL, Terraform, docs and PDFs. Groups code into subsystems with Leiden clustering. Serves an MCP server (`query_graph`, `get_node`, `shortest_path`) and an HTML graph view. | Broadest coverage: code, docs and schemas in one graph. Written in Python, so we would run it as a sidecar. |
| [colbymchenry/codegraph](https://github.com/colbymchenry/codegraph) | 72k | MIT | Rust core with tree-sitter (20+ languages). Stores the graph in local SQLite with full-text search. File watchers keep it in sync within seconds. One `codegraph_explore` MCP tool returns entry points, related symbols and snippets. | **Best fit for a Rust stack.** Local and incremental. Candidate for the default indexer on each runner. |
| [trailhq/Graft](https://github.com/trailhq/Graft) | 9.3k | MIT | Two LLM passes turn the repo into linked markdown files, one node per subsystem. No embeddings: agents read and follow the files. MCP tools for finding code and tracing calls. | Readable "map" of a big project. Complements codegraph's symbol-level graph. |
| [vitali87/code-graph-rag](https://github.com/vitali87/code-graph-rag) | 5.2k | *verify* | Tree-sitter into a Memgraph graph database, queried in natural language. | Needs a graph database server. Heavier to run. |
| [giancarloerra/SocratiCode](https://github.com/giancarloerra/SocratiCode) | 3.3k | *verify* | Hybrid search (Qdrant vectors plus graph), impact analysis and call flow. Claims support for 40M+ lines of code. | Useful reference for impact analysis ("what breaks if I change this?"). |
| [forloopcodes/contextplus](https://github.com/forloopcodes/contextplus) | 2k | *verify* | RAG plus tree-sitter plus spectral clustering, producing a hierarchical feature graph. | Reference. |
| [Muvon/octocode](https://github.com/Muvon/octocode) | 478 | *verify* | Single Rust binary: semantic search (LanceDB), knowledge graph and MCP server. | Small, but Rust-native. |
| [oraios/serena](https://github.com/oraios/serena) | 30k | *verify* | Uses language servers (LSP) for precise symbol lookup and editing, over MCP. | More precise than tree-sitter for refactors. Pair it with a graph. |

Project memory and knowledge that goes beyond code: [mex-memory/mex](https://github.com/mex-memory/mex) (team
memory kept in the repo and shared through git), [0xK3vin/MegaMemory](https://github.com/0xK3vin/MegaMemory)
(project knowledge graph over MCP), [swarmclawai/swarmvault](https://github.com/swarmclawai/swarmvault) (LLM wiki
plus knowledge graph), [claude-mem](https://github.com/thedotmack/claude-mem) (95k★).

**Takeaways**
- Nobody has built the combination we want: a **code graph plus a project graph** (tasks, decisions, PRs,
  Slack threads linked to code) shared by **all agents on all machines**. The existing tools are single-repo
  and run on one machine.
- Run a proven indexer on each runner (**codegraph** first; Graphify for docs and schemas). Build the
  project-knowledge layer and cross-machine sharing ourselves.
- The token-saving claims (for example "61% fewer tokens") come from the projects themselves.
  We should measure them on our own tasks before trusting them.

---

## 2. Skills: management, safety, evaluation, optimization

"Skills" here means the Agent Skills format (`SKILL.md` folders) that Claude Code, Codex, OpenCode, Gemini CLI
and others now load.

| Stage | Project | ★ | License | Notes |
|---|---|---|---|---|
| Manage and sync | [farion1231/cc-switch](https://github.com/farion1231/cc-switch) | 138k | MIT | **Tauri 2 + Rust + React + SQLite** desktop app. Manages providers, MCP servers, skills and prompts for Claude Code, Codex, OpenCode, OpenClaw, Hermes and others. Configures agents but doesn't run them. Proof that our stack works at scale, and a good reference for provider, MCP and skills screens. |
| Manage and sync | [qufei1993/skills-hub](https://github.com/qufei1993/skills-hub) | 1.7k | *verify* | Tauri (Rust + React). "Install once, sync everywhere" into each agent's skills folder. |
| Safety | [NVIDIA/SkillSpector](https://github.com/NVIDIA/SkillSpector) | 18k | *verify* | Scans skills for prompt injection, data exfiltration and supply-chain risks before install. |
| Evaluation | [NVIDIA/SkillEvaluator](https://github.com/NVIDIA/SkillEvaluator) | 524 | *verify* | Quality gates, overlap detection, synthetic eval sets, and live runs that measure how a skill changes agent behaviour. |
| Retrieval | [EverMind-AI/SkillCorpus](https://github.com/EverMind-AI/SkillCorpus) | 669 | *verify* | Turns many `SKILL.md` files into a searchable corpus, with skill-routing evaluation. |
| Optimization | [gepa-ai/gepa](https://github.com/gepa-ai/gepa) | 6.8k | *verify* | Reflective optimization of prompts and text artifacts, driven by execution traces. |
| Optimization, applied | [NousResearch/hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution) | 5.4k | *verify* | Uses DSPy + GEPA to evolve Hermes Agent's skills, prompts and code. The closest working example of "skills that improve themselves". |
| Content | [obra/superpowers](https://github.com/obra/superpowers) (292k★), [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) (100k★) | – | *verify* | Proven skill packs to start from. |

**Takeaways**
- Managing and syncing skills is solved (cc-switch, skills-hub). Scanning is solved (SkillSpector).
- What's missing is **optimization based on outcomes**: measure each skill version against real task results
  (tests pass, review verdict, tokens, time), then let GEPA propose improved versions for a human to approve.
  Our hub records every agent event in one log, so it has the data this loop needs.

---

## 3. Slack as a control surface

| Project | ★ | License | Design |
|---|---|---|---|
| [chenhg5/cc-connect](https://github.com/chenhg5/cc-connect) | 15.7k | *verify* | Go. Connects local Claude Code, Cursor, Gemini CLI and Codex to Slack, Discord, Telegram, Feishu and others. Most platforms don't need a public IP. |
| [refly-ai/refly](https://github.com/refly-ai/refly) | 7.5k | *verify* | Skills builder with Slack and Lark bots. |
| [amplifthq/opentag](https://github.com/amplifthq/opentag) | 1.4k | MIT | **The closest design to ours.** Mention an ACP agent in Slack, GitHub, GitLab, Linear or Lark. A self-hosted control plane queues the work to a paired **runner** machine that holds the code, worktrees and agent logins. Replies stay in the Slack thread, with approval points. TypeScript, Postgres, Docker Compose. Supports only a single runner. |
| [cyrusagents/cyrus](https://github.com/cyrusagents/cyrus) | 832 | *verify* | Background agent for Linear, Slack and GitHub. Supports Claude Code, Codex, Cursor, Gemini and OpenCode. |
| [context-machine-lab/sleepless-agent](https://github.com/context-machine-lab/sleepless-agent) | 832 | *verify* | Runs Claude Code 24/7 from Slack: task queue, isolated workspaces, commits and PRs. |
| [linxidnju/OpenTag](https://github.com/linxidnju/OpenTag) | 502 | *verify* | Slack gateway that routes threads to Claude Code, Codex, OpenCode, Docker or HTTP agents, with policies, approvals and audit logs. |
| [slackapi/slack-skills-plugin](https://github.com/slackapi/slack-skills-plugin) | 135 | *verify* | Official: a Slack MCP server plus Slack developer skills for Claude Code, Codex and Cursor. Lets agents *read and post* in Slack. |

**Takeaways**
- The standard pattern is **thread = task**: an @mention creates the task, the bot posts progress and approval
  requests in the thread, and the result ends up in the same thread.
- **Socket Mode** (the bot opens an outbound WebSocket to Slack) means our hub needs no public URL. That suits
  VPSes behind NAT.
- OpenTag already has a control-plane → runner split, but with only one runner. Ours needs many runners, a
  task graph, and structured events from agents.

---

## 4. Rust + TypeScript desktop

- **Tauri 2** (Rust backend, web frontend) is the standard choice for Rust + TypeScript desktop apps. Proven in
  this space by cc-switch (138k★), skills-hub, OpenContext, LiveAgent and openhuman (40k★, Rust agent harness).
- Rust-native agent control planes worth reading: [zeronsh/zeron](https://github.com/zeronsh/zeron) (2.3k★, GPUI
  native control plane for Claude Code, Codex, Cursor and Devin), [BrokkAi/mjolnir](https://github.com/BrokkAi/mjolnir)
  (Rust ACP control plane), [herdrdev/herdr](https://github.com/herdrdev/herdr) (Rust, Apache-2.0).
- **ACP's reference implementation is in Rust**
  ([agentclientprotocol/agent-client-protocol](https://github.com/agentclientprotocol/agent-client-protocol), 4.3k★).
  The official Claude adapter [agentclientprotocol/claude-agent-acp](https://github.com/agentclientprotocol/claude-agent-acp)
  (2.6k★) runs the Claude Agent SDK behind ACP. So a Rust runner can drive Claude Code, Codex, Gemini CLI,
  OpenCode and Goose through one protocol.

**Effect on the base decision:** the earlier main candidates, Paseo and Orca, are Node/Electron. With
Rust + TypeScript and a desktop app, building our own Tauri app plus a Rust hub and runner fits better. We
would reuse permissively licensed pieces: the ACP crate, codegraph, and patterns from cc-switch and OpenTag.
See [../architecture/overview.md](../architecture/overview.md).
