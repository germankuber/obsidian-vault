---
title: Bidirectional Streaming
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/communication
  - system-design/patterns
aliases:
  - Bidirectional Streaming
  - bidirectional-streaming
  - WebSockets
  - WebSocket
---

# Bidirectional Streaming

> [!note] Definition
> Una conexión **persistente** donde cliente y servidor pueden mandar mensajes en
> cualquier momento, en ambas direcciones. La implementación típica es WebSockets
> (sobre HTTP) o gRPC bidireccional.

## Cómo funciona

Tras un *handshake* que "upgradea" la conexión HTTP, queda un canal full-duplex
abierto: ambos extremos empujan mensajes sin pedir turno. Baja latencia, ideal
para tiempo real.

## Cuándo usarlo

> [!tip]
> Cuando ambos lados necesitan empujar en tiempo real: **chat**, juegos
> multijugador, edición colaborativa, dashboards de trading, señalización. Cuando
> el cliente también tiene que enviar, no solo recibir.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Más complejo y caro que [[Server-Sent Events]]**: si el cliente solo
>   *recibe*, WebSockets es overkill — SSE es más simple y reconecta solo.
> - **Conexiones con estado**: cada conexión abierta consume recursos del
>   servidor; escalar a millones requiere infraestructura específica (sticky
>   sessions, gateways de WS).
> - **No es request/response**: perdés el modelo simple de HTTP (caching,
>   reintentos, status codes). Manejo de reconexión y backpressure a tu cargo.
> - Proxies/firewalls a veces no juegan bien con conexiones largas.

## Patrones relacionados / alternativas

- [[Server-Sent Events]] — si solo necesitás servidor→cliente, mucho más simple.
- [[Request-Response]] — para interacciones puntuales, sin conexión persistente.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Server-Sent Events]]
- [[Request-Response]]
