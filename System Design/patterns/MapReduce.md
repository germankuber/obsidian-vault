---
title: MapReduce
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/data-processing
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - MapReduce
  - mapreduce
---

# MapReduce

> [!note] Definition
> Partir un dataset enorme entre muchas máquinas (fase **map**), cada una procesa
> su parte independiente, y combinar los resultados (fase **reduce**). Es la base
> del procesamiento **batch** de datos a gran escala.

## Cómo funciona

- **Map**: cada nodo aplica una función a su porción y emite pares clave-valor.
- **Shuffle**: el framework agrupa los valores por clave y los manda al reducer
  correspondiente.
- **Reduce**: cada reducer agrega los valores de una clave en el resultado final.

El framework (Hadoop, etc.) maneja la distribución, los reintentos y la
tolerancia a fallas. El modelo escala horizontalmente sumando nodos.

## Cuándo usarlo

> [!tip]
> Para procesar **volúmenes masivos** de datos en batch donde la latencia no
> importa: indexado, agregaciones históricas, ETL nocturno, log analytics. El
> problema debe descomponerse en map + reduce.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Alta latencia**: es batch — los resultados tardan minutos u horas. Para
>   datos en vivo, [[Stream Processing]].
> - **No todo encaja en map/reduce**: algoritmos iterativos (grafos, ML) son
>   torpes en este modelo; herramientas como Spark los hacen mejor.
> - **Overhead de IO**: la escritura a disco entre fases (en Hadoop clásico) es
>   lenta — motores en memoria la evitan.
> - Para datasets que entran en una máquina, es over-engineering.

## Patrones relacionados / alternativas

- [[Stream Processing]] — el equivalente en tiempo real (evento a evento).
- [[Lambda Architecture]] — combina batch (MapReduce) y stream.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Stream Processing]]
- [[Lambda Architecture]]
