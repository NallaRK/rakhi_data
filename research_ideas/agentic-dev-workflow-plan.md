# Agentic Development Workflow — Implementation Plan

**Document Type:** Architecture Proposal for Leadership Review  
**Date:** June 2026  
**Status:** Draft for Approval

---

## Executive Summary

This proposal introduces a three-phase approach to accelerate the development lifecycle across our 35 polyrepo environment (30 React MFE + 5 Spring Boot/MongoDB) by leveraging agentic AI development workflows.

**The core problem:** Product Owners write vague user stories in business terminology. Developers spend significant time analyzing which of the 35+ repos require changes, understanding cross-repo impacts, and manually prompting AI coding assistants. This analysis overhead slows delivery velocity and introduces routing errors.

**The solution:** A progressively maturing "Code Intelligence Platform" that enhances the story-to-implementation pipeline:

- **Phase 1 (Weeks 1–2):** Zero-infrastructure approach using VSCode Copilot custom agents, skills, instructions files, and Devin's existing DeepWiki MCP for cross-repo context. Validates the workflow with 3 MVP repos.
- **Phase 2 (Weeks 3–6):** Copy and customize Devin's DeepWiki content for all 35 repos into a self-hosted GitHub repository, eliminating per-query ACU costs and enabling offline/enhanced documentation accessible via Copilot MCP. Extends to PO use cases.
- **Phase 3 (Weeks 7–12):** Full graph intelligence platform using SCIP code indexers, ArcadeDB (Apache 2.0), and automated CI-triggered reconciliation for real-time, symbol-level cross-repo intelligence.

**Each phase is independently valuable.** Phase 2 only proceeds if Phase 1 validates developer adoption. Phase 3 only proceeds if Phase 2 reveals gaps that require symbol-level code intelligence.

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
2. **Repo Routing Overhead:** With 35 repos, developers spend significant time determining which repos require changes for a given story.
3. **Manual Copilot Prompting:** Developers manually craft prompts for each story, referencing patterns, files, and conventions from memory. No standardized workflow exists.
4. **Cross-Repo Blind Spots:** Changes in shared components or backend APIs affect multiple MFEs. Impact analysis is manual and error-prone.
5. **Inconsistent Coding Patterns:** Without codified conventions, Copilot generates code that doesn't follow project standards, requiring review rework.

---

## Phase 1: Copilot-Native Workflow (Weeks 1–2)

### Objective
Validate the agentic development workflow using zero additional infrastructure. Leverage VSCode Copilot's native agent, skill, and instruction systems with Devin's existing DeepWiki MCP for cross-repo context.

### Scope
- **MVP Repos:** 2 React MFEs + 1 Java Spring Boot repo (selected by highest story volume)
- **Users:** 5–8 developers on the MVP teams

### Cost
- Devin MCP API calls consume ACUs ($2.00–$2.25 per ACU, ~15 min of work each). Phase 1 usage is limited (MVP only), so cost is bounded. This cost driver is eliminated in Phase 2.
- No additional infrastructure, hosting, or licensing costs.

### Architecture

```
Developer → VSCode Copilot → Custom Agent (.agent.md)
                                  ├── Reads JIRA ticket (JIRA MCP)
                                  ├── Queries cross-repo docs (DeepWiki MCP)
                                  ├── Reads local repo context (@workspace)
                                  ├── Reads architecture manifest (.github/architecture-manifest.json)
                                  ├── Follows coding standards (.github/copilot-instructions.md)
                                  └── Generates → Technical Details / Implementation Plan / Code
```

### Files Created Per Repo (MVP repos only)

| File | Location | Purpose | Generation Method |
|------|----------|---------|-------------------|
| `copilot-instructions.md` | `.github/` | Project-wide coding standards, patterns, conventions | Auto-generated via Copilot `/init` + `/create-instructions`, then human-reviewed |
| `architecture-manifest.json` | `.github/` | Structured repo metadata: components, hooks, API endpoints, MongoDB collections, shared dependencies, related repos | Auto-generated via Copilot `@workspace` analysis, then human-reviewed |
| `story-analyzer.agent.md` | `.github/agents/` | Reads JIRA ticket → queries DeepWiki + manifest → generates Technical Implementation Details section | Hand-crafted once, shared across repos with repo-specific customization |
| `implementation-planner.agent.md` | `.github/agents/` | Receives technical analysis → creates ordered, file-by-file implementation plan with pattern references | Hand-crafted once, shared across repos |
| `code-reviewer.agent.md` | `.github/agents/` | Reviews implementation against original story ACs → produces coverage matrix + risk assessment | Hand-crafted once, shared across repos |
| `AGENTS.md` | Root | Always-on context for all agents: project-level architecture overview, cross-repo reference map, team structure | Hand-crafted once per repo |
| `mcp.json` | `.vscode/` | MCP server configuration: DeepWiki (Phase 1) / self-hosted wiki (Phase 2) + JIRA | Templated, one-time setup |

