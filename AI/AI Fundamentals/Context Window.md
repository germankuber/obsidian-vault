---
title: Context Window
source: https://jamwithai.substack.com/p/agent-memory-the-7-types-you-should
author: Shantanu Ladhwe, Shirin Khosravi Jam
created: 2026-07-04
tags:
  - ai/fundamentals
  - type/concept
  - status/permanent
aliases:
  - Context Window
  - Ventana de contexto
reading:
  total_words: 232
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-07-04
---

# Context Window

> [!note] Definición
> El **context window** es el **máximo de texto que un LLM puede procesar de una vez**, medido en [[Tokens]]. Todo lo que el modelo "ve" en un turno — instrucciones, historial, resultados de tools, lookups — tiene que **caber dentro** de esta ventana.

## Por qué importa

- Es el límite físico que restringe la **working memory** de un agente: una conversación larga **no entra**, y hay que elegir **qué conservar, qué resumir y qué descartar** (ver [[Agent Memory]], §1 Working memory).
- **Aunque el window fuera enorme**, no conviene llenarlo: **más tokens = más latencia y más costo** por turno. Un window grande es un techo, no un objetivo.

## Cuándo NO tratarlo como "memoria infinita"

> [!warning]
> Tener un window de un millón de tokens no reemplaza una estrategia de memoria. Meter todo el historial adentro es lento, caro y **entierra la señal en ruido**. La memoria externa + retrieval existen justamente para traer **solo lo relevante** a la ventana. Ver [[Agent Memory]].

## Related

- [[Tokens]] — la unidad en que se mide la ventana.
- [[Agent Memory]] — la working memory está limitada por esta ventana.

## Referencias

- Fuente: [Agent Memory: the 7 types you should know](https://jamwithai.substack.com/p/agent-memory-the-7-types-you-should) — Shantanu Ladhwe y Shirin Khosravi Jam, 2026-07-01. Esta nota cubre el ángulo de memoria; el resto del concepto es conocimiento general.
