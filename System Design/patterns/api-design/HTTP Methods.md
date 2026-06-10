---
title: HTTP Methods
source: https://designgurus.substack.com/p/what-is-a-rest-api-a-system-design
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/api
aliases:
  - HTTP Methods
  - HTTP Verbs
  - Métodos HTTP
---

# HTTP Methods

> [!note] Definición
> Las **acciones estandarizadas** que un client puede pedirle a un server. Mapean directamente a operaciones de base de datos (leer, escribir, borrar). Junto con el [[REST API|endpoint]], definen qué quiere hacer el client: la URL dice **dónde** está el recurso, el método dice **qué** hacer con él.

## Los cuatro métodos principales

- **GET** — recuperar datos. Operación **estrictamente read-only**: nunca debe alterar registros. Devuelve el estado JSON actual del recurso.
- **POST** — **crear** un recurso nuevo. El client manda un payload de datos nuevos y el server genera una entrada nueva en su DB. Ej: registrar un perfil nuevo.
- **PUT** — **actualizar** un recurso existente **por completo**. El client manda un payload completo que **reemplaza** el recurso en la DB. Ej: guardar la edición de un perfil.
- **DELETE** — **eliminar** permanentemente un recurso específico de la DB. El server encuentra el recurso destino y lo borra.

## Convención de diseño de endpoints

- Al diseñar endpoints se usan **sustantivos en plural, no verbos de acción**: la URL apunta a un recurso `users`, no a una acción como `fetchUsers`. La acción la define el método HTTP, no la URL.

## Relación con otros conceptos

- POST/PUT pueden necesitar [[Idempotency|idempotencia]] para que un retry no cree duplicados.
- La respuesta a cada método trae un [[HTTP Status Codes|status code]] (200 OK para GET exitoso, 201 Created para POST, etc.).

## References

- Fuente: [The Anatomy of a REST API](https://designgurus.substack.com/p/what-is-a-rest-api-a-system-design) — Arslan Ahmad (Design Gurus), 2026-02-19

## Related

- [[REST API]]
- [[HTTP Status Codes]]
- [[REST Constraints]]
- [[Idempotency]]
