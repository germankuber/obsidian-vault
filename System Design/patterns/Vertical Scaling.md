---
title: Vertical Scaling
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/scaling
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Vertical Scaling
  - vertical-scaling
  - Scale Up
---

# Vertical Scaling

> [!note] Definition
> Pasar a una máquina **más potente**: más CPU, más RAM, discos más rápidos.
> Escalás "hacia arriba" (*scale up*) la misma instancia, en vez de agregar más.

## Cómo funciona

Se reemplaza el servidor por uno con más recursos. No requiere cambios de
arquitectura: la app sigue siendo una sola instancia. Es lo primero que se
prueba porque no toca el código.

## Cuándo usarlo

> [!tip]
> Como **primer paso** de escalado, o para cargas que no paralelizan bien: una
> base relacional monolítica, software con estado en memoria difícil de
> distribuir. Simple y sin complejidad distribuida.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Tiene un techo físico**: hay un límite a cuánta CPU/RAM podés ponerle a una
>   máquina, y el costo crece **no linealmente** (las máquinas top son carísimas).
> - **Punto único de falla**: una sola máquina; si cae, cae todo. No da alta
>   disponibilidad por sí sola.
> - **Downtime al escalar**: muchas veces hay que reiniciar para cambiar de
>   instancia.
> - Pasado cierto punto, [[Horizontal Scaling]] es la única salida.

## Patrones relacionados / alternativas

- [[Horizontal Scaling]] — la alternativa que escala sin techo y da HA, a costa
  de complejidad.
- [[Auto-Scaling]] — generalmente opera sobre escalado horizontal.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Horizontal Scaling]]
