---
title: Two-Phase Commit
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/consistency
  - system-design/patterns
aliases:
  - Two-Phase Commit
  - two-phase-commit
  - 2PC
---

# Two-Phase Commit

> [!note] Definition
> Protocolo para hacer una transacción **atómica** entre varios participantes:
> fase 1 pregunta a todos "¿podés commitear?"; fase 2 commitea si **todos**
> dijeron que sí, o hace rollback si alguno dijo que no.

## Cómo funciona

Un **coordinador** dirige:
1. **Prepare**: pregunta a cada participante si puede commitear; cada uno reserva
   recursos y responde sí/no, quedando "preparado".
2. **Commit/Abort**: si todos dijeron sí, ordena commit a todos; si alguno dijo
   no (o no respondió), ordena abort.

Garantiza atomicidad: o todos commitean o ninguno.

## Cuándo usarlo

> [!tip]
> Cuando necesitás **atomicidad fuerte** entre recursos distintos (dos bases, una
> base + una cola) y la corrección lo exige por encima de la disponibilidad.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Bloqueante**: los participantes quedan "preparados" con recursos
>   reservados/locked hasta la fase 2. Si el coordinador cae justo ahí, quedan
>   colgados indefinidamente (*blocking problem*).
> - **Mata la disponibilidad**: un solo participante lento o caído bloquea toda la
>   transacción. No escala bien.
> - **Punto único de falla**: el coordinador.
> - En microservicios distribuidos, casi siempre se prefiere [[Saga]]
>   (consistencia eventual, sin locks globales) en vez de 2PC.

## Patrones relacionados / alternativas

- [[Saga]] — la alternativa moderna: transacciones locales + compensaciones, sin
  bloqueo global. Cambia atomicidad fuerte por disponibilidad.
- [[Quorum]] — consistencia en sistemas replicados sin commit atómico global.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Saga]]
- [[Quorum]]
