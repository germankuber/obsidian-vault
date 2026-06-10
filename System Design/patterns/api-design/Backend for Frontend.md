---
title: Backend for Frontend
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/api
  - system-design/patterns
aliases:
  - Backend for Frontend
  - backend-for-frontend
  - BFF
---

# Backend for Frontend

> [!note] Definition
> Crear una **capa de API separada por tipo de cliente**: un BFF móvil devuelve
> respuestas livianas, un BFF web devuelve respuestas más ricas. Cada frontend
> habla con su backend a medida.

## Cómo funciona

En vez de una API genérica que todos los clientes comparten (y que termina
sirviendo el mínimo común denominador), cada cliente tiene su BFF: agrega y
adapta los datos de los servicios downstream a las necesidades exactas de *ese*
frontend (forma del payload, campos, cantidad de round-trips).

## Cuándo usarlo

> [!tip]
> Cuando tenés **múltiples clientes con necesidades muy distintas** (móvil con
> ancho de banda limitado vs web con pantallas ricas) y una API única te obliga a
> compromisos incómodos o a sobre-fetching.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Duplicación**: cada BFF repite parte de la lógica de agregación → más código
>   que mantener y mantener sincronizado.
> - **Más servicios que operar y desplegar.**
> - Si tus clientes tienen necesidades **parecidas**, un solo [[API Gateway]] con
>   una API bien diseñada es más simple y evita la proliferación de BFFs.

## Patrones relacionados / alternativas

- [[API Gateway]] — un gateway único como alternativa más simple; el BFF es como
  tener un gateway por cliente.
- [[API Versioning]] — otra forma de servir clientes con necesidades distintas.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[API Gateway]]
- [[API Versioning]]
