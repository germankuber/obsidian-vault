---
title: Timeout
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/resilience
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Timeout
  - timeout
  - Timeout Pattern
updated: 2026-06-11
---

# Timeout

> [!note] Definition
> Fijar una duración máxima para una llamada externa: si no termina dentro del límite, se aborta y se devuelve un error o fallback en vez de esperar indefinidamente.

## Cómo funciona

Toda llamada que cruza un límite de red (otro servicio, una base, una API) puede colgarse sin devolver nunca. Un timeout pone un techo: pasado el límite, la llamada se cancela y el control vuelve. Es el patrón de resiliencia más básico — y el que habilita a los demás.

## Cuándo usarlo

> [!tip]
> **Siempre**, en toda llamada remota. Es la base sobre la que se apoyan [[Retry with Backoff]] (necesita saber cuándo un intento "falló"), [[Circuit Breaker]] (una llamada colgada debe contar como falla) y [[Bulkhead]] (sin timeout, las particiones se llenan de llamadas zombi).

## Cuándo NO usarlo / trade-offs

> [!warning]
> - El verdadero riesgo no es *si* usarlo sino **el valor**: muy corto y abortás llamadas que iban a tener éxito (falsos positivos, reintentos innecesarios); muy largo y no te protege de los cuelgues.
> - El presupuesto de timeout debe **encadenarse**: si A llama a B llama a C, el timeout de A tiene que ser mayor que el de B, o A se rinde mientras B todavía trabaja. Mal encadenado, se desperdicia trabajo.
> - Un timeout que dispara **no garantiza** que la operación no haya ocurrido del otro lado — por eso se combina con [[Idempotency]] antes de reintentar.

## Patrones relacionados / alternativas

- [[Retry with Backoff]] · [[Circuit Breaker]] · [[Bulkhead]] — todos dependen de un timeout para funcionar.
- [[Idempotency]] — porque un timeout deja la operación en estado incierto.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Circuit Breaker]]
- [[Retry with Backoff]]
- [[Bulkhead]]
