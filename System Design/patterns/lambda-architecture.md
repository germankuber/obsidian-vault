---
title: Lambda Architecture
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/data-processing
  - system-design/patterns
aliases:
  - Lambda Architecture
  - lambda-architecture
---

# Lambda Architecture

> [!note] Definition
> Correr procesamiento **batch** y **stream** en paralelo: el batch produce
> resultados exactos con alta latencia, el stream produce resultados aproximados
> con baja latencia, y una **capa de serving** fusiona ambos.

## Cómo funciona

- **Batch layer**: reprocesa todo el histórico ([[MapReduce]]/Spark) → vistas
  precisas pero "viejas".
- **Speed layer**: procesa los eventos recientes ([[Stream Processing]]) →
  cubre la ventana que el batch todavía no alcanzó.
- **Serving layer**: combina ambas vistas al responder una query, dando exactitud
  histórica + frescura reciente.

## Cuándo usarlo

> [!tip]
> Cuando necesitás **lo mejor de los dos**: la exactitud del batch y la latencia
> del stream, y podés tolerar mantener dos caminos. Analytics a gran escala donde
> el reproceso histórico debe ser exacto.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Lógica duplicada**: la misma transformación se implementa **dos veces**
>   (batch y stream) y hay que mantenerlas consistentes — la crítica principal a
>   este patrón.
> - **Complejidad operativa alta**: dos pipelines + capa de fusión.
> - **Alternativa moderna**: la *Kappa Architecture* propone un solo camino de
>   stream (reprocesando desde el log) para evitar la duplicación. Evaluala antes
>   de comprometerte con Lambda.

## Patrones relacionados / alternativas

- [[MapReduce]] — el batch layer.
- [[Stream Processing]] — el speed layer.
- [[Event Sourcing]] — un log de eventos reproducible habilita la idea de Kappa.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[MapReduce]]
- [[Stream Processing]]
