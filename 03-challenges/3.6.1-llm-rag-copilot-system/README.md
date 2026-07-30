# Design an Enterprise LLM & RAG Copilot System (ChatGPT for Enterprise)

## Executive Summary

Designing an **Enterprise LLM & RAG Copilot System** (such as Microsoft 365 Copilot, Enterprise ChatGPT, or GitHub Copilot for Internal Knowledge) requires serving billions of queries over millions of sensitive company documents while meeting strict enterprise latency SLAs ($< 400\text{ms}$ Time-To-First-Token) and rigid Role-Based Access Control (RBAC) security boundaries.

This document provides a comprehensive, production-grade High-Level Design (HLD) covering embedding pipelines, hybrid vector-sparse search, prompt prefix caching, continuous batching GPU inference, and streaming token delivery.

---

## Navigation & Artifact Index

- **[quick-overview.md](./quick-overview.md)** — Architectural summary, core metrics, and revision cheat sheet.
- **[hld-diagram.md](./hld-diagram.md)** — System architecture diagrams (12+ Mermaid diagrams).
- **[sequence-diagrams.md](./sequence-diagrams.md)** — Interaction flows & sequence diagrams.
- **[this-over-that.md](./this-over-that.md)** — Deep-dive design trade-offs & technology choices.
- **[pseudocode.md](./pseudocode.md)** — Production algorithms for HNSW ANN search, vLLM continuous batching, and token streaming.

---

## 1. Requirements & Scope

### Functional Requirements
1. **Multi-Source Knowledge Ingestion:** Real-time and batch ingestion of PDF, Docx, Slack, Confluence, and GitHub code repositories.
2. **Hybrid Semantic & Keyword Search:** Dense vector embeddings (HNSW index) combined with sparse BM25 keyword search re-ranked via Cross-Encoder models.
3. **Low-Latency Streaming Output:** Stream generated tokens using Server-Sent Events (SSE) or WebSockets.
4. **Enterprise RBAC & Document Security:** Strict document-level ACL filtering during vector retrieval.
5. **Prompt & Context Caching:** Cache common system prompts and frequent document context blocks to cut GPU costs.

### Non-Functional Requirements (SLAs & Constraints)
- **Time-To-First-Token (TTFT):** $< 400\text{ms}$ (p99).
- **Inter-Token Latency (ITL):** $< 30\text{ms}$ per token.
- **Scale:** 10,000 active concurrent enterprise users; 100 Million indexing documents.
- **Availability:** $99.99\%$ Uptime.
- **Data Security:** Zero external data leakage (all embeddings and LLM inference run inside air-gapped or private cloud VPCs).

---

## 2. High-Level Architecture

```mermaid
flowchart TD
    subgraph Client Tier
        User[Enterprise Mobile / Web Client]
    end

    subgraph API Gateway & Edge Security
        User -- WSS / SSE --> Gateway[API Gateway / Rate Limiter]
        Gateway --> Auth[OAuth2 / OIDC Auth Svc]
        Gateway --> Guard[Prompt Safety Guardrails]
    end

    subgraph Semantic Cache Layer
        Guard --> Cache{Semantic Cache: Redis}
        Cache -- Hit (<20ms) --> Gateway
    end

    subgraph Hybrid Retrieval Engine (RAG)
        Guard -- Miss --> EmbSvc[Embedding Service]
        EmbSvc --> Qdrant[(Vector DB: Qdrant Cluster)]
        Guard --> BM25[(Sparse Engine: Elasticsearch)]
        
        Qdrant -- Dense Matches --> Merge[Reciprocal Rank Fusion]
        BM25 -- Sparse Matches --> Merge
        Merge --> Reranker[Cross-Encoder Re-ranker]
    end

    subgraph Prompt Assembly & Context Construction
        Reranker --> PromptBuilder[Context Truncator & Security Filter]
    end

    subgraph GPU LLM Inference Cluster
        PromptBuilder --> vLLMRouter[vLLM Load Balancer]
        vLLMRouter --> GPU1[vLLM Worker 1: H100]
        vLLMRouter --> GPU2[vLLM Worker 2: H100]
    end

    GPU1 -- Token Streaming Stream --> Gateway
```

---

## 3. Deep-Dive Subsystem Architecture

### 1. Document Ingestion & Vector Indexing Pipeline
- **Parsing:** Documents are parsed into clean text using Apache Tika / Unstructured.io.
- **Chunking:** Semantic Chunking with sliding windows (512 tokens per chunk, 64 token overlap).
- **Embedding:** Embedding model (`bge-large-en-v1.5`, 1024 dimensions) generates dense vectors.
- **ACL Tagging:** Attach JSON metadata payload: `{"doc_id": "doc_881", "allowed_groups": ["ENG_ALL", "LEVEL_4_EXEC"]}`.

### 2. Hybrid Search & Re-ranking Algorithm
1. Query vectors are executed against Qdrant HNSW index with a payload filter constraint.
2. Sparse BM25 queries run against Elasticsearch to capture exact product codes or acronyms.
3. **Reciprocal Rank Fusion (RRF)** score formula:
   $$RRF(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
4. Top 100 merged items are passed through a Cross-Encoder Re-ranker (`bge-reranker-large`), selecting the top 5 highest-confidence chunks ($>0.82$).

---

## 4. Bottleneck Resolution & Performance Strategies

1. **GPU Memory Bottleneck (PagedAttention):** Allocates KV Cache dynamically in non-contiguous VRAM pages, supporting up to 4x batch size per H100 GPU node.
2. **First Token Delay (TTFT):** Implements **Prefix Caching** on vLLM to preserve system prompt states across user sessions.
3. **Data Security & Privacy:** Hardware-enforced Confidential Computing (NVIDIA H100 Confidential VMs) preventing memory snooping.
