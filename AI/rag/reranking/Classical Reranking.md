---
title: Classical Reranking
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-10
tags:
  - ai/rag/reranking
aliases:
  - Classical Reranking
  - classical-reranking
  - Reranking Estadístico
  - Statistical Reranking
---

# Classical Reranking

> [!note] Definición
> Mucho antes de usar transformers para reordenar pasajes, el reranking existía en **estadística** como el arte de la **corrección posterior (posterior correction)**: refinar estimaciones de probabilidad para corregir el orden de los resultados tras observar nueva información.

## La idea base

- Se parte de una **función de scoring inicial** `s(x)` y un **modelo secundario** que ajusta o transforma esos scores según features o restricciones adicionales.
- Forma general: una transformación `f` sobre `s(x)` usando un término de contexto `z` (contexto, priors o términos de calibración). `f` es lineal en regresión simple, no lineal en sistemas modernos.

## Patrones fundacionales

- **Logistic reranking** — transforma scores crudos `s(x)` en probabilidades vía **sigmoide**. Sigue muy usado en scoring de documentos y comparación pairwise.
- **Platt scaling** / **isotonic regression** — recalibran las salidas de un clasificador; conceptualmente, una forma de reranking por transformación aprendida.
- **Pairwise reranking** — optimiza el **orden relativo** en vez de scores absolutos. Familias **RankNet** y **LambdaRank**.

## Conexión con el reranking moderno

En el contexto moderno (RAG / search), el reranking se define como una transformación `T` sobre el score inicial `s(d_i, q)` del retriever, que produce un score reranqueado `r_i` para reordenar. El diseño de `T` y la **métrica de similitud** dentro de `s(·)` definen "el alma" del reranker:

- Métrica **coseno / dot-product** → cercanía geométrica; ideal para embeddings densos, limitada en razonamiento composicional.
- **Cross-encoder attention / interacciones a nivel token** → significado a nivel relacional más profundo; más lento pero mucho más discriminativo.
- `T` puede ser un modelo aprendido (transformer, MLP) o una heurística (down-weight de pasajes genéricos, boost de recencia).

> [!tip] Trade-off central
> El reranking es un equilibrio entre **fidelidad y eficiencia** (entender vs throughput). El arte está en elegir la transformación correcta para el ruido de recuperación correcto.

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Information Theory of Reranking]]
- [[Reranking Metrics]]
