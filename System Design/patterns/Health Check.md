---
title: Health Check
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/observability
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Health Check
  - health-check
  - Health Endpoint
---

# Health Check

> [!note] Definition
> Cada servicio expone un endpoint `/health` que devuelve su estado actual.
> Balanceadores y orquestadores lo consultan para saber si el servicio está
> **vivo** y **listo** para recibir tráfico.

## Cómo funciona

Dos chequeos distintos (en Kubernetes):
- **Liveness**: ¿el proceso está vivo? Si falla, se reinicia.
- **Readiness**: ¿está listo para servir? (conexión a base OK, caché cargada). Si
  falla, se saca de la rotación del [[Load Balancing|balanceador]] sin matarlo.

El [[Load Balancing|LB]]/orquestador hace *polling* periódico y actúa según la
respuesta.

## Cuándo usarlo

> [!tip]
> En **todo** servicio detrás de un balanceador u orquestador. Es lo que permite
> sacar de rotación instancias enfermas, reiniciar las colgadas y hacer deploys
> sin downtime ([[Canary Deployment]]).

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Chequeo superficial vs profundo**: un `/health` que solo devuelve 200 no
>   detecta que la base está caída (falso "sano"). Uno demasiado profundo (chequea
>   todas las dependencias) puede dar falsos negativos y *cascadas*: si una
>   dependencia común se cae, todos se marcan unhealthy a la vez.
> - **Frecuencia**: muy seguido agrega carga; muy espaciado tarda en detectar
>   fallas.
> - Confundir liveness con readiness causa reinicios innecesarios.

## Patrones relacionados / alternativas

- [[Load Balancing]] — consume los health checks para rutear.
- [[Distributed Tracing]] / métricas — observabilidad más profunda que un health
  binario.
- [[Canary Deployment]] — usa health + métricas para decidir avanzar o revertir.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Load Balancing]]
- [[Canary Deployment]]
