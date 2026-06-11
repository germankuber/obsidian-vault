---
title: Envoy
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/technology
  - status/permanent
aliases:
  - Envoy
  - Envoy Proxy
  - Envoy Gateway
---

# Envoy

> [!note] Definition
> El proxy de [[Sidecar|sidecar]] **más popular** para [[_Service Mesh|service meshes]]. Es el [[Data Plane vs Control Plane|data plane]] de facto: corre junto a cada servicio e intercepta todo su tráfico, aplicando discovery, load balancing, retries, circuit breaking, [[Mutual TLS|mTLS]] y telemetría.

## Dónde se usa

- **Data plane de [[Istio]]** — Istio usa Envoy como su sidecar.
- **AWS App Mesh** y **muchos meshes custom** lo usan por debajo.
- **Envoy Gateway** — soporta tanto norte-sur (edge) como este-oeste, alineado con la [[Kubernetes Gateway API]].

## Cuándo elegirlo / trade-offs

> [!tip]
> Es la opción por defecto cuando querés el proxy más probado y con más features. Si adoptás [[Istio]], ya estás usando Envoy.

> [!warning]
> - Más **pesado** que alternativas como linkerd2-proxy: si buscás overhead mínimo, [[Linkerd]] usa un proxy más liviano.
> - Como todo sidecar, **consume CPU/RAM (~50-100MB por instancia)** y agrega **1-10 ms de latencia por hop** (ver [[Sidecar]]).

## Tecnologías relacionadas

- [[Istio]] — el mesh más adoptado, construido sobre Envoy.
- [[Linkerd]] — usa linkerd2-proxy, más liviano que Envoy.
- [[Cilium]] — alternativa basada en eBPF que evita el modelo de sidecar.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).

## Related

- [[Sidecar]]
- [[_Service Mesh|Service Mesh]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
