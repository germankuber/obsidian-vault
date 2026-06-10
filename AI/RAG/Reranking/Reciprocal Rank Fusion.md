---
title: Reciprocal Rank Fusion
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/retrieval
  - type/pattern
  - status/permanent
aliases:
  - Reciprocal Rank Fusion
  - reciprocal-rank-fusion
  - RRF
---

# Reciprocal Rank Fusion

> [!note] Definition
> Un método de consenso para fusionar los rankings de **varios retrievers** en uno solo. Cada candidato suma una contribución inversamente proporcional a la posición que le dio cada retriever; los que aparecen consistentemente arriba en varias listas suben. Es la técnica típica del [[Derived vs Hybrid Reranking|Hybrid Reranking]].

## Fórmula

```
RRF(d) = Σ(i=1 to M) [ 1 / (k + rank_i(d)) ]
```

- `d`: candidato (documento/nodo)
- `M`: cantidad de retrievers
- `rank_i(d)`: posición que el retriever `i` le asignó a `d`
- `k`: constante de amortiguación (suele ser 60)

## La intuición

> [!tip]
> "Un candidato iluminado por varios reflectores probablemente sea importante, aunque ningún reflector solo lo haya puesto en el centro." RRF premia el **acuerdo entre fuentes** sin necesitar que los scores de los distintos retrievers estén calibrados entre sí — solo usa el *rango*, no el score crudo.

## Por qué se usa

Distintos retrievers ven el mundo distinto: los sparse ([[BM25]]) son buenos con keywords exactas, los dense ([[Bi-Encoder]]) con similitud semántica, y los de dominio con terminología especializada. RRF combina sus señales para mejorar el **recall** sin tener que unificar sus escalas de score.

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Hybrid Search]]
