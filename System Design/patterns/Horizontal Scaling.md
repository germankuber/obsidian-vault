---
title: Horizontal Scaling
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/scaling
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Horizontal Scaling
  - horizontal-scaling
  - Scale Out
---

# Horizontal Scaling

> [!note] Definition
> Agregar **más máquinas** para manejar más tráfico: 10 servidores aguantan ~10×
> lo de uno. Escalás "hacia afuera" (*scale out*), no "hacia arriba".

## Cómo funciona

Se corren múltiples instancias idénticas del servicio detrás de un
[[Load Balancing|balanceador]] que reparte requests. Para que funcione, las
instancias deben ser **stateless** (el estado vive en una base/cache compartida),
así cualquier instancia atiende cualquier request.

## Cuándo usarlo

> [!tip]
> Cuando necesitás escalar más allá de lo que una sola máquina puede, o querés
> **alta disponibilidad** (si una instancia cae, las otras siguen). Es el modelo
> de escalado de la nube y la base del [[Auto-Scaling]].

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Exige diseño stateless**: el estado en memoria local (sesiones, caché) se
>   rompe al haber varias instancias. Hay que externalizarlo — refactor no trivial.
> - **Complejidad distribuida**: balanceo, *service discovery*, consistencia de
>   datos compartidos, despliegues coordinados.
> - **No todo paraleliza**: una base relacional monolítica no se escala solo
>   agregando réplicas de app — el cuello se muda a la base ([[Sharding]]).
> - Para cargas chicas, [[Vertical Scaling]] es más simple y suele alcanzar.

## Patrones relacionados / alternativas

- [[Vertical Scaling]] — máquina más grande en vez de más máquinas; más simple,
  con techo.
- [[Load Balancing]] — imprescindible para repartir entre instancias.
- [[Auto-Scaling]] — agregar/quitar instancias automáticamente.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Vertical Scaling]]
- [[Load Balancing]]
- [[Auto-Scaling]]
