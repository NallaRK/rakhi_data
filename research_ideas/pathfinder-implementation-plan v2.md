# Pathfinder Implementation Plan

**Combined Phase 1 — Stage 1a (Weeks 1–2) + Stage 1b (Weeks 3–6)**

*A structured agentic-development workflow for a 35-repo polyrepo environment (30 React MFE + 5 Java Spring Boot), using VSCode Copilot Enterprise, Devin (one-time DeepWiki export), GitHub Actions, JIRA, and a custom MCP server.*

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architectural Decisions](#2-architectural-decisions)
3. [Stage 1a — Weeks 1–2 — MVP Pilot](#3-stage-1a--weeks-12--mvp-pilot)
4. [Stage 1b — Weeks 3–6 — Hub + MCP + Scale](#4-stage-1b--weeks-36--hub--mcp--scale)
5. [Pattern B Orchestrator Architecture](#5-pattern-b-orchestrator-architecture)
6. [Spec Kit Integration (Option B — Layered)](#6-spec-kit-integration-option-b--layered)
7. [Loop Engineering — Future Roadmap (Option C)](#7-loop-engineering--future-roadmap-option-c)
8. [Decision Tree for Sub-Agent Invocation](#8-decision-tree-for-sub-agent-invocation)
9. [Risk Assessment](#9-risk-assessment)
10. [Cost Summary](#10-cost-summary)
11. [Asset Reference](#11-asset-reference)

---

## 1. Executive Summary

### What Pathfinder is

Pathfinder is a structured AI-assisted development workflow that takes a JIRA ticket from creation to merged PR through five specialized AI sub-agents coordinated by an orchestrator. It addresses three concrete pains in the current environment:

1. **POs write business-language stories** with insufficient technical detail; developers waste hours figuring out which of the 35 repos to change.
2. **Devs juggle context across 30+ React MFEs** with shared design system components; pattern drift is constant.
3. **Cross-repo coordination is invisible** until QA — frontend and backend ship out of order, breaking environments.

Pathfinder solves these by:

- A `@story-analyzer` sub-agent that queries a custom MCP server (`pathfinder-hub`) holding repo manifests and wikis, returning ranked affected repos with file-path citations
- A `@design-analyzer` sub-agent for JIRA tickets with Figma PNG/JPG attachments, mapping designs to the internal design system
- An `@implementation-planner` sub-agent producing ordered file-by-file plans with pattern references
- A `@code-reviewer` sub-agent (independent model invocation) verifying AC coverage, pattern compliance, and cross-repo consistency
- An optional `@spec-writer` sub-agent (Spec Kit wrapper) for greenfield features only

### What it is NOT

- Not a replacement for developer judgment. Every output passes through a human gate.
- Not autonomous code shipping. Pathfinder produces analyses and plans; Copilot Agent Mode (driven by the developer) writes the code.
- Not full Loop Engineering. VSCode Copilot Enterprise lacks the native loop primitives (`/loop`, `/goal`) that Claude Code and Codex CLI provide. We adopt Loop Engineering principles (maker-checker split, stop conditions) without building autonomous loops. See Section 7.

### Combined Phase 1 timeline

| Stage | Weeks | Scope | Validation gate |
|---|---|---|---|
| 1a | 1–2 | 3 pilot repos, agents + prompts only, manual MCP-free workflow with local DeepWiki paste-ins | Week 2 review: can 3 devs successfully use the orchestrator to handle 3 real JIRA tickets end-to-end? |
| 1b | 3–6 | Build `pathfinder-hub` repo, deploy MCP server, sync mechanism, scale to all 6 pilot repos | Week 6 review: time-to-merged-PR measurably reduced; cross-repo coordination misses ≤ 1 per week |

No formal kill switch between stages, but **at end of Week 2** the team pauses to validate Stage 1a outcomes before committing to the Stage 1b build. If Stage 1a fails to show clear improvement, the team has only spent two weeks on agents and prompts — Stage 1b's MCP + hub investment is not yet made.

---

## 2. Architectural Decisions

### Decision matrix

| Decision | Choice | Rejected alternative | Why |
|---|---|---|---|
| Orchestration pattern | **Pattern B (Subagent Delegation)** | Pattern A (Handoffs); Pattern C (Automatic) | Pattern B fits JIRA-driven workflow — orchestrator branches based on ticket content without requiring developer to click handoff buttons. Confirmed working in VSCode Copilot Enterprise via `agents:` frontmatter allowlist. |
| Repo for cross-repo intelligence | **Single `pathfinder-hub` repo** | Per-repo `.pathfinder/` folders | Single source of truth, easier refresh cadence, supports MCP server hosting. Per-repo copies are derived (synced via GHA). |
| DeepWiki strategy | **One-time Devin export, paste into `docs/wiki/`** | Ongoing Devin ACU spend | Avoids per-month ACU costs. Devin's value is in the initial generation; ongoing maintenance is cheaper via Copilot + prompts. |
| Component library access | **Copilot `@workspace` on `node_modules/@your-org/design-system/`** | Hosting design system docs in `pathfinder-hub` | The TypeScript `.d.ts` files are ground truth for props. `__examples__/` show real usage. No external hosting needed. |
| MCP server stack | **Python FastMCP, no Docker** | Java Spring Boot + container | Lighter dependency footprint. FastMCP is the standard MCP framework. `python -m server` + `systemd` is sufficient. |
| Spec Kit integration | **Option B (layered, optional for greenfield only)** | Replace custom agents with Spec Kit; ignore Spec Kit entirely | Spec Kit is excellent for new features but overlaps with `@story-analyzer` for existing-code work. `@spec-writer` invokes it only when the orchestrator detects greenfield. |
| Loop Engineering adoption | **Option C (principles now, full loops as future roadmap)** | Build full loops on VSCode Copilot today | VSCode Copilot Enterprise has no native `/loop` or `/goal` command. Forcing loops via GHA cron would be over-engineering. Principles (maker-checker split, stop conditions) are adoptable now without infrastructure. |
| Pilot scope | **6 repos: 4 React MFE + 2 Spring Boot** | All 35 repos | Pilot must include cross-repo coordination (so frontend + backend both present) and design system usage (so multiple MFE). 6 is the smallest meaningful subset. |
| Branching | **Existing GitFlow preserved** | New branching model | GitFlow works; Pathfinder doesn't touch it. Sync PRs target `develop`. |

### Tech stack confirmed

- **Frontend (4 MFE pilot repos):** TypeScript, React 18, Redux Toolkit, React Query, internal design system via `@your-org/design-system`
- **Backend (2 service pilot repos):** Java 17, Spring Boot, Spring Data MongoDB
- **AI tooling:** VSCode Copilot Enterprise (Claude Opus 4.6/4.7 + GPT-5.2 + GPT-5.3-Codex available)
- **Knowledge bootstrap:** Devin DeepWiki (one-time text export only)
- **CI/CD:** GitHub Actions (in pathfinder-hub), Harness (existing deployment)
- **Issue tracking:** JIRA (existing)
- **MCP server:** Python 3.11+, FastMCP, BM25 for full-text search

### Open source only — license check

Every dependency in the asset package is OSI-approved:

| Component | License |
|---|---|
| `mcp[cli]` (Python SDK / FastMCP) | MIT |
| `PyYAML` | MIT |
| `rank-bm25` | Apache 2.0 |
| GitHub Spec Kit | MIT |
| VSCode Copilot custom agents framework | Microsoft (proprietary to Copilot, but configuration files are yours) |
| GitHub Actions used | All MIT (`actions/checkout`, `actions/setup-python`, `peter-evans/create-pull-request`) |

---

## 3. Stage 1a — Weeks 1–2 — MVP Pilot

### Goal

Prove that the agent-based workflow improves developer experience using **local-only assets** (no MCP server, no central hub). 3 pilot repos. End of Week 2: decide whether to commit to Stage 1b's hub + MCP build.

### Pilot subset for Stage 1a

Pick **3 repos** from your 6-repo pilot list. Recommended split:
- 1 React MFE that is the primary surface for a frequently-changed JIRA epic
- 1 React MFE that depends on the backend pilot repo
- 1 Spring Boot service that the second MFE consumes

This combination gives Stage 1a coverage of: in-repo work, frontend-only work, and cross-repo work.

### Week 1 — Setup

| Day | Activity | Asset used |
|---|---|---|
| 1 | Devin DeepWiki one-time export for all 6 pilot repos. Save text exports to a shared Drive/Sharepoint. | (no asset) |
| 1 | For 3 Stage 1a repos: paste DeepWiki text into `docs/wiki/` per repo using the export script | `pathfinder-hub-structure/scripts/export-deepwiki.py` |
| 2 | Generate `.github/copilot-instructions.md` for each of the 3 repos | `prompts/01-generate-copilot-instructions.md` |
| 2 | Generate `.github/architecture-manifest.json` for each of the 3 repos | `prompts/02-generate-architecture-manifest.md` + the 2 schema templates in `manifests/` |
| 3 | Optimize the pasted DeepWiki content in `docs/wiki/` using the optimizer prompt | `prompts/03-optimize-deepwiki-content.md` |
| 3 | Install the 6 custom agents at the user level (so available across all 3 repos) | All 6 files in `agents/` |
| 4 | Configure the `deepwiki-export-helper.yml` GHA in each of the 3 repos | `workflows/deepwiki-export-helper.yml` |
| 4–5 | Each of 3 pilot developers picks one of their next JIRA tickets and runs it through the orchestrator end-to-end | All assets above |

**End of Week 1 checkpoint:** Each developer has run at least one JIRA ticket through the orchestrator. Capture: pain points, what worked, what failed, surprises.

### Week 2 — Iterate + Validate

| Day | Activity |
|---|---|
| 6 | Triage Week 1 feedback. Adjust agent prompts (especially `orchestrator.agent.md` decision tree). Refine `.github/copilot-instructions.md` based on actual Copilot outputs. |
| 7–9 | Each developer runs 2–3 more JIRA tickets through Pathfinder. Measure: time-to-first-PR, code review iterations, AC coverage at first review. |
| 10 | Stage 1a retrospective. Decision: proceed to Stage 1b (yes/no/modify). |

### Stage 1a validation criteria (end of Week 2)

Proceed to Stage 1b if **all four** are true:

1. Each pilot developer successfully ran ≥ 3 JIRA tickets through the orchestrator
2. At least 50% of those tickets had measurably better outcomes (time, review cycles, or AC coverage) than the developer's prior baseline
3. No critical failure modes surfaced (e.g., orchestrator routinely picks wrong sub-agents, agents hallucinate file paths)
4. The team wants to continue (qualitative — overruling 1–3 is allowed in either direction)

If any criterion fails, hold a Stage 1a extension week with targeted fixes before committing to Stage 1b's larger investment.

### What Stage 1a does NOT include

- **No `pathfinder-hub` repo yet.** Manifests and wikis live locally in each repo's `.github/` and `docs/wiki/`. The `pathfinder-hub` MCP queries described in agent files will return "not configured" — agents fall back to `@workspace`.
- **No MCP server.** Sub-agents work without it; they just have less cross-repo context.
- **No automated sync.** Per-repo wikis drift from each other (acceptable for 2 weeks).
- **No design system mapping for unfamiliar components.** `@design-analyzer` works via `@workspace` on `node_modules/@your-org/design-system/`, which is fine.

---

## 4. Stage 1b — Weeks 3–6 — Hub + MCP + Scale

### Goal

Build the `pathfinder-hub` repo, deploy the MCP server, automate sync to consumer repos, and scale from 3 pilot repos to all 6.

### Week 3 — Build pathfinder-hub

| Day | Activity | Asset used |
|---|---|---|
| 11 | Create `pathfinder-hub` GitHub repo. Copy the entire `pathfinder-hub-structure/` scaffold from the asset zip as initial content. | `pathfinder-hub-structure/` (every file) |
| 11 | Move the 3 manifests from Stage 1a's per-repo `.github/architecture-manifest.json` into `pathfinder-hub/manifests/`. Verify they match the schema templates. | `manifests/*.template.json` |
| 12 | Move the 3 optimized wikis from Stage 1a's `docs/wiki/` into `pathfinder-hub/wiki/{repo}/`. | (existing Stage 1a content) |
| 12–13 | Generate manifests for the remaining 3 pilot repos using prompt 02. Add to `pathfinder-hub/manifests/`. | `prompts/02-generate-architecture-manifest.md` |
| 13 | Devin DeepWiki text-export → paste → optimize for the remaining 3 pilot repos. | `prompts/03-optimize-deepwiki-content.md`, `export-deepwiki.py` |
| 14–15 | Generate the 4 cross-repo files (`dependency-map.md`, `shared-components-registry.md`, `api-contracts-index.md`, `domain-glossary.md`). | `prompts/05-build-cross-repo-docs.md` |

### Week 4 — Deploy MCP server

| Day | Activity | Asset used |
|---|---|---|
| 16 | Pick a Linux host inside the corporate network (existing internal dev server is fine). Install Python 3.11+. | (no asset) |
| 16 | Clone `pathfinder-hub` onto the host. Install MCP server dependencies. | `mcp-server/requirements.txt`, `mcp-server/README.md` |
| 17 | Run the MCP server in stdio mode locally first to validate tool wiring. Test each of the 6 tools via the MCP Inspector. | `mcp-server/server.py` |
| 17 | Switch to streamable HTTP transport. Run under `systemd`. Configure on internal hostname (e.g., `pathfinder-hub.internal:8765`). | `mcp-server/server.py`, `mcp-server/README.md` |
| 18 | Add `.vscode/mcp.json` to each of the 6 pilot repos pointing at the HTTP endpoint. | (per-repo config) |
| 19 | Each pilot developer tests `@story-analyzer` and verifies the MCP tools are queried (visible in Copilot Chat tool-call trace). | All `agents/*` |
| 20 | Tune MCP server scoring logic in `find_repos.py` based on early misses. Restart server. | `mcp-server/tools/find_repos.py` |

### Week 5 — Automated sync + JIRA automation

| Day | Activity | Asset used |
|---|---|---|
| 21 | Generate a GitHub Personal Access Token with repo scope. Store as `PATHFINDER_HUB_PAT` secret in `pathfinder-hub`. | (admin task) |
| 21 | Install `workflows/sync-to-repos.yml` in `pathfinder-hub/.github/workflows/`. | `workflows/sync-to-repos.yml` |
| 22 | Test sync: make a small manifest change in `pathfinder-hub/manifests/mfe-kyc.json`. Push to main. Verify a PR opens in mfe-kyc against `develop` updating `.github/architecture-manifest.json`. | `workflows/sync-to-repos.yml` |
| 23 | Install `workflows/quarterly-refresh.yml`. Test via `workflow_dispatch`. | `workflows/quarterly-refresh.yml` |
| 24 | JIRA automation: configure a JIRA automation rule that calls the MCP server's `enhance_story` tool on story creation. (Implementation details depend on your JIRA tier — Atlassian Forge app or webhook to a small relay service.) | `mcp-server/tools/enhance_story.py` |
| 25 | Scale check: orchestrator workflow works across all 6 pilot repos. Run real JIRA tickets through each. |

### Week 6 — Validation + retrospective + rollout decision

| Day | Activity |
|---|---|
| 26–28 | Each pilot developer runs ≥ 5 JIRA tickets through full Pathfinder workflow. Track metrics: time-to-merged-PR, review cycles, cross-repo coordination misses, AC coverage at first review. |
| 29 | Stage 1b retrospective. Surface: what worked, what didn't, what's the runway for adding repos 7–35? |
| 30 | Decision: full rollout plan to 35 repos (Phase 2, out of scope for this plan) or hold and iterate at 6. |

### Stage 1b validation criteria (end of Week 6)

Pathfinder is considered "working" if:

1. Time-to-merged-PR is ≥ 20% lower than pre-pilot baseline (measured across the 6 pilot repos)
2. Cross-repo coordination misses ≤ 1 per week across the pilot team
3. AC coverage at first review ≥ 90% (story-analyzer + planner are catching what POs leave vague)
4. ≥ 4 of 6 pilot developers report it as net-positive (qualitative)
5. The MCP server has had ≥ 99% uptime in Week 5

If 3+ of these fail, the architecture needs rework before scaling. If 0–2 fail, proceed with rollout planning (the failures become Phase 2 priorities).

---

## 5. Pattern B Orchestrator Architecture

### Why Pattern B

VSCode Copilot Enterprise supports three orchestration patterns:

| Pattern | Style | Best for |
|---|---|---|
| **A — Handoffs** | Orchestrator finishes, suggests "next: invoke @sub-agent", user clicks to proceed | Workflows where developers want to review at each gate |
| **B — Subagent Delegation** | Orchestrator invokes sub-agents inline via `agents:` allowlist; user sees one result | Workflows where the routing logic is data-driven (e.g., based on ticket content) |
| **C — Automatic Delegation** | Sub-agents self-invoke based on context detection | Loosely-coupled tasks where any sub-agent might be relevant |

**Pattern B wins for JIRA-driven work.** The orchestrator reads the ticket, branches on its content (Figma attached? greenfield?), and delegates without forcing the developer to click. The developer sees a coherent summary instead of toggling between agents.

### Pattern B mechanics — verified

- Custom agent files (`.agent.md`) declare `agents: ['list', 'of', 'allowed']` in YAML frontmatter
- The orchestrator invokes a sub-agent using the built-in `agent` tool
- Sub-agents run with isolated context windows (their conversation history is not visible to the parent)
- The sub-agent's result is returned to the parent as a string

### Pattern B constraint (known)

**Sub-agents inherit the parent's model.** [GitHub issue #310138](https://github.com/microsoft/vscode/issues/310138) tracks this; not yet resolved as of June 2026. Implication: if the orchestrator runs on Claude Opus 4.6, all sub-agents do too. Cost goes up if many sub-agent invocations happen per ticket.

**Workarounds adopted in the asset package:**
- Orchestrator's `model:` lists Claude Opus 4.6 first (preferred), GPT-5.2 second (fallback)
- Sub-agents lock to the same model list — no mismatch surprises
- `@code-reviewer` runs as a fresh invocation from the orchestrator, providing an independent verification pass even though it's the same model. This is "maker-checker" within a single model class — not as strong as cross-model verification, but better than the same conversation re-grading its own work.

### Decision tree (executed by orchestrator)

```
Receive JIRA ticket
    │
    ├── Read ticket via JIRA MCP
    │
    ├── Branch 1: Has Figma PNG/JPG attachments OR mentions UI changes?
    │       │
    │       └── YES → invoke @design-analyzer (vision-capable model required)
    │
    ├── Branch 2: Is this a greenfield feature (no existing analog)?
    │       │
    │       └── YES → invoke @spec-writer (runs Spec Kit workflow)
    │
    ├── ALWAYS: invoke @story-analyzer (queries pathfinder-hub MCP)
    │
    ├── ALWAYS: invoke @implementation-planner (produces ordered file-by-file plan)
    │
    ├── Developer drives Copilot Agent Mode through the plan (sub-agent-less here)
    │
    └── ALWAYS: invoke @code-reviewer (independent verification, AC coverage matrix)
```

See `agents/orchestrator.agent.md` for the full decision tree as the orchestrator executes it.

### Why the orchestrator does NOT code itself

The orchestrator is a **coordinator role**, not an executor. Three reasons:

1. **Context isolation.** If the orchestrator wrote code, its context fills with file diffs and stops being useful for routing decisions on subsequent tickets in the same session.
2. **Pattern enforcement.** Code-writing should happen in Copilot Agent Mode, where developers can see and approve each edit. Hiding edits inside an orchestrator chat reduces visibility.
3. **Maker-checker discipline.** The orchestrator invokes `@code-reviewer` as a final step. If the orchestrator also wrote the code, the reviewer is grading its parent's work — exactly the anti-pattern Loop Engineering warns about.

---

## 6. Spec Kit Integration (Option B — Layered)

### What Spec Kit is

[GitHub Spec Kit](https://github.com/github/spec-kit) is an MIT-licensed CLI toolkit for Spec-Driven Development (SDD). It registers slash commands (`/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`, etc.) in VSCode Copilot and produces structured `spec.md → plan.md → tasks.md` artifacts before any code is written.

- Installed via: `uv tool install specify-cli --from git+https://github.com/github/spec-kit.git`
- Per-project init: `specify init --integration copilot`
- Verified working with VSCode Copilot Enterprise (one of 30+ supported integrations)

### Why Option B (layered, not central)

Spec Kit's structured workflow overlaps significantly with `@story-analyzer` → `@implementation-planner`. For a typical JIRA ticket modifying existing code, running Spec Kit is overhead — you'd be authoring a spec for work that has clear precedent in the codebase.

But Spec Kit shines for **greenfield work**: a brand-new feature with no existing analog. There, the spec/plan/tasks discipline catches ambiguities that ad-hoc planning misses.

So: keep the custom agents as primary. Layer Spec Kit on top via `@spec-writer`, which the orchestrator invokes **only** when it detects greenfield work.

### When `@spec-writer` is invoked

| Ticket type | `@spec-writer` invoked? |
|---|---|
| Modification to existing component/service | NO |
| Bug fix | NO |
| Adding to an existing pattern (e.g., another CRUD endpoint) | NO |
| Brand-new feature with no existing analog | YES |
| New service or new module | YES |
| Refactor of well-understood area | NO (use planner directly) |

`@spec-writer` itself decides whether to proceed. If invoked on a non-greenfield ticket, it declines politely and recommends the orchestrator skip it.

### What `@spec-writer` does NOT do

It does NOT run `/speckit.implement`. Implementation is handled by Copilot Agent Mode after `@implementation-planner` reviews the Spec Kit output and reconciles it with cross-repo constraints (which Spec Kit doesn't know about).

The Spec Kit artifacts (`spec.md`, `plan.md`, `tasks.md`) become inputs to `@implementation-planner`, not bypasses around it.

### Asset locations

- Agent file: `agents/spec-writer.agent.md`
- No prompts or skills needed — Spec Kit ships its own slash commands

### Cost note

Spec-driven workflows consume 20–40% more tokens per feature than vibe coding because the agent re-reads spec, plan, and tasks every turn. For the 5–10% of tickets where Spec Kit is appropriate, this is a clear win (fewer wasted implementation cycles). Layering it on top of the spine workflow keeps the cost contained.

---

## 7. Loop Engineering — Future Roadmap (Option C)

### What Loop Engineering is

A practice that crystallized in June 2026, popularized by Addy Osmani (Google), Peter Steinberger (OpenAI), and Boris Cherny (Anthropic, creator of Claude Code). The core insight:

> *"Stop prompting your coding agent. Design the loop that prompts it."*
> — Peter Steinberger, June 2026

A loop is a small program that fires the agent on a trigger (schedule, webhook, file change), gives it a goal, lets it act, runs a verifier, and decides whether to continue or stop — without the developer typing each prompt.

Five building blocks of a loop:
1. **Trigger / schedule** — cron, GHA, hook
2. **State / memory** — what was done, what's pending, what failed
3. **Decision / next action** — what to do this iteration
4. **Execution** — code change, tool call, command
5. **Verification** — separate model checks if the goal is met

Plus: **stop condition** — when does the loop terminate? Loop Engineering's most important principle: *the verifier is the bottleneck, not the model.* A weak verifier passes weak code at scale.

### Why Loop Engineering does NOT fit Pathfinder today

VSCode Copilot Enterprise has no native loop primitive. The canonical loop tools are:
- **Claude Code** — `/loop` command (shipped May 11, 2026, v2.1.139)
- **OpenAI Codex CLI** — `/goal` command + Automations tab (shipped April 30, 2026, v0.128.0)

Both run unattended. Both have separate verifier sub-agents that grade the worker's output before stopping. Neither integrates with VSCode Copilot's custom agent system directly.

Building a loop on top of VSCode Copilot today would require:
- GHA cron firing a shell script that opens VSCode headless
- The script driving Copilot Chat via UI automation (no API exists for headless chat)
- Custom verifier glue
- Token-cost tracking outside the Copilot Enterprise budget surface

That's a build with no clear payoff over just adopting Claude Code or Codex CLI directly for unattended work. Pathfinder is the wrong substrate.

### What we DO adopt from Loop Engineering — now

Three principles transferable without infrastructure:

#### 7.1 Maker-checker split

The agent that writes the code is not the agent that grades it.

- `@implementation-planner` produces the plan
- Copilot Agent Mode writes the code
- `@code-reviewer` (independent invocation, same model class) verifies AC coverage and pattern compliance

This is **not as strong** as the cross-model verification Loop Engineering recommends (e.g., Claude writes, GPT verifies). It is stronger than no verifier. Phase 2 may introduce true cross-model verification once VSCode supports per-invocation model override (issue #310138).

#### 7.2 Explicit stop conditions

Every agent file has a "Failure Modes to Avoid" section that includes stop conditions:

- `@story-analyzer` stops when it finds ≥ 1 affected repo OR confidence is below threshold
- `@design-analyzer` stops when each visible UI element has been mapped or explicitly flagged as ungap
- `@code-reviewer` stops when AC coverage matrix is built, even if it finds issues (issues are surfaced, not auto-fixed)

The orchestrator does NOT loop. Each sub-agent runs once per invocation.

#### 7.3 Comprehension-debt awareness

Loop Engineering's most-cited risk: *"Two engineers can run the same loop and get opposite results. The loop doesn't know. You do."* AI-generated code that ships without human comprehension creates invisible technical debt.

Pathfinder counters this with **hard human gates** at three points:
- After `@story-analyzer` — developer confirms the affected-repos list before planning
- After `@implementation-planner` — developer confirms the plan before Copilot Agent Mode runs
- After `@code-reviewer` — developer reviews the AC matrix before opening PR

These gates are non-negotiable in the orchestrator's instructions.

### Future roadmap — when Loop Engineering becomes adoptable

If your team eventually wants unattended loops (e.g., overnight "fix CI failures" or "triage Renovate PRs"), the migration path is:

**Phase 3 trigger conditions (any of):**
- VSCode Copilot ships native `/loop` and `/goal` commands
- Team adopts Claude Code or Codex CLI alongside Copilot for specific use cases
- Maintenance volume crosses a threshold where unattended triage is worth the safety investment

**Phase 3 migration steps:**

1. **Pick the lowest-stakes loop first.** Recommendation: dependency-update PR review. The verifier is a clean test suite; the failure mode is a closed PR.
2. **Adopt Claude Code or Codex CLI** for the loop runtime. Keep VSCode Copilot for in-editor work.
3. **Reuse Pathfinder's verifier patterns.** `@code-reviewer`'s AC coverage logic translates directly to a Claude Code verifier sub-agent.
4. **Reuse the MCP server.** Both Claude Code and Codex CLI consume MCP. The `pathfinder-hub` MCP server works with all three tools.
5. **Set hard budget caps.** Loop Engineering's most expensive failure mode is runaway iteration. Cap per-loop spend at $X per run; hard-stop after N iterations.
6. **Maintain the maker-checker split with cross-model verification.** If Claude writes, GPT-5.2 verifies (or vice versa). This is where loops get genuinely safer than single-pass generation.

The `mcp-server/` in this asset package is already loop-compatible. You don't need to rebuild it for Phase 3 — you add new agent definitions that consume it from a different IDE.

### Loop Engineering — what we are explicitly NOT doing

- Not building autonomous unattended loops on VSCode Copilot. The substrate doesn't support it.
- Not introducing scheduled agent runs in this phase. Quarterly refresh GHA is the only schedule-driven workflow, and it produces a report — it doesn't autonomously act.
- Not enabling agents to invoke each other in loops. The orchestrator delegates once per sub-agent per ticket. No cycles.

If anyone proposes "let's just add a loop here" during Stage 1b, the answer is: it doesn't fit the substrate. Roadmap it to Phase 3.

---

## 8. Decision Tree for Sub-Agent Invocation

Reproduced from `agents/orchestrator.agent.md` for quick reference. The orchestrator follows this when receiving any JIRA ticket.

### Always invoked (the spine)

| Sub-agent | Always? | Why |
|---|---|---|
| `@story-analyzer` | YES | Every ticket needs affected-repo + AC analysis |
| `@implementation-planner` | YES | Every ticket needs an ordered plan |
| `@code-reviewer` | YES | Every ticket needs verification, even trivial ones |

### Conditionally invoked

| Sub-agent | Condition | Why skip otherwise |
|---|---|---|
| `@design-analyzer` | Ticket has Figma PNG/JPG attachments OR mentions UI changes | Pure backend work doesn't need it; costs context tokens |
| `@spec-writer` | Greenfield feature, no codebase analog | Spec Kit overhead is unjustified for routine modifications |

### Never invoked in parallel

The spine (`story-analyzer` → `implementation-planner` → `code-reviewer`) is **sequential**. Each consumes the previous one's output. Don't run them in parallel.

`@design-analyzer` and `@story-analyzer` can theoretically run in parallel (different inputs), but in practice run sequentially because the planner needs both as inputs.

### What developers do between sub-agent calls

| Gate | Developer action |
|---|---|
| After `@story-analyzer` | Read affected-repos list, sanity-check confidence scores, approve or amend |
| After `@design-analyzer` (if invoked) | Read uncertainties, ask PO if needed |
| After `@spec-writer` (if invoked) | Read `spec.md`, run `/speckit.clarify` if ambiguities surfaced |
| After `@implementation-planner` | Read step-by-step plan, verify cited files exist, approve or amend |
| Between plan and code | Open Copilot Agent Mode, drive through the plan step by step |
| After Copilot Agent Mode produces code | Invoke `@code-reviewer` |
| After `@code-reviewer` | Read AC matrix, address blocking issues before PR |

These gates are non-negotiable. Skipping `@code-reviewer` on small changes is the most common shortcut and the most expensive in cumulative review-back-and-forth cost.

---

## 9. Risk Assessment

### High risks

| Risk | Impact | Mitigation |
|---|---|---|
| MCP server downtime breaks the pilot workflow | High — agents can't get cross-repo context | Run MCP server under `systemd` with auto-restart. Add the orchestrator a fallback path: if MCP returns error, downgrade to `@workspace`-only analysis with a flag in the output. |
| Manifests drift from actual code | High — agents cite paths that don't exist | Quarterly refresh GHA flags stale manifests. Sync workflow auto-syncs when hub manifests change. Manifest-generator skill makes regeneration easy. |
| Sub-agent inherits wrong model (cost) | Medium — Claude Opus on every sub-call is expensive | Track per-developer token spend in Copilot Enterprise admin. If a team is consistently over budget, switch orchestrator default to GPT-5.2. |
| Design analyzer hallucinates Figma details | High — wrong design implementations ship | The `agents/design-analyzer.agent.md` enforces "annotated vs inferred" distinction. Code reviewer specifically verifies design system component usage. |
| Pattern B is misunderstood as "less control" | Medium — developers skip gates | Hard-coded human gates in the orchestrator instructions. Stage 1a retrospective measures whether gates are being respected. |

### Medium risks

| Risk | Impact | Mitigation |
|---|---|---|
| Stage 1a developers don't use the workflow consistently | Medium — invalidates Week 2 validation | Daily 15-min check-ins in Week 1. Track adoption per developer. |
| Spec Kit adds overhead to non-greenfield work | Medium — slows down routine tickets | `@spec-writer` declines politely when invoked on non-greenfield work. Train developers to recognize the difference. |
| JIRA automation rule for `enhance_story` is too noisy | Medium — POs get spammed with auto-edits | Start with manual invocation only. Gate auto-enhancement behind a JIRA label like `pathfinder-enhance`. |
| Cross-repo files become stale | Medium — devs ignore them | Quarterly refresh + clear "last updated" stamps. If older than 90 days, the freshness GHA opens a PR demanding refresh. |

### Low risks

| Risk | Mitigation |
|---|---|
| BM25 search quality is poor on small wiki corpus | Adequate for 6 repos; Phase 3 can swap to vector search |
| GitHub Spec Kit changes its CLI surface | Pin to a known-good version in `@spec-writer`. Test before upgrading. |
| FastMCP API changes | Pin to `mcp[cli]>=1.27.0` in requirements.txt. Test before upgrading. |

### What's NOT a risk (despite intuition)

- **Loop Engineering not being adopted.** This is a deliberate choice. The substrate doesn't support it; forcing it is the risk.
- **Pattern B sub-agent inheritance of parent model.** Known constraint, accepted, workaround in place.
- **DeepWiki content going stale after one-time export.** Optimizer prompt + freshness GHA + quarterly refresh handle this.

---

## 10. Cost Summary

### One-time costs (Stages 1a + 1b combined)

| Item | Cost |
|---|---|
| Devin DeepWiki export for 6 pilot repos | One-time ACU spend (estimate: 50–100 ACU @ your current rate). After export, no further Devin spend in this phase. |
| Linux host for MCP server | If using existing internal dev server: $0. If new VM: minimal (single small instance). |
| Spec Kit installation | $0 (MIT open source) |
| GitHub Actions minutes for sync + refresh workflows | Minimal — workflows run on push to main (sync) and quarterly (refresh). Estimated < 100 minutes/month. |
| Developer time | Stage 1a: 3 developers × 2 weeks part-time on Pathfinder workflow + setup. Stage 1b: same 3 + lead time on hub build. |

### Ongoing per-month costs (steady state)

| Item | Cost |
|---|---|
| VSCode Copilot Enterprise per-seat | (existing; no increase) |
| Token spend (Claude Opus + GPT-5.2 via Copilot) | Estimated 20–40% increase per pilot developer due to multi-agent invocations. Track via Copilot Enterprise admin dashboard. |
| MCP server hosting | $0 incremental on existing infra |
| Maintenance | ~2 hours/week from one developer to refresh wikis and tune agent prompts based on feedback |

### Cost mitigation levers

1. **Orchestrator model choice.** Defaulting to GPT-5.2 instead of Claude Opus 4.6 cuts per-ticket cost ~50%. Quality difference for routing decisions is small.
2. **Skip `@spec-writer` aggressively.** Spec Kit's 20–40% token premium only earns its keep on true greenfield work.
3. **Cache `pathfinder-hub` content on the MCP server.** Currently the server reads files on every tool call. For 35 repos this becomes noticeable; for 6 it's fine. Phase 3 optimization.

### What Pathfinder saves (the payoff)

The validation criteria (Sections 3 and 4) define success as ≥ 20% faster time-to-merged-PR. Even at 20%, the token spend increase is dwarfed by the developer-time savings — assuming each pilot developer's hourly cost is far higher than the marginal token cost per ticket.

If validation fails at the 20% threshold, the costs are not justified. The kill switch is real.

---

## 11. Asset Reference

Every file referenced in this plan lives in the `pathfinder-assets.zip` companion package. Quick index:

### Agents (6 files in `agents/`)

| File | Purpose |
|---|---|
| `orchestrator.agent.md` | Pattern B lead, branches based on ticket content |
| `story-analyzer.agent.md` | Queries pathfinder-hub MCP, identifies affected repos |
| `design-analyzer.agent.md` | Maps Figma PNG/JPG to design system components (vision-capable) |
| `spec-writer.agent.md` | Wraps Spec Kit for greenfield features only |
| `implementation-planner.agent.md` | Produces ordered file-by-file plan |
| `code-reviewer.agent.md` | Independent verifier — maker-checker split |

### Prompts (6 files in `prompts/`)

| File | When to use |
|---|---|
| `01-generate-copilot-instructions.md` | Once per repo at setup |
| `02-generate-architecture-manifest.md` | Once per repo at setup; re-run after refactors |
| `03-optimize-deepwiki-content.md` | After pasting DeepWiki text into `docs/wiki/` |
| `04-build-wiki-from-scratch.md` | For repos that lack DeepWiki content |
| `05-build-cross-repo-docs.md` | Stage 1b after all manifests exist; quarterly thereafter |
| `06-figma-design-extraction.md` | Standalone Figma analysis without invoking the full agent |

### Skills (4 files in `skills/`)

| File | When it auto-activates |
|---|---|
| `deepwiki-optimizer.md` | Editing files in `docs/wiki/` or `pathfinder-hub/wiki/` |
| `manifest-generator.md` | Editing `.github/architecture-manifest.json` or `pathfinder-hub/manifests/` |
| `cross-repo-analyzer.md` | Editing files in `pathfinder-hub/cross-repo/` |
| `design-system-mapper.md` | Tasks involving design system component selection |

### Manifest templates (2 files in `manifests/`)

| File | Use for |
|---|---|
| `architecture-manifest-react-mfe.template.json` | React MFE repos (TypeScript + Redux Toolkit + React Query) |
| `architecture-manifest-spring-boot.template.json` | Java Spring Boot repos (Java 17 + Spring Data MongoDB) |

### MCP server (in `mcp-server/`)

| File | Purpose |
|---|---|
| `server.py` | FastMCP server with 6 tools |
| `requirements.txt` | Python dependencies |
| `tools/find_repos.py` | Tool: which repos a story affects (BM25-like scoring) |
| `tools/get_repo_context.py` | Tool: full manifest + wiki for one repo |
| `tools/impact_analysis.py` | Tool: all repos that reference a symbol/file |
| `tools/enhance_story.py` | Tool: PO story → enhanced story with technical details |
| `tools/search_wiki.py` | Tool: BM25 full-text search across wiki |
| `tools/cross_repo_dependencies.py` | Tool: upstream + downstream graph for one repo |
| `README.md` | Setup, stdio + HTTP transport, troubleshooting |

### Workflows (3 files in `workflows/`)

| File | Trigger |
|---|---|
| `sync-to-repos.yml` | Push to main in pathfinder-hub touching `manifests/` |
| `quarterly-refresh.yml` | Cron quarterly + manual dispatch |
| `deepwiki-export-helper.yml` | PR touching `docs/wiki/` in consumer repo (Stage 1a) |

### pathfinder-hub scaffold (in `pathfinder-hub-structure/`)

| Path | Use |
|---|---|
| `README.md` | Scaffold guide |
| `wiki/_template/` | Skeleton wiki files to copy per repo |
| `cross-repo/_templates/` | Skeleton cross-repo files |
| `manifests/` | Empty in scaffold; populated in Stage 1b |
| `scripts/export-deepwiki.py` | Stage 1a: convert DeepWiki text export → `docs/wiki/` markdown |
| `scripts/sync-manifests.sh` | Stage 1b: manual sync (alternative to GHA) |

### Top-level

| File | Purpose |
|---|---|
| `README.md` | Asset index — what's in the zip and how it maps to stages |

---

## Final notes

- **Treat this plan as a hypothesis.** Stage 1a's Week 2 validation is the empirical test. If Pathfinder doesn't measurably improve developer experience, do not proceed to Stage 1b. The two-week timebox keeps the failure cost contained.
- **The biggest risk is over-engineering.** Pattern B + 6 sub-agents + MCP server + sync workflows + cross-repo docs is a lot. Stage 1a is deliberately stripped down to validate the core hypothesis (does an agent-based workflow improve JIRA ticket handling?) before committing to the full architecture.
- **The roadmap to 35 repos is out of scope.** This plan covers the 6-repo pilot. Phase 2 (rollout to all 35) is a separate planning exercise based on Stage 1b validation results.
- **Loop Engineering is a deliberate "not now".** The substrate doesn't support it; forcing it would be over-engineering. Section 7 documents the path forward if conditions change.

Read `pathfinder-assets/README.md` for the asset zip's structure, then `pathfinder-assets/pathfinder-hub-structure/README.md` for the future hub repo scaffold guide.

For specific questions on any agent, prompt, skill, or workflow file, the file's header section explains its purpose and constraints.
