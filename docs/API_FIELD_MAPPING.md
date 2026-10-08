# TRACE API Field Mapping Contract

**Project:** TRACE — Tracing Research Across Connected Evidence & Reasoning  
**Document type:** Integration mapping contract  
**Audience:** Backend engineers implementing research-source connectors and mappers  
**Status:** Canonical reference for current and future providers  

> **Scope rule:** This document defines the **internal Paper DTO** and how external academic APIs map into it. It does not prescribe HTTP client code, MongoDB schemas, or AI behavior. Connectors and mappers **must** follow this contract.

---

## Table of Contents

1. [Overview](#1-overview)
2. [TRACE Paper DTO](#2-trace-paper-dto)
3. [Field Mapping Table](#3-field-mapping-table)
4. [Normalization Rules](#4-normalization-rules)
5. [Data Quality Rules](#5-data-quality-rules)
6. [Provider Notes](#6-provider-notes)
7. [Future Extensibility](#7-future-extensibility)
8. [Implementation Principles](#8-implementation-principles)
9. [Provider Rollout Phases](#9-provider-rollout-phases)

---

## 1. Overview

TRACE retrieves scholarly papers from multiple external academic providers. Each provider exposes a **different JSON shape**, naming conventions, nesting depth, and completeness profile.

A dedicated **field mapping contract** is required so that:

- Connector authors know exactly which provider paths feed which TRACE fields.  
- Reviewers can validate mappings without reading scattered mapper code.  
- Downstream modules (Discovery, PaperService, GraphRAG, reports, AI agents) share one data language.  
- New providers can be onboarded without renegotiating field names across the stack.

### Why connectors must normalize

External payloads are **provider-specific contracts**. TRACE must not treat OpenAlex `display_name`, Semantic Scholar `title`, and CrossRef `title[0]` as interchangeable at the application layer. Normalization happens **at the integration boundary** (mapper), immediately after a connector receives a successful response (or when raw fixtures are adapted for tests).

### Why the rest of TRACE must not depend on provider JSON

If Discovery, PaperService, or the AI layer consume OpenAlex/Semantic Scholar objects directly:

- Provider API changes break core TRACE flows.  
- Multi-source merges become inconsistent (different author shapes, year formats, DOI prefixes).  
- Swapping or A/B testing sources requires rewriting business logic.

**Rule:** Every connector pipeline must ultimately expose the **same internal Paper DTO**, regardless of external provider. Provider-specific keys must not appear outside `integrations/` (connectors + mappers).

---

## 2. TRACE Paper DTO

The **Paper DTO** is the canonical in-memory / service-boundary object for discovered scholarly works. Persistence layers may store a compatible document (e.g. MongoDB `papers`), but **service and connector contracts** are defined by this DTO.

| Field | Type (logical) | Required at DTO boundary | Purpose |
| --- | --- | --- | --- |
| `sessionId` | ObjectId / string | Yes (when persisting to a workspace) | Associates the paper with a Research Session. Discovery sets this when saving; raw connector search results may omit it until persistence. |
| `title` | string | Yes | Primary bibliographic title. Used in UI lists, reports, and deduplication fallback. |
| `authors` | string[] | Yes (may be empty array) | Ordered list of author display names. Never `null`. |
| `abstract` | string \| null | No | Free-text abstract. May be missing for many provider records. |
| `doi` | string \| null | No | Normalized DOI (no resolver URL prefix). Primary cross-source identity key when present. |
| `venue` | string \| null | No | Journal, conference, preprint server, or host venue display name. |
| `publicationYear` | integer \| null | No | Four-digit publication year. |
| `paperType` | string \| null | No | Coarse work type (e.g. journal-article, proceedings-article, preprint). Provider vocabularies are mapped into a TRACE-friendly string; exact controlled vocabulary may evolve. |
| `citationCount` | integer | Yes (default `0`) | Cited-by count when the provider exposes it; otherwise `0`. |
| `references` | string[] \| object[] \| null | No | Outbound reference list when available. Prefer lightweight identifiers (DOI/title) over dumping full provider graphs. Structure may be refined when citation GraphRAG needs stabilize. |
| `keywords` | string[] | Yes (may be empty array) | Topics, subjects, MeSH-like terms, or keywords. Never `null`. |
| `source` | string enum | Yes | Provenance of the record: `openalex` \| `semantic_scholar` \| `crossref` \| `core` \| `pubmed` \| `arxiv` \| `manual` \| `upload` \| `other`. |
| `discoveryMethod` | string \| null | No | How TRACE obtained the record (e.g. `connector_search`, `connector_get`, `user_upload`, `manual`). Useful for audit and explainability without embedding AI logic. |
| `pdfUrl` | string (URL) \| null | No | Best available full-text or PDF URL when known. Not a guarantee of OA legality—display cautiously. |
| `openAccess` | boolean \| null | No | Whether the work is marked open access by the provider. `null` if unknown. |
| `peerReviewed` | boolean \| null | No | Peer-review signal when reliably known; otherwise `null` (do not invent). |
| `retractionStatus` | string \| null | No | Retraction / withdrawal signal when available (`none`, `retracted`, `withdrawn`, or provider-mapped value). Default conceptually to `none` when unknown at persistence time. |
| `correctionStatus` | string \| null | No | Correction / erratum signal when available. |
| `venueQuality` | string \| null | No | Optional venue quality / prestige hint. Do not fabricate scores; leave `null` unless a trusted field exists. |
| `processingStatus` | string \| null | No | TRACE pipeline status for this paper (e.g. `discovered`, `ready`, `processing`, `failed`). Not an external API field. |
| `createdAt` | datetime \| null | System | Set by TRACE on create. Not mapped from providers. |
| `updatedAt` | datetime \| null | System | Set by TRACE on update. Not mapped from providers. |

### Persistence alignment note

When saving via PaperService, DTO fields map into the Mongo paper document (for example `publicationYear` → `year`, integrity fields → nested `integrity`, DOI → `externalIds.doi`). The DTO remains the **integration and orchestration contract**; storage layout may nest or rename fields as long as round-trips preserve meaning.

---

## 3. Field Mapping Table

Paths below use **dot / bracket notation** for nested JSON. Values are the **typical** fields documented or commonly returned by each API as of this contract revision. Where TRACE cannot verify a stable path, the cell is marked **TBD** or **Future**.

| TRACE Field | OpenAlex | Semantic Scholar | CrossRef | CORE | PubMed (Future) | arXiv (Future) |
| --- | --- | --- | --- | --- | --- | --- |
| `sessionId` | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* |
| `title` | `display_name` (fallback: `title`) | `title` | `title[0]` (message work) | `title` | TBD | Future (`title`) |
| `authors` | `authorships[].author.display_name` | `authors[].name` | `author[]` → `given` + `family` | `authors[]` (string or `{ name }`) | TBD | Future (author list) |
| `abstract` | `abstract` **or** reconstruct from `abstract_inverted_index` when present | `abstract` | `abstract` (often JATS/XML; strip tags) | `abstract` / `description` | TBD | Future (`summary`) |
| `doi` | `doi` (may be URL; strip resolver) | `externalIds.DOI` | `DOI` | `doi` | TBD | Future (rarely; often null) |
| `venue` | `primary_location.source.display_name` or `host_venue.display_name` | `venue` or `journal.name` | `container-title[0]` | `publisher` or `journals[0].title` | TBD | Future (`journal-ref` / category) |
| `publicationYear` | `publication_year` | `year` | `published.date-parts[0][0]` (fallback print/online date-parts) | `yearPublished` / `year` | TBD | Future (from `published`) |
| `paperType` | `type` | `publicationTypes[0]` | `type` | `documentType` | TBD | Future (`preprint`) |
| `citationCount` | `cited_by_count` | `citationCount` | `is-referenced-by-count` | `citationCount` when present; else `0` | TBD | Future (usually unavailable → `0`) |
| `references` | TBD (`referenced_works` ids — map carefully) | `references` / reference payloads when requested | TBD (reference component may require extra calls) | TBD | Future | Future |
| `keywords` | `keywords[].display_name` or concept labels when used | `fieldsOfStudy[]` | `subject[]` | `topics[]` when present | TBD (MeSH) | Future (categories) |
| `source` | constant `"openalex"` | constant `"semantic_scholar"` | constant `"crossref"` | constant `"core"` | `"pubmed"` | `"arxiv"` |
| `discoveryMethod` | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* |
| `pdfUrl` | `open_access.oa_url` or `primary_location.pdf_url` | `openAccessPdf.url` | TBD (may require link relations) | `downloadUrl` / `sourceFulltextUrls[0]` | TBD | Future (PDF abs URL) |
| `openAccess` | `open_access.is_oa` | `isOpenAccess` or implied by `openAccessPdf` | TBD | `isOpenAccess` when present | TBD | Future (typically OA) |
| `peerReviewed` | *(usually unknown → `null`)* | *(usually unknown → `null`)* | *(usually unknown → `null`)* | *(usually unknown → `null`)* | Future | Future (`false` for preprint) |
| `retractionStatus` | TBD | TBD | TBD | TBD | Future | Future |
| `correctionStatus` | TBD | TBD | TBD | TBD | Future | Future |
| `venueQuality` | *(do not invent → `null`)* | *(do not invent → `null`)* | *(do not invent → `null`)* | *(do not invent → `null`)* | Future | Future |
| `processingStatus` | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* | *(TRACE-assigned)* |
| `createdAt` | *(TRACE system)* | *(TRACE system)* | *(TRACE system)* | *(TRACE system)* | *(TRACE system)* | *(TRACE system)* |
| `updatedAt` | *(TRACE system)* | *(TRACE system)* | *(TRACE system)* | *(TRACE system)* | *(TRACE system)* | *(TRACE system)* |

### Mapping discipline

- Prefer the **primary** path in each cell; document fallbacks in connector mapper comments when used.  
- Do **not** invent peer-review, venue quality, or retraction values when the provider does not supply them.  
- Rows marked **TBD** require a design note + PR update to this document before use in production mappers.

---

## 4. Normalization Rules

All mappers must apply the following rules before returning a Paper DTO.

### 4.1 Authors

- Convert author **objects** into an ordered **string array** of display names.  
- Skip empty names.  
- **Deduplicate** consecutive or exact duplicate author strings (case-insensitive preferred).  
- Never leave `authors` as `null` — use `[]` when unknown.

### 4.2 Missing values

- Use `null` for optional scalar fields that are absent or unusable (`abstract`, `doi`, `venue`, `pdfUrl`, booleans of unknown truth, etc.).  
- Use empty arrays (`[]`) for list fields that are absent (`authors`, `keywords`).  
- Do not use empty string and `null` interchangeably for the same optional scalar across providers—prefer `null` for “unknown / absent,” and `""` only when an empty string is intentionally stored (prefer `null` for venue/abstract when missing).

### 4.3 Publication year

- Coerce to a **four-digit integer** when possible.  
- From ISO dates (`2019-05-01`) take the year component.  
- From CrossRef `date-parts`, use the first element of the first date part.  
- If unparsable, set `publicationYear` to `null` (never `NaN` or `0` as a fake year).

### 4.4 DOI

- Trim whitespace.  
- Lowercase the DOI string for storage/compare.  
- Strip common resolver prefixes:  
  - `https://doi.org/`  
  - `http://doi.org/`  
  - `https://dx.doi.org/`  
  - `http://dx.doi.org/`  
- If the remaining value is empty, set `doi` to `null`.

### 4.5 Keywords / subjects

- Flatten nested keyword objects to strings (`display_name`, etc.).  
- Trim each keyword.  
- Drop empties; optionally dedupe case-insensitively.  
- Never leave `keywords` as `null`.

### 4.6 Strings and whitespace

- Trim `title`, `venue`, and other human-readable strings.  
- Collapse excessive internal whitespace in abstracts after XML/HTML stripping (CrossRef).

### 4.7 Citation counts

- Coerce to a non-negative integer.  
- Invalid / missing → `0`.

### 4.8 Provider-specific leakage

- Ignore fields not listed in this contract (raw ids, scores, inverted indices after abstract reconstruction, request diagnostics, etc.).  
- Do not attach `rawOpenAlex`, `_source`, or similar bags onto the DTO returned to Discovery.  
- If raw payloads must be retained for debugging, persistence may store a capped `raw` blob **inside PaperService / repository only**, never as part of the public connector DTO contract.

### 4.9 Source and discovery method

- Set `source` to the connector’s fixed id.  
- Set `discoveryMethod` in Discovery (or the calling service), not by guessing from provider JSON.

---

## 5. Data Quality Rules

| Rule | Requirement |
| --- | --- |
| Title required | Reject or skip records with missing/blank `title` after trim. |
| Authors never null | Always `string[]` (possibly empty). |
| Citation count default | Always an integer ≥ 0; default `0`. |
| Abstract optional | `string` or `null`. |
| PDF URL optional | Valid URL string or `null`. Do not invent. |
| DOI optional | Normalized string or `null`. |
| Empty records | Reject completely empty / unusable papers (no title and no DOI). |
| Session association | Persistence requires a valid `sessionId` belonging to the authenticated user (enforced outside connectors). |
| Source required | Every DTO must set `source`. |
| No fabricated integrity | `peerReviewed`, `retractionStatus`, `correctionStatus`, `venueQuality` must not be guessed. |

Records that fail hard rules (especially missing title) must not be passed to PaperService as valid discoveries.

---

## 6. Provider Notes

### 6.1 OpenAlex

- **Best at:** Broad scholarly graph coverage, OA signals, citation counts, concept/keyword-style metadata, stable work ids.  
- **Metadata quality:** Generally strong for title/authors/year/citations; abstracts may require inverted-index reconstruction.  
- **TRACE use:** **Default discovery source** for multi-hop literature intake and session paper seeding.

### 6.2 Semantic Scholar

- **Best at:** Paper-centric search, citation-oriented metadata, fields of study, convenient PDF OA links when present.  
- **Metadata quality:** Good titles and abstracts for many CS/AI works; coverage varies by domain.  
- **TRACE use:** Secondary discovery source; useful for complementary ranking and citation context alongside OpenAlex.

### 6.3 CrossRef

- **Best at:** DOI registry metadata, bibliographic correctness, publisher-registered titles/authors/container titles.  
- **Metadata quality:** High for DOI-backed works; abstracts often sparse or XML-encoded; citation counts via `is-referenced-by-count` when provided.  
- **TRACE use:** DOI enrichment, validation, and bibliographic normalization after initial discovery.

### 6.4 CORE

- **Best at:** Aggregated open-access full-text oriented discovery across repositories.  
- **Metadata quality:** Variable by repository; useful `downloadUrl` / full-text hints when present.  
- **TRACE use:** OA-oriented expansion and full-text URL candidates for later (non-connector) parsing pipelines.

### 6.5 PubMed (Future)

- Biomedical literature and MeSH-oriented indexing. Mapping rows remain **TBD** until a connector is scheduled.

### 6.6 arXiv (Future)

- Preprint metadata and categories; peer-review typically false/unknown. Mapping rows remain **Future** until scheduled.

---

## 7. Future Extensibility

Adding a new research provider must be an **integration-local** change:

1. **Connector** — Implement the shared connector interface (`search`, `getPaper`, `normalize`) under `integrations/`.  
2. **Mapping** — Add explicit rows to **§3** of this document and implement a provider mapper that emits the Paper DTO.  
3. **Registration** — Register the connector in the factory/registry (`createConnector` / source allow-list).

**No downstream module should require modification** solely because a new provider was added:

- Discovery continues to call `createConnector(sourceId)` and Paper DTO APIs.  
- PaperService continues to accept Paper DTOs only.  
- AI / GraphRAG / report modules continue to consume TRACE papers, not vendor JSON.

Any new DTO field requires an update to **§2** and a migration plan for PaperService/persistence.

---

## 8. Implementation Principles

Architectural rules for TRACE research-source integrations:

1. **Connector** retrieves (or will retrieve) **provider JSON** from the external API and may perform transport concerns (timeouts, retries, rate limits).  
2. **Mapper** converts provider JSON into the **Paper DTO** defined in this document—no TRACE feature logic beyond field mapping and normalization.  
3. **Discovery** orchestrates connectors and works **only** with Paper DTOs (plus session/auth context).  
4. **PaperService** never receives provider-specific objects; it accepts Paper DTOs (or TRACE-normalized paper inputs derived from them).  
5. **AI layer** never communicates directly with external academic APIs; agents call TRACE services that already hold normalized papers or invoke Discovery/connectors through approved application services.  
6. **This document is the source of truth.** If code and this contract disagree, update the contract via review **or** fix the code—never leave silent drift.

---

## 9. Provider Rollout Phases

External connectors are enabled incrementally. Discovery and PaperService already consume the shared connector interface, so later phases require **only** connector HTTP + mapping work—not orchestration redesign.

```text
Phase 1  →  OpenAlex
Phase 2  →  Semantic Scholar
Phase 3  →  CrossRef
Phase 4  →  CORE
```

| Phase | Provider | Role in TRACE | Status target |
| --- | --- | --- | --- |
| **1** | **OpenAlex** | Default discovery source; works search + work-by-id; primary citation/OA metadata | Live HTTP connector |
| **2** | **Semantic Scholar** | Secondary discovery; complementary paper search and fields of study | Live HTTP connector |
| **3** | **CrossRef** | DOI enrichment and bibliographic validation | Scaffold → live HTTP |
| **4** | **CORE** | OA / full-text oriented expansion | Scaffold → live HTTP |

**Rules for each phase**

- Implement `search`, `getPaper`, and `normalize` against this mapping contract.  
- Do not change Discovery or PaperService contracts when moving to the next phase.  
- Do not enable Phase *N+1* HTTP until Phase *N* returns stable Paper DTOs in integration tests.  
- Future providers (PubMed, arXiv) begin only after Phases 1–4 are complete unless product priority changes.

---

## Document control

| Item | Value |
| --- | --- |
| Path | `docs/API_FIELD_MAPPING.md` |
| Related code (informative) | `backend/src/integrations/**`, Discovery, PaperService |
| Related design | `docs/DATABASE_DESIGN.md` (`papers` collection) |
| Change process | PR required for mapping or DTO field changes |

**End of API Field Mapping Contract**
