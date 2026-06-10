---
title: REST API
source: https://designgurus.substack.com/p/what-is-a-rest-api-a-system-design
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/api
  - type/pattern
  - status/permanent
aliases:
  - REST API
  - REST
  - Representational State Transfer
---

# REST API

> [!note] Definición
> **API** = Application Programming Interface: un **contrato digital** entre dos programas independientes que dicta qué requests se pueden hacer y cómo se formatean los datos. **REST** = Representational State Transfer: **no es una librería ni un lenguaje**, sino un **estilo arquitectónico** para diseñar aplicaciones en red. Cuando una interfaz sigue los principios de REST, es una REST API.

## La idea central

- Todo se trata como un **recurso digital**. Cuando el client necesita interactuar con un recurso, le pide al server una **representación de su estado actual**. El server empaqueta ese estado en texto y lo transfiere por la red.
- La mayoría de las implementaciones modernas usan **JSON** (JavaScript Object Notation): formato liviano de pares clave-valor en texto, fácil de parsear para máquinas y legible para humanos.
- Resuelve el problema central de los sistemas distribuidos: frontends y backends son entidades **completamente independientes** (distintos lenguajes y lógica); sin un protocolo estandarizado, conectarlos requiere código custom frágil.

## Los 5 principios

REST se define por cinco constraints (detalle en [[REST Constraints]]): **Client-Server Separation**, **Statelessness** (el más crítico), **Cacheability**, **Uniform Interface**, **Layered System**.

## El ciclo request/response

El client **siempre inicia** la comunicación; el server reacciona. Toda interacción empieza con un **HTTP request** a un destino llamado **endpoint**.

### Anatomía del request (4 componentes)

- **Endpoint / resource path** — una URL que apunta a un recurso. Convención: **sustantivos en plural, no verbos** (`users`, no `fetchUsers`). Es la dirección digital del recurso; la acción la define el método.
- **[[HTTP Methods|HTTP method]]** — la acción (GET / POST / PUT / DELETE), que mapea a operaciones de DB.
- **Headers** — metadata oculta con contexto operativo: credenciales de seguridad (tokens de autenticación) y qué formato de datos espera recibir el client.
- **Body / payload** — los datos que se transfieren. En POST/PUT, contiene la información nueva a guardar, típicamente en JSON.

### Anatomía de la response

- Un **[[HTTP Status Codes|status code]]** de 3 dígitos (200/201 éxito, 4xx error del client, 5xx error del server).
- Un **body**: si el request fue exitoso, el server consulta la DB, formatea el resultado en un objeto JSON limpio, y el client lo parsea a variables usables.

## Conceptos avanzados (system design)

Para sistemas distribuidos grandes, el ciclo básico no alcanza:

- **[[Pagination]]** — una DB con millones de registros no puede devolverlos todos en una response (consume bandwidth, crashea la memoria del client). Se divide en chunks con parámetros `limit` y `offset` en la URL.
- **[[Rate Limiting]]** — restringe cuántos requests hace un client en una ventana de tiempo, para proteger el backend de sobrecarga. Excederlo devuelve **429 Too Many Requests**.
- **[[API Versioning]]** — incluir un número de versión en la URL; al hacer cambios estructurales, se publica una versión nueva junto a la vieja para no romper clients existentes.
- **[[Idempotency]]** — ejecutar una operación N veces produce el mismo estado que ejecutarla una. Crítico ante redes inestables: si un POST se manda pero la red cae antes de la confirmación, el client reintenta seguro sin crear duplicados.

## References

- Fuente: [The Anatomy of a REST API: Client-Server Architecture Explained](https://designgurus.substack.com/p/what-is-a-rest-api-a-system-design) — Arslan Ahmad (Design Gurus), 2026-02-19

## Related

- [[REST Constraints]]
- [[HTTP Methods]]
- [[HTTP Status Codes]]
- [[Pagination]]
- [[Rate Limiting]]
- [[API Versioning]]
- [[Idempotency]]
