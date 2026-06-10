---
title: Cache Stampede Prevention
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/caching
  - system-design/patterns
aliases:
  - Cache Stampede Prevention
  - cache-stampede-prevention
  - Thundering Herd
  - Dogpile Effect
---

# Cache Stampede Prevention

> [!note] Definition
> Evitar que, cuando una entrada de cache **popular expira**, miles de requests
> simultáneos sufran el miss a la vez y golpeen todos juntos la base (una
> *estampida* / thundering herd) que puede tumbarla.

## Cómo funciona

Tres técnicas habituales:
- **Request coalescing (lock)**: solo el primer request que ve el miss recarga;
  los demás esperan y reusan ese resultado.
- **Expiración probabilística temprana**: se renueva la entrada *antes* de que
  expire, con probabilidad creciente al acercarse el TTL — así un solo request
  la refresca y el resto sigue sirviendo lo cacheado.
- **Locks distribuidos**: un lock por clave garantiza que solo un proceso
  recarga en todo el cluster.

## Cuándo usarlo

> [!tip]
> Cuando tenés **claves muy calientes** detrás de [[Cache-Aside]]: una página de
> inicio, un "documento All-Hands", un trending. Sin protección, su expiración es
> un mini-DDoS contra tu propia base.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Agrega complejidad y latencia**: los requests que esperan el lock pagan esa
>   espera; un lock mal liberado puede colgar a todos.
> - **El coalescing crea un punto de serialización** sobre la clave caliente.
> - Para claves de tráfico bajo o parejo, no hace falta — es over-engineering.

## Patrones relacionados / alternativas

- [[Cache-Aside]] — el patrón donde más aparece la estampida.
- [[Rate Limiting]] — protege la base por otra vía (limitando requests).
- Relacionado con el [[Retry with Backoff]] *jitter*: la misma idea de
  desincronizar para evitar herds.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Cache-Aside]]
- [[Rate Limiting]]
