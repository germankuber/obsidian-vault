---
title: Service Mesh — Mapa del tema
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/moc
  - status/permanent
aliases:
  - Service Mesh MOC
  - Service Mesh Index
---

# Service Mesh — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del subtema *service mesh* dentro de [[_System Design|System Design]]: cómo se gestiona el tráfico **este-oeste** (servicio↔servicio) en microservicios, y su relación con el [[API Gateway]] (norte-sur). Empezá por el fundamento y bajá. Abrí esta nota, no la carpeta.

## 🧭 Fundamento

- [[North-South vs East-West Traffic]] — la nota madre del cluster: norte-sur (gateway, clientes externos) vs este-oeste (mesh, servicios internos), el **marco de decisión por escala**, el flujo de un request de punta a punta, los 5 errores comunes y la respuesta modelo de entrevista. El punto de partida.

## ⚙️ Mecanismos

- [[Sidecar]] — el proxy ligero junto a cada servicio que intercepta todo su tráfico; mueve los networking concerns del código a la infra. Aloja los números de overhead (50-100MB, 1-10 ms/hop).
- [[Data Plane vs Control Plane]] — las dos partes de la mesh: sidecars que ejecutan (data plane) + componente central que configura (control plane).

## 🛡️ Capacidades

- [[Mutual TLS]] — cifrado y autenticación mutua de todo el tráfico interno; cert por sidecar con rotación automática (zero-trust).
- [[Service Discovery]] — encontrar instancias sanas de otros servicios sin IPs hardcodeadas.
- [[Traffic Management]] — canary, traffic mirroring y fault injection internos vía sidecars.
- También transversales: [[Circuit Breaker]] y [[Distributed Tracing]] — la mesh los provee sin código en cada servicio.

## 🧰 Implementaciones

- [[Envoy]] — el sidecar/proxy más popular; data plane de facto.
- [[Istio]] — la mesh más adoptada (Envoy + istiod); potente pero compleja.
- [[Linkerd]] — alternativa liviana, foco en simplicidad y performance.
- [[Cilium]] — mesh basado en eBPF, sin sidecars, overhead near-zero.

## 🔭 Convergencia

- [[Kubernetes Gateway API]] — estándar vendor-neutral que unifica routing norte-sur y este-oeste; empuja a fusionar gateway y mesh en una sola capa.

## 🔗 Conexión con el resto del grafo

- Subtema de [[_System Design|System Design]]. Cruza con [[API Gateway]] (la puerta norte-sur, complementaria a la mesh), [[Backend for Frontend]] (donde va la lógica que NO debe ir en el gateway) y [[Canary Deployment]] (el release progresivo que el traffic management implementa).

## 🌱 Por escribir (semillas del grafo)

- [[Kong]] — API gateway open-source con tier enterprise; candidato a promover.
- [[AWS App Mesh]] — mesh managed para AWS.
- [[Consul]] — service discovery liviano, alternativa al mesh con pocos servicios.
- [[eBPF]] — la tecnología de kernel detrás de [[Cilium]].
- [[Gloo|Solo.io/Gloo]] — unified control plane para gateway Y mesh.

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "System Design/patterns/Service Mesh"
WHERE file.name != this.file.name
SORT file.name ASC
```
