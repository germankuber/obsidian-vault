---
title: Quorum
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/consistency
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - Quorum
  - quorum
  - Quorum Consensus
updated: 2026-06-11
---

# Quorum

> [!note] Definition
> En un sistema replicado con **N** réplicas, exigir que **W** confirmen una escritura y **R** coincidan en una lectura, con **W + R > N**. Esa desigualdad garantiza que toda lectura ve al menos una réplica con la última escritura.

## Cómo funciona

Como los conjuntos de W (escritura) y R (lectura) se solapan cuando W + R > N, al menos una réplica leída tiene el dato más nuevo. Ajustando W y R se elige el balance:
- **W alto, R bajo**: escrituras más lentas/consistentes, lecturas rápidas.
- **W bajo, R alto**: lo inverso.
- `W = R = (N/2)+1`: balance típico (mayoría).

Es la base de la consistencia ajustable en sistemas tipo Dynamo/Cassandra.

## Cuándo usarlo

> [!tip]
> En **almacenes distribuidos replicados** donde querés *tunear* el trade-off consistencia↔latencia↔disponibilidad sin un líder único. Permite seguir operando aunque algunas réplicas estén caídas (mientras alcances el quórum).

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **No da consistencia fuerte total** por sí solo: sin mecanismos extra (read repair, hinted handoff) puede haber lecturas inconsistentes transitorias.
> - **Latencia**: esperar W o R respuestas es más lento que tocar una sola réplica.
> - **Más complejo de razonar** que un modelo líder-seguidor ([[Primary-Replica]]).
> - Para datos donde un líder único alcanza, Primary-Replica es más simple.

## Patrones relacionados / alternativas

- [[Primary-Replica]] — alternativa con líder único, más simple, menos flexible.
- [[Two-Phase Commit]] — atomicidad entre recursos distintos (problema distinto).
- [[Vector Clocks]] — para detectar/resolver conflictos entre réplicas.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Primary-Replica]]
- [[Vector Clocks]]
