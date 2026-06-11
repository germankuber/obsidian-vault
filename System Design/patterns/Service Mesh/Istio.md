---
title: Istio
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/technology
  - status/permanent
aliases:
  - Istio
  - istiod
---

# Istio

> [!note] Definition
> La [[_Service Mesh|Service Mesh]] **más adoptada**. Usa [[Envoy]] como [[Data Plane vs Control Plane|data plane]] (un sidecar Envoy por servicio) y `istiod` como control plane. Muy potente, pero **compleja**.

## Cómo funciona

- **Control plane: `istiod`** — mantiene el registro de servicios e instancias y empuja las políticas a cada sidecar Envoy.
- **Data plane: sidecars [[Envoy]]** — aplican [[Mutual TLS|mTLS]], routing, retries, circuit breaking y telemetría por request.
- Su gateway soporta la [[Kubernetes Gateway API]], manejando edge e interno desde un único control plane.

## Cuándo elegirlo / trade-offs

> [!tip]
> Para arquitecturas **grandes (50+ servicios)**, alto tráfico y varios equipos, donde necesitás el set de features más completo y la mayor comunidad/ecosistema.

> [!warning]
> - **Potente pero complejo**: es el ejemplo canónico del overhead que hace que un mesh no valga la pena con pocos servicios. Para 10-30 servicios suele convenir [[Linkerd]] (más liviano y simple).
> - Hereda el overhead de los sidecars [[Envoy]] (~50-100MB y 1-10 ms por hop). Para overhead near-zero, mirá [[Cilium]] (eBPF).

## Tecnologías relacionadas

- [[Linkerd]] — alternativa liviana, foco en simplicidad y performance.
- [[Cilium]] — mesh basado en eBPF, sin sidecars.
- [[AWS App Mesh]] — mesh managed para AWS.
- [[Envoy]] — su data plane.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).

## Related

- [[_Service Mesh|Service Mesh]]
- [[Envoy]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
