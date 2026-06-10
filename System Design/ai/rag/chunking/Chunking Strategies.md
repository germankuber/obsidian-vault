---
title: Chunking Strategies
source: https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking
author: togoAI Labs
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/patterns
aliases:
  - Chunking Strategies
  - chunking-strategies
  - RAG Chunking
  - Chunking
---

# Chunking Strategies

> [!note] Definición
> Un **chunk** es un segmento contiguo de texto: la **unidad atómica de
> recuperación** en RAG. Se embebe en un vector, se indexa, se recupera por
> similitud y se le pasa al LLM como contexto. Un buen chunk debe ser a la vez
> **autocontenido, específico, conciso y coherente** — requisitos que entran en
> conflicto entre sí.

## Por qué importa tanto

- El chunking explica el **60-80% de la varianza** de calidad de retrieval (datos
  de producción 2026).
- Más influyente que la elección de modelo de embeddings, de reranker o de LLM.
- Ejemplo brutal: sentence-transformers básico + chunks semánticos = **78% MRR@10**,
  vs. Voyage AI state-of-the-art + chunks naive de 500 chars = **62%**. 16 puntos
  de diferencia, **costo cero**. Antes de gastar en mejores embeddings o
  [[Reranking]], arreglá el chunking.

> [!warning] Recuperación = compresión con pérdida
> Cómo chunkeás define el **techo** de lo que el sistema puede recuperar. Si la
> respuesta queda partida por un corte naive, ninguna sofisticación de embeddings
> la recupera. Los límites de chunk fijan el máximo alcanzable.

## El problema profundo: similitud ≠ suficiencia contextual

- Objetivo matemático: maximizar la relevancia total de los `k` chunks
  recuperados dentro del presupuesto de la ventana de contexto.
- **Relevancia = similitud semántica + suficiencia contextual.** Alta similitud
  NO implica contexto suficiente.
- Ejemplo: el chunk *"reemplazo de batería cubierto por garantía"* matchea bien
  la query *"período de garantía"* pero **no la responde**.
- [[Chunk Overlap]] mitiga esto; las estrategias semánticas avanzadas (Part 2)
  lo atacan de frente.

## El baseline de producción

> [!example] Punto de partida para la mayoría
> - **Tamaño de chunk:** 400-512 tokens
> - **Overlap:** 15-20% (ver [[Chunk Overlap]])
> - **Jerarquía de separadores:** `["\n\n", "\n", ". ", " ", ""]`

## El espectro de estrategias

| Estrategia | MRR@10 | Coherencia | Velocidad | Mejor para |
|---|---|---|---|---|
| [[Fixed-Size Chunking]] · character | 58-62% | 0.45 | `O(n)` | solo prototipo |
| [[Fixed-Size Chunking]] · token | 62-68% | 0.51 | ~5K chunks/s | baseline |
| [[Sentence-Based Chunking]] | 71-75% | 0.78 | `O(n)`+ | prosa |
| [[Paragraph-Based Chunking]] | 76-80% | 0.84 | adaptativo | prosa estructurada |
| [[Recursive Character Splitting]] | 74-78% | alta | ~2K chunks/s | contenido mixto |
| [[Markdown-Aware Chunking]] | — | — | — | docs técnicas |
| [[Code-Aware Chunking]] | — | — | — | repos de código |

Transversal a todas: [[Chunk Overlap]]. Avanzada: [[Hierarchical Chunking]].

## Árbol de decisión por tipo de contenido

> [!tip]
> - Prosa estructurada (artículos, libros, reportes) → [[Paragraph-Based Chunking]] o [[Sentence-Based Chunking]]
> - Contenido mixto (docs + tablas + código + texto) → [[Recursive Character Splitting]]
> - Docs técnicas en markdown → [[Markdown-Aware Chunking]]
> - Repositorios de código → [[Code-Aware Chunking]]
> - Dumps no estructurados (chat logs, social) → [[Fixed-Size Chunking]] token-based

## Mapeo por nivel de madurez

- **MVP / arranque** → [[Recursive Character Splitting]], 512 tokens, 20% overlap.
- **Producción** → sentence o paragraph-based, 256-512 tokens, 15-20% overlap.
- **Alto riesgo** (legal, médico, financiero) → estrategias semánticas avanzadas
  (Part 2 del artículo).

## El trade-off de fondo: precisión vs recall vs eficiencia

- **Chunks grandes**: alto recall, baja precisión, desperdician ventana de contexto.
- **Chunks chicos**: alta precisión, bajo recall, obligan a recuperar más (redundante).
- **Zona Goldilocks**: 256-512 tokens para la mayoría de los casos.

## Medición (no adivines)

- Evaluá sobre **tus** datos con **tus** queries reales.
- Métricas: accuracy de retrieval (MRR@10 o NDCG@10), score de coherencia, tasa
  de retrieval fallido. Iterá según el dominio.

## References

- Fuente: [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs, 2026-02-09

## Related

- [[_RAG|RAG]]
- [[Enterprise RAG Assistant]]
- [[Hierarchical Chunking]]
- [[Hybrid Search]]
