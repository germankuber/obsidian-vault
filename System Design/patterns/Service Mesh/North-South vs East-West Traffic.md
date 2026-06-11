---
title: North-South vs East-West Traffic
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/infrastructure
  - type/concept
  - status/permanent
aliases:
  - North-South vs East-West
  - Tráfico Norte-Sur vs Este-Oeste
  - North-South Traffic
  - East-West Traffic
---

# North-South vs East-West Traffic

> [!note] Definition
> Dos direcciones de tráfico en una arquitectura de microservicios. **Norte-sur**: cliente externo ↔ servicios internos (cruza el borde internet público ↔ red privada). **Este-oeste**: servicio ↔ servicio dentro de la red privada (ej. Order Service → Inventory Service). La distinción decide qué tecnología usás: [[API Gateway]] para norte-sur, [[_Service Mesh|Service Mesh]] para este-oeste.

> [!tip] La frase que lo resume
> "The gateway handles the front door. The mesh handles the hallways." El gateway es la **puerta de entrada** (clientes externos); la mesh son los **pasillos** (servicios internos hablándose); el [[Sidecar]] es el mecanismo que hace funcionar la mesh.

## Por qué se confunden gateway, mesh y sidecar

Los tres se solapan en features (rate limiting, auth, load balancing) y los tres usan proxies por debajo. Pero operan en **capas distintas**, manejan **tráfico distinto** y resuelven **problemas distintos**:

- **API Gateway** → edge, único punto de entrada para tráfico **externo** (norte-sur).
- **Service Mesh** → gestor del tráfico **interno** (este-oeste).
- **Sidecar Proxy** → el mecanismo que potencia la mesh: corre junto a cada servicio e intercepta todo su tráfico.

Coste de equivocarse:
- Gateway donde se necesita mesh → tráfico interno **sin cifrado, sin observabilidad, sin resiliencia**.
- Mesh donde se necesita gateway → clientes externos **sin auth, sin rate-limiting, sin versioning**.

## Norte-sur (clientes externos)

- Cliente externo (web/móvil/partners) ↔ servicios internos. **Cruza el borde** internet público ↔ red privada.
- No controlás al cliente → necesitás: **auth**, **rate limiting**, **request transformation**, **API versioning**.
- Resuelto por el [[API Gateway]] en el edge.

## Este-oeste (servicio a servicio)

- Servicio ↔ servicio interno. **Se queda en la red privada.**
- Controlás ambos lados, pero la red es poco fiable → necesitás: **encryption (mTLS)**, **service discovery**, **load balancing**, **retries**, **circuit breaking**, **observabilidad**.
- Resuelto por la [[_Service Mesh|Service Mesh]] vía [[Sidecar|sidecars]].

> [!note]
> El modelo tradicional (gateway → norte-sur, mesh → este-oeste) es un buen punto de partida. La realidad moderna difumina fronteras (gateways hacen más routing este-oeste, meshes más ingress), pero la distinción core sigue válida para decisiones arquitectónicas.

## Marco de decisión (por escala)

> [!tip] Cuándo usar cada uno
> - **API Gateway** → tenés clientes externos que necesitan punto de entrada gestionado; necesitás auth/rate-limiting/versioning/transformation en el edge; exponés APIs públicas o de partners. Aplica a **virtualmente todo sistema con tráfico externo**.
> - **Service Mesh** → tenés **más de 10 a 15 servicios internos** comunicándose; necesitás [[Mutual TLS|mTLS]] para todo el tráfico interno; querés observabilidad consistente sin código; necesitás [[Traffic Management|traffic management]] (canary, fault injection) interno.
> - **Ambos** → microservicios maduros con tráfico externo **e** interno. Gateway = norte-sur en el edge; mesh = este-oeste interno. **Se complementan, no compiten.**

Guía concreta por tamaño:

| Escala | Stack recomendado |
|---|---|
| Monolito o 3-5 servicios | Gateway + HTTP directo entre servicios. **Sin mesh**. Shared HTTP client library con retry y circuit breaking. |
| 10-30 servicios, tráfico moderado | Gateway + mesh liviano ([[Linkerd]]): menor overhead que [[Istio]], aporta mTLS y observabilidad sin la complejidad. |
| 50+ servicios, alto tráfico, varios equipos | Stack completo: gateway ([[Kong]] o AWS API Gateway) + mesh completo ([[Istio]] o [[Cilium]]). El overhead se paga solo en consistencia/seguridad/observabilidad. |
| Serverless / managed | AWS Lambda con API Gateway: la plataforma maneja gran parte de lo que aporta un mesh (routing, auth, algo de observabilidad). Un mesh separado suele ser innecesario en arquitectura fully serverless. |

> [!warning] Saltate el mesh con menos de 10 servicios
> El overhead operacional (control plane, recursos de sidecars, complejidad de config) supera los beneficios. Usá una shared HTTP client library con retry logic, o service discovery liviano como [[Consul]].

