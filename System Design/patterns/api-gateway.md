---
title: API Gateway
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/api
  - system-design/patterns
aliases:
  - API Gateway
  - api-gateway
---

# API Gateway

> [!note] Definition
> Un **punto de entrada único** que rutea, autentica, limita (rate-limit) y
> transforma los requests antes de reenviarlos a los servicios backend.

## Cómo funciona

Todos los clientes pegan al gateway, no a los servicios directamente. El gateway
concentra *cross-cutting concerns*: autenticación/autorización, [[Rate Limiting]],
ruteo, agregación de respuestas, terminación TLS, logging. Los servicios backend
se simplifican porque no repiten esa lógica.

## Cuándo usarlo

> [!tip]
> En arquitecturas de **microservicios**, para no exponer decenas de servicios al
> mundo ni duplicar auth/rate-limiting en cada uno. Da un frente único y
> consistente a los clientes.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Punto único de falla y cuello de botella**: todo el tráfico pasa por ahí →
>   hay que hacerlo redundante y escalable, o se vuelve el límite del sistema.
> - **Riesgo de "gateway gordo"**: si le metés lógica de negocio, se transforma en
>   un monolito encubierto. Debe quedarse en concerns transversales.
> - **Latencia extra**: un salto más en cada request.
> - Para un solo servicio o un monolito, es innecesario — un [[Reverse Proxy]]
>   simple alcanza.

## Patrones relacionados / alternativas

- [[Reverse Proxy]] — un gateway es un reverse proxy con features de API encima.
- [[Backend for Frontend]] — varios gateways especializados por tipo de cliente.
- [[Rate Limiting]] — una de sus funciones centrales.
- [[Load Balancing]] — suele combinarse con el ruteo del gateway.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Reverse Proxy]]
- [[Backend for Frontend]]
- [[Rate Limiting]]
