---
title: RAG — Mapa del tema
created: 2026-06-08
tags:
  - ai/rag
  - moc
aliases:
  - RAG
  - RAG MOC
  - Retrieval-Augmented Generation
---

# RAG — Mapa del tema

> [!note] Cómo usar esta nota
> Es el índice (MOC) de la carpeta `ai/rag`. Empezá por arriba y bajá: cada
> sección va de lo fundamental a lo avanzado. Abrí esta nota, no la carpeta.

## 🚀 Empezá por acá

- [[Enterprise RAG Assistant]] — la visión de sistema completa: requisitos, las
  dos pipelines (ingesta + consulta), control de acceso y cuellos de botella.
  Todo lo demás de abajo son piezas de este diseño.

## 📥 Ingesta — preparar los datos

- [[Change Data Capture]] — captar cambios de las fuentes (webhooks + Kafka) sin
  re-escanear todo.
- **[[_Chunking|Chunking]]** 📁 — subtema con carpeta propia (`chunking/`): cómo trocear
  los documentos. 8 estrategias + overlap. Abrí su MOC.

## 🔎 Recuperación — encontrar el contexto

- [[Hybrid Search]] — combinar búsqueda densa (vectores) y dispersa (BM25).
- [[BM25]] — el lado *sparse*: ranking léxico por frecuencia de términos (matchea
  palabras exactas, no significado).
- [[ACL Filtering en RAG]] — la "regla de oro": filtrar permisos en una sola
  etapa, dentro de la consulta.

## 🎯 Reranking — reordenar por relevancia

- **[[_Reranking|Reranking]]** 📁 — subtema con carpeta propia (`reranking/`):
  reordenar por relevancia. El hub conceptual es [[Reranking]]; la carpeta tiene
  el espectro Bi-Encoder → ColBERT → Cross-Encoder → LLM, más BM25 y RRF. Abrí
  su MOC.

## 🌱 Por escribir (semillas del grafo)

Conceptos ya enlazados desde las notas de arriba pero que todavía no tienen nota
propia — candidatos a promover cuando un próximo artículo aporte material:

- [[Derived vs Hybrid Reranking|Derived Reranking]] · [[Derived vs Hybrid Reranking|Hybrid Reranking]] · [[Multimodal Reranking]] · [[Agent Reranking]]
- [[GraphRAG]] · [[Query Rewriting]] · [[Redis Cache]] · [[Ghost Chunk Problem]] · [[Server-Sent Events]]

## 🔍 Todas las notas del dominio RAG (auto)

```dataview
LIST
FROM "AI/rag"
WHERE file.name != this.file.name
SORT file.path ASC
```
