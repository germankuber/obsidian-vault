---
title: Event Sourcing
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/storage
  - system-design/patterns
aliases:
  - Event Sourcing
  - event-sourcing
---

# Event Sourcing

> [!note] Definition
> En vez de guardar el **estado actual** de una entidad, guardar la **secuencia
> de eventos** que ocurrieron. El estado actual se deriva reproduciendo todos los
> eventos desde el principio.

## Cómo funciona

El *event store* es append-only: nunca se actualiza ni borra, solo se agregan
eventos (`CuentaAbierta`, `DineroDepositado`, `DineroRetirado`). Para obtener el
saldo, se reproducen los eventos. Para no reproducir todo cada vez, se guardan
*snapshots* periódicos. Da una auditoría perfecta y permite reconstruir el estado
a cualquier punto del pasado.

## Cuándo usarlo

> [!tip]
> Cuando la **historia importa tanto como el estado**: finanzas, auditoría,
> sistemas donde "¿cómo llegamos a este estado?" es una pregunta de negocio.
> Encaja naturalmente con [[CQRS]] (los eventos alimentan modelos de lectura) y
> con [[Event-Driven Architecture]].

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Complejidad alta**: reproducir eventos, snapshots, versionado de eventos
>   (¿qué pasa cuando cambia el schema de un evento viejo?). Es mucho más difícil
>   que un CRUD.
> - **Las queries son indirectas**: no podés hacer un `SELECT` simple del estado
>   actual; necesitás proyecciones/modelos de lectura ([[CQRS]]).
> - **Consistencia eventual** entre el event store y las proyecciones.
> - Para un CRUD común donde solo importa el estado actual, es **sobre-ingeniería**
>   clara.

## Patrones relacionados / alternativas

- [[CQRS]] — casi siempre va de la mano: los eventos construyen los modelos de
  lectura.
- [[Event-Driven Architecture]] — los eventos del store pueden publicarse.
- [[Write-Ahead Log]] — un log parecido, pero como detalle de recuperación, no
  como fuente de verdad del dominio.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[CQRS]]
- [[Event-Driven Architecture]]
