---
title: Sidecar
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/infrastructure
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Sidecar
  - Sidecar Proxy
updated: 2026-06-11
---

# Sidecar

> [!note] Definition
> Un **proxy/proceso ayudante ligero que corre junto al servicio principal** (en el mismo pod de Kubernetes o el mismo host) y maneja *cross-cutting concerns* sin modificar el servicio. Todo el tráfico hacia/desde el servicio pasa por él, y **el servicio no sabe que existe**: envía requests a lo que cree que es el destino y el sidecar los **intercepta**.

## Cómo funciona

El sidecar comparte el ciclo de vida y la red local del servicio principal, pero es un proceso aparte. Se encarga de cosas transversales —proxy de red, logging, recolección de métricas, sincronización de config, certificados— de forma que el servicio se concentre solo en su lógica de negocio. Es el bloque de construcción de un [[Service Mesh]].

Mueve al nivel de infra los *networking concerns* que de otro modo irían en el código: [[Service Discovery|service discovery]], [[Load Balancing|load balancing]], retries/timeouts, [[Circuit Breaker|circuit breaking]], [[Mutual TLS|mutual TLS]] y telemetría (latencia, error rate, traces por llamada).

**El problema que resuelve**: sin sidecars, cada servicio implementa retry/circuit-breaking/TLS/discovery en su propio código → **una librería por lenguaje**. Con servicios en Go, Java, Python y Node.js = cuatro implementaciones de la misma lógica. Cambiar una timeout policy → actualizar cuatro librerías y redeployar cada servicio. Con el sidecar, el servicio solo hace HTTP/gRPC; cambiar la retry policy = **actualizar la config del sidecar, sin cambios de código ni redeployments**.

Implementaciones: [[Envoy]] (de lejos el más popular; usado por [[Istio]], [[AWS App Mesh]] y muchos meshes custom) · linkerd2-proxy (de [[Linkerd]], más liviano que Envoy).

## Cuándo usarlo

> [!tip]
> Cuando querés agregar capacidades (observabilidad, seguridad, proxy) a un servicio **sin tocar su código**, o de forma **uniforme** a servicios escritos en distintos lenguajes. Ideal en Kubernetes.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Recursos por instancia**: cada sidecar consume CPU/RAM, **típicamente 50-100MB por instancia**.
> - **Latencia por hop**: cada request atraviesa un hop de red adicional, añadiendo **1 a 10 milisegundos** de latencia. Una cadena de 5 servicios = 5-50 ms solo por el mesh.
> - **Escala de gestión**: en un sistema con **200 servicios × 5 réplicas = 1.000 instancias de sidecar** a gestionar.
> - **Mitigación**: los meshes basados en eBPF ([[Cilium]]) mueven la funcionalidad del proxy al kernel de Linux, reduciendo el overhead por servicio a near-zero.
> - **Otra pieza en el ciclo de vida**: hay que versionar, actualizar y monitorear el sidecar junto al servicio.
> - Para una funcionalidad que es **propia** del servicio (no transversal), meterla en un sidecar es una indirección innecesaria — va en el servicio.

## Patrones relacionados / alternativas

- [[Service Mesh]] — una flota de sidecars coordinados por un control plane.
- [[Data Plane vs Control Plane]] — el sidecar es la unidad del data plane.
- [[Envoy]] / [[Linkerd]] — implementaciones concretas de sidecar.
- [[Reverse Proxy]] — el sidecar suele actuar como proxy local del servicio.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Enriquecido con: [API Gateway vs Service Mesh vs Sidecar](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus) — interceptación de tráfico, problema de "una librería por lenguaje" y números de overhead.
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_Service Mesh|Service Mesh]]
- [[Service Mesh]]
- [[Data Plane vs Control Plane]]
- [[Reverse Proxy]]
