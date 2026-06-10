---
title: Server-Sent Events
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/communication
  - system-design/patterns
aliases:
  - Server-Sent Events
  - server-sent-events
  - SSE
---

# Server-Sent Events

> [!note] Definition
> Push **unidireccional** del servidor al cliente sobre una conexión HTTP de larga
> duración: el servidor envía eventos a medida que ocurren, sin que el cliente los
> pida.

## Cómo funciona

El cliente abre una conexión HTTP que queda abierta; el servidor va escribiendo
eventos en ese stream (`text/event-stream`). El navegador los recibe vía
`EventSource`. Reconexión automática y simple. Es el transporte típico para
**streamear la respuesta token-a-token de un LLM** (ver [[Enterprise RAG Assistant]]).

## Cuándo usarlo

> [!tip]
> Para flujos **servidor→cliente** donde el cliente solo recibe: streaming de
> respuestas de IA, feeds en vivo, notificaciones, barras de progreso. Más simple
> que WebSockets cuando no necesitás que el cliente también empuje.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Unidireccional**: el cliente no puede mandar por el mismo canal. Si necesitás
>   ida y vuelta en tiempo real (chat, juegos, colaboración), usá
>   [[Bidirectional Streaming]] (WebSockets).
> - **Una conexión abierta por cliente**: a gran escala consume sockets/recursos
>   del servidor.
> - **Solo texto** (UTF-8); para binario, WebSockets.
> - Proxies/balanceadores mal configurados pueden bufferear o cortar conexiones
>   largas.

## Patrones relacionados / alternativas

- [[Bidirectional Streaming]] — WebSockets, cuando hace falta ida y vuelta.
- [[Webhooks]] — push pero servidor-a-servidor, no a un browser.
- [[Enterprise RAG Assistant]] — usa SSE para streamear la generación.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Bidirectional Streaming]]
- [[Webhooks]]
- [[Enterprise RAG Assistant]]
