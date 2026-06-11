---
title: Connection Pooling
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/scaling
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Connection Pooling
  - connection-pooling
updated: 2026-06-11
---

# Connection Pooling

> [!note] Definition
> Mantener un **pool de conexiones reutilizables** a la base (u otro recurso) en vez de abrir una nueva por cada request. Las conexiones se prestan y se devuelven al pool.

## Cómo funciona

Abrir una conexión (TCP + handshake + auth) es caro. El pool pre-abre N conexiones; cada request toma una prestada, la usa y la devuelve. Limita además cuántas conexiones concurrentes golpean la base — la protege de saturarse.

## Cuándo usarlo

> [!tip]
> Prácticamente **siempre** que una app de alto tráfico hable con una base. Esencial cuando escalás horizontalmente: sin pool, N instancias × M requests abren una avalancha de conexiones que tumba la base.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Dimensionar el pool es un equilibrio**: muy chico y los requests esperan conexión libre (latencia, timeouts); muy grande y saturás la base (que tiene su propio límite de conexiones).
> - **El total se multiplica por instancia**: 50 instancias × pool de 20 = 1000 conexiones a la base. Hay que pensar el número *agregado*, no por instancia (de ahí los *poolers* externos como PgBouncer).
> - **Conexiones zombi**: conexiones rotas en el pool dan errores intermitentes; necesita validación/health checks de conexión.

## Patrones relacionados / alternativas

- [[Horizontal Scaling]] / [[Auto-Scaling]] — multiplican la presión sobre la base; el pool la contiene.
- [[Bulkhead]] — pools separados por dependencia para aislar saturación.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Horizontal Scaling]]
- [[Bulkhead]]
