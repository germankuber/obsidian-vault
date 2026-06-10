---
title: Reranking Metrics
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-10
tags:
  - ai/rag/reranking
aliases:
  - Reranking Metrics
  - reranking-metrics
  - Métricas de Reranking
  - NDCG
  - MRR
---

# Reranking Metrics

> [!note] Definición
> Un reranker es tan bueno como su evaluación: la calidad se mide por **qué tan
> bien el nuevo orden refleja la relevancia real**. Se cuantifica con métricas
> basadas en ranking.

## Las métricas

- **NDCG (Normalized Discounted Cumulative Gain)** — premia colocar los ítems
  relevantes **más arriba**. Un NDCG alto significa que el reranker no solo
  encuentra buenos resultados, sino que **los pone en el orden correcto**.
- **MRR (Mean Reciprocal Rank)** — mide **qué tan pronto aparece el primer
  resultado relevante**. Especialmente útil en QA o sistemas de diálogo, donde
  **la respuesta de arriba es la que más importa**.
- **Precision@k** — de los top-k reordenados, cuántos son realmente relevantes.
- **Recall@k** — de todos los relevantes, cuántos quedaron en el top-k.

## Cómo se relacionan con las dos familias

> [!tip] Métrica según objetivo
> - [[Derived vs Hybrid Reranking|Derived reranking]] optimiza **precisión** →
>   mirá Precision@k y NDCG.
> - [[Derived vs Hybrid Reranking|Hybrid reranking]] optimiza **recall** →
>   mirá Recall@k (reduce falsos negativos).

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Derived vs Hybrid Reranking]]
- [[Information Theory of Reranking]]
