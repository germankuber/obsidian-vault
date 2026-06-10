---
title: HTTP Status Codes
source: https://designgurus.substack.com/p/what-is-a-rest-api-a-system-design
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/api
aliases:
  - HTTP Status Codes
  - Status Codes
  - Códigos de Estado HTTP
---

# HTTP Status Codes

> [!note] Definición
> Números estandarizados de **tres dígitos** que indican el resultado final de un request. Se agrupan en **rangos** que dicen de un vistazo qué pasó. El client los lee para saber si reintentar, corregir el request, o procesar la respuesta.

## 2xx — Éxito

- **200 OK** — el request fue exitoso y el server devuelve los datos pedidos.
- **201 Created** — un recurso nuevo se generó exitosamente tras un [[HTTP Methods|POST]].

## 4xx — Error del cliente

Implica que el client formateó mal el request o pidió datos inválidos. **Requieren que el client corrija el request.**

- **400 Bad Request** — el payload JSON tiene errores de sintaxis.
- **401 Unauthorized** — el client no proveyó credenciales de autenticación válidas.
- **404 Not Found** — el client pidió un endpoint o recurso que **no existe** en el server.
- **429 Too Many Requests** — el client excedió el límite de requests permitidos (ver [[Rate Limiting]]). Los headers suelen incluir un timestamp de cuándo puede reanudar.

## 5xx — Error del servidor

El client mandó un request perfectamente válido, pero el server falló al procesarlo por un problema interno.

- **500 Internal Server Error** — el código del backend crasheó fatalmente o encontró un bug inesperado de la DB.

## References

- Fuente: [The Anatomy of a REST API](https://designgurus.substack.com/p/what-is-a-rest-api-a-system-design) — Arslan Ahmad (Design Gurus), 2026-02-19

## Related

- [[REST API]]
- [[HTTP Methods]]
- [[Rate Limiting]]
