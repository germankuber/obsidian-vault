---
title: Sidecar
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/infrastructure
  - system-design/patterns
aliases:
  - Sidecar
  - sidecar
---

# Sidecar

> [!note] Definition
> Desplegar un **proceso ayudante junto al servicio principal** (en el mismo pod/
> grupo de contenedores) que maneja *cross-cutting concerns* sin modificar el
> servicio.

## Cómo funciona

El sidecar comparte el ciclo de vida y la red local del servicio principal, pero
es un proceso aparte. Se encarga de cosas transversales —proxy de red, logging,
recolección de métricas, sincronización de config, certificados— de forma que el
servicio se concentre solo en su lógica de negocio. Es el bloque de construcción
de un [[Service Mesh]].

## Cuándo usarlo

> [!tip]
> Cuando querés agregar capacidades (observabilidad, seguridad, proxy) a un
> servicio **sin tocar su código**, o de forma **uniforme** a servicios escritos
> en distintos lenguajes. Ideal en Kubernetes.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Recursos por instancia**: cada servicio carga su sidecar → más CPU/RAM y un
>   hop de red local extra. A gran escala, suma.
> - **Otra pieza en el ciclo de vida**: hay que versionar, actualizar y
>   monitorear el sidecar junto al servicio.
> - Para una funcionalidad que es **propia** del servicio (no transversal), meterla
>   en un sidecar es una indirección innecesaria — va en el servicio.

## Patrones relacionados / alternativas

- [[Service Mesh]] — una flota de sidecars coordinados por un control plane.
- [[Reverse Proxy]] — el sidecar suele actuar como proxy local del servicio.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Service Mesh]]
- [[Reverse Proxy]]
