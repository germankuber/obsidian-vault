---
title: Rate Limiting
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/api
  - system-design/patterns
aliases:
  - Rate Limiting
  - rate-limiting
  - Throttling
---

# Rate Limiting

> [!note] Definition
> Restringir cuántos requests puede hacer un cliente en una ventana de tiempo,
> usando algoritmos como *token bucket*, *fixed window* o *sliding window*.

## Cómo funciona

- **Token bucket**: un balde se rellena a tasa fija; cada request consume un
  token; sin tokens, se rechaza. Permite ráfagas hasta el tamaño del balde.
- **Fixed window**: N requests por ventana fija (ej. por minuto). Simple, pero
  permite picos en el borde entre ventanas.
- **Sliding window**: ventana móvil, reparte más parejo, evita el pico de borde.

Al exceder, se responde `429 Too Many Requests`.

## Cuándo usarlo

> [!tip]
> Para **proteger** el sistema de abuso, scraping, bugs de clientes en loop y
> picos que tumban servicios. También para *fairness* (que un cliente no monopolice)
> y para tiers de pricing. Suele vivir en el [[API Gateway]].

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Definir los límites es delicado**: muy estricto y rechazás tráfico
>   legítimo; muy laxo y no protegés.
> - **Estado distribuido**: con varias instancias, el contador debe ser
>   compartido (Redis) o cada nodo limita por separado y el límite real es N×.
> - **Identificar al cliente** es complejo: por IP (NAT agrupa usuarios), por API
>   key, por user — cada uno con problemas.
> - Debe devolver headers claros (`Retry-After`) para que el cliente coopere con
>   [[Retry with Backoff]].

## Patrones relacionados / alternativas

- [[API Gateway]] — el lugar natural donde aplicarlo.
- [[Retry with Backoff]] — cómo el cliente debe reaccionar al 429.
- [[Cache Stampede Prevention]] — otra defensa de la base ante avalanchas.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[API Gateway]]
- [[Retry with Backoff]]
