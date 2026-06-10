---
title: Reverse Proxy
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/infrastructure
  - system-design/patterns
aliases:
  - Reverse Proxy
  - reverse-proxy
---

# Reverse Proxy

> [!note] Definition
> Un servidor que se sienta **entre los clientes y los backends**, manejando
> terminación SSL, compresión, caché y ruteo. El cliente le habla a él, no al
> backend.

## Cómo funciona

A diferencia de un *forward proxy* (que representa al cliente), el reverse proxy
representa al servidor: recibe el tráfico entrante y lo reenvía al backend
apropiado, ocultando la topología interna. Nginx, HAProxy, Envoy son ejemplos.
Concentra concerns de red (TLS, gzip, caché de respuestas, [[Load Balancing]]).

## Cuándo usarlo

> [!tip]
> Casi siempre frente a servicios web: para terminar TLS en un solo lugar, cachear
> respuestas, comprimir, y rutear/balancear. Es la base sobre la que se construye
> un [[API Gateway]] o un [[Content Delivery Network]].

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Salto y posible cuello de botella**: agrega latencia y, si no es redundante,
>   un punto único de falla.
> - **Otra pieza para configurar y operar** (reglas de ruteo, certificados,
>   timeouts).
> - Para una app simple sin necesidad de TLS-offload, caché ni balanceo, agrega
>   complejidad sin beneficio.

## Patrones relacionados / alternativas

- [[API Gateway]] — un reverse proxy con features de API (auth, rate limiting).
- [[Load Balancing]] — función típica de un reverse proxy.
- [[Content Delivery Network]] — reverse-proxy-caché distribuido globalmente.
- [[Sidecar]] — en un [[Service Mesh]], el sidecar hace de proxy local.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[API Gateway]]
- [[Load Balancing]]
- [[Service Mesh]]
