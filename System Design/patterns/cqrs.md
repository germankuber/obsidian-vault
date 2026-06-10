---
title: CQRS
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/storage
  - system-design/patterns
aliases:
  - CQRS
  - cqrs
  - Command Query Responsibility Segregation
---

# CQRS

> [!note] Definition
> *Command Query Responsibility Segregation*: separar el modelo de **escritura**
> (optimizado para consistencia y reglas de negocio) del modelo de **lectura**
> (optimizado para queries rápidas), posiblemente en bases o schemas distintos.

## Cómo funciona

Las escrituras (*commands*) van a un modelo normalizado que valida invariantes.
Las lecturas (*queries*) usan modelos desnormalizados, pre-calculados para cada
vista. Un mecanismo (eventos, replicación) mantiene los modelos de lectura
sincronizados con las escrituras. Cada lado escala y se modela de forma
independiente.

## Cuándo usarlo

> [!tip]
> Cuando lecturas y escrituras tienen **cargas o formas muy distintas**: muchas
> lecturas con vistas complejas vs. escrituras con lógica de negocio pesada.
> Brilla junto a [[Event Sourcing]] y en sistemas de alto tráfico de lectura.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Consistencia eventual**: el modelo de lectura va por detrás del de
>   escritura. Si tu UI necesita leer-tras-escribir inmediato, duele.
> - **Duplica modelos y código**: dos representaciones que mantener
>   sincronizadas. Más superficie de bugs.
> - Es uno de los patrones **más sobre-aplicados**: para un CRUD normal con
>   lecturas y escrituras parecidas, un solo modelo es más simple y correcto.
>   Usalo solo cuando la asimetría lo justifica.

## Patrones relacionados / alternativas

- [[Event Sourcing]] — combinación clásica: los eventos alimentan los modelos de
  lectura.
- [[Primary-Replica]] — las réplicas de lectura son una forma "ligera" de separar
  read/write sin todo el peso de CQRS.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Event Sourcing]]
- [[Primary-Replica]]
