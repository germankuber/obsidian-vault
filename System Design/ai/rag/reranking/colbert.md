---
title: ColBERT
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/retrieval
aliases:
  - ColBERT
  - colbert
  - CoLBERT
  - Late Interaction
  - MaxSim
---

# ColBERT

> [!note] Definition
> Un modelo de *late interaction* que busca el punto medio entre [[Bi-Encoder]] y [[Cross-Encoder]]: codifica query y documento en embeddings **a nivel token** (de forma independiente, así que son precomputables) y los compara con la operación **MaxSim**. Gana razonamiento fino sin pagar la atención cruzada completa.

## La intuición

Cada token de la query busca su **mejor coincidencia** en el documento. La suma de esas mejores coincidencias es el score. Como los embeddings por token se calculan por separado, se conserva la escalabilidad del bi-encoder, pero las interacciones token-a-token aportan la finura que le falta.

## Formulación (MaxSim)

```
score(q, d) = Σ(i ∈ q) max(j ∈ d) cos(v_q_i, v_d_j)
```

Más formalmente, con `Q = [q_1..q_m]` y `D = [d_1..d_n]`:

```
score(Q, D) = Σ(i=1 to m) max_j cos(q_i, d_j)
```

> [!tip] Por qué es eficiente
> El "late interaction" evita la atención cruzada entre *cada* par de tokens query-documento (lo que hace caro al [[Cross-Encoder]]). Solo computa, por token de la query, el máximo coseno contra los tokens del documento — un ahorro enorme de cómputo manteniendo razonamiento granular.

## Familia relacionada

El mismo principio de late interaction se extiende a otras modalidades — ColPali y ColQwen para visión-lenguaje — relevante para [[Multimodal Reranking]].

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Bi-Encoder]]
- [[Cross-Encoder]]
