---
title: Bi-Encoder
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/retrieval
  - type/technology
  - status/permanent
aliases:
  - Bi-Encoder
  - bi-encoder
  - Siamese Encoder
---

# Bi-Encoder

> [!note] Definition
> Una red siamesa que codifica la query y el nodo **por separado**, cada uno en su propio embedding, y compara los dos vectores con similitud coseno. Como los embeddings de los documentos se pueden precalcular, es el método semántico más rápido y escalable.

## Formulación

```
v_q = enc_q(q),  v_d = enc_d(d)
score(q, d) = cos(v_q, v_d)
```

## Fortalezas y debilidades

> [!tip] Fortalezas
> Extremadamente rápido; los embeddings de los nodos son precomputables y se indexan una sola vez; escala a millones de nodos. Es el caballo de batalla de la recuperación inicial.

> [!warning] Debilidades
> Trata query y nodo de forma independiente, así que pierde señales relacionales sutiles (negación, orden de tokens). Puede rankear demasiado alto nodos semánticamente parecidos pero contextualmente irrelevantes — el mismo problema de "similitud ≠ relevancia" que el [[Reranking]] busca corregir.

## Dónde encaja

Es el extremo "rápido y grosero" del espectro de rerankers semánticos. Su contraparte precisa es el [[Cross-Encoder]]; [[ColBERT]] busca el punto medio entre ambos.

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Cross-Encoder]]
- [[ColBERT]]
