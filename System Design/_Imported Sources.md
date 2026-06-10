---
title: Imported Sources
created: 2026-06-08
tags:
  - meta/import-log
aliases:
  - Imported Sources
  - Import Log
  - Fuentes Importadas
---

# Imported Sources

> [!note] Qué es esto
> El registro de todas las páginas/artículos importados a este vault. **Antes de
> importar una URL nueva, se valida contra esta lista**: si ya está, se avisa en
> vez de re-importar. Cada entrada deja la URL original, la fecha y qué notas
> generó (o mergeó).

| Fecha      | Fuente                                                                                                                                             | Notas creadas / mergeadas                                                                                                                                                                                                                                                            |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 2026-06-08 | [Enterprise RAG Assistant](https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design) — Avani Chaskar                              | `ai/rag/`: [[Enterprise RAG Assistant]], [[ACL Filtering en RAG]], [[Hybrid Search]], [[Reranking]], [[Hierarchical Chunking]], [[Change Data Capture]] + MOC [[_RAG|RAG]]                                                                                                                |
| 2026-06-08 | [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai                 | `ai/rag/`: merge [[Reranking]] + nuevas [[Bi-Encoder]], [[Cross-Encoder]], [[ColBERT]], [[Reciprocal Rank Fusion]], [[LLM as Reranker]], [[BM25]]                                                                                                                                    |
| 2026-06-08 | [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus | MOC [[_System Design|System Design]] + 49 notas en `patterns/` (los 50 patrones; CDC reusó la nota de `ai/rag/`)                                                                                                                                                                                    |
| 2026-06-08 | [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs       | `ai/rag/`: hub [[Chunking Strategies]] + [[Fixed-Size Chunking]], [[Sentence-Based Chunking]], [[Paragraph-Based Chunking]], [[Recursive Character Splitting]], [[Markdown-Aware Chunking]], [[Code-Aware Chunking]], [[Chunk Overlap]] (enlaza [[Hierarchical Chunking]] existente) |

| 2026-06-10 | [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok | `ai/agents/`: hub [[Agent Harness]] + [[Harness Responsibilities]], [[Sandboxing]], [[Permission Enforcement]], [[Harness Maturity Spectrum]], [[OpenAI Codex]], [[Claude Code]], [[AI Framework]], [[Orchestrator]], [[Multi-User Agent Design]] + MOC [[_AI Agents|AI Agents]] |

| 2026-06-10 | [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai **(re-import con curl crudo)** | `ai/rag/reranking/`: +nuevas [[Classical Reranking]], [[Information Theory of Reranking]], [[Reranking Metrics]], [[Node vs Document Reranking]], [[Derived vs Hybrid Reranking]], [[Multimodal Reranking]], [[Agent Reranking]]; enriquecida [[LLM as Reranker]] + MOC. (El import original 2026-06-08 vía WebFetch había perdido ~64% del contenido.) |

| 2026-06-10 | [Grounded Eval Harness: Building an AI That Fact-Checks Itself](https://substack.com/home/post/p-197984224) — Avani (alwaysavani) | `ai/agents/`: [[Grounded Eval Harness]] (caso), [[Generator-Evaluator Pattern]], [[LangGraph]] + MOC [[_AI Agents|AI Agents]] actualizado. Semillas: [[Grounding]], [[Hallucinations]], [[Evals]], [[RLHF]]. |

| 2026-06-10 | [Harness Engineering: Evolution of MLOps to AutoMLOps](https://substack.com/home/post/p-198249792) — Jam with AI | `ai/mlops/` (dominio nuevo): hub [[AutoMLOps]] + [[AutoResearch]], [[MLOps Maturity Stages]], [[Offline vs Business Metrics]], [[Three-Tier Evaluation Pipeline]] + MOC [[_MLOps|MLOps]]. Enlaza [[Sandboxing]]/[[Agent Harness]]/[[Generator-Evaluator Pattern]] (ai/agents) y [[Reranking Metrics]] (ai/rag). |

| 2026-06-10 | [System Design Deep Dive: Architecting Idempotent APIs](https://designgurus.substack.com/p/system-design-deep-dive-architecting) — Arslan Ahmad (Design Gurus) | `patterns/idempotency/` (subtema nuevo): enriquecida [[Idempotency]] + nuevas [[Idempotency Key]], [[Idempotency Architecture]] + MOC [[_Idempotency|Idempotency]]; [[Distributed Lock]] en `patterns/`. Contenido pegado (Jina truncaba por paywall). |