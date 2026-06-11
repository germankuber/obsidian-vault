---
title: Traffic Management
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/concept
  - status/permanent
aliases:
  - Traffic Management
  - Gestión de tráfico
  - Traffic Mirroring
  - Fault Injection
---

# Traffic Management

> [!note] Definition
> El control fino de **cómo fluye el tráfico interno** entre servicios, aplicado por la [[_Service Mesh|Service Mesh]] vía [[Sidecar|sidecars]] sin tocar el código. Permite enviar porcentajes de tráfico a versiones distintas, copiar tráfico a otro lado, o inyectar fallas a propósito — todo configurable desde el [[Data Plane vs Control Plane|control plane]].

## Capacidades

- **Canary deployments** — enviar un % chico (ej. 5%) a la versión nueva y subir gradualmente según métricas. La mesh hace el split de tráfico; ver el patrón completo en [[Canary Deployment]].
- **Traffic mirroring** (shadowing) — **copiar** el tráfico de producción y mandar la copia a staging, sin que el usuario lo note. Probás la versión nueva con carga real sin riesgo, porque la respuesta del mirror se descarta.
- **Fault injection** — inyectar **fallas o latencia a propósito** (chaos testing) para verificar que el sistema resiste: ¿el circuit breaker abre?, ¿el retry funciona?, ¿la degradación es elegante?

> [!tip] Esto es lo que un [[API Gateway]] NO hace
> El traffic management **interno fino** (split por %, mirroring, fault injection entre microservicios) es de la mesh. Un gateway hace ruteo en el edge, no este control granular este-oeste.

## Cuándo usarlo

> [!tip]
> Cuando tenés suficientes servicios y despliegues frecuentes como para necesitar **releases progresivos seguros** ([[Canary Deployment|canary]]), validar con tráfico real ([[Traffic Management|mirroring]]) o probar resiliencia activamente (fault injection). Requiere una mesh ya desplegada.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - Estas capacidades **presuponen un mesh** (y su overhead); no las adoptes solo por el canary si tenés < 10 servicios.
> - El **canary necesita métricas confiables** para decidir avanzar o revertir (ver trade-offs en [[Canary Deployment]]).
> - El **mirroring** puede duplicar efectos secundarios si el sistema espejo escribe en recursos reales: hay que aislar staging.

## Conceptos relacionados

- [[Canary Deployment]] — el release progresivo que la mesh implementa con su split de tráfico.
- [[_Service Mesh|Service Mesh]] — provee estas capacidades transversalmente.
- [[Distributed Tracing]] / [[Health Check]] — las señales para decidir en un canary.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).

## Related

- [[_Service Mesh|Service Mesh]]
- [[Canary Deployment]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
