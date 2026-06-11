---
title: Service Discovery
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/concept
  - status/permanent
aliases:
  - Service Discovery
  - Descubrimiento de servicios
---

# Service Discovery

> [!note] Definition
> El mecanismo por el que un servicio **encuentra las instancias sanas** de otro servicio sin tener IPs hardcodeadas. En un mundo donde las instancias aparecen, mueren y cambian de IP todo el tiempo (autoscaling, Kubernetes), hace falta un **registro dinámico** que diga "Inventory Service vive hoy en estas N direcciones".

## Cómo funciona

- En una [[_Service Mesh|Service Mesh]]: el [[Data Plane vs Control Plane|control plane]] mantiene un **registro de todos los servicios y sus instancias**. Cuando el [[Sidecar|sidecar]] de origen necesita llamar a `inventory-service`, **consulta endpoints vía el control plane**, recibe la lista de instancias sanas y elige una con [[Load Balancing|load balancing]].
- El servicio hace una **llamada HTTP plana** a un nombre lógico (`inventory-service:8080`); el sidecar traduce ese nombre a una instancia real y viva. El servicio nunca conoce IPs.

> [!example] En el flujo de request
> Order Service → llama a `inventory-service:8080` → su sidecar busca endpoints vía control plane → selecciona una instancia sana → cifra con mTLS → reenvía.

## Cuándo usarlo

> [!tip]
> Indispensable en cualquier sistema con **instancias dinámicas** (microservicios en Kubernetes, autoscaling). Sin discovery, cada cambio de topología obliga a reconfigurar clientes a mano.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - Para **pocos servicios estáticos**, un mesh completo es excesivo solo por discovery: alcanza un registro liviano como [[Consul]] o incluso DNS.
> - El registro es **estado crítico**: si está desactualizado, los clientes apuntan a instancias muertas (de ahí la importancia de combinarlo con [[Health Check|health checks]]).

## Conceptos relacionados

- [[Load Balancing]] — elegir **cuál** de las instancias descubiertas usar.
- [[Health Check]] — el discovery debe devolver solo instancias **sanas**.
- [[Consul]] — opción de service discovery liviano, útil sin mesh.
- [[_Service Mesh|Service Mesh]] — lo integra junto con LB, mTLS y observabilidad.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).

## Related

- [[_Service Mesh|Service Mesh]]
- [[Sidecar]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
