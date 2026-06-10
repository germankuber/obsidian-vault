---
title: Idempotency
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/resilience
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Idempotency
  - idempotency
  - Idempotencia
  - Idempotent Operations
---

# Idempotency

> [!note] Definition
> Diseñar una operación para que ejecutarla **N veces produzca el mismo resultado
> que ejecutarla una vez**. Reintentar una operación idempotente es seguro: no
> duplica efectos.

## Cómo funciona

La técnica habitual es una [[Idempotency Key|idempotency key]]: el cliente manda un identificador único por operación; el servidor registra qué keys ya procesó y, si una se repite, devuelve el resultado original en vez de volver a ejecutar. Operaciones naturalmente idempotentes (un `GET`, un `PUT` que setea un valor absoluto, un `DELETE`) no necesitan key; las que acumulan efecto (`POST` que cobra, incrementa, encola) sí.

El diseño completo del lado servidor —tracking record, estados processing/completed/failed, cacheo de la respuesta, race conditions, TTL— está en [[Idempotency Architecture]]. Acá queda el concepto; ahí, el cómo.

## Operaciones seguras vs inseguras

- **Naturalmente idempotentes**: lecturas. Un `GET` veinte veces deja la DB igual → reintentar es 100% seguro.
- **No idempotentes por defecto**: las que modifican estado. Un `POST` crea/acumula → reintentar duplica (en un ledger financiero, entradas duplicadas). Hay que **construir** idempotencia manualmente en estos.

## Cuándo usarlo

> [!tip]
> Imprescindible cuando hay **reintentos** o entregas *at-least-once*:
> [[Retry with Backoff]], colas de mensajes ([[Message Queue]], [[Pub-Sub|Pub/Sub]]),
> [[Webhooks]]. En sistemas distribuidos el reenvío es inevitable, así que
> cualquier operación con efectos secundarios (pagos, órdenes, envíos) debe ser
> idempotente.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **No es gratis**: requiere almacenar las keys procesadas (con su TTL) y
>   chequearlas en cada request — estado y latencia extra.
> - **Definir la "misma" operación es sutil**: ¿dos compras idénticas son un
>   reintento o dos compras legítimas? La key debe reflejar la intención del
>   cliente, no solo el payload.
> - Para operaciones de **solo lectura** o ya naturalmente idempotentes, agregar
>   maquinaria de keys es sobre-ingeniería.
> - La ventana de deduplicación es finita: si el reintento llega después de que
>   la key expiró, se vuelve a ejecutar.

## Patrones relacionados / alternativas

- [[Retry with Backoff]] — la razón principal por la que necesitás idempotencia.
- [[Message Queue]] / [[Pub-Sub|Pub/Sub]] — entregan *at-least-once*: el consumidor debe
  ser idempotente.
- [[Timeout]] — deja la operación en estado incierto, que la idempotencia
  resuelve al reintentar.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Deep-dive de arquitectura: [System Design Deep Dive: Architecting Idempotent APIs](https://designgurus.substack.com/p/system-design-deep-dive-architecting) — Arslan Ahmad (Design Gurus), 2026-03-05
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Idempotency Key]]
- [[Idempotency Architecture]]
- [[Distributed Lock]]
- [[Retry with Backoff]]
- [[Message Queue]]
