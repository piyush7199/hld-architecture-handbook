# Design Decisions & Trade-Offs: Enterprise LLM & RAG Copilot

## 1. Vector Database: Qdrant vs. Pinecone vs. pgvector

### Decision: Choose **Qdrant (Self-Hosted / Managed)**
- **Why over Pinecone:** Enterprise data privacy requires running vector infrastructure within private VPCs / air-gapped networks. Pinecone is closed-source SaaS only.
- **Why over pgvector:** While `pgvector` is convenient for small projects, Qdrant's HNSW implementation delivers $10\text{x}$ higher throughput at scale under concurrent payload filter loads (100M+ vectors).

---

## 2. Serving Engine: vLLM vs. Ollama vs. TGI

### Decision: Choose **vLLM**
- **Why over Ollama:** Ollama is designed for single-user local desktop deployment. vLLM supports multi-user production serving with PagedAttention and continuous batching.
- **Why over HuggingFace TGI:** vLLM provides superior prefix caching mechanics and higher throughput per dollar on NVIDIA H100 hardware.

---

## 3. Context Retrieval: Hybrid (Dense + Sparse) vs. Pure Dense

### Decision: Choose **Hybrid Search (Dense + Sparse BM25 + Re-ranker)**
- **Why over Pure Dense:** Dense vectors struggle with exact key matches like error codes (`ERR_4091_KAFKA_TIMEOUT`). BM25 ensures exact keyword precision, while cross-encoder re-ranking guarantees top relevancy.
