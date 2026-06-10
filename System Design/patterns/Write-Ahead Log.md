---
title: Write-Ahead Log
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/storage
  - system-design/patterns
aliases:
  - Write-Ahead Log
  - write-ahead-log
  - WAL
---

# Write-Ahead Log

> [!note] Definition
> Toda operación se escribe primero en un **log secuencial** en disco, *antes* de
> aplicarse al almacenamiento principal. Si el sistema crashea, se reproduce el
> log para recuperar las operaciones incompletas.

## Cómo funciona

La regla es "log antes de aplicar": el cambio se persiste en el WAL (append-only,
secuencial → rápido) y recién después se modifica la estructura de datos real. En
el arranque tras un crash, el sistema reproduce el WAL desde el último
*checkpoint* y queda consistente. Es la base de la durabilidad (la "D" de ACID)
en casi toda base de datos.

## Cuándo usarlo

> [!tip]
> Cuando necesitás **durabilidad y recuperación ante crash**: bases de datos,
> sistemas de archivos, almacenes de eventos. También es lo que habilita la
> replicación — las réplicas de [[Primary-Replica]] consumen el WAL del primario.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Doble escritura**: cada cambio se escribe dos veces (log + datos), lo que
>   cuesta IO. El fsync del log es además un punto de latencia.
> - **El log crece**: requiere *checkpointing* y truncado periódico, o llena el
>   disco.
> - Es un patrón de **infraestructura de almacenamiento**, no algo que
>   implementes a nivel app salvo que estés construyendo un motor de datos.

## Patrones relacionados / alternativas

- [[Event Sourcing]] — relacionado pero distinto: el WAL es un detalle interno de
  recuperación; Event Sourcing hace del log la fuente de verdad del *dominio*.
- [[Primary-Replica]] — la replicación se alimenta del WAL.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Event Sourcing]]
- [[Primary-Replica]]
