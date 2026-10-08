# TRACE AI Orchestration Architecture

**Project:** TRACE — Tracing Research Across Connected Evidence & Reasoning  
**Document:** `docs/AI_ORCHESTRATION.md`  
**Status:** Authoritative architecture contract (frozen for implementation planning)  
**Scope:** AI layer design only — not an implementation guide for code in this change  
**Audience:** Backend engineers, AI integration owners, and reviewers of the agent pipeline  

> **Scope rule:** This document defines the TRACE AI architecture, orchestrator rules, agent contracts, provenance requirements, GraphRAG role, shared research state, failure handling, and non-goals. It does **not** ship agents, prompts, LLM clients, orchestrator code, controllers, routes, or services. Implementation must follow this document as the single source of truth for the AI layer.

---

## Document Control

| Field | Value |
| --- | --- |
| Version | 1.0.0 |
| Effective date | 2026-08-08 |
| Classification | Architecture contract |
| Supersedes | Informal “Connector Agent” placeholder naming in older UI/agent stubs |
| Related docs | `docs/database_design.md`, GraphRAG retrieval layer (`backend/src/graphRag/`), Knowledge Graph Builder (`backend/src/knowledgeGraph/`) |
| Change policy | Material changes require explicit architecture review; do not silently diverge in code |

### Agent roster (final)

| # | Agent ID | Label |
| --- | --- | --- |
| 1 | `planner` | Planner Agent |
| 2 | `explorer` | Explorer Agent |
| 3 | `evidence_analyst` | Evidence Analyst Agent |
| 4 | `critic` | Critic Agent |
| 5 | `synthesizer` | Synthesizer Agent |

**Agent count: 5.** There is **no** Knowledge Graph Agent and **no** separate Connector Agent in the final architecture.

Naming note: earlier frontend/backend stubs used a fifth slot labeled “Connector.” That role is **replaced** by **Evidence Analyst**, which reasons over the already-built knowledge graph rather than constructing it.

---

## Table of Contents

