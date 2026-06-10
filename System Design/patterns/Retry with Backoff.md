---
title: Retry with Backoff
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/resilience
  - system-design/patterns
aliases:
  - Retry with Backoff
  - retry-with-backoff
  - Retry Pattern
  - Exponential Backoff
---

# Retry with Backoff

> [!note] Definition
> Ante una falla, reintentar la operación tras demoras **crecientes** (1s, 2s,
> 4s, 8s…) en vez de inmediatamente, sumando *jitter* (un offset aleatorio) para
> que muchos clientes no se sincronicen y golpeen a la vez.

## Cómo funciona

El backoff exponencial separa los reintentos cada vez más, dándole aire al
servicio para recuperarse. El **jitter** rompe la sincronización: sin él, miles
de clientes que fallaron en el mismo instante reintentan exactamente juntos —una
*estampida* (thundering herd) que vuelve a tumbar al servicio que recién se
levantaba. Siempre se pone un **tope de intentos**.

## Cuándo usarlo

> [!tip]
> Cuando la falla es probablemente **transitoria**: un paquete perdido, una pausa
> de GC, un pico momentáneo. El reintento, bien espaciado, suele tener éxito sin
> intervención.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **No reintentar operaciones no [[Idempotency|idempotentes]]** sin cuidado: un
>   reintento de "cobrar tarjeta" puede cobrar dos veces. Reintento ⇒ necesitás
>   idempotencia.
> - **No reintentar fallas permanentes**: un 4xx (request inválido), un "no
>   existe" o un error de auth no mejora reintentando — solo gastás tiempo y
>   amplificás carga. Reintentá solo errores potencialmente transitorios (5xx,
>   timeouts).
> - **Reintentos infinitos esconden caídas reales** y amplifican la carga sobre
>   un servicio ya caído. Por eso el tope y el [[Circuit Breaker]] encima.
> - El backoff agrega **latencia** percibida: el usuario espera 1+2+4s antes del
>   error final.

## Patrones relacionados / alternativas

- [[Circuit Breaker]] — cuando la falla deja de ser transitoria, el breaker corta
  los reintentos por completo.
- [[Idempotency]] — requisito para reintentar con seguridad.
- [[Dead Letter Queue]] — a dónde mandar lo que no se pudo procesar tras agotar
  los reintentos.
- [[Timeout]] — define cuándo un intento "falló" y dispara el siguiente.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Circuit Breaker]]
- [[Idempotency]]
- [[Dead Letter Queue]]
