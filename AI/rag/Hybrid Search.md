---
title: Hybrid Search
source: https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design
author: Avani Chaskar
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/retrieval
aliases:
  - Hybrid Search
  - hybrid-search
  - Dense and Sparse Search
---

# Hybrid Search

> [!note] Definition
> Combinar **búsqueda densa** (vectorial, por similitud semántica) con **búsqueda
> dispersa** (por palabras clave exactas, típicamente BM25) en una sola consulta,
> para capturar tanto el significado como los términos literales.

## Por qué no alcanza con vectores solos

- **Densa (vectores):** encuentra texto conceptualmente parecido aunque no
  comparta palabras. Falla con identificadores exactos, códigos de error,
  nombres propios o siglas que "no significan" nada semánticamente.
- **Dispersa (BM25):** encuentra coincidencias léxicas exactas. Falla cuando el
  usuario usa sinónimos o parafrasea.

La búsqueda híbrida corre ambas y fusiona los resultados, cubriendo las
debilidades mutuas. El conjunto candidato resultante suele pasar luego por un
paso de [[Reranking]] para quedarse con lo mejor.

> [!tip]
> En un RAG empresarial, la query reescrita y la identidad/ACL del usuario entran
> juntas a este paso, de modo que la búsqueda híbrida ya respeta
> [[ACL Filtering en RAG]] desde el origen.

## References

- Fuente: [System Design Deep Dive: How to Design an Enterprise RAG Assistant](https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design) — Avani Chaskar, 2026-05-17

## Related

- [[Enterprise RAG Assistant]]
- [[Reranking]]
- [[ACL Filtering en RAG]]
