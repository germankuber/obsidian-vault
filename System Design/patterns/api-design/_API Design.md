---
title: API Design — Mapa del tema
created: 2026-06-10
tags:
  - system-design/api
  - moc
aliases:
  - API Design MOC
  - API Design Index
---

# API Design — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del subtema *API design* dentro de [[_System Design|System Design]]: cómo diseñar las interfaces que exponen los servicios. Abrí esta nota, no la carpeta.

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

- [[REST]] · [[GraphQL]] · [[gRPC]] · [[Webhooks]] (esta última ya existe en `patterns/`) · [[OpenAPI]] — estilos y contratos de API, candidatos a promover.

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "System Design/patterns/api-design"
WHERE file.name != this.file.name AND !contains(file.path, "pagination")
SORT file.name ASC
```
