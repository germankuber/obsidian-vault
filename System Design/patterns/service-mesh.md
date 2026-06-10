---
title: Service Mesh
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/infrastructure
  - system-design/patterns
aliases:
  - Service Mesh
  - service-mesh
---

# Service Mesh

> [!note] Definition
> Una capa de infraestructura dedicada a la comunicación **servicio-a-servicio**,
> donde un proxy [[Sidecar]] junto a cada servicio maneja balanceo, reintentos,
> circuit breaking, mTLS y observabilidad — sin tocar el código del servicio.

## Cómo funciona

Cada instancia corre con un sidecar (ej. Envoy) que intercepta todo el tráfico de
entrada/salida. Un *control plane* (Istio, Linkerd) configura todos los sidecars
centralmente. Así, [[Retry with Backoff]], [[Circuit Breaker]], mTLS y métricas
viven en la malla, no replicados en cada servicio en cada lenguaje.

## Cuándo usarlo

> [!tip]
> En arquitecturas con **muchos microservicios** (decenas+) donde querés políticas
> de red, seguridad (mTLS) y observabilidad **uniformes** sin reimplementarlas en
> cada servicio y lenguaje.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Complejidad operativa alta**: el control plane y los sidecars son otra capa
>   compleja que aprender, desplegar y debuggear. Muchos equipos la adoptan antes
>   de necesitarla.
> - **Overhead de recursos y latencia**: un proxy extra por pod consume CPU/RAM y
>   agrega un par de hops.
> - Para **pocos servicios**, librerías de resiliencia en la app + un
>   [[API Gateway]] alcanzan, con mucha menos complejidad.

## Patrones relacionados / alternativas

- [[Sidecar]] — el mecanismo que implementa la malla.
- [[API Gateway]] — maneja tráfico *norte-sur* (cliente↔sistema); la mesh maneja
  *este-oeste* (servicio↔servicio).
- [[Circuit Breaker]] / [[Retry with Backoff]] — la mesh los provee
  transversalmente.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Sidecar]]
- [[API Gateway]]
