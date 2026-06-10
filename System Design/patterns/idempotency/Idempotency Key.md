---
title: Idempotency Key
source: https://designgurus.substack.com/p/system-design-deep-dive-architecting
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/resilience
  - system-design/api
aliases:
  - Idempotency Key
  - Clave de Idempotencia
---

# Idempotency Key

> [!note] Definición
> Un string único de caracteres que viaja **junto al payload** y permite al servidor identificar requests duplicados. Es el eje de toda arquitectura idempotente: el sistema **no puede confiar en el payload** para detectar duplicados (dos transacciones legítimas distintas pueden tener exactamente los mismos valores), así que necesita este identificador dedicado.

## Quién la genera y cuándo

- **El cliente** es el responsable estricto de generarla.
- Se genera **antes** del primer intento de request.
- Se inyecta en los **headers** del request (un header es metadata que acompaña al payload).
- **Crucial**: ante un retry por fallo de red, el cliente debe adjuntar la **misma key exacta** al reintento. Si cambia la key, el servidor lo ve como una operación nueva.

## Qué se usa como key — UUID

- Los ingenieros usan abrumadoramente un **[[UUID]]** (Universally Unique Identifier).
- Es un label algorítmico estandarizado con **unicidad matemática**: un cliente puede generar millones sin crear nunca un duplicado.

## Validación de integridad — payload hashing

> [!warning] La key sola no alcanza
> Un cliente con bugs puede reintentar con la **misma key pero un payload alterado**. Si el servidor solo chequea la key, devolvería la respuesta cacheada de una operación distinta. Por eso se valida con **payload hashing**: el servidor hashea el payload original y lo guarda junto a la key; cuando llega un duplicado, hashea el nuevo payload y compara. Si los hashes **no coinciden**, rechaza el request.

## Reglas de comunicación

- Si llega un request **sin** el header de la key → el servidor lo rechaza con **HTTP 400 (bad request)**.
- El servidor debe enforzar agresivamente la presencia de la key.

## References

- Fuente: [System Design Deep Dive: Architecting Idempotent APIs](https://designgurus.substack.com/p/system-design-deep-dive-architecting) — Arslan Ahmad (Design Gurus), 2026-03-05

## Related

- [[Idempotency]]
- [[Idempotency Architecture]]
- [[Distributed Lock]]
- [[UUID]]
