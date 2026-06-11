---
title: Data Plane vs Control Plane
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/concept
  - status/permanent
aliases:
  - Data Plane
  - Control Plane
  - Plano de datos vs Plano de control
---

# Data Plane vs Control Plane

> [!note] Definition
> Las **dos partes** de una [[_Service Mesh|Service Mesh]]. El **data plane** son los [[Sidecar|sidecars]] junto a cada servicio: el código que toca cada request (cifra, enruta, reintenta, mide). El **control plane** es el componente central que **configura y gestiona** todos los sidecars: no toca el tráfico, decide las políticas que el data plane aplica.

## Cómo funciona

- **Control plane** (istiod de [[Istio]], control plane de [[Linkerd]]): mantiene un registro de todos los servicios y sus instancias. Cuando desplegás una política (ej. "todo el tráfico entre Order Service y Payment Service debe usar [[Mutual TLS|mTLS]]"), empuja esa config a **cada sidecar relevante**.
- **Data plane** (los sidecars, típicamente [[Envoy]]): aplica las políticas **en tiempo real**. Cada request entre servicios fluye por los sidecars, que aplican encryption / routing / retries / circuit-breaking / telemetría **sin que los servicios lo sepan**.

> [!example] El reparto en una frase
> El control plane **decide** (políticas, registro de servicios, distribución de certificados). El data plane **ejecuta** (intercepta cada paquete y aplica la política).

## Por qué importa la separación

- El **enforcement es distribuido** (un sidecar por instancia, cerca del tráfico) pero la **gestión es centralizada** (una sola fuente de verdad de las políticas).
- Cambiás una política una vez en el control plane → se propaga a 1.000 sidecars sin tocar código ni redeployar servicios.
- Es el mismo patrón mental que en redes (SDN) y en bases distribuidas: separar el plano que mueve los datos del plano que toma decisiones.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - El **control plane es una pieza más que operar**: desplegar, versionar y debuggear (es parte del overhead que hace que un mesh no valga la pena con < 10 servicios).
> - Si el control plane cae, los sidecars suelen seguir con la **última config conocida** (degradan, no se caen), pero pierden la capacidad de reconfigurarse y de descubrir servicios nuevos.
> - Más componentes = más superficie de falla y de aprendizaje.

## Conceptos relacionados

- [[Sidecar]] — la unidad del data plane.
- [[_Service Mesh|Service Mesh]] — el todo que compone ambos planos.
- [[Istio]] / [[Linkerd]] — implementaciones, cada una con su control plane.
- [[Envoy]] — el proxy más usado como data plane.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).

## Related

- [[_Service Mesh|Service Mesh]]
- [[Sidecar]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
