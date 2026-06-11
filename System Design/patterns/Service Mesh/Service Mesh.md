---
title: Service Mesh
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/infrastructure
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Service Mesh
  - Malla de servicios
updated: 2026-06-11
---

# Service Mesh

> [!note] Definition
> Una capa de infraestructura dedicada a la comunicación **servicio-a-servicio** (tráfico este-oeste). Tiene **dos partes**: el **data plane** (los proxies [[Sidecar]] junto a cada servicio, que aplican balanceo, reintentos, circuit breaking, [[Mutual TLS|mTLS]] y observabilidad) y el **control plane** (que configura todos los sidecars centralmente) — sin tocar el código del servicio. Ver [[Data Plane vs Control Plane]].

## Cómo funciona

Cada instancia corre con un sidecar (ej. Envoy) que intercepta todo el tráfico de entrada/salida. Un *control plane* (Istio, Linkerd) configura todos los sidecars centralmente. Así, [[Retry with Backoff]], [[Circuit Breaker]], mTLS y métricas viven en la malla, no replicados en cada servicio en cada lenguaje.

La división es clave (ver [[Data Plane vs Control Plane]]): el **control plane** (`istiod` en [[Istio]], el control plane de [[Linkerd]]) mantiene el registro de servicios e instancias y, al desplegar una política (ej. "todo el tráfico entre Order Service y Payment Service usa mTLS"), la empuja a cada sidecar relevante. El **data plane** la aplica en tiempo real: cada request fluye por los sidecars que cifran/enrutan/reintentan/miden sin que los servicios lo sepan.

## Qué hace que un [[API Gateway]] NO hace

- **[[Mutual TLS|mTLS]] en todas partes** — cifra automáticamente todo el tráfico interno; un cert por sidecar, rotación automática, zero-trust a nivel de infra.
- **Observabilidad distribuida** — métricas, logs y [[Distributed Tracing|distributed traces]] de cada llamada automáticamente, **sin una línea de instrumentación**.
- **[[Traffic Management|Traffic management]] interno** — canary ([[Canary Deployment]]) enviando 5% a la versión nueva, traffic mirroring copiando producción a staging, fault injection para chaos testing.
- **Enforcement language-agnostic** — Go/Java/Python/Rust, mismas políticas, sin librerías por lenguaje.

Un gateway opera en el edge (norte-sur); la mesh, en el interior (este-oeste). Ver [[North-South vs East-West Traffic]].

## Cuándo usarlo

> [!tip]
> En arquitecturas con **muchos microservicios** (decenas+) donde querés políticas de red, seguridad (mTLS) y observabilidad **uniformes** sin reimplementarlas en cada servicio y lenguaje.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Complejidad operativa alta**: el control plane y los sidecars son otra capa compleja que aprender, desplegar y debuggear. Muchos equipos la adoptan antes de necesitarla.
> - **Overhead de recursos y latencia**: un proxy extra por pod consume CPU/RAM y agrega un par de hops.
> - Para **pocos servicios**, librerías de resiliencia en la app + un [[API Gateway]] alcanzan, con mucha menos complejidad.

## Patrones relacionados / alternativas

- [[Sidecar]] — el mecanismo (data plane) que implementa la malla.
- [[Data Plane vs Control Plane]] — las dos partes de la mesh.
- [[API Gateway]] — maneja tráfico *norte-sur* (cliente↔sistema); la mesh maneja *este-oeste* (servicio↔servicio). Ver [[North-South vs East-West Traffic]].
- [[Mutual TLS]] · [[Service Discovery]] · [[Traffic Management]] — capacidades que la mesh provee transversalmente.
- [[Circuit Breaker]] / [[Retry with Backoff]] — la mesh los provee transversalmente.
- Implementaciones: [[Istio]] · [[Linkerd]] · [[Cilium]] · [[Envoy]] (data plane).

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Enriquecido con: [API Gateway vs Service Mesh vs Sidecar](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus) — split data/control plane y diferencias con el gateway.
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_Service Mesh|Service Mesh]]
- [[North-South vs East-West Traffic]]
- [[Sidecar]]
- [[Data Plane vs Control Plane]]
- [[API Gateway]]
