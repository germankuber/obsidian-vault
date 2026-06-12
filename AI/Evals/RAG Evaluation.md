---
title: RAG Evaluation
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - RAG Evaluation
  - Evaluación de RAG
  - RAG Failure Diagnosis
  - Synthetic Query Generation
updated: 2026-06-11
---

# RAG Evaluation

> [!note] Definición
> El ángulo **eval** de un sistema [[RAG|RAG (Retrieval Augmented Generation)]]: el AI (1) recupera info de una DB y (2) la usa para generar la respuesta. La clave es que tiene **dos failure modes que se evalúan POR SEPARADO**: el **retrieval** falla (no encuentra la info) vs el **generation** falla (usa mal la info que sí encontró).

## RAG Failure Diagnosis

`diagnose_rag_failure` localiza dónde se rompió:
1. **Chequear retrieval** — si el target no está en el **top 5** → `failure_point = 'RETRIEVAL'`.
2. **Chequear calidad del documento** recuperado.
3. **Chequear generation** — si la respuesta es incorrecta **pese a un buen retrieval** → `failure_point = 'GENERATION'`.

> [!tip] Fix según el punto de falla
> - **Retrieval falla** → distintas estrategias de chunking, metadata filters, hybrid search (keyword+semantic), query expansion, **reranking models** (ver [[Reranking]]), tokenizers domain-specific.
> - **Generation falla** → mejorar el system prompt, few-shot examples, chain-of-thought, instrucciones de grounding explícitas, requisitos de citación.

## Synthetic Query Generation

- Para testear retrieval sin queries reales, un LLM genera, dada una receta, **UNA pregunta realista** que dependa de un detalle técnico preciso (métodos, settings de electrodomésticos, prep de ingredientes, timing, temperatura).
- Devuelve JSON `{'query': '...?', 'salient_fact': '<quote/paraphrase>'}`.
- El **`salient_fact` es el ground truth**: sabés qué documento contiene la respuesta. Ejemplo: *"¿A qué temperatura horneo las cookies?"* → salient: `'350 degrees F for 8-10 minutes'`.

## Métricas de retrieval

> [!info] No se redefinen acá
> Las métricas **Recall@K** ("¿el doc correcto apareció en el top K?", evaluadores RecallAt1/RecallAt3/RecallAt5) y **MRR** ("si lo encontró, ¿qué tan alto rankeó?" = 1/rank del primer doc relevante, sino 0.0) ya están documentadas en [[Reranking Metrics]] — referencialas ahí. El algoritmo de baseline [[BM25]] (keyword search, lib `rank_bm25`) tampoco se redefine.

## Domain-Specific Tokenizer

> [!warning] El tokenizer importa para retrieval keyword-based
> Los tokenizers estándar **quitan números**, pero en recetas `375` (temperatura), `9x13` (tamaño de molde) y `1/2` (medida) son **términos de búsqueda críticos**. El regex `_TOKEN_RE` preserva dimensiones (`9x13`), fracciones (`1/2`), números (`375`), unidades de temperatura y palabras; y normaliza: `'degrees f'`→`'degreesf'`, `'mins'`/`'minutes'`→`'min'`.

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Cap 6.

## Related

- [[Reranking Metrics]]
- [[BM25]]
- [[Reranking]]
- [[Pipeline and Multi-Turn Evaluation]]
- [[Eval Lifecycle]]
- [[Evals]]
