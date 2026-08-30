# Enterprise Multimodal RAG Platform — PRD
## (Vertex AI RAG Engine Implementation)

## 1. Overview

Build an enterprise RAG platform on Google Cloud that ingests PDFs (text, tables, images), Confluence, SharePoint, Word, and JSON content, and serves grounded, cited answers through an ADK agent. This revision uses **Vertex AI RAG Engine** as the managed retrieval backbone (in place of Vertex AI Search / Agent Search used in the prior design), with a custom augmentation layer for tagging, table handling, and image captioning to achieve tight, filterable metadata.

## 2. Problem Statement

Users need fast, accurate answers grounded in a large, mixed-format internal corpus (thousands of PDFs plus Confluence documentation), including data locked in tables and figures — without hallucination and with traceable citations back to source page/section.

## 3. Goals

- Ingest and index PDFs with text, tables, and images with layout-aware chunking
- Provide grounded, cited answers via an ADK agent at ~1,000–2,000 queries/hour for ~1,000 users
- Support metadata-based filtering (doc type, source, date, category) at query time
- Build a knowledge graph layer capturing entity relationships across documents, enabling multi-hop and relationship-aware retrieval beyond flat chunk similarity
- Maintain a governance/audit catalog of all ingested content, independent of the serving index
- Fully automatable via Terraform IaC, with a manual path for quick POC

## 4. Non-Goals

- Replacing existing systems of record — this platform is additive, read-only against source systems
- Real-time (sub-second) ingestion — batch/near-real-time ingestion is acceptable

## 5. Users & Scale

- ~1,000 internal users
- 1,000–2,000 queries/hour peak
- Initial corpus: thousands of PDFs + Confluence space(s), growing over time

## 6. Functional Requirements

| ID | Requirement |
|---|---|
| FR1 | Ingest PDFs (incl. scanned/OCR), Word, JSON, Confluence, SharePoint sources |
| FR2 | Parse tables as structured data, not flattened text |
| FR3 | Generate searchable descriptions for embedded images/charts |
| FR4 | Tag each chunk with entities, doc type, category, source, date |
| FR5 | Support metadata filtering and boosting at query time |
| FR6 | Return answers with citations (source doc, page) |
| FR7 | Track ingestion status per document (pending/parsing/tagging/indexed/failed) with retry |
| FR8 | Maintain a separate governance catalog (audit trail, raw table data) |
| FR9 | Support incremental re-ingestion when source documents change |
| FR10 | Extract entity relationships from ingested content and expose them for multi-hop, relationship-aware queries |

## 7. Non-Functional Requirements

- **Latency:** P95 query-to-answer under ~4 seconds (retrieval + generation)
- **Availability:** 99.5% for the query path
- **Security:** IAP + Okta-authenticated access; least-privilege IAM per service
- **Scalability:** Corpus growth to tens of thousands of documents without re-architecture
- **Cost transparency:** Ingestion and query costs tracked per environment (dev/staging/prod)

## 8. Success Metrics

- Retrieval precision/groundedness score (via eval framework) above target threshold
- % of answers with valid, correct citations
- P95 latency within SLA
- Ingestion pipeline failure rate < 2%, with automatic retry recovering > 90% of transient failures

## 9. Assumptions & Constraints

- Already operating on Google Cloud — no cloud migration required
- Vertex AI RAG Engine file limits apply: 20 MB max per file, 500 pages max per PDF (larger PDFs are split upstream)
- RAG Engine allows **at most one corpus per generation request** — corpus segmentation strategy must account for this (see TDD §3.7)
- Embedding model is locked to a corpus at creation time — changing models requires corpus recreation and full re-import

## 10. Phased Rollout

1. **Phase 0 — POC:** Manual corpus creation, single source (subset of PDFs), validate layout parsing + retrieval quality
2. **Phase 1 — Pilot:** Full ingestion pipeline (Cloud Run/Workflows/Firestore), one corpus, limited user group
3. **Phase 2 — GA:** Multi-source ingestion (Confluence, SharePoint), full metadata/tagging layer, BigQuery catalog, eval framework in CI
4. **Phase 3 — Knowledge Graph:** Entity/relation extraction promoted from BigQuery staging into a queryable graph store, agent-side graph-aware retrieval for multi-hop questions
5. **Phase 4 — Extensions:** Cross-corpus federation, further graph enrichment (e.g., user-behavior or collaborative edges)

---
