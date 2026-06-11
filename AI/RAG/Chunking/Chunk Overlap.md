---
title: Chunk Overlap
source: https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking
author: togoAI Labs
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - Chunk Overlap
  - chunk-overlap
  - Overlap Strategy
updated: 2026-06-11
---

# Chunk Overlap

> [!note] Definición
> Compartir **O tokens** entre chunks adyacentes para crear una "zona buffer" que capture información a caballo entre dos chunks. Técnica **transversal**: se combina con cualquier estrategia de [[Chunking Strategies|chunking]].

## Cómo funciona

- Chunks adyacentes comparten `O` tokens (típico 10-20% del tamaño).
- `stride efectivo = chunk_size − overlap`.
- Ejemplo: chunks de 512 tokens + 102 de overlap → stride efectivo de 410 tokens.
- Resuelve la **suficiencia contextual**: si la respuesta se parte entre dos chunks, al menos uno tiene contexto suficiente.

## Ratios y guía

- **10%** — ligero; contenido bien estructurado con límites claros. → +3-5% accuracy.
- **15-20%** — **baseline de producción**, buen balance. → +7-12% accuracy (*sweet spot*).
- **25-30%** — pesado; contenido técnico denso o cortes de borde riesgosos. → +10-15% accuracy, pero **rendimientos decrecientes**.
- Regla práctica: el overlap debería cubrir **≥ 1 oración completa**, idealmente 2-3.

## Análisis de costo (por 1M tokens de fuente)

| | Sin overlap | Con 20% overlap |
|---|---|---|
| Chunks | 1.953 | 2.439 |
| Tokens indexados | 1.000.000 | 1.248.768 |
| Almacenamiento | base | **+24,8%** |

- Costo de embedding: $0.10 → $0.125 por 1M tokens (+$0.025).
- Para una KB de 100M tokens: **+$2.50** one-time de embedding, +25MB storage.
- Beneficio: **+7-12%** de accuracy de retrieval.

## Veredicto

> [!tip]
> **15-20% vale la pena para casi toda aplicación.** El costo (storage/cómputo) es marginal; la mejora de calidad es sustancial (menos alucinaciones, mejor UX, más tareas completadas).

## Cuándo NO usarlo / trade-offs

> [!warning]
> - >30% raramente justifica el costo (rendimientos decrecientes).
> - **Duplicación en resultados**: chunks solapados pueden devolver texto repetido al LLM → conviene deduplicar tras el retrieval.
> - 0% ahorra espacio pero arriesga partir respuestas.

## References

- Fuente: [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs, 2026-02-09

## Related

- [[Chunking Strategies]]
- [[Fixed-Size Chunking]]
