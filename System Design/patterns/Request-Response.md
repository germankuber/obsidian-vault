---
title: Request-Response
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/communication
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Request-Response
  - request-response
---

# Request-Response

> [!note] Definition
> El cliente manda un request y **espera** la respuesta antes de seguir. Es el
> patrón de comunicación más simple — el de REST y gRPC unario.

## Cómo funciona

Comunicación síncrona y bloqueante (desde la perspectiva lógica): un request, una
respuesta, correlacionados uno a uno. El cliente conoce el contrato y sabe qué
esperar de vuelta.

## Cuándo usarlo

> [!tip]
> El **default** cuando el cliente necesita el resultado para continuar: leer un
> dato, validar, ejecutar un comando y confirmar. Simple de razonar, fácil de
> debuggear.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Acopla temporalmente**: el cliente queda esperando; si el servidor está
>   lento o caído, el cliente sufre (de ahí [[Timeout]] y [[Circuit Breaker]]).
> - **No sirve para trabajo lento o desacoplado**: subir un video, procesar un
>   batch. Para eso, [[Message Queue]].
> - **No escala a fan-out**: un evento que interesa a muchos consumidores no
>   encaja — ahí va [[Pub-Sub|Pub/Sub]].

## Patrones relacionados / alternativas

- [[Message Queue]] — para trabajo asíncrono donde el cliente no espera.
- [[Pub-Sub|Pub/Sub]] — para un emisor y muchos receptores.
- [[Timeout]] / [[Circuit Breaker]] — protegen al cliente que espera.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Message Queue]]
- [[Pub-Sub|Pub/Sub]]
