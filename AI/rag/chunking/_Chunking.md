---
title: Chunking
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - type/moc
  - status/permanent
aliases:
  - Chunking
  - Chunking MOC
---

# Chunking

> [!note] Cómo usar esta nota
> Índice del subtema *chunking* dentro de [[_RAG|RAG]]. Empezá por el hub y bajá.

## 🚀 Empezá por acá

- [[Chunking Strategies]] — **el hub.** Por qué el chunking explica el 60-80% de
  la calidad de retrieval, el baseline de producción (400-512 tokens, 15-20%
  overlap) y el framework de selección por tipo de contenido.

## Estrategias (de más simple a más consciente de la estructura)

- [[Fixed-Size Chunking]] — por caracteres o tokens; baseline grosero.
- [[Sentence-Based Chunking]] — agrupa oraciones completas.
- [[Paragraph-Based Chunking]] — agrupa párrafos; mejor coherencia en prosa.
- [[Recursive Character Splitting]] — el todoterreno de LangChain.
- [[Markdown-Aware Chunking]] — respeta headers/código en docs técnicas.
- [[Code-Aware Chunking]] — corta por función/clase en repos.

## Técnicas transversales y avanzadas

- [[Chunk Overlap]] — solape entre chunks (15-20%); aplica a todas.
- [[Hierarchical Chunking]] — chunks hijo (búsqueda) + padre (contexto del LLM).

## Conexión

- Subtema de [[_RAG|RAG]] · alimenta la [[Hybrid Search]] del pipeline de consulta.

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "AI/rag/chunking"
WHERE file.name != this.file.name
SORT file.name ASC
```
