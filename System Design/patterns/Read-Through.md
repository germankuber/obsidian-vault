---
title: Read-Through
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/caching
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Read-Through
  - read-through
updated: 2026-06-11
---

# Read-Through

> [!note] Definition
> La **cache misma** carga el dato desde la base cuando hay un miss. La aplicación habla solo con la cache, nunca directamente con la base.

## Cómo funciona

A diferencia de [[Cache-Aside]] (donde la app coordina cache y base), acá la cache encapsula la lógica de carga: ante un miss, la cache va a la base, se puebla y devuelve. La app ve una sola interfaz. Suele ofrecerlo la librería/capa de cache (no la implementás a mano).

## Cuándo usarlo

> [!tip]
> Cuando querés **centralizar la lógica de carga** en la cache y simplificar el código de la app — útil cuando muchos servicios comparten la misma estrategia. Va de la mano con [[Write-Through]].

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Acoplás la cache a la base**: la cache necesita saber cómo cargar de la fuente, lo que la hace menos genérica y más difícil de testear/mockear.
> - **Mismo miss frío inicial** que cache-aside.
> - Si querés control fino sobre *cómo* y *qué* se carga (queries custom, transformaciones), [[Cache-Aside]] te da más control en la app.

## Patrones relacionados / alternativas

- [[Cache-Aside]] — la alternativa donde la app controla la carga (más común).
- [[Write-Through]] — su compañero del lado escritura para cache consistente.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Cache-Aside]]
- [[Write-Through]]
