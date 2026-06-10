---
title: Sharding
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/storage
  - system-design/patterns
aliases:
  - Sharding
  - sharding
  - Partitioning
  - Horizontal Partitioning
---

# Sharding

> [!note] Definition
> Partir los datos entre varios servidores, cada uno con una porción. Una **shard
> key** determina en qué servidor vive cada registro. Escala lo que
> [[Primary-Replica]] no puede: las **escrituras**.

## Cómo funciona

Se elige una shard key (p. ej. `user_id`) y una función que la mapea a un shard.
Cada shard es una base independiente que maneja su subconjunto de datos. Las
queries que incluyen la shard key van directo al shard correcto; las que no,
tienen que abanicarse (*scatter-gather*) a todos.

## Cuándo usarlo

> [!tip]
> Cuando el volumen de datos o el throughput de **escritura** supera lo que un
> solo nodo aguanta. Es el último recurso de escalado de datos, después de
> agotar réplicas de lectura y caché.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Es una de las decisiones más difíciles de revertir.** Re-shardear datos en
>   producción es doloroso. No shardees antes de necesitarlo.
> - **Elegir la shard key es crítico**: una mala key da *hot shards* (un shard
>   recibe todo el tráfico). [[Consistent Hashing]] ayuda a repartir parejo.
> - **Las queries cross-shard son caras**: joins y agregaciones que cruzan shards
>   pierden la ventaja; las transacciones cross-shard requieren [[Two-Phase Commit]]
>   o [[Saga]].
> - Agrega complejidad operativa enorme (rebalanceo, backups por shard, etc.).

## Patrones relacionados / alternativas

- [[Consistent Hashing]] — cómo mapear claves a shards minimizando el remapeo al
  agregar/quitar nodos.
- [[Primary-Replica]] — escala lecturas; sharding escala escrituras. Se combinan.
- [[Two-Phase Commit]] / [[Saga]] — para transacciones que cruzan shards.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Consistent Hashing]]
- [[Primary-Replica]]
