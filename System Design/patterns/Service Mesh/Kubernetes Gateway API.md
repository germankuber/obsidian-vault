---
title: Kubernetes Gateway API
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/concept
  - status/permanent
aliases:
  - Kubernetes Gateway API
  - Gateway API
---

# Kubernetes Gateway API

> [!note] Definition
> Un estándar emergente **vendor-neutral** para definir traffic routing **norte-sur y este-oeste** en Kubernetes. Reemplaza el viejo recurso **Ingress** con una semántica de routing más rica que sirve tanto para el edge como para el tráfico interno.

## Por qué importa

- El recurso **Ingress** clásico solo cubría norte-sur (entrada) y de forma limitada. La Gateway API unifica **ambas direcciones** ([[North-South vs East-West Traffic]]) bajo una sola API.
- **[[Envoy]] Gateway** y el **gateway de [[Istio]]** la soportan, manejando edge e interno **desde un único control plane**.
- Es una de las patas de la **tendencia de convergencia**: empuja a que [[API Gateway]] y [[_Service Mesh|Service Mesh]] se fusionen en una sola capa de infra con políticas distintas por tipo de tráfico.

## Cuándo usarlo / trade-offs

> [!tip]
> Cuando querés una forma **estándar y portable** de declarar routing en Kubernetes, sin atarte a la API propietaria de un Ingress controller o un mesh específico.

> [!warning]
> - Es un estándar **en maduración**: el soporte y la cobertura de features varían entre implementaciones.
> - Convive con (y por ahora no elimina del todo) las APIs específicas de cada gateway/mesh.

## Conceptos relacionados

- [[API Gateway]] — el componente edge que esta API ayuda a configurar de forma estándar.
- [[_Service Mesh|Service Mesh]] — el tráfico interno que la API también cubre.
- [[North-South vs East-West Traffic]] — las dos direcciones que unifica.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).

## Related

- [[_Service Mesh|Service Mesh]]
- [[API Gateway]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
