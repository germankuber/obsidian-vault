---
title: Circuit Breaker
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/resilience
  - system-design/patterns
aliases:
  - Circuit Breaker
  - circuit-breaker
  - Disyuntor
---

# Circuit Breaker

> [!note] Definition
> Envuelve las llamadas a una dependencia y, si falla repetidamente, "abre el
> circuito": deja de llamarla y devuelve un fallback de inmediato, dándole tiempo
> a recuperarse. Cada tanto deja pasar una llamada de prueba para ver si ya
> volvió.

## Cómo funciona

Tres estados:
- **Closed** — las llamadas fluyen normal; se cuentan las fallas.
- **Open** — superado un umbral de fallas, todas las llamadas fallan al instante
  sin tocar la dependencia.
- **Half-Open** — tras un timeout, deja pasar **una** llamada de prueba. Si sale
  bien, vuelve a Closed; si falla, vuelve a Open.

El estado Half-Open es la parte sutil: sin esa prueba, un breaker abierto nunca
se enteraría de que la dependencia ya se recuperó.

## Cuándo usarlo

> [!tip]
> Cuando una dependencia puede fallar de forma **sostenida** y seguir llamándola
> solo empeora las cosas: agota tu pool de threads/conexiones, encola trabajo
> condenado y propaga la caída. Ideal frente a servicios remotos, APIs externas,
> bases bajo estrés. Se combina con [[Retry with Backoff]] (que cubre la falla
> *transitoria*) y [[Bulkhead]] (que contiene el contagio).

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **No es para fallas transitorias** — para un blip de red, un [[Retry with Backoff]]
>   simple alcanza; un breaker ahí es sobre-ingeniería.
> - **Tunear el umbral y el reset es trabajo real**: muy sensible y "trips" con
>   ruido (rechazás tráfico sano); muy laxo y no protegés nada.
> - **Agrega estado y complejidad**: hay que decidir el fallback (¿error?,
>   ¿valor cacheado?, ¿cola?) y monitorear el estado del breaker.
> - En un monolito o una llamada in-process que no puede "caerse sola", no aporta.

## Patrones relacionados / alternativas

- [[Retry with Backoff]] — para la falla transitoria; el breaker toma la posta
  cuando la falla es sostenida.
- [[Timeout]] — casi siempre va *debajo* de un breaker: sin timeout, una llamada
  colgada nunca cuenta como falla.
- [[Bulkhead]] — aísla recursos para que el problema no se propague.
- [[Graceful Degradation]] — qué servís cuando el circuito está abierto.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Retry with Backoff]]
- [[Bulkhead]]
- [[Timeout]]
