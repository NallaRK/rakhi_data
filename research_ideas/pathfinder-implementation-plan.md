# Pathfinder — AI-Guided Development Platform

## Implementation Plan

**Document Type:** Architecture Proposal for Leadership Review
**Date:** June 2026
**Status:** Draft for Approval
**Version:** 2.0

---

## Naming Conventions

| Element | Name | Rationale |
|---------|------|-----------|
| Platform / Initiative | **Pathfinder** | Describes the core capability: finding the implementation path from a business story through 35 repos to working code. Follows the industry convention of pairing an evocative codename with a clear capability (cf. Netflix "Paved Road," Spotify "Backstage," Google "Kythe"). |
| Central Knowledge Repository | **pathfinder-hub** | Lowercase kebab-case per GitHub naming conventions. `-hub` suffix signals aggregation point. Contains cross-repo documentation, architecture manifests, and MCP server. |
| Custom Agents (collective) | **Pathfinder Agents** | The set of `.agent.md` files that automate the story → analysis → plan → implement → review pipeline. |
| GitHub Actions Workflow | `pathfinder-index.yml` | Phase 3 CI workflow for SCIP indexing. Prefixed for easy identification across 35 repos. |
| Cross-tool Context File | `AGENTS.md` | Industry standard (Linux Foundation's Agentic AI Foundation, adopted by 60,000+ repos). Not renamed — using the standard maximizes cross-tool compatibility. |

---

## Executive Summary

This proposal introduces **Pathfinder**, a three-phase approach to accelerate the development lifecycle across our 35-repo polyrepo environment (30 React MFE + 5 Spring Boot / MongoDB) by standardizing how AI coding assistants understand our codebase and execute implementation workflows.

**The core problem:** Product Owners write user stories in business terminology without technical context. Developers then spend significant time manually analyzing which of 35+ repos require changes, understanding cross-repo impacts, and crafting ad-hoc prompts for AI coding assistants. This analysis overhead slows delivery velocity and introduces routing errors.

**The solution:** Pathfinder — a progressively maturing platform that enhances the story-to-implementation pipeline:

- **Phase 1 — Validate (Weeks 1–2):** Zero-infrastructure approach using VSCode Copilot custom agents, instructions files, and Devin's existing DeepWiki MCP for cross-repo context. Validates the workflow with 3 MVP repos and 5–8 developers.
- **Phase 2 — Scale (Weeks 3–6):** Self-hosted knowledge hub (`pathfinder-hub` repo) containing enhanced documentation for all 35 repos, eliminating per-query Devin ACU costs. Adds MCP server, cross-repo intelligence, and PO-facing story assistant.
- **Phase 3 — Deepen (Weeks 7–14):** Full graph intelligence platform using SCIP code indexers and ArcadeDB for real-time, symbol-level cross-repo analysis. Automated CI-triggered reconciliation on every merge.

**Each phase is independently valuable.** Phase 2 proceeds only if Phase 1 validates developer adoption. Phase 3 proceeds only if Phase 2 reveals gaps requiring symbol-level code intelligence. If Phase 1 alone proves sufficient, we expand it to all 35 repos without further infrastructure.

---

## Environment Context

| Dimension | Detail |
|-----------|--------|
| Frontend | 30 React MFE polyrepos (TypeScript, Redux Toolkit, React Query) |
| Backend | 5 Java Spring Boot repos (Java 17, Spring Data MongoDB) |
| Database | MongoDB with `@Document` annotated classes |
| Shared Libraries | Common design system + shared utility packages across all 30 MFEs |
| Branching | GitFlow (develop / release/* / main) across all repos |
| CI/CD | GitHub Actions → Harness deployment pipeline |
| Developers | 100+ across multiple teams |
| Existing Tools | VSCode Copilot Enterprise, Devin (with DeepWiki for all 35 repos), GitHub Actions |

---

## Current Pain Points

1. **Vague Stories:** POs write business-level stories without technical context (affected repos, APIs, database impacts). Developers must reverse-engineer this during grooming or after ticket assignment.
2. **Repo Routing Overhead:** With 35 repos, developers spend significant time determining which repos require changes for a given story. No systematic method exists.
3. **Ad-hoc Copilot Prompting:** Developers manually craft prompts for each story, referencing patterns, files, and conventions from memory. Quality varies by individual. No standardized workflow exists.
4. **Cross-Repo Blind Spots:** Changes in shared components or backend APIs affect multiple MFEs. Impact analysis is manual, error-prone, and often discovered late in testing.
5. **Inconsistent Code Generation:** Without codified conventions, Copilot generates code that doesn't follow project standards, requiring rework during code review.

---

## Phase 1 — Validate: Copilot-Native Workflow (Weeks 1–2)

### Objective

Validate that a standardized, agent-driven development workflow reduces story analysis time and improves cross-repo routing accuracy, using zero additional infrastructure. Leverage VSCode Copilot's native custom agent system with Devin's existing DeepWiki MCP for cross-repo context.

### Scope

- **MVP Repos:** 2 React MFEs + 1 Java Spring Boot repo (selected by highest story volume)
- **Users:** 5–8 developers on the MVP teams
- **Stories:** Minimum 10 stories processed through the workflow during the 2-week pilot

### Cost

| Item | Cost | Notes |
|------|------|-------|
| Devin MCP ACUs | ~$20–50 total (bounded) | ~10–25 queries during MVP at $2.00–$2.25/ACU. Eliminated in Phase 2. |
| Copilot Enterprise | $0 incremental | Existing license |
| Infrastructure | $0 | No servers, no hosting |
| People | 1 developer, 2 weeks | Includes learning curve for Copilot agent system |

### Architecture

```
Developer → VSCode Copilot → Pathfinder Agent (.agent.md)
                                  ├── Reads JIRA ticket (JIRA MCP)
                                  ├── Queries cross-repo docs (DeepWiki MCP — Phase 1 only)
                                  ├── Reads local repo context (@workspace)
                                  ├── Reads architecture manifest (.github/architecture-manifest.json)
                                  ├── Follows coding standards (.github/copilot-instructions.md)
                                  └── Outputs → Technical Details / Implementation Plan / Code Changes
```

### Pathfinder Agent Files (Per Repo)

| File | Location | Intent | Generation Method |
|------|----------|--------|-------------------|
| `copilot-instructions.md` | `.github/` | Project-wide coding standards, component patterns, naming conventions, testing approach, state management rules, import conventions. Ensures all Copilot-generated code follows team standards. | Copilot `/init` + `/create-instructions` → human review and refinement |
| `architecture-manifest.json` | `.github/` | Structured repo metadata: components with file paths, hooks, API endpoints, MongoDB collections, shared design-system dependencies, related repos. Machine-readable for MCP tools. | Copilot `@workspace` analysis → human review |
| `story-analyzer.agent.md` | `.github/agents/` | **The entry point.** Reads a JIRA ticket via MCP, queries DeepWiki for cross-repo context, reads local manifest, and generates a "Technical Implementation Details" section identifying affected repos, components, APIs, database changes, and shared-component impacts. Hands off to planner. Model preference: reasoning-optimized (Claude Opus 4.6 or GPT-5.2). | Hand-crafted template. Repo-specific customization limited to the `tools` list and domain context. |
| `implementation-planner.agent.md` | `.github/agents/` | Receives technical analysis. Produces an ordered, file-by-file implementation plan with explicit pattern references ("follow the hook pattern in src/hooks/useKycForm.ts"). Each step is a single-file change, independently testable. Flags cross-repo steps. Hands off to Agent mode. Model preference: code-optimized. | Shared template across all repos. |
| `code-reviewer.agent.md` | `.github/agents/` | Reviews implementation against original story's acceptance criteria. Produces a coverage matrix (AC → code change → test), pattern-compliance check, and cross-repo risk assessment. | Shared template across all repos. |
| `AGENTS.md` | Root | Always-on context for all agents and all AI tools (Copilot, Claude Code, Devin). Contains: project architecture overview, tech stack, cross-repo reference map, team ownership, domain boundaries. Per Linux Foundation Agentic AI Foundation standard. | Hand-crafted once per repo. Updated when architecture changes significantly. |
| `mcp.json` | `.vscode/` | MCP server configuration. Phase 1: DeepWiki MCP + JIRA MCP. Phase 2: `pathfinder-hub` MCP replaces DeepWiki. Phase 3: adds graph MCP endpoint. | Templated. Committed to repo for team-wide sharing. |

**Subdirectory instruction files (as needed per repo):**

| File | Location | Intent |
|------|----------|--------|
| `.instructions.md` | `src/hooks/` | Hook-specific conventions: naming, React Query patterns, Zod validation, error handling, JSDoc requirements |
| `.instructions.md` | `src/components/` | Component conventions: container/presenter split, prop patterns, colocated tests, styling approach |
| `.instructions.md` | `src/services/` | API service conventions: apiClient usage from @org/shared-utils, error mapping, response typing |
| `.instructions.md` | `src/main/java/.../controller/` | (Java) REST controller conventions: annotation patterns, DTO usage, validation |
| `.instructions.md` | `src/main/java/.../service/` | (Java) Service layer conventions: transaction boundaries, exception handling, repository patterns |

### Developer Workflow (Phase 1)

```
Step 1:  Developer opens Copilot Chat, selects @story-analyzer from agent dropdown
Step 2:  Developer types JIRA ticket ID (e.g., "PROJ-1234")
         ─── This is the only typed prompt. All subsequent steps are guided. ───
Step 3:  Agent reads JIRA ticket (MCP) → queries DeepWiki for cross-repo context (MCP) →
         reads local architecture manifest → generates Technical Implementation Details
Step 4:  Developer reviews technical analysis (2–3 min)
         ├── Confirms affected repos, components, APIs are correct
         ├── Flags any missing context or incorrect assumptions
         └── IF analysis has errors → developer corrects inline and agent adjusts
Step 5:  Developer clicks "Plan Implementation" handoff button
Step 6:  Planner agent produces ordered, file-by-file plan with pattern references
Step 7:  Developer reviews plan (2–3 min)
         ├── Confirms step ordering and file paths
         ├── Adjusts scope if needed
         └── Flags steps requiring changes in OTHER repos as separate tickets
Step 8:  Developer clicks "Start Implementation" → Copilot switches to Agent mode
Step 9:  Copilot implements plan step-by-step. Developer reviews each file change.
         ├── Keep / Undo controls per file
         └── Copilot self-corrects on build/test failures
Step 10: Developer selects @code-reviewer → validates coverage against story ACs
```

**What this replaces:** 10–15 manually written prompts per story, each requiring the developer to remember file paths, patterns, and conventions. Pathfinder standardizes this into a repeatable, guided workflow.

**What this does NOT replace:** Developer judgment. Every output (analysis, plan, code) requires human review and approval. The developer remains the decision-maker at each step.

### Success Metrics

| Metric | Baseline (pre-Pathfinder) | Target (end of Phase 1) | How Measured |
|--------|---------------------------|-------------------------|--------------|
| Story analysis time | 2–4 hours (estimate) | < 30 minutes | Developer self-report at retro |
| Manual prompts per story | 10–15 | 1 (ticket ID) + guided interactions | Observation during pilot |
| Cross-repo routing accuracy | Developer memory / guesswork | Agent-identified with file-path citations | Compare agent output vs. actual PR scope |
| Pattern compliance in generated code | Inconsistent (review feedback) | Follows copilot-instructions.md | Code review rejection rate |
| Developer satisfaction | N/A | ≥ 7/10 "would recommend to team" | Anonymous survey at Week 2 retro |

### Decision Gate: End of Week 2

| Signal | Action |
|--------|--------|
| Developers report measurable time savings AND identify gaps in DeepWiki's cross-repo context quality | **Proceed to Phase 2** |
| Workflow works well enough with DeepWiki MCP; no significant gaps | **Stay in Phase 1.** Expand agent files to remaining 32 repos. Skip Phase 2 infrastructure. |
| Developers don't adopt the agent workflow or find it slower than manual prompting | **Stop.** Conduct retrospective to understand why. Iterate on agent design before re-attempting. |
| Developers adopt workflow but DeepWiki MCP ACU costs project to exceed budget at 100+ developer scale | **Proceed to Phase 2** (cost-driven, even if quality is acceptable) |

---

## Phase 2 — Scale: Self-Hosted Knowledge Hub (Weeks 3–6)

### Objective

Eliminate Devin ACU dependency by copying, enhancing, and self-hosting codebase documentation for all 35 repos in a central GitHub repository (`pathfinder-hub`). Extend Pathfinder to PO story-writing use cases. Establish the MCP server that becomes the durable intelligence layer for all subsequent phases.

### Why Not Continue Using Devin MCP?

| Concern | Detail |
|---------|--------|
| **Cost** | Devin ACUs ($2.00–$2.25 each) for private repo API access. At 100+ developers making 3–5 queries/day, monthly cost is unpredictable and potentially significant. |
| **Latency** | Remote MCP calls add network round-trip to every agent interaction. Self-hosted content serves in milliseconds. |
| **Customization** | DeepWiki generates generic documentation. Pathfinder-hub content can be enhanced with: cross-repo dependency maps, business-domain glossaries, PO-facing architecture descriptions, and project-specific context DeepWiki doesn't capture. |
| **Availability** | No dependency on Devin's service uptime, API versioning, or pricing changes. |
| **Data control** | JIRA ticket content and codebase documentation stay within org-controlled infrastructure. |

### DeepWiki Export Method

DeepWiki blocked web scraping. The documented extraction methods are:

| Method | How | ACU Cost | Recommended? |
|--------|-----|----------|--------------|
| DeepWiki MCP `read_wiki_contents` tool | Programmatic extraction via official MCP API. Call per-repo, save output as markdown. | ~1–2 ACU per repo (one-time) | **Yes — primary method.** ~$70–$160 total for 35 repos. One-time cost. |
| Devin session | Task Devin: "Export the DeepWiki documentation for [repo] as markdown files." | ~2–4 ACU per repo | Backup method if MCP tool output is insufficient. |
| Manual copy-paste | Copy from DeepWiki web UI per repo. | $0 | Last resort. Time-intensive for 35 repos. |

**Recommended approach:** Use DeepWiki MCP `read_wiki_contents` in a scripted batch (Week 3, Day 1). Budget ~$100–$160 in ACUs as a one-time export cost. This eliminates all ongoing Devin costs.

### Architecture

**Source of truth:** `pathfinder-hub` repo is the single source of truth for all cross-repo documentation and manifests. Per-repo `.github/architecture-manifest.json` files are generated FROM `pathfinder-hub` via a sync mechanism, not the other way around. This eliminates the dual-source-of-truth problem.

```
┌───────────────────────────────────────────────────┐
│          pathfinder-hub (GitHub Repository)         │
│                                                   │
│   wiki/                                           │
│   ├── mfe-checkout/   (overview, architecture,    │
│   ├── mfe-kyc/         components, api-contracts) │
│   ├── payment-service/                            │
│   └── ... (all 35 repos)                          │
│                                                   │
│   manifests/                                      │
│   ├── mfe-checkout.json   ← AUTHORITATIVE SOURCE  │
│   ├── mfe-kyc.json                                │
│   └── ... (all 35 repos)                          │
│                                                   │
│   cross-repo/                                     │
│   ├── dependency-map.md                           │
│   ├── shared-components-registry.md               │
│   ├── api-contracts-index.md                      │
│   └── domain-glossary.md                          │
│                                                   │
│   mcp-server/                                     │
│   ├── server.py         (FastAPI MCP server)      │
│   └── requirements.txt                            │
│                                                   │
│   scripts/                                        │
│   ├── sync-manifests.sh (push manifests to repos) │
│   └── export-deepwiki.py (one-time export script) │
└──────────────────────┬────────────────────────────┘
                       │
           ┌───────────┴───────────┐
           │                       │
   ┌───────▼────────┐    ┌────────▼────────┐
   │ VSCode Copilot  │    │  PO Interface   │
   │ (Developers)    │    │  (see options   │
   │ MCP stdio/SSE   │    │   below)        │
   └─────────────────┘    └─────────────────┘
```

### Manifest Sync Mechanism

```
pathfinder-hub/manifests/mfe-checkout.json  (authoritative)
          │
          │  GitHub Action: on push to pathfinder-hub
          │  runs sync-manifests.sh
          ▼
mfe-checkout/.github/architecture-manifest.json  (derived copy)
```

When a manifest is updated in `pathfinder-hub`, a GitHub Action automatically opens PRs in the affected repos to sync the derived copies. This ensures developers always have local access to their manifest even when the MCP server is unavailable, while `pathfinder-hub` remains the single authoritative source.

### MCP Server Tools

| Tool | Intent | Primary User |
|------|--------|--------------|
| `find_repos_for_story` | Given story text, identify affected repos with confidence scores and reasoning | Developer, PO |
| `get_repo_context` | Return full wiki + manifest for a specific repo | Developer |
| `impact_analysis` | Given a component, endpoint, or collection name, return all repos that use or depend on it | Developer, PO |
| `enhance_story` | Generate Technical Details section from story text + cross-repo context | Developer (grooming), PO |
| `search_wiki` | Full-text search across all 35 repos' documentation | Developer, PO |
| `cross_repo_dependencies` | Return upstream and downstream dependency graph for a given repo | Developer |

### PO Interface Options (Evaluated, Not Prescribed)

The document previously assumed OpenWebUI as the PO interface. This needs validation against PO workflow reality.

| Option | Integration Effort | PO Adoption Risk | Recommendation |
|--------|-------------------|-------------------|----------------|
| **A: JIRA Automation Rule** | Medium (JIRA API + Pathfinder MCP). When PO creates a story, automation calls `enhance_story` and appends Technical Details as a comment. | **Lowest risk.** POs stay in JIRA. No new tool. | **Recommended for Phase 2 MVP.** Validate before building a standalone UI. |
| **B: Confluence Macro / JIRA Plugin** | High (custom plugin development). Embed Pathfinder query directly in JIRA/Confluence. | Low risk if built well. | Consider for Phase 3 if JIRA automation proves limiting. |
| **C: OpenWebUI Chat** | Low (connect OpenWebUI to Pathfinder MCP). Separate web UI for brainstorming. | **Highest risk.** POs must adopt a new tool outside their workflow. | Defer unless POs explicitly request a chat-based exploration tool. |
| **D: Slack Bot** | Medium (Slack MCP or bot framework). PO types story idea in a Slack channel, bot responds with enhanced story. | Medium risk. POs already use Slack. | Good alternative if team prefers async brainstorming. |

### Wiki Maintenance Process

| Trigger | Action | Automation Level | Owner |
|---------|--------|-----------------|-------|
| New repo added to project | Generate wiki + manifest using Copilot. Add to `pathfinder-hub`. | Semi-automated | Platform owner |
| Major architectural change | Re-generate affected repo's wiki section using Copilot. | Semi-automated | Team that made the change |
| Quarterly refresh | GitHub Action runs Copilot analysis on all 35 repos, diffs against existing wiki, opens PR with stale sections flagged. | Automated trigger, human review | Platform owner |
| Shared component contract change | Update `shared-components-registry.md`. | Manual | Design system team |
| New API endpoint added | Detectable from Spring Boot `@RestController` changes in PR. GitHub Action flags for manifest update. | Semi-automated | Backend team |
| **Fallback / degradation** | If `pathfinder-hub` MCP is unreachable, agents fall back to per-repo `.github/architecture-manifest.json` (derived copy) + `@workspace` context. Reduced cross-repo awareness but not a full outage. | Automatic (agent fallback logic) | N/A |

### Rollout Plan

| Week | Activity |
|------|----------|
| Week 3 | Export DeepWiki content for 3 MVP repos (scripted via MCP). Create `pathfinder-hub` repo. Build MCP server. Test with MVP developer team. |
| Week 4 | Export and enhance remaining 32 repos. Build cross-repo documents (dependency map, shared component registry, API index, domain glossary). |
| Week 5 | Deploy MCP server (self-hosted VM or GitHub-hosted process). Update all repos' `.vscode/mcp.json`. Implement JIRA automation for PO story enhancement (Option A). |
| Week 6 | Train POs on enhanced story workflow. Full team rollout of Pathfinder agents to all 35 repos. Measure adoption. Retrospective. |

### Decision Gate: End of Week 6

| Signal | Action |
|--------|--------|
| Developers consistently request symbol-level queries the wiki cannot answer (e.g., "find every file that calls this function across all 35 repos") | **Proceed to Phase 3** |
| Wiki-level context is sufficient for story analysis and implementation planning | **Stay in Phase 2.** Invest in wiki quality and maintenance automation instead of graph infrastructure. |
| POs request richer interactive exploration beyond JIRA automation | Evaluate adding OpenWebUI or Slack bot (Options C/D) as a Phase 2.5 enhancement. |

---

## Phase 3 — Deepen: Graph Intelligence Platform (Weeks 7–14)

> **Timeline adjusted from original 6 weeks to 8 weeks.** Industry data on developer platform rollouts consistently shows 1.5–2x timeline overrun. The original 6-week estimate left no buffer for SCIP indexer integration issues, graph schema iteration, or developer onboarding. 8 weeks is more realistic.

### Objective

Add real-time, symbol-level code intelligence using SCIP indexers and a graph database (ArcadeDB), triggered automatically by CI/CD pipeline events. This enables precise queries that static documentation cannot answer: blast radius analysis, cross-repo type resolution, dangling reference detection, and automated architectural drift alerts.

### When This Phase Is Justified

Phase 3 is justified only if Phase 2 reveals these specific, documented gaps:

| Gap | Example Query Phase 2 Cannot Answer |
|-----|--------------------------------------|
| Symbol-level cross-referencing | "Which files in MFE-order import and call `usePaymentV2` from MFE-checkout?" |
| Blast radius with type resolution | "If I change `FormField` props in the design system, which MFEs pass the old prop shape?" |
| Automated staleness detection | "The wiki says the API takes `PaymentRequest`, but the code now uses `PaymentRequestV2`." |
| Dangling reference detection | "MFE-checkout renamed a hook but MFE-order still imports the old name." |

**If these gaps are not observed during Phase 2, do not build Phase 3.** The operational cost of maintaining a graph database is only justified by queries the wiki cannot serve.

### Architecture

```
35 Repos (GitHub Actions CI)
        │
        │  POST index.scip on merge to develop/release/main
        ▼
┌─────────────────────────────────────────────────────┐
│     Pathfinder Intelligence Server (single VM)       │
│                                                     │
│   Ingestion API (FastAPI)                           │
│        │                                            │
│        ▼                                            │
│   Branch-Aware Router                               │
│   (main → PO-facing graph, release/* → dev graph)   │
│        │                                            │
│        ▼                                            │
│   ArcadeDB (Apache 2.0, JVM)                       │
│   Graph + Document + Vector                         │
│        │                                            │
│        ▼                                            │
│   Cross-Repo Consistency Checker                    │
│   (dangling refs, version skew, contract drift)     │
│        │                                            │
│        ▼                                            │
│   MCP Server (SSE for POs, stdio for developers)    │
│   Graph tools: symbol_lookup, blast_radius,         │
│   consistency_check, collection_usage               │
│                                                     │
│   + All Phase 2 wiki/manifest tools (retained)      │
└─────────────────────────────────────────────────────┘
```

### CI/CD Changes (All 35 Repos)

| File | Location | Intent |
|------|----------|--------|
| `pathfinder-index.yml` | `.github/workflows/` | GitHub Action: on merge to develop/release/main, runs SCIP indexer, uploads `index.scip` to Pathfinder server. Identical across all repos (templated). |

**Indexers used (all Apache 2.0):**

- React MFE repos: `scip-typescript` — requires `npm ci` + `tsc` compilation in CI. Adds ~30–60 seconds to CI pipeline.
- Java Spring Boot repos: `scip-java` — uses `semanticdb` compiler plugin. Requires Gradle/Maven build step. Adds ~45–90 seconds to CI pipeline.

### Graph Database Schema

**Node types:**

| Type | Key Properties | Description |
|------|---------------|-------------|
| Symbol | name, kind, filePath, repo, branch, startLine, endLine, commitSha, indexedAt | Functions, classes, hooks, components, interfaces, Spring beans |
| File | path, repo, branch, language | Source files |
| MongoCollection | name, repo | MongoDB collections (from `@Document` annotations) |

**Edge types:**

| Type | From → To | Description |
|------|-----------|-------------|
| DEFINED_IN | Symbol → File | Where a symbol is defined |
| REFERENCES | Symbol → Symbol | One symbol references another |
| IMPORTS | File → File | File-level import relationships |
| CALLS | Symbol → Symbol | Function/method call relationships |
| QUERIES_COLLECTION | Symbol → MongoCollection | Spring Data repository → MongoDB collection mapping |

### New MCP Tools (Additive to Phase 2)

| Tool | Intent | Example Query |
|------|--------|---------------|
| `symbol_lookup` | Find where a symbol is defined and all its references across repos | "Where is `usePaymentV2` defined and who calls it?" |
| `blast_radius` | Given a symbol, return all transitively affected files and repos | "If I change `FormField` props, what breaks across all 30 MFEs?" |
| `consistency_check` | Return current cross-repo inconsistencies (dangling refs, renamed symbols still imported under old names) | "Are there broken imports after today's merges?" |
| `collection_usage` | All code paths that read or write a MongoDB collection | "What code touches the `kyc_submissions` collection?" |
| `recent_changes` | Symbols added, removed, or renamed since a given date or commit | "What changed in payment-service since last release?" |

### Reconciliation Model

| Event | Action | Staleness |
|-------|--------|-----------|
| PR merges to develop / release/* / main | GitHub Action runs SCIP indexer → uploads `index.scip` → server atomically replaces that repo's graph partition | ~1–2 minutes after merge |
| Multiple repos merge simultaneously | ArcadeDB handles concurrent writes natively (ACID transactions, no write serialization queue) | Each repo updated independently |
| Cross-repo inconsistency detected | Stored as Warning nodes in graph. Surfaced via `consistency_check` tool and optional Slack notification. | Real-time after index update |
| Branch-aware queries | PO queries default to `main`. Developer queries default to `release/*` with fallback to `main` for repos without the release branch. | Automatic, per-query |

### Rollout Plan

| Week | Activity |
|------|----------|
| Week 7 | Deploy ArcadeDB on VM (`java -jar`, no Docker). Design and test graph schema. Build ingestion API. |
| Week 8 | Add SCIP indexing to CI for 3 MVP repos. Test full pipeline: merge → index → upload → graph insert → query. |
| Week 9 | Build enhanced MCP server with graph-backed tools. Test with MVP developers alongside existing wiki tools. |
| Week 10 | Iterate on graph queries based on developer feedback. Tune false-positive rate on `consistency_check`. |
| Week 11 | Roll `pathfinder-index.yml` workflow to all 35 repos (templated, identical). |
| Week 12 | Integrate graph tools into JIRA automation and/or PO interface. Add consistency dashboard. |
| Week 13–14 | Full team rollout. Measure impact. Retrospective. Buffer for issues discovered during rollout. |

### Operational Requirements

| Dimension | Detail |
|-----------|--------|
| Infrastructure | 1 Linux VM (8 CPU, 16 GB RAM) for ArcadeDB + MCP server + ingestion API |
| Storage | ~10–50 GB for graph database (35 repos, 3 branches each) |
| Backup | Nightly backup of ArcadeDB data directory + retain last 30 days of `.scip` index files for rebuild |
| Monitoring | JVM metrics (heap, GC), API response latency, index freshness per repo (alert if any repo > 24 hours stale) |
| Maintenance | ~2–4 hours/week ongoing: SCIP indexer version updates, graph schema evolution, query performance tuning |
| Ownership | **Named owner required.** Platform engineering team or a designated individual. Not "the team" — a specific person accountable for uptime, updates, and developer support. |

---

## Complete File Inventory

### Per-Repo Files (each of the 35 repos)

| File | Phase | Created By |
|------|-------|------------|
| `.github/copilot-instructions.md` | 1 | Copilot `/init` + `/create-instructions` → human review |
| `.github/architecture-manifest.json` | 1 (local), 2 (synced from hub) | Phase 1: Copilot `@workspace` analysis. Phase 2+: derived from `pathfinder-hub` via sync. |
| `.github/agents/story-analyzer.agent.md` | 1 | Hand-crafted template, repo-specific `tools` list and domain context |
| `.github/agents/implementation-planner.agent.md` | 1 | Shared template across all repos |
| `.github/agents/code-reviewer.agent.md` | 1 | Shared template across all repos |
| `AGENTS.md` | 1 | Hand-crafted per repo. Updated on major architecture changes. |
| `.vscode/mcp.json` | 1 | Templated. Updated in Phase 2 (hub MCP) and Phase 3 (graph MCP). |
| `src/hooks/.instructions.md` | 1 | Copilot-generated, human-reviewed (React repos only) |
| `src/components/.instructions.md` | 1 | Copilot-generated, human-reviewed (React repos only) |
| `src/services/.instructions.md` | 1 | Copilot-generated, human-reviewed |
| `.github/workflows/pathfinder-index.yml` | 3 | Templated GitHub Action (identical across all repos) |

### Central Repository (`pathfinder-hub`)

| File / Directory | Phase | Intent |
|------------------|-------|--------|
| `wiki/{repo-name}/*.md` | 2 | Enhanced, curated documentation per repo |
| `manifests/{repo-name}.json` | 2 | Authoritative architecture manifests for all 35 repos |
| `cross-repo/dependency-map.md` | 2 | Inter-repo dependency graph (which repos depend on which) |
| `cross-repo/shared-components-registry.md` | 2 | Design system component usage matrix across all 30 MFEs |
| `cross-repo/api-contracts-index.md` | 2 | All REST endpoints across all 5 backend services |
| `cross-repo/domain-glossary.md` | 2 | Business term → technical component mapping (PO-facing) |
| `mcp-server/server.py` | 2 | MCP server: wiki + manifest query tools |
| `mcp-server/graph_tools.py` | 3 | Graph-backed MCP tools (additive to Phase 2 tools) |
| `scripts/export-deepwiki.py` | 2 | One-time DeepWiki export script |
| `scripts/sync-manifests.sh` | 2 | Pushes manifests from hub to per-repo derived copies |
| `.github/workflows/sync-to-repos.yml` | 2 | GitHub Action: on manifest change, opens PRs in affected repos |
| `.github/workflows/quarterly-refresh.yml` | 2 | GitHub Action: triggers wiki freshness check across all 35 repos |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **Developers don't adopt Pathfinder workflow** | Medium | High | Phase 1 MVP with 5–8 developers validates before scaling. Stop/iterate if adoption fails. |
| **AI hallucination in story analysis** | High | Medium | Story analyzer agent MUST cite specific file paths from manifest/wiki. Implementation planner verifies cited paths exist via `@workspace` before generating plan. Code reviewer validates against actual codebase. Human review at every step. |
| **DeepWiki content too generic for implementation** | Medium | Medium | Phase 2 enhances with project-specific context, cross-repo maps, and domain glossary. |
| **Wiki documentation goes stale** | High | Medium | Quarterly automated refresh (GitHub Action). Manifest sync on every `pathfinder-hub` update. Named owner accountable. |
| **SCIP indexing adds CI time** | Low | Low | scip-typescript adds ~30–60 sec per build. Monitor. If unacceptable, run indexing as a separate, non-blocking workflow. |
| **ArcadeDB operational burden (Phase 3)** | Medium | Medium | Single VM, JVM-native. Nightly backup. 2–4 hr/week maintenance budget. Named owner. |
| **MCP server downtime during active sprint** | Medium | Medium | Agent fallback to per-repo `.github/architecture-manifest.json` + `@workspace`. Degraded cross-repo awareness but not a full workflow outage. |
| **Devin MCP deprecation or pricing change** | Low | Low | Phase 1 only. Phase 2 eliminates dependency entirely. |
| **Security: JIRA ticket content exposure** | Medium | High | JIRA MCP scoped to read-only access. MCP server deployed within org network. No ticket content stored in `pathfinder-hub`. Review with InfoSec before deployment. |
| **Manifest dual-source divergence** | Medium | Medium | `pathfinder-hub` is authoritative. Per-repo copies are derived via automated sync. Sync GitHub Action prevents silent drift. |

---

## Cost Summary

| Phase | Infrastructure | Tool Cost | People Cost |
|-------|---------------|-----------|-------------|
| Phase 1 | $0 | Devin ACUs: ~$50 (bounded MVP). Existing Copilot Enterprise license. | 1 developer × 2 weeks (includes learning curve) |
| Phase 2 | $0 (GitHub repo) or ~$50/mo (small VM for MCP server) | DeepWiki export: ~$100–$160 one-time ACU cost. All tools open source thereafter. | 1 developer × 4 weeks (setup + rollout) |
| Phase 3 | ~$100–$200/mo (VM for ArcadeDB + MCP server) | $0 (ArcadeDB Apache 2.0, SCIP indexers Apache 2.0) | 1–2 developers × 8 weeks (build + rollout). ~2–4 hr/week ongoing maintenance. |

---

## Constraints Honored

| Constraint | How Honored |
|------------|-------------|
| Open source tools only | ArcadeDB (Apache 2.0), SCIP indexers (Apache 2.0), OpenWebUI (MIT if used), MCP server (custom, in-house) |
| No Docker | ArcadeDB runs as `java -jar`. MCP server runs as Python process. SCIP indexers run as `npx` / Gradle CLI tools in CI runners. |
| No Yarn | All npm operations use `npm ci` / `npx`. No Yarn dependency anywhere in the pipeline. |
| Leverage existing tools | VSCode Copilot Enterprise (existing), Devin (existing, Phase 1 only), GitHub Actions (existing), JIRA (existing) |

---

## Approval Request

We request approval to proceed with **Phase 1 immediately:**

- **Cost:** ~$50 in Devin ACUs + 1 developer for 2 weeks
- **Risk:** Zero infrastructure. Fully reversible. If it fails, we've lost 2 weeks and learned why.
- **Decision gate at Week 2** determines Phase 2 progression based on measured outcomes, not assumptions.

---

## Appendices (Available on Request)

- **A:** Complete `.agent.md` file specifications for all three Pathfinder agents
- **B:** `copilot-instructions.md` template for React MFE and Spring Boot repos
- **C:** `architecture-manifest.json` schema specification with example
- **D:** MCP server API specification and tool contracts
- **E:** SCIP indexing GitHub Action workflow template
- **F:** ArcadeDB graph schema DDL and sample Cypher queries

---

*Document prepared for architecture review. Detailed implementation specifications available in the appendices upon request.*