## Cómo trabajan juntos (flujo de un request)

> [!example] App móvil → Order Service → Inventory Service
> 1. App móvil envía HTTPS a `api.yourcompany.com`. DNS resuelve a CDN/load balancer → reenvía al [[API Gateway]].
> 2. El **gateway** termina TLS, valida el JWT, chequea el rate limit del cliente y enruta al Order Service.
> 3. El request entra al cluster y golpea el [[Sidecar]] ([[Envoy]]) del Order Service: aplica políticas de mesh, registra telemetría y reenvía al container.
> 4. El Order Service procesa y necesita Inventory Service: hace una **llamada HTTP plana** a `inventory-service:8080` (no sabe que hay un sidecar).
> 5. La saliente la intercepta el sidecar del Order Service: busca endpoints de Inventory Service vía control plane ([[Service Discovery]]), selecciona una instancia sana con load balancing, cifra con [[Mutual TLS|mTLS]] y reenvía.
> 6. Llega a una instancia de Inventory Service, pasa por **su** sidecar (descifra mTLS, registra telemetría) y alcanza el container. La respuesta vuelve por el mismo path en reversa.

Reparto de responsabilidades:
- **Gateway** → edge concerns: auth, rate limiting.
- **Mesh** → internal concerns: mTLS, discovery, load balancing, observabilidad.
- **Sidecars** → la interceptación real + el enforcement.

## La tendencia de convergencia

> [!note]
> En pocos años gateway y mesh probablemente se **fusionen en una sola capa de infra** con políticas distintas por tipo de tráfico. Por ahora, entenderlos separados sigue siendo esencial.

- **[[Kubernetes Gateway API]]** — estándar emergente vendor-neutral para definir traffic routing norte-sur **y** este-oeste. Reemplaza el viejo recurso Ingress con semántica de routing más rica que sirve para edge e interno. [[Envoy]] Gateway y el gateway de [[Istio]] soportan ambos casos desde un único control plane.
- **eBPF-based meshes ([[Cilium]])** — eliminan el modelo de sidecar moviendo la funcionalidad del proxy al kernel de Linux ([[eBPF]]). Reduce el overhead por servicio de megabytes de memoria y milisegundos de latencia a near-zero. A medida que madura, el argumento del coste operacional contra los meshes se debilita.
- **Unified control planes ([[Gloo|Solo.io/Gloo]])** — una sola capa de gestión para gateway **Y** mesh. En vez de Kong + Istio separados, una sola plataforma para ambos tipos de tráfico.

## Errores comunes

> [!warning] Los 5 errores de microservicios
> 1. **Mesh con 5 servicios** — el overhead de Istio supera el beneficio; una HTTP client library con retries/circuit-breaking es dramáticamente más simple y suficiente.
> 2. **Business logic en el API gateway** — el gateway enruta/autentica/rate-limita; NO debe agregar datos de varios servicios, transformar business objects ni implementar workflow logic. Eso va en los servicios o en una capa [[Backend for Frontend|BFF]].
> 3. **Ignorar la latencia del sidecar** — 1-10 ms por request; una cadena de 5 servicios = **5-50 ms añadidos** solo por el mesh. Para cargas latency-sensitive, medí el overhead antes.
> 4. **No cifrar el tráfico interno** — "está dentro de la VPC, es seguro" es peligroso. Debe cifrarse con [[Mutual TLS|mTLS]]; un mesh lo hace trivial. Sin mesh, implementar TLS en cada servicio es difícil de hacer consistentemente.
> 5. **Tratar el gateway como sustituto del mesh** — un gateway puede enrutar entre servicios pero no aporta sidecars por servicio, mTLS automático, distributed tracing ni el traffic management fino del mesh. Resuelven problemas distintos.

## En entrevistas de System Design

Aparece en comunicación de microservicios, diseño de API o resiliencia. Distinguir estas tecnologías señala madurez arquitectónica.

> [!example] Respuesta modelo (verbatim, en inglés)
> "For external traffic, I would use an API gateway like Kong or AWS API Gateway. It handles authentication via JWT validation, rate limits per API key, and routes requests to the appropriate backend service. For internal traffic between the 20+ microservices, I would deploy a service mesh using Istio with Envoy sidecars. This gives us automatic mTLS encryption for all service-to-service calls, distributed tracing without code changes, and circuit breaking to prevent cascading failures. The gateway handles the front door. The mesh handles the hallways."

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).
- Marco de decisión, flujo de request y errores comunes: del artículo fuente.

## Related

- [[_Service Mesh|Service Mesh]]
- [[API Gateway]]
- [[Sidecar]]
- [[Data Plane vs Control Plane]]
- [[Kubernetes Gateway API]]
- [[_System Design|System Design]]
