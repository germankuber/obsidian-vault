---
title: API Design — Mapa del tema
created: 2026-06-10
tags:
  - system-design/api
  - type/moc
  - status/permanent
aliases:
  - API Design MOC
  - API Design Index
updated: 2026-06-10
---

# API Design — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del subtema *API design* dentro de [[_System Design|System Design]]: cómo diseñar las interfaces que exponen los servicios. Abrí esta nota, no la carpeta.

## 🧱 Fundamentos REST

- [[REST API]] — qué es REST (estilo arquitectónico, recursos, JSON) y el ciclo request/response. El punto de partida.
- [[REST Constraints]] — los 5 principios: client-server, statelessness, cacheability, uniform interface, layered system.
- [[HTTP Methods]] — GET/POST/PUT/DELETE y su mapeo a operaciones de DB.
- [[HTTP Status Codes]] — 2xx éxito, 4xx error de cliente, 5xx error de server (200, 201, 400, 401, 404, 429, 500).

## 🚪 Entrada y ruteo

- [[API Gateway]] — punto de entrada único que rutea, autentica, limita y transforma requests antes del backend.
- [[Backend for Frontend]] — una capa de API por tipo de cliente (móvil liviano, web más rico).

## 🛡️ Control de tráfico

- [[Rate Limiting]] — limitar cuántos requests hace un cliente por ventana (token bucket, fixed/sliding window).

## 🔄 Evolución

- [[API Versioning]] — mantener varias versiones a la vez para no romper clientes viejos.

## 📄 Paginación

- **[[_Pagination|Pagination]]** 📁 — subtema con carpeta propia (`pagination/`): devolver listas grandes en pedazos. Offset vs Cursor vs Keyset, decision framework, híbridos. Abrí su MOC.

## 🔗 Conexión

- Subtema de [[_System Design|System Design]]. Cruza con [[Idempotency]] (APIs con efectos secundarios deben ser idempotentes) y [[Distributed Tracing]] (observabilidad de requests).

## 🌱 Por escribir (semillas del grafo)

- [[GraphQL]] · [[gRPC]] · [[OpenAPI]] — otros estilos/contratos de API, candidatos a promover. ([[REST API]] ya tiene nota; [[Webhooks]] existe en `patterns/`.)
- [[JSON]] · [[HTTP]] · [[Stateful vs Stateless]] — conceptos base enlazados desde [[REST API]]/[[REST Constraints]].

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "System Design/patterns/API Design"
WHERE file.name != this.file.name AND !contains(file.path, "pagination")
SORT file.name ASC
```
