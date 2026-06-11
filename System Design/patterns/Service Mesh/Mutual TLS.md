---
title: Mutual TLS
source: https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar
author: Arslan Ahmad (Design Gurus)
created: 2026-06-11
tags:
  - system-design/security
  - type/concept
  - status/permanent
aliases:
  - mTLS
  - Mutual TLS
  - TLS mutuo
---

# Mutual TLS

> [!note] Definition
> TLS donde **ambos lados se autentican mutuamente** con certificados (no solo el servidor, como en HTTPS normal). En una [[_Service Mesh|Service Mesh]] se aplica a **todo el tráfico interno**: cada servicio prueba su identidad antes de hablar, y la comunicación va cifrada — un modelo **zero-trust** a nivel de infraestructura.

## Cómo funciona en una mesh

- **Un certificado por [[Sidecar|sidecar]]**: el [[Data Plane vs Control Plane|control plane]] emite y distribuye certs a cada sidecar, con **rotación automática**.
- Cada llamada servicio-a-servicio la **cifra el sidecar de origen** y la **descifra el sidecar de destino**. Los servicios solo ven HTTP/gRPC plano; no saben del cifrado.
- El resultado es **mTLS en todas partes** automáticamente: cifra todo el tráfico este-oeste sin que nadie escriba una línea de cripto.

> [!tip] Esto es lo que un [[API Gateway]] NO hace
> Un gateway cubre el edge (norte-sur). El mTLS pervasivo del tráfico interno (este-oeste) es de la mesh: cert por sidecar, rotación automática, zero-trust de infra. Un gateway no aporta sidecars por servicio.

## Cuándo usarlo

> [!tip]
> Siempre que el tráfico interno deba ser **confidencial y autenticado** — que es casi siempre. Con un mesh es trivial activarlo; sin mesh, implementar TLS consistentemente en cada servicio y lenguaje es difícil y propenso a errores.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **El antipatrón a evitar**: "está dentro de la VPC, es seguro" → falso. La red interna no es de fiar; el tráfico interno debe cifrarse.
> - Sin un mesh, hacer mTLS bien (emisión, distribución y **rotación** de certs en cada servicio) es trabajo manual difícil de mantener consistente.
> - El cifrado/descifrado suma algo de CPU y latencia, parte del overhead del [[Sidecar]] (1-10 ms por hop).

## Conceptos relacionados

- [[_Service Mesh|Service Mesh]] — lo provee transversalmente.
- [[Data Plane vs Control Plane]] — el control plane gestiona los certs; el data plane cifra.
- [[Sidecar]] — donde vive el cert y ocurre el cifrado.

## References

- Fuente: [API Gateway vs Service Mesh vs Sidecar Proxy: A Decision Framework](https://designgurus.substack.com/p/api-gateway-vs-service-mesh-vs-sidecar) — Arslan Ahmad (Design Gurus).
- Detalle de cert-por-sidecar y rotación: del artículo fuente.

## Related

- [[_Service Mesh|Service Mesh]]
- [[North-South vs East-West Traffic]]
- [[_System Design|System Design]]
