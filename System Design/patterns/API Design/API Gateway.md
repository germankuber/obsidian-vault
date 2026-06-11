---
title: API Gateway
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/api
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - API Gateway
  - api-gateway
updated: 2026-06-11
---

# API Gateway

> [!note] Definition
> Un **punto de entrada único** que rutea, autentica, limita (rate-limit) y transforma los requests antes de reenviarlos a los servicios backend.

## Cómo funciona

Todos los clientes pegan al gateway, no a los servicios directamente. El gateway concentra *cross-cutting concerns*: autenticación/autorización, [[Rate Limiting]], ruteo, agregación de respuestas, terminación TLS, logging. Los servicios backend se simplifican porque no repiten esa lógica.

El gateway vive en el **edge** y maneja el tráfico **norte-sur** (clientes externos ↔ sistema; ver [[North-South vs East-West Traffic]]). Concretamente, en el edge: valida auth (API keys, OAuth2 tokens, **claims de JWT**) antes de que el request llegue al servicio; aplica rate limiting por cliente o API key (ej. "1.000 requests/hora por developer"; "tier premium 10×"); termina **TLS/HTTPS**; cachea respuestas frecuentes; y hace [[API Versioning|versioning]] (`/v1/users` vs `/v2/users` → backends distintos, transparente al cliente) y transformación de formato público↔interno.

Implementaciones: [[Kong]], AWS API Gateway, Azure API Management, NGINX, [[Envoy]] Gateway. Kong/NGINX son open-source con tier enterprise; AWS/Azure son fully managed.

## Cuándo usarlo

> [!tip]
> En arquitecturas de **microservicios**, para no exponer decenas de servicios al mundo ni duplicar auth/rate-limiting en cada uno. Da un frente único y consistente a los clientes.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Punto único de falla y cuello de botella**: todo el tráfico pasa por ahí → hay que hacerlo redundante y escalable, o se vuelve el límite del sistema.
> - **Riesgo de "gateway gordo"**: si le metés lógica de negocio (agregar datos de varios servicios, transformar business objects, workflow logic), se transforma en un monolito encubierto. El gateway solo enruta/autentica/rate-limita; esa lógica va en los servicios o en una capa [[Backend for Frontend|BFF]].
> - **Latencia extra**: un salto más en cada request.
> - Para un solo servicio o un monolito, es innecesario — un [[Reverse Proxy]] simple alcanza.

## Patrones relacionados / alternativas

- [[Reverse Proxy]] — un gateway es un reverse proxy con features de API encima.
- [[Backend for Frontend]] — varios gateways especializados por tipo de cliente.
- [[Rate Limiting]] — una de sus funciones centrales.
- [[Load Balancing]] — suele combinarse con el ruteo del gateway.
- [[_Service Mesh|Service Mesh]] — la contraparte para el tráfico *este-oeste* (servicio↔servicio); gateway y mesh **se complementan, no compiten**. Ver [[North-South vs East-West Traffic]].

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Enriquecido con: [API Gateway vs Service Mesh vs Sidecar](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus) — framing norte-sur/edge/JWT/TLS/versioning y la relación con la mesh.
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Reverse Proxy]]
- [[Backend for Frontend]]
- [[Rate Limiting]]
- [[North-South vs East-West Traffic]]
- [[_Service Mesh|Service Mesh]]