**Additional instruction files (as needed per repo):**

| File | Location | Purpose |
|------|----------|---------|
| `.instructions.md` | `src/hooks/` | Hook-specific conventions (naming, React Query patterns, error handling) |
| `.instructions.md` | `src/components/` | Component-specific conventions (container/presenter, prop patterns) |
| `.instructions.md` | `src/services/` | API service conventions (apiClient usage, error mapping) |
| `.instructions.md` | `src/main/java/.../controller/` | (Java) REST controller conventions |
| `.instructions.md` | `src/main/java/.../service/` | (Java) Service layer conventions |

### Developer Workflow (Phase 1)

```
Step 1: Developer selects @story-analyzer agent in Copilot Chat
Step 2: Developer types JIRA ticket ID (e.g., "PROJ-1234")
Step 3: Agent reads JIRA ticket via MCP → queries DeepWiki for cross-repo context →
        reads local manifest → generates Technical Implementation Details
Step 4: Developer reviews technical details (2–3 min)
Step 5: Developer clicks "Plan Implementation" handoff button
Step 6: Planner agent creates ordered, file-by-file implementation plan
Step 7: Developer reviews plan (2–3 min)
Step 8: Developer clicks "Start Implementation" → Copilot Agent mode executes the plan
Step 9: Developer reviews each file change as Copilot implements
Step 10: Developer selects @code-reviewer → validates coverage against story ACs
```

**Developer prompts written: 1** (the JIRA ticket ID). Everything else is agent-driven.

### Success Metrics (measured at Week 2 retro)

| Metric | Baseline (estimate) | Target |
|--------|---------------------|--------|
| Story analysis time (grooming to implementation start) | 2–4 hours | < 30 minutes |
| Copilot prompts per story | 10–15 manual prompts | 1 prompt (ticket ID only) |
| Cross-repo routing accuracy | Developer memory/guesswork | Agent-identified with references |
| Pattern compliance in generated code | Inconsistent | Follows copilot-instructions.md |

### Decision Gate: End of Week 2

- **Proceed to Phase 2 if:** Developers report measurable time savings AND identify gaps in DeepWiki's cross-repo context quality.
- **Stay in Phase 1 if:** Workflow works well enough with DeepWiki MCP. Expand to remaining repos without Phase 2 infrastructure.
- **Stop if:** Developers don't adopt the agent workflow or find it slower than manual prompting.

---

## Phase 2: Self-Hosted Documentation Intelligence (Weeks 3–6)

### Objective
Eliminate Devin ACU costs for documentation queries by copying, enhancing, and self-hosting DeepWiki content for all 35 repos. Create a centralized documentation repository accessible via MCP to both developers (VSCode Copilot) and POs (web UI).

### Why Not Continue Using Devin MCP?

| Concern | Detail |
|---------|--------|
| **Cost** | Devin charges ACUs ($2.00–$2.25 each, ~15 min of work) for private repo API access. With 100+ developers making multiple queries daily, ACU costs scale unpredictably. |
| **Latency** | Remote MCP calls to Devin's servers add network latency to every agent interaction. Self-hosted content is faster. |
| **Customization** | DeepWiki generates generic documentation. Self-hosted copies can be enhanced with project-specific context: cross-repo dependency maps, domain glossaries, PO-facing architecture descriptions. |
| **Availability** | No dependency on Devin's service availability or API changes. |

### Architecture