1. [Purpose of the TRACE AI Layer](#1-purpose-of-the-trace-ai-layer)
2. [AI Architecture](#2-ai-architecture)
3. [Research Orchestrator](#3-research-orchestrator)
4. [Agent Responsibilities](#4-agent-responsibilities)
5. [GraphRAG Role](#5-graphrag-role)
6. [Shared Research State](#6-shared-research-state)
7. [Agent Contract](#7-agent-contract)
8. [Machine-Readable Output Contracts](#8-machine-readable-output-contracts)
9. [Provenance](#9-provenance)
10. [Failure Handling](#10-failure-handling)
11. [Bounded Research Loop](#11-bounded-research-loop)
12. [Database Responsibilities](#12-database-responsibilities)
13. [Explainability](#13-explainability)
14. [AI Model Abstraction](#14-ai-model-abstraction)
15. [Execution Sequence](#15-execution-sequence)
16. [Non-Goals](#16-non-goals)
17. [Future Extensions](#17-future-extensions)
18. [Implementation Guardrails](#18-implementation-guardrails)

---

## 1. Purpose of the TRACE AI Layer

TRACE does not exist to return a one-shot chat answer.

The AI layer must:

1. Decompose a research question into an actionable plan  
2. Discover relevant literature through existing TRACE services  
3. Use connected graph evidence via the deterministic GraphRAG retrieval layer  
4. Analyze relationships across papers without inventing edges  
5. Identify supporting and conflicting evidence  
6. Identify research gaps  
7. Assess evidence quality and confidence  
8. Produce an explainable research report  
9. Preserve provenance from final claims back to source papers and graph evidence  

Every important claim in the final report must be traceable to evidence.

---

## 2. AI Architecture

### 2.1 Five agents only

TRACE’s AI layer consists of exactly five agents:

1. **Planner Agent** — plans the research  
2. **Explorer Agent** — retrieves literature and graph context  
3. **Evidence Analyst Agent** — analyzes retrieved evidence and existing graph relationships  
4. **Critic Agent** — validates support, contradictions, gaps, and confidence  
5. **Synthesizer Agent** — writes the structured research report  

### 2.2 Knowledge Graph Builder is not an agent

TRACE already has a **deterministic Knowledge Graph Builder** that creates and persists:

- Paper nodes  
- Author nodes  
- Concept nodes  
- Venue nodes  
- Graph relationships (`AUTHOR_OF`, `HAS_CONCEPT`, `PUBLISHED_IN`, `RELATED_TO`, `CITES` when available)  

| Component | Responsibility |
| --- | --- |
| **Knowledge Graph Builder** | Builds and persists the graph from canonical papers |
| **Evidence Analyst** | Analyzes the existing graph and retrieved evidence |

**Do not** create a Knowledge Graph Agent.  
**Do not** allow the Evidence Analyst to rebuild, mutate topology, or invent relationships outside retrieved/persisted graph data.

### 2.3 Layered stack (conceptual)

```
┌──────────────────────────────────────────────────────────┐
│                 Research Orchestrator                    │
│         (order, state, retries, refinement)              │
└───────────────┬──────────────────────────────────────────┘
                │
     ┌──────────┼──────────┬──────────────┬──────────────┐
     ▼          ▼          ▼              ▼              ▼
 Planner    Explorer  Evidence Analyst  Critic     Synthesizer
                │            │             │
                ▼            ▼             ▼
        ┌─────────────────────────────────────────┐
        │ Tools / deterministic TRACE services    │
        │ Discovery · Canonicalization · GraphRAG │
        │ KG Builder (infra) · Papers · Reports   │
        └─────────────────────────────────────────┘
```

Agents reason. Services retrieve, canonicalize, persist, and score deterministically.

---

## 3. Research Orchestrator

### 3.1 Ownership

The **Research Orchestrator** is the only component allowed to:

- Own execution order  
- Own state transitions for a research run  
- Invoke agents  
- Decide when retries or a bounded refinement loop are permitted  
- Attach a unique `runId` to the execution  
- Coordinate writes through existing backend services  

### 3.2 Hard rule: agents do not call agents

Agents **must not** directly invoke other agents.

All inter-agent sequencing flows through the Orchestrator.

### 3.3 Canonical workflow

```
User Query
    ↓
Orchestrator
    ↓
Planner
    ↓
Explorer
    ↓
Evidence Analyst
    ↓
Critic
    ↓
Synthesizer
    ↓
Research Report
```

Optional single refinement cycle (see [§11](#11-bounded-research-loop)):

```
Critic → Orchestrator → Explorer → Evidence Analyst → Critic → Synthesizer
```

### 3.4 Orchestrator responsibilities

| Responsibility | Description |
| --- | --- |
| Validation | Ensure session ownership, non-empty query, runnable session state |
| Run identity | Create `runId` for the execution |
| Sequencing | Call agents in contract order |
| Package assembly | Pass only validated structured outputs downstream |
| Retries | Apply bounded, agent-specific retries |
| Refinement | Allow at most N evidence-refinement cycles (default 1) |
| Persistence coordination | Trigger service-level writes (activities, report shell updates, explainability) |
| Terminal status | Mark run completed, partial, or failed without uncontrolled restart |

### 3.5 Orchestrator must NOT

- Perform LLM reasoning itself as a sixth “hidden agent”  
- Allow recursive peer-to-peer agent calls  
- Restart discovery from scratch for ordinary synthesizer failures  
- Emit a fully confident report when critic validation is incomplete  
- Bypass provenance requirements  

---

## 4. Agent Responsibilities

For each agent: purpose, inputs, outputs, tools, read/write scope, prohibitions, failure behavior, and provenance.

---

### 4.1 Planner Agent

#### Purpose

Decompose the research question into a structured research plan that downstream agents can execute without improvising scope.

#### Inputs

| Input | Source |
| --- | --- |
| Research question | User / session |
| Session context | Session metadata, domain, filters, selected sources |
| Available research sources | Discovery provider registry (e.g. OpenAlex, Semantic Scholar) |
| Prior session artifacts (optional) | Existing papers/report status markers only — not free-form chat memory |

#### Outputs (structured)

- `researchObjective`  
- `subQuestions[]`  
- `searchQueries[]`  
- `evidenceRequirements[]`  
- `researchDimensions[]`  
- `stoppingCriteria`  
- `provenance` / plan rationale metadata (structured, not hidden CoT)  

#### Tools it may use

- None required for retrieval  
- May read source/provider capability metadata via services  

#### Data it may read

- Research session shell  
- Provider capabilities  
- Optional high-level counts (e.g. existing paper count)  

#### Data it may write

- None directly  
- Orchestrator may persist plan artifacts into run metadata / activities / report objective fields via services  

#### What it must NOT do

- Retrieve papers  
- Call Discovery, GraphRAG, or KG Builder  
- Invent paper references  
- Call other agents  

#### Failure behavior

Planner failure is **fatal** for the run. Orchestrator stops execution and records a failed/incomplete activity. No Explorer invocation.

#### Provenance requirements

Each search query and evidence requirement should retain linkage to the originating sub-question / research dimension IDs in the structured plan.

---

### 4.2 Explorer Agent

#### Purpose

Execute the research plan by retrieving literature and graph context through existing TRACE services, assembling an evidence package for analysis.

#### Inputs

| Input | Source |
| --- | --- |
| Validated Planner output | Orchestrator |
| Session ID, run ID | Orchestrator |
| Filters / source preferences | Session / plan |

#### Outputs (structured)

- Retrieved paper references (`paperId`, providers, ranks/scores)  
- Discovery summaries (sources requested/succeeded/failed)  
- GraphRAG context snapshots (nodes, edges, paths, evidence items)  
- Insufficiency signals (`insufficientEvidence`, missing dimensions)  
- Follow-up retrieval log (bounded)  

#### Tools it may use

- Discovery service (OpenAlex, Semantic Scholar, future registered providers)  
- GraphRAG retrieval service (`POST /api/v1/graph-rag/retrieve` / service equivalent)  
- Document listing for user uploads (metadata only at this stage unless full-text APIs exist)  
- Knowledge Graph **read** APIs (not rebuild)  

#### Data it may read

- Session (owned)  
- Papers for the session  
- Knowledge graph for the session  
- Documents metadata  
- Planner output  

#### Data it may write

- None directly to MongoDB  
- Orchestrator/services may persist newly discovered papers via existing discovery/canonicalize/persist pipeline when Explorer requests search execution through those services  

#### What it must NOT do

- Write the final research report  
- Perform critic-level confidence assignment for the whole report  
- Build or rewrite the knowledge graph topology  
- Call Critic / Synthesizer / Planner  
- Drop paper IDs, provider attribution, or GraphRAG node/path IDs  

#### Failure behavior

| Case | Behavior |
| --- | --- |
| Partial provider failure | Continue if remaining evidence meets minimum usability |
| Total discovery failure | Mark evidence package failed/empty; Orchestrator may stop or continue with explicit insufficiency |
| GraphRAG empty graph | Return empty graph context; continue with paper-level evidence if any |

#### Provenance requirements

Every retrieved item must preserve:

- `sessionId`  
- `paperId`  
- `nodeId` where applicable  
- `source` / provider list  
- retrieval scores / matched fields when from GraphRAG  
- path IDs when from traversal  

---

### 4.3 Evidence Analyst Agent

#### Purpose

Reason over retrieved evidence and the **existing** knowledge graph to produce structured analytical findings: concepts, relationships, patterns, support strength, and weakly supported areas.

#### Inputs

| Input | Source |
| --- | --- |
| Explorer evidence package | Orchestrator |
| GraphRAG context | Explorer / GraphRAG tool results already retrieved |
| Planner dimensions / questions | Planner output |
| Canonical papers (read) | Session papers |

#### Outputs (structured)

- Important concepts  
- Relationship observations grounded in graph edges / evidence paths  
- Thematic patterns  
- Supporting evidence sets per finding  
- Weakly supported findings  
- Structured analytical findings with evidence IDs / paper IDs  

#### Tools it may use

- GraphRAG context already in the evidence package (primary)  
- Optional additional GraphRAG calls only if Orchestrator grants a tool call within the analyst stage budget (preferred: consume Explorer package to avoid duplicate retrieval)  
- Read-only paper/graph services  

#### Data it may read

- Papers, graph, GraphRAG context, plan, Explorer outputs  

#### Data it may write

- None directly  
- Orchestrator may store analysis artifacts in run/explainability payloads via services  

#### What it must NOT do

- **Build, mutate, or “fix” the knowledge graph**  
- Invent relationships not present in graph edges or explicit paper metadata  
- Produce final report prose sections  
- Call other agents  

#### Failure behavior

- Retry **once**  
- If still failing: mark analysis incomplete; Orchestrator continues with incomplete flag; Critic must not certify high confidence  

#### Provenance requirements

Every analytical finding **must** reference:

- `findingId`  
- `evidenceIds[]` and/or `paperIds[]`  
- optional `nodeIds[]` / `edgeIds[]` / `pathIds[]`  

Unsupported speculation is forbidden unless explicitly typed as `hypothesis` or `insufficient_evidence`.

---

### 4.4 Critic Agent

#### Purpose

Validate claim-to-evidence support, detect contradictions and gaps, assign structured confidence, and block unjustified certainty.

#### Inputs

| Input | Source |
| --- | --- |
| Research question + plan | Planner |
| Evidence package | Explorer |
| Analytical findings | Evidence Analyst |
| Optional GraphRAG paths | For verifying support paths |

#### Outputs (structured)

- Evidence quality assessments  
- Claim support checks  
- Contradictions  
- Research gaps  
- Insufficient-evidence flags  
- Structured confidence assessments (overall + per finding where applicable)  
- Blockers / weak-conclusion flags  

#### Tools it may use

- GraphRAG (verify evidence paths)  
- Read-only paper/graph services  

#### Data it may read

- Full evidence package, analyst findings, graph context, papers  

#### Data it may write

- None directly  
- Orchestrator persists critic results into report/explainability fields via services  

#### What it must NOT do

- Perform new open-ended literature discovery beyond granted verification tools  
- Clear insufficiency flags without new evidence  
- Call Synthesizer directly  
- Allow a “fully confident” status when validation is incomplete  

#### Failure behavior

- Retry **once**  
- If validation remains incomplete: Orchestrator **must not** allow a fully confident final report  
- Synthesizer may still run with reduced confidence and explicit gaps  

#### Provenance requirements

Every criticism, contradiction, gap, and confidence judgment must retain links to the findings/evidence/paper IDs it evaluates.

---

### 4.5 Synthesizer Agent

#### Purpose

Synthesize the provided evidence package into the structured Research Report without performing new research retrieval.

#### Inputs

| Input | Source |
| --- | --- |
| Original research question | User/session |
| Research plan | Planner |
| Retrieved evidence | Explorer |
| Evidence analysis | Evidence Analyst |
| Critic findings | Critic |
| Confidence / gap / contradiction package | Critic |

#### Outputs (structured report sections)

1. Research Summary  
2. Key Findings  
3. Supporting Literature  
4. Research Gap Analysis  
5. Contradictions  
6. Confidence  
7. References  
8. Recommendations  

#### Tools it may use

- **None for retrieval**  
- May format citations from the provided paper list only  

#### Data it may read

- The assembled evidence package only (plus session title/question)  

#### Data it may write

- None directly  
- Orchestrator writes report content through Research Report service  

#### What it must NOT do

- Call Discovery or GraphRAG  
- Introduce papers not present in the evidence package  
- Invent findings not supported by Analyst/Critic packages  
- Suppress critic gap/contradiction flags  
- Call other agents  

#### Failure behavior

- Preserve previous artifacts (papers, graph, prior draft report)  
- Allow retry of Synthesizer **without** repeating discovery  
- Do not restart the entire research run  

#### Provenance requirements

Important claims must include source/evidence references. Claims without evidence must be typed as:

- `hypothesis`  
- `insufficient_evidence`  
- `recommendation`  

---

## 5. GraphRAG Role

### 5.1 GraphRAG is not an agent

GraphRAG is a **deterministic retrieval service/tool**.

It is available to agents through the Orchestrator/tool interface. It does not appear in the agent roster.

### 5.2 Who uses GraphRAG

| Agent | GraphRAG usage |
| --- | --- |
| Planner | No |
| Explorer | **Primary** — evidence + graph context retrieval |
| Evidence Analyst | Uses GraphRAG context for relationship-aware analysis |
| Critic | May use GraphRAG to verify evidence paths |
| Synthesizer | **Must not** perform new GraphRAG retrieval |

### 5.3 Foundation and restrictions

The existing deterministic GraphRAG retrieval layer (`backend/src/graphRag/`) remains the foundation:

- Token/field relevance scoring  
- Graph traversal from seed nodes  
- Structured context with provenance  

**Not in scope for this architecture stage** (unless later promoted under Future Extensions):

- Embeddings / vector databases  
- Community detection / DRIFT-style GraphRAG variants  
- LLM-based graph rewriting  

---

## 6. Shared Research State

### 6.1 No separate conversational memory

TRACE does **not** introduce a chat memory subsystem or `AgentMemory` collection.

The existing data model remains the source of truth:

```
Research Session
    ├── Documents
    ├── Papers
    ├── Knowledge Graph
    ├── Research Report
    ├── Explainability
    └── Research Activities
```

### 6.2 Run identity

Each TRACE AI execution receives a unique **`runId`**.

Attach `runId` to:

- Relevant research activity metadata  
- Report `generationMetadata`  
- Explainability / reasoning provenance payloads where stored  

### 6.3 Agent I/O pattern

Each agent:

1. Receives structured input from the Orchestrator  
2. Returns structured output to the Orchestrator  
3. Never depends on free-form prose from another agent as its sole contract  

Persisted durable state lives in Mongo collections via services; ephemeral run packages may be held in orchestrator memory for the duration of the run.

---

## 7. Agent Contract

Every agent follows the same execution contract:

```
Input
  ↓
Input validation (schema)
  ↓
LLM reasoning (via LLM adapter)
  ↓
Structured output
  ↓
Output validation (schema)
  ↓
Return to Orchestrator
```

Rules:

1. Invalid input → agent stage fails (no silent coercion of critical fields)  
2. Invalid output → retry once where policy allows; else stage failure  
3. Free-form narrative fields are optional annotations only; **machine-readable fields are mandatory**  
4. Agents never call other agents  
5. Agents never own Mongo writes  

---

## 8. Machine-Readable Output Contracts

The following schemas are architectural contracts. Field names may be refined at implementation time if semantically equivalent and versioned.

### 8.1 Common envelopes

```json
{
  "agentId": "planner|explorer|evidence_analyst|critic|synthesizer",
  "runId": "string",
  "sessionId": "ObjectIdstring",
  "status": "completed|partial|failed",
  "errors": [],
  "provenance": {
    "producedAt": "ISO-8601",
    "model": { "provider": "string", "name": "string" },
    "inputArtifactIds": []
  }
}
```

### 8.2 Planner output

```json
{
  "researchObjective": "string",
  "subQuestions": [{ "id": "sq1", "text": "string" }],
  "searchQueries": [{ "id": "q1", "query": "string", "subQuestionIds": ["sq1"] }],
  "evidenceRequirements": [{ "id": "er1", "description": "string", "priority": "required|optional" }],
  "researchDimensions": [{ "id": "d1", "label": "string" }],
  "stoppingCriteria": {
    "minPapers": 0,
    "maxDiscoveryCalls": 0,
    "notes": "string"
  }
}
```

### 8.3 Explorer output

```json
{
  "papers": [{
    "paperId": "string",
    "title": "string",
    "providers": ["openalex"],
    "relevanceScore": 0,
    "nodeId": "string|null"
  }],
  "graphContext": {
    "nodes": [],
    "edges": [],
    "paths": [],
    "evidence": []
  },
  "discovery": {
    "sourcesRequested": [],
    "sourcesSucceeded": [],
    "sourcesFailed": [],
    "partial": false
  },
  "insufficientEvidence": false,
  "missingDimensions": [],
  "followUpRetrievals": []
}
```

### 8.4 Evidence Analyst output

```json
{
  "concepts": [{ "id": "c1", "label": "string", "nodeIds": [], "paperIds": [] }],
  "relationships": [{
    "id": "r1",
    "type": "string",
    "description": "string",
    "edgeIds": [],
    "pathIds": [],
    "paperIds": []
  }],
  "themes": [{ "id": "t1", "label": "string", "paperIds": [], "evidenceIds": [] }],
  "findings": [{
    "id": "f1",
    "statement": "string",
    "supportLevel": "strong|moderate|weak|insufficient",
    "paperIds": [],
    "evidenceIds": [],
    "nodeIds": [],
    "claimType": "finding|hypothesis|insufficient_evidence"
  }]
}
```

### 8.5 Critic output

```json
{
  "claimChecks": [{
    "findingId": "f1",
    "supported": true,
    "issues": [],
    "evidenceIds": [],
    "paperIds": []
  }],
  "contradictions": [{
    "id": "x1",
    "summary": "string",
    "paperIds": [],
    "findingIds": []
  }],
  "researchGaps": [{
    "id": "g1",
    "summary": "string",
    "severity": "critical|major|minor",
    "relatedFindingIds": [],
    "paperIds": []
  }],
  "confidence": {
    "overall": 0,
    "breakdown": {},
    "fullyConfidentAllowed": false
  },
  "refinementRecommended": false,
  "refinementReasons": []
}
```

### 8.6 Synthesizer output

```json
{
  "report": {
    "executiveSummary": "string",
    "keyFindings": [],
    "supportingEvidence": [],
    "researchGaps": {},
    "contradictions": [],
    "confidence": 0,
    "references": [],
    "recommendations": []
  },
  "claimBindings": [{
    "claimId": "cl1",
    "section": "keyFindings",
    "paperIds": [],
    "evidenceIds": [],
    "findingIds": [],
    "claimType": "finding|hypothesis|insufficient_evidence|recommendation"
  }]
}
```

---

## 9. Provenance

Provenance is a core TRACE requirement, not an optional debug trail.

### 9.1 Provenance chain

```
Final Report Claim
    ↓
Critic / Analyst Finding
    ↓
Evidence Item
    ↓
Paper / Graph Node
    ↓
Provider / Source
```

### 9.2 Rules

1. Important claims require evidence references (`paperIds` and/or `evidenceIds`).  
2. Graph-grounded relationship statements should cite `edgeIds` and/or `pathIds` when available.  
3. Claims without evidence must be explicitly marked as `hypothesis`, `insufficient_evidence`, or `recommendation`.  
4. Agents must not strip provider/source fields while transforming evidence.  
5. Orchestrator rejects synthesizer outputs that introduce unbound high-confidence claims.  

---

## 10. Failure Handling

| Stage | Policy |
| --- | --- |
| **Planner failure** | Stop execution. Record failure activity. No downstream agents. |
| **Explorer partial provider failure** | Continue if sufficient evidence remains; mark `discovery.partial`. |
| **Explorer total failure / empty usable evidence** | Prefer stop or continue with explicit insufficiency; do not pretend success. |
| **Evidence Analyst failure** | Retry once. Otherwise mark analysis incomplete and continue cautiously. |
| **Critic failure** | Retry once. Do **not** produce a fully confident final report if validation remains incomplete. |
| **Synthesizer failure** | Preserve previous artifacts. Retry synthesizer without repeating discovery. |

**Global rule:** Never restart the entire research run unnecessarily.

---

## 11. Bounded Research Loop

### 11.1 Allowed refinement path

```
Planner
  ↓
Explorer
  ↓
Evidence Analyst
  ↓
Critic
```

If Critic identifies a **critical** evidence gap and recommends refinement:

```
Critic
  ↓
Orchestrator
  ↓
Explorer
  ↓
Evidence Analyst
  ↓
Critic
  ↓
Synthesizer
```

### 11.2 Limits

| Setting | Default | Notes |
| --- | --- | --- |
| `maxRefinementCycles` | **1** | Configurable |
| Agent-to-agent recursion | **Forbidden** | Only Orchestrator may re-invoke Explorer |
| After limit reached | Continue | Produce report with remaining gaps clearly marked |

Uncontrolled autonomous loops are forbidden.

---

## 12. Database Responsibilities

Agents do **not** directly manipulate MongoDB. They use existing backend services/repositories via the Orchestrator/tools.

### 12.1 Who writes what

| Collection | Typical writers (services / infra) | Agent influence |
| --- | --- | --- |
| `research_sessions` | Session service; Orchestrator may update status / workspace pointers | Indirect |
| `papers` | Discovery + canonicalization + paper persistence pipeline (Explorer-triggered searches) | Explorer requests via services |
| `knowledge_graphs` | Knowledge Graph Builder / graph services | **Not** Evidence Analyst; build is infra/deterministic |
| `research_reports` | Report service (Orchestrator after Synthesizer / shells) | Synthesizer content via Orchestrator |
| `research_activities` | Activity service (Orchestrator and existing domain flows) | Stage start/complete/fail events |
| `explainability` | Explainability service (Orchestrator after critic/synthesis) | Critic / Analyst structured metadata |
| `research_documents` | Document upload services | Not core agent writes |

### 12.2 Ownership boundary

- **Orchestrator** = execution-level coordination  
- **Agents** = reasoning components  
- **Services/repositories** = database owners  

---

## 13. Explainability

Explainability is a first-class TRACE output.

### 13.1 Record where appropriate

- Evidence references  
- Source attribution  
- Confidence  
- Structured reasoning / provenance chains  
- Contradictions  
- Research gaps  

### 13.2 Hidden chain-of-thought

Do **not** expose or persist hidden chain-of-thought transcripts as user-facing explainability.

Store **structured** reasoning/provenance metadata only (IDs, support levels, matched evidence, path references).

---

## 14. AI Model Abstraction

Do not couple agent contracts to a single LLM vendor.

### 14.1 LLM adapter interface (conceptual)

```
LLMAdapter.complete({
  system,
  messages,
  responseSchema,
  temperature?,
  timeout?
}) → { content, raw?, modelMeta }
```

### 14.2 Supported target backends (future adapters)

- Local models  
- OpenAI-compatible HTTP APIs  
- Other providers behind the same interface  

Agent input/output contracts remain independent of the underlying model.

---

## 15. Execution Sequence

Final sequence:

```
User Query
    ↓
Research Orchestrator          (create runId, validate session)
    ↓
Planner
    ↓
Explorer                       (Discovery + GraphRAG)
    ↓
Evidence Analyst               (analyze evidence + existing graph)
    ↓
Critic
    ↓
Optional one-time refinement   (Orchestrator → Explorer → Analyst → Critic)
    ↓
Synthesizer
    ↓
Research Report                (via report service)
    ↓
Explainability + Activities    (via explainability / activity services)
```

Parallel note: Knowledge Graph Builder may run as deterministic infrastructure when papers change (e.g. after discovery), **outside** the five-agent roster. Evidence Analyst always consumes the graph; it does not replace the builder.

---

## 16. Non-Goals

The AI layer will **NOT**:

1. Replace the deterministic Discovery layer  
2. Replace Canonicalization  
3. Replace the Knowledge Graph Builder  
4. Directly manipulate MongoDB from agents  
5. Allow agents to call each other  
6. Expose hidden chain-of-thought  
7. Introduce uncontrolled autonomous loops  
8. Introduce unnecessary AI agents (including a Knowledge Graph Agent)  
9. Introduce embeddings / vector databases at this stage  
10. Treat GraphRAG as an agent  
11. Create an `AgentMemory` collection as a conversational memory system  
12. Allow Synthesizer to perform new research retrieval  

---

## 17. Future Extensions

Documented only as future possibilities — **not** part of the current implementation contract:

- Semantic / vector retrieval  
- Richer GraphRAG global search / community summaries  
- Additional research sources beyond OpenAlex + Semantic Scholar  
- `AgentExecutions` collection if durable per-stage execution persistence becomes necessary  
- Human-in-the-loop review gates  
- Parallel agent execution within Orchestrator-controlled bounds  
- More advanced research planning / multi-objective planning  

Do not implement these under the cover of this document.

---

## 18. Implementation Guardrails

When implementation begins in later tasks:

1. Implement Orchestrator before enabling multi-agent HTTP entrypoints  
2. Implement LLM adapter before provider-specific code spreads into agents  
3. Validate every agent I/O schema at boundaries  
4. Keep GraphRAG and KG Builder deterministic unless an explicit later RFC changes that  
5. Align frontend agent labels with this roster (`evidence_analyst`, not `connector`) when the AI UI phase lands  
6. Preserve `runId` on activities and report generation metadata  

---

## Appendix A — Responsibility Matrix (summary)

| Concern | Owner |
| --- | --- |
| Execution order | Orchestrator |
| Research plan | Planner |
| Literature + GraphRAG retrieval | Explorer |
| Graph topology construction | Knowledge Graph Builder (deterministic infra) |
| Graph/evidence analysis | Evidence Analyst |
| Validation / gaps / confidence | Critic |
| Final report synthesis | Synthesizer |
| Mongo writes | Backend services via Orchestrator |
| Provenance integrity | All agents + Orchestrator enforcement |

---

## Appendix B — Glossary

| Term | Meaning |
| --- | --- |
| TRACE | Tracing Research Across Connected Evidence & Reasoning |
| runId | Unique identifier for one orchestrated AI execution |
| Evidence package | Orchestrator-assembled bundle of papers, GraphRAG context, and stage outputs |
| Fully confident report | Report state where Critic set `fullyConfidentAllowed=true` and no critical validation gaps remain |
| Refinement cycle | One Orchestrator-approved re-entry to Explorer → Analyst → Critic |

---

**End of architecture contract.**  
This document is documentation-only for the present task; no runtime behavior changes are authorized by this file alone.
