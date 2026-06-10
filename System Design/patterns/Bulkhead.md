---
title: Bulkhead
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/resilience
  - system-design/patterns
aliases:
  - Bulkhead
  - bulkhead
  - Mamparo
  - Resource Isolation
---

# Bulkhead

> [!note] Definition
> Aislar cargas de trabajo en **pools de recursos separados** (threads,
> conexiones, instancias) para que una carga que agota su pool no afecte a las
> demás. El nombre viene de los mamparos estancos de un barco: una sección
> inundada no hunde todo el casco.

## Cómo funciona

En vez de un único pool compartido para todas las dependencias, cada una recibe
su propia partición. Si la dependencia A se vuelve lenta y satura su pool, las
llamadas a B y C siguen teniendo recursos. La falla queda **contenida** en su
compartimento.

## Cuándo usarlo

> [!tip]
> Cuando varias dependencias comparten un recurso finito (un thread pool, un
> connection pool) y una sola dependencia lenta podría consumirlo entero y tirar
> abajo todo el servicio. Especialmente útil en servicios que llaman a muchas
> dependencias heterogéneas.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Particionar desperdicia capacidad**: pools separados no se prestan recursos
>   entre sí. Reservás N threads para una dependencia que la mayor parte del
>   tiempo no los usa — peor utilización que un pool único.
> - **Más configuración y tuning**: hay que dimensionar cada partición; mal
>   dimensionada, una dependencia se queda corta mientras otra desperdicia.
> - Para un servicio con **una sola** dependencia, no hay nada que aislar.
> - No detecta la falla, solo la contiene — se combina con [[Circuit Breaker]]
>   (que sí corta) y [[Timeout]].

## Patrones relacionados / alternativas

- [[Circuit Breaker]] — corta la dependencia saturada; el bulkhead evita que su
  saturación contagie a las otras. Suelen ir juntos.
- [[Timeout]] — sin timeouts, una partición se llena de llamadas colgadas.
- [[Load Balancing]] — distribuye carga *entre* instancias; el bulkhead aísla
  *dentro* de una.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Circuit Breaker]]
- [[Timeout]]
