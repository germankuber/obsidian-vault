---
title: Linkerd
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/technology
  - status/permanent
aliases:
  - Linkerd
  - linkerd2-proxy
---

# Linkerd

> [!note] Definition
> Una [[_Service Mesh|Service Mesh]] **liviana**, alternativa a [[Istio]], con foco en **simplicidad y performance**. Usa su propio sidecar, **linkerd2-proxy**, más liviano que [[Envoy]].

## Cuándo elegirlo / trade-offs

> [!tip]
> Es la recomendación para el rango **10-30 servicios, tráfico moderado**: aporta [[Mutual TLS|mTLS]] y observabilidad **con menos overhead y complejidad que [[Istio]]**. Cuando querés los beneficios de un mesh sin pagar el costo operativo de Istio.

> [!warning]
> - Menos features y ecosistema que [[Istio]]: para casos muy grandes o necesidades muy específicas, Istio o [[Cilium]] pueden encajar mejor.
> - Sigue siendo un mesh basado en [[Sidecar|sidecars]]: hay overhead por instancia (menor que Envoy, pero no cero). Para near-zero, [[Cilium]] (eBPF).

## Tecnologías relacionadas

- [[Istio]] — más potente y completo, pero más complejo.
- [[Cilium]] — basado en eBPF, sin sidecars.
- [[Envoy]] — el proxy que Linkerd evita usar a favor de uno propio más liviano.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).

## Related

- [[_Service Mesh|Service Mesh]]
- [[Istio]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
