---
title: Graceful Degradation
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/reliability
  - system-design/patterns
aliases:
  - Graceful Degradation
  - graceful-degradation
  - Degradación elegante
---

# Graceful Degradation

> [!note] Definition
> Cuando fallan componentes **no críticos**, seguir sirviendo una experiencia
> **degradada pero funcional** en vez de fallar por completo. El sistema pierde
> features, no disponibilidad.

## Cómo funciona

Se identifica qué es esencial y qué es "nice to have". Si el servicio de
recomendaciones se cae, la página de producto se sirve igual, solo sin la sección
"También te puede gustar". Se apoya en fallbacks: valores por defecto, datos
cacheados, secciones ocultas. Frecuentemente combinado con [[Circuit Breaker]]
(cuyo fallback *es* la versión degradada).

## Cuándo usarlo

> [!tip]
> En sistemas de cara al usuario donde **disponibilidad parcial > caída total**:
> e-commerce, streaming, redes sociales. Mejor un carrito que funciona sin
> recomendaciones que un error 500 completo.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Requiere clasificar criticidad** de cada dependencia, y diseñar el fallback
>   de cada una — trabajo de diseño explícito, no sale gratis.
> - **Degradación silenciosa peligrosa**: si nadie nota que una feature está caída
>   (porque "degradó elegante"), un problema real puede quedar oculto. Necesita
>   alertas igual.
> - **No aplica a lo esencial**: no podés "degradar" el checkout o la
>   autenticación — eso debe ser correcto o fallar visible.

## Patrones relacionados / alternativas

- [[Circuit Breaker]] — su fallback suele ser la respuesta degradada.
- [[Bulkhead]] — aísla la falla para que solo se degrade lo afectado.
- [[Cache-Aside]] — servir datos cacheados (aunque viejos) es una forma de
  degradar con gracia.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[Circuit Breaker]]
- [[Bulkhead]]
