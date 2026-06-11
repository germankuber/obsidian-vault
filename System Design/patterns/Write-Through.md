---
title: Write-Through
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/caching
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Write-Through
  - write-through
updated: 2026-06-11
---

# Write-Through

> [!note] Definition
> Cada escritura va **a la vez** a la cache y a la base. La cache siempre está actualizada porque nunca se escribe en la base sin escribir también en la cache.

## Cómo funciona

`cache.set(k, v)` y `db.set(k, v)` ocurren juntos (la escritura "atraviesa" la cache hacia la base). Una lectura posterior siempre encuentra el dato fresco en cache. Se suele combinar con [[Read-Through]] para tener cache consistente en lectura y escritura.

## Cuándo usarlo

> [!tip]
> Cuando necesitás que la cache **nunca sirva datos stale** y leés seguido lo que escribís. La consistencia cache-base es fuerte.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Cada escritura paga la latencia de los dos** (cache + base) — escrituras más lentas que escribir solo a la base.
> - **Cachea cosas que quizás nunca se lean**: escribís a la cache datos que tal vez nadie pida, desperdiciando memoria. ([[Cache-Aside]] solo cachea lo que se pide.)
> - No mejora el throughput de escritura; si tu cuello es escribir rápido, mirá [[Write-Behind]].

## Patrones relacionados / alternativas

- [[Write-Behind]] — escribe rápido a la cache y vuelca a la base async (más rápido, menos durable).
- [[Cache-Aside]] — no cachea en la escritura, solo en la lectura.
- [[Read-Through]] — su compañero natural del lado lectura.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Write-Behind]]
- [[Cache-Aside]]
- [[Read-Through]]
