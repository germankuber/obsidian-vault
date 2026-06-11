---
title: Saga
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/consistency
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Saga
  - saga
  - Saga Pattern
updated: 2026-06-11
---

# Saga

> [!note] Definition
> Una secuencia de **transacciones locales**: cada servicio hace su transacción y publica un evento. Si un paso falla, se ejecutan **transacciones compensatorias** que deshacen los pasos anteriores. Da consistencia sin un commit atómico global.

## Cómo funciona

En vez de una transacción distribuida ([[Two-Phase Commit]]), cada servicio commitea localmente y avisa. Dos estilos:
- **Coreografía**: cada servicio escucha eventos y reacciona (descentralizado).
- **Orquestación**: un orquestador central dirige los pasos y las compensaciones.

Si "reservar pago" falla tras "reservar stock", la saga ejecuta "liberar stock" (compensación). No hay rollback global: hay *undo* explícito por paso.

## Cuándo usarlo

> [!tip]
> Para transacciones de negocio que **cruzan varios microservicios** (reserva de viaje: vuelo + hotel + auto) donde 2PC sería demasiado bloqueante. Prioriza disponibilidad y desacople.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Consistencia eventual, no atomicidad**: hay una ventana donde el sistema está parcialmente aplicado. La UI debe tolerarlo.
> - **Compensaciones difíciles de diseñar**: no todo se puede "deshacer" limpiamente (un email ya enviado). Hay que pensar cada *undo*.
> - **Complejidad de flujo**: la coreografía se vuelve una maraña de eventos; la orquestación centraliza pero agrega un componente.
> - Requiere [[Idempotency]] (eventos pueden repetirse).

## Patrones relacionados / alternativas

- [[Two-Phase Commit]] — atomicidad fuerte pero bloqueante; la saga es el trade-off opuesto.
- [[Event-Driven Architecture]] — base de la saga coreografiada.
- [[Idempotency]] — necesaria para procesar eventos repetidos.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Two-Phase Commit]]
- [[Event-Driven Architecture]]
- [[Idempotency]]
