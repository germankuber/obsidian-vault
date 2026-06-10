---
title: Hierarchical Chunking
source: https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design
author: Avani Chaskar
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/ingestion
  - type/pattern
  - status/permanent
aliases:
  - Hierarchical Chunking
  - hierarchical-chunking
  - Parent-Child Chunking
---

# Hierarchical Chunking

> [!note] Definition
> Trocear un documento en dos niveles: chunks **hijos** pequeños para la búsqueda
> semántica, y chunks **padres** más grandes para darle contexto al LLM. Se busca
> con los chicos pero se le entrega al modelo el bloque grande que los contiene.

## La tensión que resuelve

- Chunks **chicos** (~256 tokens) → mejores para *recuperar*: el embedding es
  más enfocado y la similitud más precisa.
- Chunks **grandes** (~1024 tokens) → mejores para *generar*: preservan el
  contexto alrededor de la frase relevante, evitando respuestas mutiladas.

La jerarquía permite tener ambos: la búsqueda matchea el hijo de 256 tokens, pero
al LLM se le pasa el padre de 1024 que lo envuelve. Así se gana precisión de
recuperación sin sacrificar contexto.

## References

- Fuente: [System Design Deep Dive: How to Design an Enterprise RAG Assistant](https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design) — Avani Chaskar, 2026-05-17

## Related

- [[Enterprise RAG Assistant]]
- [[Hybrid Search]]
