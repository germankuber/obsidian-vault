---
title: Cache-Aside
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/caching
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Cache-Aside
  - cache-aside
  - Lazy Loading
---

# Cache-Aside

> [!note] Definition
> La aplicación mira la cache primero; si hay *miss*, lee de la base, devuelve el
> dato y **lo guarda en la cache** para la próxima. La cache se llena "a demanda".

## Cómo funciona

La app es la dueña de la lógica: `dato = cache.get(k)`; si es null, `dato =
db.get(k)` y `cache.set(k, dato)`. La cache no sabe nada de la base — es la app
la que coordina. Para invalidar, se borra la entrada y el próximo acceso la
recarga.

## Cuándo usarlo

> [!tip]
> El patrón de caché **por defecto** para cargas read-heavy. Solo se cachea lo
> que realmente se pide. Resiliente: si la cache se cae, la app sigue yendo a la
> base (más lento, pero funciona).

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **El primer acceso siempre es un miss** (cache fría) → latencia. Se mitiga
>   con *warming*.
> - **Riesgo de datos stale**: si alguien actualiza la base sin invalidar la
>   cache, servís viejo hasta que expire el TTL. La invalidación es "el problema
>   difícil".
> - Cuando una entrada popular expira, miles de requests pegan a la base a la vez
>   → ver [[Cache Stampede Prevention]].
> - Para cargas write-heavy donde casi no se relee, cachear no rinde.

## Patrones relacionados / alternativas

- [[Read-Through]] — igual idea pero la **cache** carga de la base, no la app.
- [[Write-Through]] / [[Write-Behind]] — estrategias de escritura que mantienen
  la cache fresca y evitan parte del staleness.
- [[Cache Stampede Prevention]] — para el problema de expiración simultánea.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Read-Through]]
- [[Cache Stampede Prevention]]
