---
title: Distributed Tracing
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/observability
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Distributed Tracing
  - distributed-tracing
updated: 2026-06-11
---

# Distributed Tracing

> [!note] Definition
> Seguir un request mientras fluye por **múltiples servicios**: cada servicio agrega un *span* con timestamp a un *trace* compartido, mostrando el camino, la latencia en cada paso y dónde está el cuello de botella.

## Cómo funciona

Al entrar el request se genera un **trace ID** que se propaga por cada llamada (en headers). Cada servicio crea *spans* (su tramo de trabajo) anidados, con inicio/fin y metadata. Un colector (Jaeger, Zipkin, OpenTelemetry) ensambla los spans en un árbol que reconstruye el viaje completo del request.

En una [[_Service Mesh|Service Mesh]], los traces (y métricas/logs) se emiten **automáticamente por cada [[Sidecar|sidecar]], sin una sola línea de instrumentación** en el código del servicio: el sidecar intercepta cada llamada y registra el span. Es una de las cosas que un [[API Gateway]] no da y la mesh sí.

## Cuándo usarlo

> [!tip]
> En arquitecturas con **muchos servicios** ([[Event-Driven Architecture]], microservicios), donde un request toca decenas de componentes y la pregunta "¿por qué esto tardó 3 segundos?" no se responde mirando un solo log.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Instrumentación en todos lados**: el trace ID debe propagarse en cada servicio y cada llamada; un eslabón sin instrumentar rompe la cadena.
> - **Overhead y costo**: generar y almacenar spans cuesta; por eso se usa *sampling* (no se trazan todos los requests), que puede perder justo el que te interesa.
> - **No reemplaza logs ni métricas**: es una de las tres patas de la observabilidad, no todas.
> - Para un monolito o pocos servicios, logs + métricas alcanzan.

## Patrones relacionados / alternativas

- [[Health Check]] — observabilidad binaria (vivo/no); el tracing es de grano fino.
- [[Event-Driven Architecture]] — donde más se necesita por el flujo implícito.
- [[_Service Mesh|Service Mesh]] — provee observabilidad distribuida (traces/métricas/logs) automáticamente vía sidecars.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Enriquecido con: [API Gateway vs Service Mesh vs Sidecar](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus) — la mesh emite traces por sidecar sin código.
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Health Check]]
- [[Event-Driven Architecture]]
- [[_Service Mesh|Service Mesh]]
