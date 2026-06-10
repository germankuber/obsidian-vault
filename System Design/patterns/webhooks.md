---
title: Webhooks
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/communication
  - system-design/patterns
aliases:
  - Webhooks
  - webhooks
  - HTTP Callbacks
---

# Webhooks

> [!note] Definition
> En vez de que el cliente haga *polling* preguntando "¿ya pasó?", el servidor
> **empuja** el evento a una URL que el cliente registró, cuando algo ocurre.

## Cómo funciona

El cliente registra una URL de callback. Cuando ocurre el evento, el servidor le
hace un `POST` con el payload. Es push de servidor a servidor sobre HTTP. Lo usan
Stripe, GitHub, etc., para avisar "pago confirmado", "push recibido". Es el
mecanismo de captura de cambios en ingestas (ver [[Change Data Capture]]).

## Cuándo usarlo

> [!tip]
> Cuando un sistema externo necesita enterarse de eventos **sin hacer polling**
> constante (que es lento y desperdicia recursos). Ideal para integraciones entre
> SaaS.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **El receptor debe estar accesible y disponible**: si su endpoint está caído,
>   el evento se pierde salvo reintentos. Necesita [[Retry with Backoff]] del lado
>   emisor y [[Idempotency]] del lado receptor (pueden llegar duplicados).
> - **Seguridad**: hay que firmar/verificar los payloads (HMAC) — una URL pública
>   es un vector de ataque.
> - **Sin garantía de orden** ni de entrega exacta.
> - Para streaming continuo hacia un browser, [[Server-Sent Events]] o WebSockets
>   encajan mejor.

## Patrones relacionados / alternativas

- [[Server-Sent Events]] / [[Bidirectional Streaming]] — push hacia clientes
  (browsers), no entre servidores.
- [[Change Data Capture]] — los webhooks suelen ser su disparador.
- [[Retry with Backoff]] / [[Idempotency]] — para entrega confiable.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Server-Sent Events]]
- [[Change Data Capture]]
