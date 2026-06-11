---
title: API Versioning
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/api
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - API Versioning
  - api-versioning
updated: 2026-06-11
---

# API Versioning

> [!note] Definition
> Mantener **varias versiones de la API a la vez** para que los clientes viejos sigan funcionando mientras la API evoluciona y los nuevos usan la última.

## Cómo funciona

Estrategias comunes: versión en la **URL** (`/v1/`, `/v2/`), en un **header** (`Accept: application/vnd.api.v2+json`), o por *content negotiation*. Cada versión mantiene su contrato; los cambios incompatibles van a una versión nueva, no rompen la existente.

## Cuándo usarlo

> [!tip]
> Siempre que la API tenga **consumidores externos** que no controlás (apps móviles ya instaladas, integraciones de terceros): no podés actualizarlos a todos de golpe. Versionar te deja evolucionar sin romperlos.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Costo de mantener varias versiones**: cada una es código y tests vivos. Sin una política de *deprecation*, se acumulan versiones zombi para siempre.
> - **Mejor evitar breaking changes**: muchos cambios pueden ser retrocompatibles (agregar campos opcionales) sin nueva versión. Versioná solo cuando el cambio es genuinamente incompatible.
> - Para una API **interna** donde controlás todos los clientes, podés actualizarlos en conjunto y versionar menos.

## Patrones relacionados / alternativas

- [[Backend for Frontend]] — otra forma de servir necesidades distintas sin versionar la API compartida.
- [[API Gateway]] — buen lugar para rutear versiones y gestionar deprecations.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Backend for Frontend]]
- [[API Gateway]]