```
                    ┌──────────────────────────────────────┐
                    │   project-intelligence (GitHub Repo)  │
                    │                                      │
                    │   wiki/                               │
                    │   ├── mfe-checkout/                   │
                    │   │   ├── overview.md                 │
                    │   │   ├── architecture.md             │
                    │   │   ├── components.md               │
                    │   │   └── api-contracts.md            │
                    │   ├── mfe-kyc/                        │
                    │   │   └── ...                         │
                    │   ├── payment-service/                │
                    │   │   └── ...                         │
                    │   └── ...  (all 35 repos)             │
                    │                                      │
                    │   manifests/                          │
                    │   ├── mfe-checkout.json               │
                    │   ├── mfe-kyc.json                    │
                    │   └── ...  (all 35 repos)             │
                    │                                      │
                    │   cross-repo/                         │
                    │   ├── dependency-map.md               │
                    │   ├── shared-components-registry.md   │
                    │   ├── api-contracts-index.md          │
                    │   └── domain-glossary.md              │
                    │                                      │
                    │   mcp-server/                         │
                    │   ├── server.py (FastAPI MCP server)  │
                    │   └── requirements.txt                │
                    └──────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
            ┌───────▼────────┐            ┌────────▼────────┐
            │ VSCode Copilot  │            │  PO Web UI      │
            │ (MCP stdio or   │            │  (OpenWebUI or  │
            │  SSE to server) │            │  simple chat)   │
            │                 │            │                 │
            │ Developers:     │            │ POs:            │
            │ Story analysis, │            │ Story brainstorm│
            │ implementation  │            │ impact queries  │
            └─────────────────┘            └─────────────────┘
```

### Files & Deliverables

**Central `project-intelligence` repository:**

