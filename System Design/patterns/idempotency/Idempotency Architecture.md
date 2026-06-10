---
title: Idempotency Architecture
source: https://designgurus.substack.com/p/system-design-deep-dive-architecting
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/resilience
  - system-design/api
aliases:
  - Idempotency Architecture
  - Arquitectura Idempotente
  - Idempotent API Architecture
---

# Idempotency Architecture

> [!note] Definición
> El diseño del **lado servidor** que hace seguros los retries automáticos: cómo un payment gateway (u otra API con efectos secundarios) detecta y absorbe requests duplicados. Implementa el concepto de [[Idempotency]] vía una [[Idempotency Key|key]] + una capa de tracking. *"Shifts the burden of safety from the chaotic network to the intelligent server."*

## El problema de fondo

- Sobre redes impredecibles, cualquier componente puede fallar: un router puede cortar la conexión **justo antes de que la confirmación de éxito llegue al cliente**. El cliente asume que todo falló y **reintenta**.
- Un backend mal diseñado procesa el retry como un **comando nuevo** → en un pago, **deduce fondos por segunda vez**, destruyendo la integridad de datos.
- Los retries automáticos son la respuesta estándar a timeouts, pero reintentar a ciegas causa **procesamiento duplicado catastrófico**.

## La capa de almacenamiento (tracking)

- Se usa un **in-memory datastore** (datos en RAM, no en disco) para validar en **milisegundos** — chequear una DB lenta en cada request destruiría el rendimiento. Es el *single source of truth* de la validación.
- El **tracking record** guarda **tres piezas**:
  1. El **string de la key** único.
  2. El **status** actual de la transacción.
  3. Una **copia serializada de la respuesta de éxito final** (para cachear).

## El flujo paso a paso

1. **Validation & status locking** — al llegar un request, el server **frena todo procesamiento financiero**, extrae la key del header y consulta el datastore. Si no existe → crea un record con status **`processing`** (este status le dice al resto del sistema distribuido que hay una transacción corriendo, para coordinarse).
2. **Procesar el ledger** — con el record establecido, ejecuta la lógica pesada contra el **financial ledger** (DB append-only de créditos/débitos). Las operaciones de DB se **agrupan para que tengan éxito o fallen juntas** (atomicidad).
3. **Cachear la respuesta** — **antes** de responder al cliente: cambia el status de `processing` a **`completed`** y guarda el **texto exacto de la respuesta de éxito** en el datastore. Recién entonces transmite el éxito al cliente.
4. **Interceptar el retry** — si llega un duplicado (misma key), el datastore **encuentra el record existente** con status `completed`: el server **aborta** la lógica financiera, recupera la **respuesta cacheada** y la devuelve. El ledger queda **intacto**.

## Edge cases

- **Race conditions** → [[Distributed Lock]] (unique constraint: acepta el primer request concurrente, rechaza el segundo).
- **Payload alterado en el retry** → payload hashing (ver [[Idempotency Key]]): si el hash del payload nuevo no coincide con el guardado, **rechaza**.
- **Expiración de datos (TTL)** — un datastore in-memory tiene capacidad limitada. Una **Time to Live policy** borra automáticamente cada record tras un límite; se configuran típicamente en **24 horas** (los retries pasan en segundos/minutos, así que 24h es buffer de sobra). Tras expirar, purga la key para liberar RAM, sin intervención manual.
- **Fallas internas del server** (no de red) — si falla la lógica o la conexión al ledger, el server marca el status **`failed`** y **NO debe cachear la respuesta de fallo** (cachear un fallo impediría recuperarse). En el próximo retry, al ver `failed`, **permite reintentar** la lógica (el fallo pudo ser un glitch temporal).
- **Request in-progress** — si llega un retry mientras el original sigue ejecutando (status `processing`), el server **no puede devolver una respuesta cacheada** (aún no existe) → rechaza con **HTTP 409 (conflict)**, informando al cliente que espere unos segundos antes de re-chequear. Evita polling agresivo.
- **Sin key** → **HTTP 400 (bad request)**.

## Los tres estados del tracking record

`processing` (corriendo) → `completed` (éxito, respuesta cacheada) · `failed` (error interno, NO se cachea, se permite reintentar).

## References

- Fuente: [System Design Deep Dive: Architecting Idempotent APIs](https://designgurus.substack.com/p/system-design-deep-dive-architecting) — Arslan Ahmad (Design Gurus), 2026-03-05

## Related

- [[Idempotency]]
- [[Idempotency Key]]
- [[Distributed Lock]]
- [[Retry with Backoff]]
- [[Timeout]]
