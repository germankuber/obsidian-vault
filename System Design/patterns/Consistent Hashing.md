---
title: Consistent Hashing
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/storage
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - Consistent Hashing
  - consistent-hashing
  - Hash Ring
updated: 2026-06-11
---

# Consistent Hashing

> [!note] Definition
> Un método de hashing donde agregar o quitar un servidor solo obliga a remapear una **fracción** de las claves, en vez de reshufflear todo. Servidores y claves se ubican en un **anillo virtual**.

## Cómo funciona

Tanto las claves como los servidores se hashean a posiciones de un anillo (0…2³²). Cada clave pertenece al primer servidor que aparece girando en sentido horario. Cuando un servidor se va, solo sus claves migran al siguiente; cuando entra uno, solo se lleva las claves de un tramo. Los **nodos virtuales** (varias posiciones por servidor) reparten la carga más pareja.

## Cuándo usarlo

> [!tip]
> Cuando el conjunto de servidores **cambia con frecuencia** y un hashing ingenuo (`hash(key) % N`) sería desastroso — porque cambiar `N` remapea *casi todas* las claves. Clave en cachés distribuidas, [[Sharding]] dinámico, y sistemas de almacenamiento tipo Dynamo.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Sin nodos virtuales, el reparto es desparejo**: pocos servidores en el anillo dejan tramos de tamaños muy distintos → *hot spots*.
> - **No resuelve los hot keys**: si una sola clave es ultra-popular, vive en un solo nodo igual. Eso es problema de caché/replicación, no del hashing.
> - Más complejo de implementar y depurar que un módulo simple; si tu `N` es **fijo**, el `hash % N` clásico alcanza y es más simple.

## Patrones relacionados / alternativas

- [[Sharding]] — el caso de uso principal: mapear claves a shards.
- [[Primary-Replica]] — replicación encima de los nodos del anillo.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Sharding]]
