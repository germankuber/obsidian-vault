---
title: Derived vs Hybrid Reranking
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-10
tags:
  - ai/rag/reranking
aliases:
  - Derived vs Hybrid Reranking
  - derived-vs-hybrid-reranking
  - Derived Reranking
  - Hybrid Reranking
  - Precision vs Recall Reranking
---

# Derived vs Hybrid Reranking

> [!note] Definición
> Las **dos estrategias** de reranking de los pipelines modernos. **Derived**
> rerankea *después* de la recuperación, solo entre los Top-K ya recuperados.
> **Hybrid** rerankea *durante* la recuperación, combinando scores de múltiples
> fuentes/modalidades y aplicando una fusión encima.

```
Derived reranking:  Query -> Retriever -> top-K nodes -> Reranker -> Generator
Hybrid retrieval:   Query -> many retrievers -> Fusion/Reranker -> Generator
```

## Derived Reranking — precisión

- Toma los Top-K recuperados y **re-evalúa con una noción más fina de relevancia**.
  Es "derived" porque **no genera candidatos nuevos**: deriva un mejor orden de lo
  que ya hay.
- **Objetivo: maximizar precisión** — alta proporción de verdaderos positivos (TP)
  vs falsos positivos (FP) en el top. Minimiza el **ruido FP**.
- Opera sobre los Top-K nodes aplicando [[Bi-Encoder|bi-encoders]] o
  [[Cross-Encoder|cross-encoders]].
- *"No ensancha la red; pule lo que ya pescaste."*

> [!warning] Su límite
> Derived reranking **no puede recuperar lo que no está en vista**: solo reordena
> lo que el retriever ya trajo. Si un documento crítico quedó afuera del Top-K
> (terminología inusual, enterrado en un reporte largo), derived no lo arregla.

## Hybrid Reranking — recall

- Parte de una suposición más humilde: **quizás la primera red se perdió algo**.
- En vez de reordenar los mismos candidatos, **combina señales de múltiples
  retrievers** y considera un **pool de candidatos más grande** (N > K).
- **Objetivo: mejorar recall** — reducir **falsos negativos (FN)**, los relevantes
  que se escaparon de la malla.
- Reconoce que **distintos retrievers ven el mundo distinto**:
  - **Sparse** (como [[BM25]]) — matches de keywords exactos.
  - **Dense** (bi-encoders) — similitud semántica.
  - **Domain-specific** — terminología especializada.
- Solo después de expandir el conjunto candidato aplica reranking encima.
- Fusión típica: [[Reciprocal Rank Fusion]] (RRF).

> [!example] La metáfora del panel de expertos
> El lexicógrafo nota las palabras con precisión; el semantista nota el
> significado en general; el experto de dominio nota la relevancia en contexto. Su
> juicio combinado suele ser más preciso que cualquier perspectiva individual.

## El resumen

> [!tip] Red de pesca
> - **Derived** = precisión: pulir la primera pesca, zoom en los mejores pocos
>   peces (textura, color, detalle). Sharpening.
> - **Hybrid** = recall: tirar una red más ancha y multi-señal porque la primera
>   pudo perderse algo, y *recién después* rerankear.

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Bi-Encoder]]
- [[Cross-Encoder]]
- [[BM25]]
- [[Reciprocal Rank Fusion]]
- [[Reranking Metrics]]
