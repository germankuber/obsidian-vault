---
title: Fixed-Size Chunking
source: https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking
author: togoAI Labs
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Fixed-Size Chunking
  - fixed-size-chunking
  - Character-Based Splitting
  - Token-Based Splitting
updated: 2026-06-11
---

# Fixed-Size Chunking

> [!note] Definición
> Partir el texto en fragmentos de tamaño fijo **ignorando la estructura** del contenido. Dos variantes: por **caracteres** o por **tokens**. Son las estrategias más simples y las de peor coherencia.

## Variante 1 — Character-Based

**Cómo funciona**
- Corta cada N caracteres, sin importar el contenido. Overlap opcional.
- Algoritmo: itera el texto con `stride = chunk_size - overlap`.

**Números**
- Velocidad: `O(n)`, overhead mínimo (la más rápida).
- Tamaños uniformes salvo el último chunk.
- Retrieval (MSMARCO): **58-62% MRR@10**. Coherencia: **0.45/1.0**.

**Problema central**
- Destroza límites semánticos: corta a mitad de palabra/frase/idea.
- Ejemplos del artículo: `"Series"` → `"Ser"` + `"ies"`; `"fast charging"` → `"fast"` + `"charging"`.

## Variante 2 — Token-Based

**Cómo funciona**
- Corta cada N **tokens** usando el tokenizer del modelo de embeddings.
- Garantiza que cada chunk entre en el límite de tokens del modelo.
- Algoritmo: encode → partir el array de tokens → decode cada chunk a texto.

**Token vs carácter (clave)**
- Inglés (GPT/Claude): ~1 token ≈ **4 caracteres** ≈ **0.75 palabras**.
- Código: ~1 token ≈ 2-3 caracteres (mayor densidad).
- Scripts no latinos: varía mucho.
- Los modelos se definen por límite de **tokens**, no de caracteres → por eso importa tokenizar.

**Números**
- Velocidad: `O(n)` + overhead de tokenización; ~**5000 chunks/seg** (M2).
- Retrieval: **62-68% MRR@10**. Coherencia: **0.51/1.0** (+6% vs character).
- Mejora: rara vez corta a mitad de palabra, pero **sí corta a mitad de frase**.

**Librería**: `tiktoken` (tokenización estilo OpenAI, p. ej. `cl100k_base` para GPT-4).

## Cuándo usarlo

> [!tip]
> - **Character**: solo prototipo, texto 100% uniforme (logs de sensores), o baseline para comparar mejoras.
> - **Token**: cuando necesitás control estricto del tamaño de input, sobre contenido poco estructurado (posts sociales, chat logs, reviews).

## Cuándo NO usarlo / trade-offs

> [!warning]
> - Ninguna es **production-ready** para aplicaciones sensibles a calidad.
> - No respetan unidades semánticas → coherencia baja (0.45-0.51).
> - Para prosa real, [[Sentence-Based Chunking]] o [[Paragraph-Based Chunking]] suben 9-18 puntos de MRR@10 **sin costo extra**.
> - El daño se mitiga parcialmente con [[Chunk Overlap]].

## References

- Fuente: [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs, 2026-02-09

## Related

- [[Chunking Strategies]]
- [[Chunk Overlap]]
- [[Sentence-Based Chunking]]
