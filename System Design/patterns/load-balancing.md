---
title: Load Balancing
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/scaling
  - system-design/patterns
aliases:
  - Load Balancing
  - load-balancing
  - Load Balancer
---

# Load Balancing

> [!note] Definition
> Distribuir los requests entrantes entre varios servidores usando algoritmos
> como *round-robin*, *least connections*, *weighted* o *IP hash*. Es lo que hace
> posible el [[Horizontal Scaling]].

## Cómo funciona

El balanceador es la puerta de entrada: recibe el tráfico y elige a qué instancia
mandarlo. Hace *health checks* ([[Health Check]]) para no rutear a instancias
caídas. Algoritmos: round-robin (rota parejo), least-connections (al menos
ocupado), weighted (según capacidad), IP hash (mismo cliente → misma instancia,
para sticky sessions).

## Cuándo usarlo

> [!tip]
> Siempre que tengas **más de una instancia** de un servicio. Reparte carga, da
> alta disponibilidad (saca de rotación lo que falla) y permite despliegues sin
> downtime ([[Canary Deployment]]).

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Puede ser un punto único de falla**: si el LB cae, cae todo → necesitás LBs
>   redundantes.
> - **Sticky sessions complican el balanceo**: si las instancias guardan estado de
>   sesión, atás cada cliente a una instancia y perdés reparto parejo (mejor:
>   hacer las instancias stateless).
> - **Health checks mal calibrados**: muy laxos rutean a instancias muertas; muy
>   agresivos sacan instancias sanas.
> - Capa 4 (TCP) vs capa 7 (HTTP) es una decisión con trade-offs de features vs
>   rendimiento.

## Patrones relacionados / alternativas

- [[Horizontal Scaling]] — el LB es su requisito.
- [[Health Check]] — alimenta las decisiones de ruteo del LB.
- [[Reverse Proxy]] — a menudo el mismo componente hace ambos.
- [[API Gateway]] — superpone ruteo + auth + rate limiting sobre el balanceo.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Horizontal Scaling]]
- [[Health Check]]
- [[Reverse Proxy]]
