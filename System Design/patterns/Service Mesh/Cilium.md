---
title: Cilium
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/technology
  - status/permanent
aliases:
  - Cilium
---

# Cilium

> [!note] Definition
> Una [[_Service Mesh|Service Mesh]] basada en **[[eBPF]]** que **elimina el modelo de [[Sidecar|sidecar]]**: mueve la funcionalidad del proxy al **kernel de Linux** en vez de correr un proceso proxy por servicio.

## Por qué importa

- El modelo de sidecar cuesta **megabytes de memoria (~50-100MB por instancia)** y **milisegundos de latencia (1-10 ms por hop)**. Cilium baja ese overhead por servicio a **near-zero** al hacer el trabajo en el kernel con [[eBPF]].
- A medida que esta tecnología madura, el **argumento del coste operacional contra los meshes se debilita**: es parte de la tendencia de convergencia que podría fusionar gateway y mesh en una sola capa de infra.

## Cuándo elegirlo / trade-offs

> [!tip]
> Para sistemas **grandes y latency/resource-sensitive** donde el overhead acumulado de los sidecars duele (ej. 1.000 sidecars = 200 servicios × 5 réplicas). El enfoque eBPF lo evita.

> [!warning]
> - **Más nuevo y menos maduro** que [[Istio]]/[[Linkerd]]; depende de capacidades del kernel ([[eBPF]]) y de un perfil operativo distinto.
> - El modelo mental (kernel-level, no proxy por pod) es diferente y puede requerir más expertise de plataforma.

## Tecnologías relacionadas

- [[eBPF]] — la tecnología de kernel que hace posible el enfoque sin sidecars.
- [[Istio]] / [[Linkerd]] — meshes basados en sidecars, el modelo que Cilium reemplaza.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).

## Related

- [[_Service Mesh|Service Mesh]]
- [[Sidecar]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
