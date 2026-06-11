---
title: Event-Driven Architecture
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/communication
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - Event-Driven Architecture
  - event-driven-architecture
  - EDA
updated: 2026-06-11
---

# Event-Driven Architecture

> [!note] Definition
> Los servicios se comunican **emitiendo y reaccionando a eventos** en vez de hacerse llamadas directas. Un servicio publica "pasó X"; otros reaccionan sin que el emisor sepa quiénes son.

## Cómo funciona

En lugar de A llama a B llama a C (orquestación acoplada), A emite un evento y B, C reaccionan de forma independiente (coreografía). El transporte suele ser [[Pub-Sub|Pub/Sub]] o [[Message Queue]]. Cada servicio es autónomo: se agregan o quitan reactores sin tocar al emisor.

## Cuándo usarlo

> [!tip]
> Cuando querés **desacoplar** servicios y que el sistema sea extensible: agregar una nueva reacción a un evento existente sin modificar el productor. Encaja con microservicios, [[Event Sourcing]] y flujos asíncronos.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **El flujo se vuelve implícito**: no hay un lugar donde leas "qué pasa cuando se crea una orden" — está repartido entre reactores. Debuggear el camino end-to-end es difícil (mitigás con [[Distributed Tracing]]).
> - **Consistencia eventual** en todo el sistema.
> - **Coreografía vs orquestación**: flujos complejos con muchos pasos pueden ser más claros con una [[Saga]] orquestada que con una maraña de eventos.
> - Para un flujo simple y síncrono, [[Request-Response]] es más directo.

## Patrones relacionados / alternativas

- [[Pub-Sub|Pub/Sub]] / [[Message Queue]] — el transporte de los eventos.
- [[Event Sourcing]] — fuente natural de eventos de dominio.
- [[Saga]] — orquestación de flujos multi-paso sobre eventos.
- [[Distributed Tracing]] — para recuperar visibilidad del flujo.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Pub-Sub|Pub/Sub]]
- [[Event Sourcing]]
- [[Saga]]
