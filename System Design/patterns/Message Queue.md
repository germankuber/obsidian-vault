---
title: Message Queue
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/communication
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Message Queue
  - message-queue
  - Task Queue
updated: 2026-06-11
---

# Message Queue

> [!note] Definition
> El productor deja un mensaje en una cola; el consumidor lo procesa **a su ritmo**, sin que el productor espere. Cada mensaje lo procesa **un** consumidor.

## Cómo funciona

La cola desacopla productor y consumidor en el tiempo: el productor encola y sigue; uno o más workers consumen cuando pueden. Absorbe picos (buffer), permite escalar consumidores horizontalmente y reintentar trabajo fallido. Entrega típica *at-least-once* → los consumidores deben ser [[Idempotency|idempotentes]].

## Cuándo usarlo

> [!tip]
> Para **trabajo asíncrono** que no necesita respuesta inmediata: enviar emails, procesar imágenes, generar reportes. También para **amortiguar** un productor rápido frente a un consumidor lento.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Consistencia eventual**: el trabajo se hace "después"; si el usuario espera el resultado ya, no encaja.
> - **Entrega at-least-once** ⇒ duplicados posibles ⇒ exige [[Idempotency]].
> - **Más infraestructura**: el broker (Kafka, RabbitMQ, SQS) es otra pieza que operar y monitorear; mensajes que no se procesan necesitan [[Dead Letter Queue]].
> - El **orden** no siempre está garantizado entre particiones.

## Patrones relacionados / alternativas

- [[Pub-Sub|Pub/Sub]] — cuando cada mensaje debe ir a **varios** consumidores, no a uno.
- [[Dead Letter Queue]] — para los mensajes que fallan repetidamente.
- [[Idempotency]] — necesaria por la entrega at-least-once.
- [[Event-Driven Architecture]] — las colas son su columna vertebral.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Pub-Sub|Pub/Sub]]
- [[Dead Letter Queue]]
- [[Idempotency]]
