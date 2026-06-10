---
title: Pub/Sub
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/communication
  - system-design/patterns
aliases:
  - Pub/Sub
  - pub-sub
  - Publish-Subscribe
  - PubSub
---

# Pub/Sub

> [!note] Definition
> El *publisher* emite a un **topic**; **varios** subscribers reciben cada
> mensaje. A diferencia de una [[Message Queue]] (donde cada mensaje va a un solo
> consumidor), acá cada subscriber recibe su copia.

## Cómo funciona

El emisor no conoce a los receptores — publica a un topic y el broker se encarga
del fan-out a todos los suscriptos. Desacopla emisor y receptores en identidad y
en número: agregás consumidores sin tocar al productor.

## Cuándo usarlo

> [!tip]
> Cuando un mismo evento le interesa a **múltiples** consumidores independientes:
> "se creó una orden" → facturación, inventario, notificaciones, analytics, todos
> reaccionan. Es la base de [[Event-Driven Architecture]].

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **No sabés quién consumió qué** ni si todos procesaron — el desacople dificulta
>   el rastreo y el debugging del flujo de extremo a extremo.
> - **Entrega y orden**: garantías variables según el broker; puede haber
>   duplicados ⇒ [[Idempotency]].
> - **Acoplamiento oculto por eventos**: muchos suscriptores a un evento crean
>   dependencias difíciles de ver. Cambiar el schema del evento rompe consumidores
>   silenciosamente.
> - Si solo un consumidor procesa cada mensaje, usá [[Message Queue]].

## Patrones relacionados / alternativas

- [[Message Queue]] — uno-a-uno en vez de uno-a-muchos.
- [[Event-Driven Architecture]] — Pub/Sub es su mecanismo de transporte típico.
- [[Webhooks]] — fan-out hacia sistemas externos vía HTTP.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Message Queue]]
- [[Event-Driven Architecture]]
