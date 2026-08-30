# EnterpriseRAG

Enterprise Multimodal RAG Platform on Google Cloud, built on **Vertex AI RAG Engine**. It ingests PDFs (text, tables, images), Confluence, SharePoint, Word, and JSON content, and serves grounded, cited answers through an ADK agent — with metadata-based filtering and a knowledge graph layer for multi-hop, relationship-aware retrieval.

## Status

This repository currently contains design documentation only (Phase 0 — POC not yet started). No application code, build system, or tests exist yet.

## Documentation

- [`rag-engine-prd.md`](rag-engine-prd.md) — Product Requirements Document: problem statement, goals, functional/non-functional requirements, phased rollout
- [`rag-engine-tdd.md`](rag-engine-tdd.md) — Technical Design Document: architecture, component design, evaluation strategy, risks

## Architecture

![High-level architecture](architecture-diagram-hl.svg)

Four zones: **Sources** feed the **Ingestion Pipeline**, which writes chunks, metadata, and staged entity relations into **Storage & Indexing**; the **Query & Generation** layer retrieves from there (vector + graph) and returns cited answers to the user.

| Zone | Detail diagram |
|---|---|
| Ingestion Pipeline | [`architecture-diagram-ll-ingestion.svg`](architecture-diagram-ll-ingestion.svg) |
| Storage & Indexing | [`architecture-diagram-ll-storage.svg`](architecture-diagram-ll-storage.svg) |
| Query & Generation | [`architecture-diagram-ll-query.svg`](architecture-diagram-ll-query.svg) |

See `rag-engine-tdd.md` §1 for the full text flow and component-by-component design.

## Phased rollout

1. **Phase 0 — POC:** Manual corpus creation, single source, validate layout parsing + retrieval quality
2. **Phase 1 — Pilot:** Full ingestion pipeline (Cloud Run/Workflows/Firestore), one corpus, limited user group
3. **Phase 2 — GA:** Multi-source ingestion, full metadata/tagging layer, BigQuery catalog, eval framework in CI
4. **Phase 3 — Knowledge Graph:** Entity/relation extraction promoted into a queryable graph store, graph-aware retrieval
5. **Phase 4 — Extensions:** Cross-corpus federation, further graph enrichment