| Directory | Content | Source |
|-----------|---------|--------|
| `wiki/{repo-name}/` | Enhanced documentation per repo (copied from DeepWiki, augmented with Copilot) | Export from DeepWiki → enhance with Copilot → human review |
| `manifests/{repo-name}.json` | Architecture manifests (promoted from Phase 1's per-repo `.github/` files) | Copied from each repo's `.github/architecture-manifest.json` |
| `cross-repo/dependency-map.md` | Which repos depend on which, shared component usage matrix | Generated via Copilot analysis of all manifests |
| `cross-repo/shared-components-registry.md` | Design system components: who uses what, prop contracts | Generated via Copilot `@workspace` + DeepWiki |
| `cross-repo/api-contracts-index.md` | All REST endpoints across all 5 backend services | Generated from manifests + Spring Boot controller analysis |
| `cross-repo/domain-glossary.md` | Business term → technical component mapping (for PO queries) | Hand-crafted with PO input |
| `mcp-server/` | Lightweight MCP server that serves wiki + manifest content as tools | Python FastAPI, ~200–300 lines |

**MCP server tools exposed:**

| Tool | Purpose | Primary User |
|------|---------|--------------|
| `find_repos_for_story` | Given story text, identify affected repos with confidence scores | Developer, PO |
| `get_repo_context` | Return full wiki + manifest for a specific repo | Developer |
| `impact_analysis` | Given a component/endpoint/collection, return all repos that use it | Developer, PO |
| `enhance_story` | Generate Technical Details section from story text + cross-repo context | Developer (grooming), PO |
| `search_wiki` | Full-text search across all 35 repos' documentation | Developer, PO |
| `cross_repo_dependencies` | Return dependency graph for a given repo | Developer |

**Updated per-repo files:**

| File | Change from Phase 1 |
|------|---------------------|
| `.vscode/mcp.json` | Points to self-hosted MCP server instead of Devin MCP |
| `.github/agents/story-analyzer.agent.md` | Updated to use `project-intelligence` MCP tools instead of DeepWiki MCP |
| `.github/agents/implementation-planner.agent.md` | No change |
| `.github/agents/code-reviewer.agent.md` | No change |

### Wiki Maintenance Process

| Trigger | Action | Automation Level |
|---------|--------|-----------------|
| New repo added to project | Generate wiki + manifest using Copilot, add to `project-intelligence` repo | Semi-automated (Copilot generates, human reviews and merges PR) |
| Major architectural change (new service, API redesign) | Re-generate affected repo's wiki section using Copilot | Semi-automated |
| Quarterly refresh | Re-run Copilot analysis on all 35 repos, diff against existing wiki, update stale sections | Semi-automated (GitHub Action triggers Copilot analysis, creates PR) |
| Shared component contract change | Update `shared-components-registry.md` | Manual (design system team owns this) |
| New API endpoint added | Update `api-contracts-index.md` and repo manifest | Semi-automated (detectable from Spring Boot controller changes in PR) |

### Rollout Plan

| Week | Activity |
|------|----------|
| Week 3 | Export DeepWiki content for 3 MVP repos. Set up `project-intelligence` repo. Build MCP server. Test with MVP teams. |
| Week 4 | Export and enhance DeepWiki for remaining 32 repos. Build cross-repo documents. |
| Week 5 | Deploy MCP server (self-hosted or as GitHub-hosted process). Update all repos' `.vscode/mcp.json`. |
| Week 6 | Deploy PO-facing web UI (OpenWebUI or equivalent). Train POs on story brainstorming workflow. Measure adoption. |

### PO Workflow (New in Phase 2)

```
Step 1: PO opens web chat UI connected to project-intelligence MCP
Step 2: PO types story idea in business language:
        "I want clients to copy KYC forms from previous submissions"
Step 3: System queries wiki + manifests → identifies affected repos, services, APIs
Step 4: System generates draft story with:
        - User story (As a / I want / So that)
        - Acceptance criteria (Given / When / Then)
        - Technical Details section (affected repos, components, APIs, MongoDB collections)
Step 5: PO reviews, adjusts business language, pushes to JIRA
Step 6: Developer picks up JIRA ticket → Phase 1 agent workflow takes over
```

### Decision Gate: End of Week 6

- **Proceed to Phase 3 if:** Developers consistently request symbol-level queries that wiki/manifests cannot answer (e.g., "find every file that calls this function across all 35 repos", "what breaks if I change this TypeScript interface").
- **Stay in Phase 2 if:** Wiki-level context is sufficient for story analysis and implementation planning.

---

## Phase 3: Graph Intelligence Platform (Weeks 7–12)

### Objective
Add real-time, symbol-level code intelligence using SCIP indexers and a graph database, triggered automatically by CI/CD pipeline events. This enables queries that static documentation cannot answer: precise blast radius analysis, cross-repo type resolution, dangling reference detection, and automated architectural drift alerts.

### When This Phase Is Justified

Phase 3 is only justified if Phase 2 reveals these specific gaps:

| Gap | Example Query Phase 2 Cannot Answer |
|-----|--------------------------------------|
| Symbol-level cross-referencing | "Which files in MFE-order import and call `usePaymentV2` from MFE-checkout?" |
| Blast radius with type resolution | "If I change `FormField` props in the design system, which MFEs pass the old prop shape?" |
| Automated staleness detection | "The wiki says the API takes `PaymentRequest`, but the code now uses `PaymentRequestV2`" |
| Dangling reference detection | "MFE-checkout renamed a hook but MFE-order still imports the old name" |

### Architecture

```
35 Repos (GitHub Actions CI)
        │
        │  POST index.scip on merge to develop/release/main
        ▼
┌─────────────────────────────────────────────────────┐
│        Code Intelligence Server (single VM)          │
│                                                     │
│   Ingestion API (FastAPI)                           │
│        │                                            │
│        ▼                                            │
│   Branch-Aware Router                               │
│   (main → PO graph, release/* → dev graph)          │
│        │                                            │
│        ▼                                            │
│   ArcadeDB (Apache 2.0, JVM)                       │
│   Graph + Document + Vector (built-in MCP)          │
│        │                                            │
│        ▼                                            │
│   Cross-Repo Consistency Checker                    │
│   (dangling refs, version skew, contract drift)     │
│        │                                            │
│        ▼                                            │
│   MCP Server (SSE for POs, stdio for developers)    │
│   Enhanced tools: symbol_lookup, blast_radius,      │
│   type_resolution, consistency_warnings             │
│                                                     │
│   + Phase 2 wiki/manifest tools (retained)          │
└─────────────────────────────────────────────────────┘
```

### CI/CD Changes (All 35 repos)

| File | Location | Purpose |
|------|----------|---------|
| `code-intelligence.yml` | `.github/workflows/` | GitHub Action: runs SCIP indexer on merge to develop/release/main, uploads index.scip to central server |

**Indexers used (all Apache 2.0, open source):**
- React MFE repos: `scip-typescript` (requires `npm ci` + `tsc` compilation in CI)
- Java Spring Boot repos: `scip-java` (uses `semanticdb` compiler plugin, requires Gradle/Maven build)

### Graph Database Schema (ArcadeDB)

| Node Type | Properties | Description |
|-----------|------------|-------------|
| Symbol | name, kind, filePath, repo, branch, startLine, endLine, commitSha | Functions, classes, hooks, components, interfaces |
| File | path, repo, branch, language | Source files |
| MongoCollection | name, repo | MongoDB collection (from `@Document` annotations) |

| Edge Type | From → To | Description |
|-----------|-----------|-------------|
| DEFINED_IN | Symbol → File | Where a symbol is defined |
| REFERENCES | Symbol → Symbol | One symbol references another |
| IMPORTS | File → File | Import relationships |
| CALLS | Symbol → Symbol | Function/method calls |
| QUERIES_COLLECTION | Symbol → MongoCollection | Spring Data repository → MongoDB collection |

### New MCP Tools (Additive to Phase 2)

| Tool | Purpose | Answers |
|------|---------|---------|
| `symbol_lookup` | Find where a symbol is defined and all references | "Where is `usePaymentV2` defined and who calls it?" |
| `blast_radius` | Given a symbol change, return all transitively affected files/repos | "If I change `FormField` props, what breaks?" |
| `consistency_check` | Return current cross-repo inconsistencies | "Are there dangling imports after today's merges?" |
| `collection_usage` | All code paths that query a MongoDB collection | "What code reads/writes the `kyc_submissions` collection?" |
| `recent_changes` | Symbols added/removed/renamed since a given date | "What changed in payment-service since last release?" |

### Updated Per-Repo Files

| File | Change from Phase 2 |
|------|---------------------|
| `.github/workflows/code-intelligence.yml` | **New.** SCIP indexing step added to CI. |
| `.vscode/mcp.json` | Updated to include graph-backed MCP endpoint alongside wiki MCP. |
| `.github/agents/story-analyzer.agent.md` | Updated to use graph tools (`blast_radius`, `symbol_lookup`) in addition to wiki tools. |

### Reconciliation Model

| Event | Action | Staleness |
|-------|--------|-----------|
| PR merges to develop/release/main | GitHub Action runs SCIP indexer → uploads index.scip → server atomically replaces repo's graph partition | ~1–2 minutes after merge |
| Multiple repos merge simultaneously | ArcadeDB handles concurrent writes natively (no write serialization queue needed) | Each repo independent |
| Cross-repo inconsistency detected | Stored as queryable Warning nodes in graph, surfaced via `consistency_check` tool | Real-time after index update |
| Branch-aware queries | PO queries filter to `main` branch. Developer queries filter to `release/*` with fallback to `main`. | Automatic per query |

### Rollout Plan

| Week | Activity |
|------|----------|
| Week 7 | Deploy ArcadeDB on VM (single `java -jar`, no Docker). Design graph schema. Build ingestion API. |
| Week 8 | Add SCIP indexing to CI for 3 MVP repos. Test index upload → graph insert pipeline. |
| Week 9 | Build enhanced MCP server with graph-backed tools. Test with MVP developers. |
| Week 10 | Roll SCIP CI workflow to all 35 repos (templated GitHub Action, identical across repos). |
| Week 11 | Integrate graph tools into PO web UI. Add consistency dashboard. |
| Week 12 | Full team rollout. Measure impact. Retrospective. |

### Operational Requirements

| Dimension | Detail |
|-----------|--------|
| Infrastructure | 1 Linux VM (8 CPU, 16 GB RAM) for ArcadeDB + MCP server + ingestion API |
| Storage | ~10–50 GB for graph database (35 repos, 3 branches each) |
| Backup | Nightly backup of ArcadeDB data directory + retain last N .scip index files |
| Monitoring | JVM metrics (heap, GC), API latency, index freshness per repo |
| Maintenance | ~2–4 hours/week ongoing: SCIP indexer updates, schema evolution, query tuning |
| Ownership | Platform engineering team or designated individual |

---

## Complete File Inventory Across All Phases

### Per-Repo Files (each of the 35 repos)

| File | Phase | Created By |
|------|-------|------------|
| `.github/copilot-instructions.md` | 1 | Copilot `/init` + `/create-instructions` → human review |
| `.github/architecture-manifest.json` | 1 | Copilot `@workspace` analysis → human review |
| `.github/agents/story-analyzer.agent.md` | 1 | Hand-crafted template, customized per repo |
| `.github/agents/implementation-planner.agent.md` | 1 | Shared template across all repos |
| `.github/agents/code-reviewer.agent.md` | 1 | Shared template across all repos |
| `AGENTS.md` | 1 | Hand-crafted per repo (project context, cross-repo references) |
| `.vscode/mcp.json` | 1 | Templated, updated in Phase 2 and Phase 3 |
| `src/hooks/.instructions.md` | 1 | Copilot-generated, human-reviewed |
| `src/components/.instructions.md` | 1 | Copilot-generated, human-reviewed |
| `src/services/.instructions.md` | 1 | Copilot-generated, human-reviewed |
| `.github/workflows/code-intelligence.yml` | 3 | Templated GitHub Action (identical across repos) |

### Central Repository (`project-intelligence`)

| File/Directory | Phase | Purpose |
|----------------|-------|---------|
| `wiki/{repo-name}/*.md` | 2 | Enhanced documentation per repo |
| `manifests/{repo-name}.json` | 2 | Consolidated architecture manifests |
| `cross-repo/dependency-map.md` | 2 | Inter-repo dependency graph |
| `cross-repo/shared-components-registry.md` | 2 | Design system usage matrix |
| `cross-repo/api-contracts-index.md` | 2 | All REST endpoints index |
| `cross-repo/domain-glossary.md` | 2 | Business term → technical mapping |
| `mcp-server/server.py` | 2 | MCP server (wiki + manifest queries) |
| `mcp-server/graph_tools.py` | 3 | Graph-backed MCP tools (additive) |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Developers don't adopt agent workflow | Medium | High (entire initiative fails) | Phase 1 MVP with 5–8 developers validates before scaling |
| DeepWiki content is too generic for implementation | Medium | Medium | Phase 2 enhances with project-specific context + manifests |
| SCIP indexing adds CI time | Low | Low | SCIP-TS adds ~30–60 sec per build. Acceptable for value gained. |
| Wiki documentation goes stale | High | Medium | Semi-automated quarterly refresh + PR-triggered updates for major changes |
| ArcadeDB operational burden (Phase 3) | Medium | Medium | Single VM, JVM-native. Nightly backup. 2–4 hr/week maintenance budget. |
| Devin MCP deprecation or API change | Low | Low (only Phase 1 risk) | Phase 2 eliminates Devin dependency entirely |

---

## Cost Summary

| Phase | Infrastructure Cost | Tool Cost | People Cost |
|-------|--------------------|-----------| ------------|
| Phase 1 | $0 | Devin ACUs (bounded, MVP only) + existing Copilot Enterprise license | 1 developer, 1 week setup + 1 week testing |
| Phase 2 | $0 (GitHub repo) or ~$50/mo (small VM for MCP server) | $0 (all open source) | 1 developer, 2 weeks setup + 2 weeks rollout |
| Phase 3 | ~$100–200/mo (VM for ArcadeDB + MCP) | $0 (ArcadeDB Apache 2.0, SCIP indexers Apache 2.0) | 1–2 developers, 4 weeks build + 2 weeks rollout. ~2–4 hr/week ongoing. |

---

## Constraints Honored

| Constraint | How Honored |
|------------|-------------|
| Open source tools only | ArcadeDB (Apache 2.0), SCIP indexers (Apache 2.0), OpenWebUI (MIT), MCP server (custom, in-house) |
| No Docker | ArcadeDB runs as `java -jar`. MCP server runs as Python process. SCIP indexers run as npm/Gradle CLI tools in CI. |
| No Yarn | All npm operations use `npm ci` / `npx`. No Yarn dependency anywhere. |
| Leverage existing tools | VSCode Copilot Enterprise (existing), Devin (existing, Phase 1 only), GitHub Actions (existing) |

---

## Approval Request

We request approval to proceed with **Phase 1 immediately** (zero cost, zero infrastructure, 1 developer for 2 weeks) with a decision gate at Week 2 to determine Phase 2 progression.

---

*Document prepared for architecture review. Detailed implementation specifications (agent files, instruction templates, MCP server code, CI workflows) available on request.*
