---
title: Node vs Document Reranking
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-10
tags:
  - ai/rag/reranking
aliases:
  - Node vs Document Reranking
  - node-vs-document-reranking
  - Node vs Document
  - Reranking de Nodos vs Documentos
---

# Node vs Document Reranking

> [!note] Definición
> En la mayoría de los sistemas RAG (incluidos frameworks como **LlamaIndex** o **LangChain**) la información **no** se guarda como documentos completos sino como **nodes** (chunks de texto que entran en el límite de contexto). Por eso, cuando decimos "rerankear documentos", a menudo rerankeamos **nodes**.

> [!example] La metáfora del estante
> El knowledge base es un **estante** (documentos); cada libro está roto en **páginas** (nodes). Al consultar, los retrievers te dan una **pila de páginas**. A veces la mejor página está enterrada en un libro; a veces muchas buenas vienen del mismo libro. Cada documento tiene varios nodes; cada node lleva contexto local + metadata (fuente, título, vector/embedding).

## Los dos niveles

- **Node-level reranking** — foco en **relevancia semántica fina**. Como reordenar páginas sobre una mesa: muy local, rápido de inspeccionar, bueno cuando el snippet relevante es autocontenido.
- **Document-level reranking** — foco en **coherencia de fuente y contexto global**. Como elegir qué libro leer: valora coherencia global y señales de rango más largo.

> [!tip] Design hint
> - Si el generador necesita **contexto largo** o la respuesta requiere **síntesis entre nodes** → preferí document-level o estrategias mixtas que promuevan diversidad de fuentes.
> - Si las respuestas son del tamaño de un snippet (FAQ, Q/A corto) → node-level suele ser mejor.

Por qué importa: la relevancia a menudo **se esconde dentro de los documentos, no entre ellos**. Un solo documento puede tener diez nodes (nueve irrelevantes, uno perfecto).

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Derived vs Hybrid Reranking]]
- [[Chunking Strategies]]
