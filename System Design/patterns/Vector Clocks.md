---
title: Vector Clocks
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/consistency
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - Vector Clocks
  - vector-clocks
---

# Vector Clocks

> [!note] Definition
> Rastrear la **causalidad** en un sistema distribuido manteniendo un vector de
> timestamps lógicos (uno por nodo). Comparando vectores se identifica si un
> evento ocurrió antes que otro, o si fueron **concurrentes**.

## Cómo funciona

Cada nodo mantiene un contador por cada nodo del sistema. Al hacer un evento,
incrementa su propio contador; al recibir un mensaje, toma el máximo elemento a
elemento y luego incrementa el suyo. Comparando dos vectores:
- Si uno es ≤ el otro en todos los componentes → hay orden causal.
- Si ninguno domina → eventos **concurrentes** (posible conflicto).

No usan tiempo de reloj físico (que no es confiable entre máquinas), sino orden
lógico.

## Cuándo usarlo

> [!tip]
> En almacenes distribuidos sin líder (estilo Dynamo) para **detectar conflictos**
> entre versiones de un mismo dato escritas en réplicas distintas, y decidir cuál
> gana o si hay que mergear.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **El vector crece con la cantidad de nodos**: en sistemas con muchos
>   escritores, los vectores se vuelven grandes (overhead de almacenamiento y red).
> - **Detectan conflictos pero no los resuelven**: la lógica de merge (qué hacer
>   con dos versiones concurrentes) queda a cargo de la aplicación.
> - **Complejo de implementar y razonar.**
> - Si tenés un líder único ([[Primary-Replica]]) que ordena las escrituras, no
>   hacen falta.

## Patrones relacionados / alternativas

- [[Quorum]] — control de consistencia que reduce (pero no elimina) conflictos.
- [[Primary-Replica]] — un líder único evita la necesidad de detectar causalidad.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Quorum]]
- [[Primary-Replica]]
