# Multimodal RAG for Automotive Engineering
### A Complete Technical Reference — Architecture, Concepts & Roadmap

---

## Table of Contents

1. [What is RAG?](#1-what-is-rag)
2. [Hybrid RAG](#2-hybrid-rag)
3. [Full Multimodal RAG](#3-full-multimodal-rag)
4. [Hybrid Multimodal RAG](#4-hybrid-multimodal-rag)
5. [Recommended Architecture for This Project](#5-recommended-architecture-for-this-project)
6. [Project Roadmap](#6-project-roadmap)

---

## 1. What is RAG?

**Retrieval-Augmented Generation (RAG)** is an AI architecture pattern that gives a Large Language Model (LLM) access to an external knowledge base at inference time — instead of relying solely on what was baked into its weights during training.

### Why RAG exists

An LLM trained on general data cannot know:
- Your proprietary assembly manuals
- Updated torque specifications from last quarter
- Internal part IDs specific to your factory line

RAG solves this by **retrieving relevant context on-the-fly** and injecting it into the prompt before the model generates its answer.

### Basic RAG Flow

```mermaid
flowchart LR
    A([User Query]) --> B[Embed Query\nvector]
    B --> C[(Vector\nDatabase)]
    C -->|Top-K similar chunks| D[Context Builder]
    D --> E[LLM Prompt\n= Query + Context]
    E --> F([Answer])

    subgraph Ingestion Phase
        G[Documents] --> H[Chunker]
        H --> I[Embedder]
        I --> C
    end
```

### Core Components

| Component | Role | Example Tools |
|-----------|------|---------------|
| **Chunker** | Splits documents into manageable pieces | LangChain, Docling |
| **Embedder** | Converts text chunks to dense vectors | `text-embedding-3-large`, BGE |
| **Vector DB** | Stores and retrieves vectors by similarity | Qdrant, Weaviate, Pinecone |
| **LLM** | Generates the final answer from context | Claude, GPT-4o |

### Limitations of Naive RAG

- Splitting documents naively **shreds table context** (a torque value without its column header is meaningless)
- Pure semantic search **misses exact tokens** like part IDs (`M8-bolt-2025`)
- No image understanding — PDFs with blueprints are treated as blind spots
- No guaranteed citation → hallucination risk

---

## 2. Hybrid RAG

**Hybrid RAG** addresses the retrieval gap by combining two fundamentally different search strategies.

### The Problem with Pure Semantic Search

Semantic search finds *conceptually similar* content. But in engineering:

> `"47 Nm torque"` and `"74 Nm torque"` are semantically similar — they are safety-critically different.

> `"M8-bolt-2025"` must be found by exact token match, not by cosine similarity to "fastener".

### Hybrid = Dense + Sparse

```mermaid
flowchart TB
    Q([User Query]) --> A[Dense Encoder\nBGE / text-embedding]
    Q --> B[Sparse Encoder\nBM25 / SPLADE]

    A -->|Semantic similarity| C[(Vector Index)]
    B -->|Keyword match| D[(Inverted Index)]

    C --> E[Result Set A]
    D --> F[Result Set B]

    E --> G[Reciprocal Rank Fusion\nRRF]
    F --> G

    G --> H[Unified Ranked Results]
    H --> I[LLM → Answer]
```

### Reciprocal Rank Fusion (RRF)

RRF merges two ranked lists without needing to normalize scores:

```
RRF_score(doc) = Σ  1 / (k + rank_in_list_i)
```

- `k = 60` is a common default
- A document ranked #1 in both lists scores higher than one ranked #1 in only one list
- Safe to combine lists from systems with incompatible score scales

### When to use each retriever

| Query Type | Best Retriever | Example |
|------------|---------------|---------|
| Conceptual question | Dense (semantic) | "How does the rear subframe absorb crash energy?" |
| Exact part lookup | Sparse (BM25) | "Find specs for M8-bolt-2025" |
| Numeric spec | Structured DB (SQL) | "What is the torque for bolt A17 at node 4?" |
| Mixed intent | Hybrid (both) | "What bolts does the firewall assembly use and what are their torque values?" |

---

## 3. Full Multimodal RAG

**Full Multimodal RAG** extends the pipeline to handle non-text modalities natively — images, diagrams, blueprints — as first-class retrieval targets.

### Why text-only RAG fails for automotive

- Assembly blueprints are 2D spatial diagrams. Their meaning cannot be captured as text.
- A mechanic uploading a photo of a cracked frame needs to match that image against the structural manual — not a text description of the frame.
- Tables in PDFs, when OCR'd naively, lose spatial relationships between cells and headers.

### Architecture: Caption-Augmented vs. Native Multimodal

#### Approach A — Caption-Augmented (simpler, weaker)

```mermaid
flowchart LR
    IMG[Blueprint Image] --> VLM[VLM Captioner\ne.g. LLaVA]
    VLM -->|Text caption| EMB[Text Embedder]
    EMB --> VDB[(Vector DB)]

    Q([Query: photo of part]) --> VLM2[VLM: describe photo]
    VLM2 --> EMB2[Text Embedder]
    EMB2 -->|similarity search| VDB
    VDB --> LLM[LLM Answer]
```

**Weakness:** The caption is a lossy compression of the image. If the caption misses a dimension label, that data is gone forever.

#### Approach B — Native Multimodal Embeddings (stronger)

```mermaid
flowchart LR
    IMG[Blueprint Image] --> MENC[Multimodal Encoder\nCLIP / ColPali / BLIP-2]
    TXT[Text Chunk] --> MENC
    MENC -->|Shared embedding space| VDB[(Vector DB)]

    Q([Photo Query]) --> MENC2[Same Encoder]
    MENC2 -->|Cross-modal similarity| VDB
    VDB --> CTX[Retrieved: text + images]
    CTX --> VLM[VLM Generator\nGPT-4o / Claude]
    VLM --> ANS([Answer with image grounding])
```

**Strength:** A photo query directly retrieves diagrams in the same embedding space. No caption intermediary.

### Key Models for Native Multimodal Embedding

| Model | Strength | Best For |
|-------|----------|---------|
| **ColPali** | Late interaction on page images | PDF-native retrieval without OCR |
| **CLIP / SigLIP** | Image-text alignment | Cross-modal search |
| **BLIP-2** | Image captioning + VQA | Visual QA on diagrams |
| **GPT-4o / Claude** | Generation from mixed context | Final answer synthesis |

---

## 4. Hybrid Multimodal RAG

**Hybrid Multimodal RAG** is the synthesis of all the above: it combines dense + sparse retrieval with multimodal embeddings, structured data stores for exact numerics, and a routing layer that directs each query to the correct retrieval path.

This is the architecture that makes sense for high-stakes engineering domains.

### Why the combination is necessary

No single retrieval strategy handles all automotive query types:

```mermaid
quadrantChart
    title Query Type vs Retrieval Strategy
    x-axis Exact Token Match --> Semantic Understanding
    y-axis Text Only --> Visual / Multimodal
    quadrant-1 Native VLM Embedding
    quadrant-2 Text Semantic Search
    quadrant-3 BM25 / SQL Exact Match
    quadrant-4 Multimodal + BM25 Hybrid
    Part ID lookup: [0.05, 0.15]
    Torque spec table: [0.1, 0.2]
    Concept question: [0.85, 0.2]
    Blueprint cross-reference: [0.5, 0.85]
    Photo-to-manual match: [0.2, 0.9]
    Mixed spec + visual: [0.6, 0.7]
```

### Full Hybrid Multimodal RAG Architecture

```mermaid
flowchart TB
    subgraph INPUT
        UQ([User Query\nText or Image])
    end

    UQ --> QR[Query Router\nClassifier]

    QR -->|Numeric / part ID| SDB[(Structured DB\nPostgreSQL / DuckDB)]
    QR -->|Semantic concept| VSS[Dense Vector Search\nQdrant]
    QR -->|Exact keyword| BM25[BM25 Sparse Index\nElasticsearch]
    QR -->|Visual / blueprint| VLM_R[VLM Embedding Search\nColPali / CLIP]

    SDB --> FUSE[Result Fusion\nRRF + Source Map]
    VSS --> FUSE
    BM25 --> FUSE
    VLM_R --> FUSE

    FUSE --> CTX[Context Builder\nwith Provenance Tags]
    CTX --> GEN[Generator LLM\nClaude / GPT-4o]
    GEN --> CIT[Citation Enforcer\nGuardrail Layer]
    CIT --> ANS([Answer\n+ Source + Page + Revision])

    subgraph INGESTION
        direction TB
        PDF[PDFs] --> DOCLING[Docling Parser\nlayout-aware]
        XML[AUTOSAR / DITA / KBL] --> XPARSER[lxml Parser\ntyped extraction]
        IMG[Blueprints / Photos] --> VLM_E[VLM Encoder\nCOLPali]

        DOCLING --> TABLES[Tables → SQL rows]
        DOCLING --> CHUNKS[Text Chunks → Vector DB]
        DOCLING --> IMAGES[Images → VLM Index]
        XPARSER --> TABLES
        VLM_E --> IMAGES
    end
```

---

## 5. Recommended Architecture for This Project

### System Design Principles

Given the automotive safety context (ISO 26262, IATF 16949), three principles are non-negotiable:

1. **Numerics are never retrieved by similarity** — exact match from typed storage only
2. **Every answer carries a verifiable source trail** — document, page, section, revision hash
3. **The LLM is a formatter, not an oracle** — it assembles retrieved facts, never infers specs

### Component Stack

```mermaid
flowchart LR
    subgraph Ingestion Stack
        A[Docling] -->|tables, images, text| B[PostgreSQL\nTyped Specs]
        A --> C[Qdrant\nVector DB]
        A --> D[Elasticsearch\nBM25 Index]
        E[lxml\nXML Parser] --> B
        F[ColPali] --> C
    end

    subgraph Retrieval Stack
        G[Query Router\nfine-tuned classifier] --> B
        G --> C
        G --> D
        B --> H[RRF Fusion]
        C --> H
        D --> H
    end

    subgraph Generation Stack
        H --> I[Context Builder\n+ provenance]
        I --> J[Claude / GPT-4o\nwith system prompt guardrails]
        J --> K[Citation Validator\npost-processing]
        K --> L([Response\n+ source map])
    end
```

### Technology Choices Explained

| Layer | Tool | Why |
|-------|------|-----|
| PDF parsing | **Docling** | Preserves table structure as DataFrames, handles multi-column layouts, outputs image bounding boxes |
| XML parsing | **lxml** | Full XPath support, schema validation, parent-child traversal for AUTOSAR/DITA |
| Vector DB | **Qdrant** | Native sparse+dense hybrid search in one system, payload filtering for metadata |
| Sparse index | **Elasticsearch BM25** | Industry standard for exact token retrieval, integrates with Qdrant via federation |
| Structured DB | **PostgreSQL** | Typed columns for torque specs, part IDs, tolerances; ACID compliance for safety traceability |
| Multimodal embed | **ColPali** | Embeds full PDF pages as images — no OCR pipeline needed, preserves spatial layout |
| LLM Generator | **Claude 3.5 Sonnet** | Long context window, strong instruction following for citation constraints |
| Orchestration | **LangGraph** | Stateful agent graph, supports conditional routing between retrieval paths |

### Query Router Logic

```mermaid
flowchart TD
    Q([Incoming Query]) --> C1{Contains image?}
    C1 -->|Yes| VLM_PATH[VLM Embedding Path]
    C1 -->|No| C2{Contains part ID\nor exact number?}
    C2 -->|Yes| C3{Is it a lookup\nor a spec?}
    C3 -->|Spec / numeric| SQL_PATH[Structured DB Path]
    C3 -->|Part reference| HYBRID_PATH[BM25 + Vector Hybrid]
    C2 -->|No| C4{Conceptual\nor procedural?}
    C4 -->|Conceptual| VEC_PATH[Dense Vector Only]
    C4 -->|Procedural| HYBRID_PATH
```

### Citation & Guardrail Pattern

Every chunk in the vector DB and every row in the structured DB must carry a provenance payload:

```json
{
  "doc_id": "chassis-assembly-v4.2",
  "revision_hash": "sha256:a3f9c...",
  "page": 47,
  "section": "3.4.2",
  "table_id": "T-14",
  "row": 3,
  "col": "Torque (Nm)",
  "extraction_method": "docling-table",
  "extracted_at": "2025-04-01T09:00:00Z"
}
```

The system prompt enforces citation at generation time:

```
You are a structural engineering assistant. You MUST:
1. Answer ONLY using the retrieved context provided below.
2. End every numeric claim with [Source: {doc_id}, Page {page}, Section {section}].
3. If the retrieved context does not contain the answer, respond:
   "This specification was not found in the retrieved documents. 
    Please consult [doc_id] directly or contact the responsible engineer."
4. NEVER estimate, interpolate, or infer numeric values.
```

---

## 6. Project Roadmap

### Overview Timeline

```mermaid
gantt
    title Automotive Multimodal RAG — Build Roadmap
    dateFormat  YYYY-MM-DD
    section Phase 1 · Foundation
    Environment setup & tooling          :p1a, 2025-05-01, 7d
    PDF ingestion pipeline (Docling)     :p1b, after p1a, 10d
    Structured DB schema + XML parser    :p1c, after p1a, 10d
    Basic text-only RAG (smoke test)     :p1d, after p1b, 7d

    section Phase 2 · Hybrid Retrieval
    Qdrant setup + dense embeddings      :p2a, after p1d, 7d
    BM25 / Elasticsearch index           :p2b, after p1d, 7d
    RRF fusion layer                     :p2c, after p2a, 5d
    Query router (rule-based v1)         :p2d, after p2c, 5d
    Evaluation harness (RAGAS)           :p2e, after p2d, 5d

    section Phase 3 · Multimodal
    ColPali integration for blueprints   :p3a, after p2e, 10d
    Cross-modal retrieval (photo→manual) :p3b, after p3a, 7d
    Multi-page blueprint stitcher        :p3c, after p3a, 5d
    VLM generator integration            :p3d, after p3b, 5d

    section Phase 4 · Safety & Compliance
    Citation enforcer + guardrails       :p4a, after p3d, 7d
    Provenance payload standardization   :p4b, after p3d, 5d
    Hallucination red-team testing       :p4c, after p4a, 7d
    Audit log + revision hash tracking   :p4d, after p4b, 5d

    section Phase 5 · Production
    Query router v2 (ML classifier)      :p5a, after p4c, 7d
    API layer + auth                     :p5b, after p4d, 7d
    Engineer-facing UI                   :p5c, after p5b, 10d
    Load testing + SLA validation        :p5d, after p5c, 5d
    Pilot deployment (1 factory line)    :p5e, after p5d, 5d
```

### Phase Breakdown

#### Phase 1 — Foundation (Weeks 1–3)

**Goal:** Reliable ingestion of PDFs and XML without data loss.

```mermaid
flowchart LR
    A[Raw PDFs\nXML files] --> B[Docling\nlayout parser]
    B --> C[Tables → PostgreSQL]
    B --> D[Text → Chunks]
    B --> E[Images → saved with bbox]
    F[AUTOSAR XML] --> G[lxml parser]
    G --> C
    D --> H[Basic RAG\nsmoke test]
```

Key deliverables:
- `AutomotivePart` data class with typed spec fields
- Table-to-SQL pipeline preserving column headers
- Chunk metadata schema with page + section provenance
- First end-to-end query returning a torque value with source citation

---

#### Phase 2 — Hybrid Retrieval (Weeks 4–6)

**Goal:** Part ID lookup precision + semantic coverage working together.

```mermaid
flowchart LR
    A[Text Chunks] --> B[Dense Embedder\nBGE-M3]
    A --> C[BM25 Indexer]
    B --> D[(Qdrant)]
    C --> E[(Elasticsearch)]
    F([Query]) --> G[RRF Fusion]
    D --> G
    E --> G
    G --> H[Top-K Results\nwith provenance]
```

Key deliverables:
- Qdrant collection with payload metadata
- BM25 index with field boosting on part IDs
- RRF merger returning unified ranked list
- RAGAS evaluation baseline (context precision, recall, answer faithfulness)

---

#### Phase 3 — Multimodal (Weeks 7–9)

**Goal:** Blueprint retrieval from image queries; visual context in answers.

```mermaid
flowchart TB
    A[Blueprint\nPDF Pages] --> B[ColPali\nPage Encoder]
    B --> C[(Qdrant\nMultimodal Index)]

    D([Engineer uploads\nphoto of part]) --> E[ColPali\nQuery Encoder]
    E -->|similarity search| C
    C --> F[Matching blueprint pages]
    F --> G[VLM Generator\nClaude with vision]
    G --> H([Answer with\nvisual grounding])
```

Key deliverables:
- ColPali-indexed blueprint library
- Multi-page stitcher for fold-out diagrams
- Cross-modal query path (image input → text + image output)
- VLM prompt template with citation enforcement for visual sources

---

#### Phase 4 — Safety & Compliance (Weeks 10–11)

**Goal:** Zero-hallucination guarantee on numeric specs; audit-ready output.

```mermaid
flowchart LR
    A[LLM Response] --> B[Citation Parser\nregex + JSON]
    B --> C{All numerics\ncited?}
    C -->|No| D[Block response\nReturn clarification]
    C -->|Yes| E[Provenance Verifier\ncross-check DB]
    E -->|Mismatch| D
    E -->|Match| F[Audit Log\nappend entry]
    F --> G([Validated Response\n+ source map])
```

Key deliverables:
- Citation enforcer post-processing layer
- Revision hash tracking for every source document
- Red-team test suite: numeric swap attacks, hallucination probes
- Audit log schema compatible with ISO 26262 traceability requirements

---

#### Phase 5 — Production (Weeks 12–16)

**Goal:** Factory-floor deployment with engineer-facing UI and SLA compliance.

```mermaid
flowchart TB
    subgraph User Facing
        A[Web UI\nReact] --> B[REST API\nFastAPI]
        C[Mobile App\nfactory floor] --> B
    end

    subgraph Backend
        B --> D[Query Router v2\nfine-tuned classifier]
        D --> E[Retrieval Orchestrator\nLangGraph]
        E --> F[All retrieval paths]
        F --> G[Generator + Guardrails]
        G --> H[Response Cache\nRedis]
    end

    subgraph Ops
        I[Monitoring\nLangSmith] --> E
        J[Doc Ingestion\nCI trigger on update] --> F
        K[Audit DB\nPostgreSQL] --> G
    end
```

Key deliverables:
- FastAPI service with auth and rate limiting
- LangGraph-orchestrated agent with stateful routing
- LangSmith tracing for every query (compliance requirement)
- Pilot on one production line with 20 engineers

---

### Success Metrics by Phase

| Phase | Metric | Target |
|-------|--------|--------|
| 1 | Table extraction fidelity | > 99% rows correctly parsed |
| 2 | Part ID retrieval precision@5 | > 0.95 |
| 2 | Semantic recall@10 | > 0.85 |
| 3 | Blueprint retrieval MRR | > 0.80 |
| 4 | Numeric hallucination rate | 0% on test suite |
| 4 | Citation completeness | 100% of numeric claims cited |
| 5 | P95 query latency | < 3 seconds |
| 5 | Engineer satisfaction (pilot) | > 4.2 / 5.0 |

---

## Appendix: Key Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Table split across PDF pages | Missing torque rows | Docling continuation detection + table ID stitching |
| VLM embeds semantics not magnitudes | Wrong numeric retrieval | Separate structured DB; numerics never embedded |
| Outdated document version in index | Wrong spec served | Revision hash in provenance; ingestion CI on document update |
| Multi-column PDF layout shredded | Context loss | Docling `do_table_structure=True` + layout-aware chunking |
| Cross-modal query hits wrong part family | Wrong blueprint returned | Re-ranker with part family filter as Qdrant payload condition |
| LLM infers beyond retrieved context | Hallucination | System prompt hard constraints + post-generation citation check |

---

*Document version: 1.0 — April 2025*
*Architecture: Hybrid Multimodal RAG for Automotive Structural Engineering*
