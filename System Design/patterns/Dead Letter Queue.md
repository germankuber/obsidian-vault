---
title: Dead Letter Queue
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/reliability
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Dead Letter Queue
  - dead-letter-queue
  - DLQ
updated: 2026-06-11
---

# Dead Letter Queue

> [!note] Definition
> Cuando un mensaje no se puede procesar tras varios reintentos, se mueve a una **cola aparte** (la *dead letter queue*) en vez de bloquear la cola principal o perderse silenciosamente.

## Cómo funciona

El consumidor intenta procesar; si falla N veces (envenenamiento: payload inválido, bug, dependencia caída), el broker reenruta el mensaje a la DLQ. La cola principal sigue fluyendo; los mensajes problemáticos quedan apartados para inspección, corrección y reproceso manual o automático.

## Cuándo usarlo

> [!tip]
> En cualquier sistema con [[Message Queue]] o [[Pub-Sub|Pub/Sub]]: garantiza que un mensaje "veneno" no frene a todos los que vienen detrás ni desaparezca sin dejar rastro. Es la red de seguridad del procesamiento asíncrono.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **La DLQ hay que monitorearla**: una DLQ que se llena y nadie mira es trabajo perdido en silencio. Necesita alertas.
> - **Decidir qué reintentar vs descartar**: algunos fallos son transitorios (reintentar), otros permanentes (a la DLQ ya). Mal calibrado, mandás a la DLQ cosas que se habrían recuperado, o reintentás eternamente veneno.
> - **Orden**: sacar un mensaje del medio rompe el orden estricto si tu sistema lo requería.

## Patrones relacionados / alternativas

- [[Message Queue]] / [[Pub-Sub|Pub/Sub]] — la DLQ es su complemento de confiabilidad.
- [[Retry with Backoff]] — los reintentos previos a mandar a la DLQ.
- [[Idempotency]] — necesaria al reprocesar desde la DLQ.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Message Queue]]
- [[Retry with Backoff]]
