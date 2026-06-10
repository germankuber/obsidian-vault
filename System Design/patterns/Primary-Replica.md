---
title: Primary-Replica
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/storage
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Primary-Replica
  - primary-replica
  - Master-Slave
  - Leader-Follower
---

# Primary-Replica

> [!note] Definition
> Un nodo **primario** toma todas las escrituras; una o más **réplicas** copian
> esas escrituras y sirven lecturas. Si el primario cae, una réplica es promovida
> a primario.

## Cómo funciona

El primario propaga su log de cambios a las réplicas (replicación síncrona o
asíncrona). Las lecturas se reparten entre réplicas para escalar el read
throughput; las escrituras siguen yendo a un único primario, que es la fuente de
verdad.

## Cuándo usarlo

> [!tip]
> Cuando tenés **muchas más lecturas que escrituras** (el caso común) y querés
> escalar lectura y tener tolerancia a fallas. Es la base de casi toda base
> relacional en producción.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Lag de replicación**: con replicación asíncrona, una réplica puede servir
>   datos viejos — lees algo que recién escribiste y no aparece (*read-your-writes*
>   roto). Mitigás leyendo del primario lo recién escrito, o con [[Quorum]].
> - **El primario es cuello de botella de escritura**: no escala writes. Para eso
>   necesitás [[Sharding]].
> - **El failover no es instantáneo ni gratis**: promover una réplica lleva
>   segundos y puede perder las últimas escrituras no replicadas; mal hecho, da
>   *split-brain* (dos primarios).

## Patrones relacionados / alternativas

- [[Sharding]] — para escalar **escrituras** (Primary-Replica solo escala
  lecturas). Suelen combinarse: cada shard con su primario y réplicas.
- [[Quorum]] — alternativa para consistencia en lectura/escritura replicada.
- [[CQRS]] — las réplicas de lectura son una forma natural de separar read/write.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Sharding]]
- [[Quorum]]
- [[CQRS]]
