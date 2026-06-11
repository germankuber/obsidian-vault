---
title: Auto-Scaling
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/scaling
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Auto-Scaling
  - auto-scaling
  - Autoscaling
updated: 2026-06-11
---

# Auto-Scaling

> [!note] Definition
> Agregar o quitar instancias **automáticamente** según el tráfico: suma cuando la CPU (u otra métrica) supera un umbral, resta cuando el tráfico baja.

## Cómo funciona

Un controlador observa métricas (CPU, memoria, requests/s, largo de cola) y ajusta el número de instancias contra políticas definidas. Sobre [[Horizontal Scaling]] + [[Load Balancing]]: las instancias nuevas entran a la rotación del balanceador automáticamente. Ahorra costo (pagás por lo que usás) y absorbe picos.

## Cuándo usarlo

> [!tip]
> Cuando el tráfico es **variable o impredecible**: picos diarios, campañas, estacionalidad. En la nube es casi default para servicios stateless.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **El arranque no es instantáneo**: bootear una instancia lleva segundos o minutos; ante un pico súbito llegás tarde. Mitigás con *pre-warming* o escalado predictivo.
> - **Flapping**: umbrales mal puestos hacen subir/bajar instancias sin parar (caro e inestable) → hace falta *cooldown* e histéresis.
> - **El cuello puede no ser la app**: si escalás la app pero la base no aguanta, solo trasladás el problema ([[Sharding]], [[Connection Pooling]]).
> - Requiere instancias **stateless** y arranque rápido.

## Patrones relacionados / alternativas

- [[Horizontal Scaling]] — auto-scaling lo automatiza.
- [[Load Balancing]] — integra las instancias nuevas/removidas.
- [[Connection Pooling]] — para que escalar la app no reviente la base.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Horizontal Scaling]]
- [[Load Balancing]]
