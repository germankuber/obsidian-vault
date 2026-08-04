---
title: Fine-tuning
source: https://jamwithai.substack.com/p/agent-memory-the-7-types-you-should
author: Shantanu Ladhwe, Shirin Khosravi Jam
created: 2026-07-04
tags:
  - ai/fundamentals
  - type/concept
  - status/permanent
aliases:
  - Fine-tuning
  - Fine tuning
  - Ajuste fino
reading:
  total_words: 181
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-07-04
---

# Fine-tuning

> [!note] Definición
> **Fine-tuning** es **seguir entrenando** un modelo pre-entrenado sobre **tus propios ejemplos**, ajustando sus **weights** (parámetros). Modifica la **parametric memory** del modelo: lo que sabe "de fábrica".

## Para qué sirve

- **Cambiar estilo y comportamiento**: tono, formato de respuesta, seguir un patrón de tarea específico. Ahí es fuerte.

## Cuándo NO usarlo

> [!warning]
> Es una **mala forma de guardar hechos que cambian**. Los weights quedan **congelados en el momento del training**: no podés meter la mano y corregir un dato puntual, y cualquier hecho actual, privado o volátil queda desactualizado. Para eso servís mejor con **memoria externa + retrieval** (una fuente confiable que vos controlás), no re-entrenando el modelo. Ver [[Agent Memory]], §6 Parametric memory.

## Related

- [[Agent Memory]] — el fine-tuning modifica la parametric memory (tipo 6).
- [[RLHF]] — otra forma de post-training que ajusta comportamiento.

## Referencias

- Fuente: [Agent Memory: the 7 types you should know](https://jamwithai.substack.com/p/agent-memory-the-7-types-you-should) — Shantanu Ladhwe y Shirin Khosravi Jam, 2026-07-01. Esta nota cubre el ángulo de memoria; el resto del concepto es conocimiento general.
