# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

This repository currently contains **design documentation only** — there is no application code, build system, package manifest, or test suite yet. Do not assume any build/lint/test commands exist; check for their introduction before referencing them. The repo holds:

- `rag-engine-prd.md` — Product Requirements Document
- `rag-engine-tdd.md` — Technical Design Document
- `architecture-diagram-hl.svg` — high-level architecture (Sources → Ingestion → Storage & Indexing → Query & Generation)
- `architecture-diagram-ll-ingestion.svg`, `architecture-diagram-ll-query.svg`, `architecture-diagram-ll-storage.svg` — low-level diagrams for each zone, referenced inline by the TDD

When implementation work begins, prefer extending this file with real commands (install/build/lint/test/run) rather than inferring them.

## What this system is

An **Enterprise Multimodal RAG Platform** on Google Cloud, built on **Vertex AI RAG Engine** as the managed retrieval backbone. It ingests PDFs (text/tables/images), Confluence, SharePoint, Word, and JSON content, and serves grounded, cited answers through an ADK agent, at a target of ~1,000–2,000 queries/hour for ~1,000 users with P95 query-to-answer latency under ~4 seconds.

Full requirements: `rag-engine-prd.md`. Full design: `rag-engine-tdd.md` (read this before implementing any component — it is the source of truth for data schemas, service boundaries, and design rationale summarized below).

## Architecture (from the TDD)

Four zones, diagrammed in `architecture-diagram-hl.svg` and detailed per-zone in the `architecture-diagram-ll-*.svg` files:

```
Sources (GCS landing: PDFs, Confluence export, SharePoint, JSON)
   → Cloud Workflows (orchestration)
        → Cloud Run job: Document AI layout parser (parse)
        → Cloud Run job: table extraction + LLM summary → BigQuery raw tables
        → Cloud Run job: image extraction + Gemini captioning → GCS images
        → Cloud Run job: LLM tagging (entities, doc type, category)
        → Cloud Run job: layout-aware chunk assembly
        → Vertex AI RAG Engine: corpus.import_files (chunks + metadata)
        → BigQuery: catalog write (audit, table refs, image refs, tags, entity relations)
   → Firestore: per-document status tracking throughout

Nightly graph promotion job:
BigQuery (entity_relations, staged) → dedup/merge → Spanner Graph (nodes + edges)

Query path:
ADK Agent → RAG Engine retrieveContexts (metadata_filter + top_k)
          → (optional) Spanner Graph traversal for multi-hop/relational intent
          → (optional) BigQuery exact lookup via table_ref
          → Gemini generation (grounded) → cited answer to user
```

### Key architectural decisions to preserve when implementing

- **Standalone Document AI parsing before RAG Engine import, not RAG Engine's built-in parser.** This is deliberate (TDD §3.2, §5 risks): running Document AI as its own Cloud Run stage lets table/image extraction and tagging happen *before* chunks reach RAG Engine, giving full control over per-chunk metadata. RAG Engine's own import-time parser must stay disabled to avoid double-processing.
- **One corpus per generation request is a hard RAG Engine limit** (PRD §9). Corpora are segmented by logical domain/source (e.g. `corpus-product-specs`, `corpus-confluence`), and the ADK agent routes to/across corpora per query rather than using one giant corpus (TDD §3.7).
- **Embedding model is locked at corpus creation.** Changing it requires full corpus recreation and re-import — treat model selection as a reviewed, documented decision, not a config tweak.
- **The BigQuery catalog (`rag_catalog` dataset) is a governance/audit layer, not on the serving path.** It mirrors chunk metadata for compliance/reporting and stages data for the graph layer; it must stay in sync with the RAG Engine index because a single ingestion pipeline writes both, and Firestore only marks a document `indexed` after both writes succeed (TDD §5).
- **The knowledge graph (Spanner Graph) is additive, not a replacement for vector retrieval.** The ADK agent classifies query intent first and only invokes graph traversal for relational/multi-hop queries, to avoid adding latency to purely semantic queries (TDD §3.12).
- **Entity/relation extraction reuses the existing tagging pass** (§3.5), constrained to a versioned, controlled ontology (entity types, predicate vocabulary, type constraints) fed into the extraction prompt as a closed vocabulary — not left to free-form LLM labeling.
- **Firestore (`ingestion_status`, keyed by `doc_id`)** tracks per-document pipeline status (`pending/parsing/tagging/indexed/failed`) with retry counts; Cloud Workflows applies exponential-backoff retries (max 3) per step before marking a document `failed`.
- **The chunk/catalog metadata schema is uniform** across RAG Engine chunk metadata and the BigQuery `chunks` table: `doc_id`, `chunk_id`, `page`, `content_type` (`text | table_summary | image_caption`), `doc_type`, `category`, `entities`, `source`, `date`, `table_ref`, `image_uri` (TDD §3.5). Any pipeline stage that produces chunks must populate this schema.
- **Agents follow up on structured data rather than trusting LLM paraphrase**: retrieved contexts carrying `table_ref` should trigger a BigQuery lookup for exact numeric values instead of relying on the chunk's natural-language table summary (TDD §3.9).

### Phased rollout (PRD §10)

Implementation is expected to land in this order — check which phase is current before assuming a component (e.g. the graph layer, multi-source ingestion, eval-in-CI) is already in scope:

0. POC — manual corpus creation, single source, validate parsing/retrieval quality
1. Pilot — full ingestion pipeline (Cloud Run/Workflows/Firestore), one corpus, limited users
2. GA — multi-source ingestion, full metadata/tagging layer, BigQuery catalog, eval framework in CI
3. Knowledge Graph — entity/relation extraction promoted into Spanner Graph, graph-aware agent retrieval
4. Extensions — cross-corpus federation, further graph enrichment

### Evaluation strategy (TDD §4)

A golden query dataset (version-controlled, with expected answers/sources/chunk IDs, including adversarial "no good answer" cases) is scored offline pre-merge/pre-deploy on retrieval (context precision/recall) and generation (groundedness, citation accuracy, answer relevance) metrics, with CI-blocking thresholds and regression comparison against baseline — not just absolute pass/fail. When adding ingestion pipeline code, tagging prompts, chunking config, or corpus config changes, expect/wire this eval to run and gate promotion.
