---
title: Canary Deployment
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/observability
  - system-design/patterns
aliases:
  - Canary Deployment
  - canary-deployment
  - Canary Release
---

# Canary Deployment

> [!note] Definition
> Desplegar la versión nueva a un **subconjunto chico** de servidores, rutear un
> pequeño porcentaje del tráfico (1-5%), monitorear las métricas, y aumentar
> gradualmente si todo está sano — o hacer **rollback** si las métricas se
> degradan.

## Cómo funciona

El nombre viene del "canario en la mina". Se libera la versión nueva a pocos
usuarios reales; se observan errores, latencia y métricas de negocio
([[Distributed Tracing]], [[Health Check]]). Si se mantienen sanas, se sube el
tráfico por etapas (5% → 25% → 100%); si empeoran, se revierte afectando a pocos.

## Cuándo usarlo

> [!tip]
> Para **reducir el riesgo de cada release** en producción: en vez de exponer a
> todos a un bug nuevo, lo descubrís con un grupo chico. Ideal con despliegues
> frecuentes y tráfico suficiente para sacar señal estadística.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Necesitás métricas confiables y automáticas** para decidir avanzar/revertir;
>   sin buena observabilidad, el canary es solo un deploy lento.
> - **Convivencia de versiones**: dos versiones corriendo a la vez deben ser
>   compatibles (esquema de base, contratos de API → [[API Versioning]]).
> - **Tráfico bajo** no da señal estadística para detectar regresiones a tiempo.
> - Estado/migraciones de base complican el rollback (revertir código es fácil,
>   revertir datos no).

## Patrones relacionados / alternativas

- [[Health Check]] / [[Distributed Tracing]] — proveen las señales para decidir.
- [[Load Balancing]] — reparte el % de tráfico al canary.
- [[API Versioning]] — para que las versiones convivan sin romperse.
- *Blue-Green deployment* — alternativa: dos entornos completos, switch instantáneo
  (sin exposición gradual).

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Health Check]]
- [[Distributed Tracing]]
- [[API Versioning]]
