---
title: Distributed Lock
source: https://designgurus.substack.com/p/system-design-deep-dive-architecting
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/consistency
  - system-design/resilience
aliases:
  - Distributed Lock
  - Lock Distribuido
  - Distributed Locks
---

# Distributed Lock

> [!note] Definición
> Una restricción que **garantiza que solo un thread de servidor pueda insertar una key específica en un momento dado**. Actúa como un *"semáforo inquebrantable"* (unbreakable traffic light) para la base de datos del backend. Resuelve las **race conditions** en sistemas distribuidos de alto volumen.

## El problema que ataca — race conditions

- Una **race condition** ocurre cuando dos procesos concurrentes intentan manipular **el mismo dato simultáneamente**.
- Escenario: un cliente dispara por error **dos requests idénticos en el mismo milisegundo**. Ambos llegan al servidor a la vez, ambos consultan el datastore al mismo tiempo, y como **ninguno creó todavía un record**, ambas queries vuelven vacías → los dos intentan ejecutar la lógica de deducción financiera. Las verificaciones básicas de seguridad ([[Idempotency Architecture|el chequeo de key existente]]) **se evaden por completo**.

## Cómo lo resuelve

- Se implementa como un **unique constraint** estricto en la base de datos.
- Cuando los dos requests concurrentes intentan crear el tracking record, la DB **acepta el primero** e **instantáneamente rechaza el segundo con un error estructural**.
- El servidor **captura ese error de DB** y detiene la ejecución duplicada. El segundo request queda bloqueado de forma segura **antes de tocar dato financiero alguno**.

## Conexión en el vault

- Es la pieza que hace **a prueba de concurrencia** la [[Idempotency Architecture|arquitectura idempotente]]: sin él, dos retries simultáneos burlan el chequeo de [[Idempotency Key|key]] existente.
- Relacionado con [[Quorum]] y [[Two-Phase Commit]] como mecanismos de coordinación en sistemas distribuidos.

## References

- Fuente: [System Design Deep Dive: Architecting Idempotent APIs](https://designgurus.substack.com/p/system-design-deep-dive-architecting) — Arslan Ahmad (Design Gurus), 2026-03-05

## Related

- [[Idempotency Architecture]]
- [[Idempotency]]
- [[Idempotency Key]]
- [[Race Condition]]
