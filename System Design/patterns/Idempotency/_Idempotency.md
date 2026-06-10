---
title: Idempotency — Mapa del tema
created: 2026-06-10
tags:
  - system-design/resilience
  - type/moc
  - status/permanent
aliases:
  - Idempotency MOC
  - Idempotency Index
updated: 2026-06-10
---

# Idempotency — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del subtema *idempotency* dentro de [[_System Design|System Design]]. Empezá por el concepto y bajá al mecanismo y la arquitectura. Abrí esta nota, no la carpeta.

## 🚀 Empezá por acá

- [[Idempotency]] — **el concepto.** Ejecutar N veces = ejecutar una vez. Por qué los sistemas distribuidos lo necesitan, operaciones seguras (GET/PUT/DELETE) vs inseguras (POST), y los trade-offs.

## 🔑 El mecanismo

- [[Idempotency Key]] — el identificador único que el cliente genera y reusa en cada retry; UUID, headers, payload hashing, HTTP 400 si falta.

## 🏗️ La arquitectura del lado servidor

- [[Idempotency Architecture]] — el deep-dive: in-memory datastore, tracking record (key + status + respuesta cacheada), el flujo de 4 pasos, estados processing/completed/failed, TTL 24h, HTTP 409 si in-progress, manejo de fallas internas.

## 🔗 Conexión

- [[Distributed Lock]] (en `patterns/`) — resuelve las race conditions de la arquitectura idempotente; concepto de concurrencia reusable más allá de idempotencia.
- [[Retry with Backoff]] — la razón principal por la que existe la idempotencia.
- [[Timeout]] · [[Message Queue]] · [[Webhooks]] — todos generan reenvíos que exigen idempotencia.

## 🌱 Por escribir (semillas del grafo)

- [[UUID]] — el identificador que se usa como key (el artículo enlaza a un post propio sobre por qué no usar UUIDs estándar como primary key).
- [[Race Condition]] · [[Payload Hashing]] — hoy viven inline en las notas de arriba; promover si un futuro artículo aporta material.

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "System Design/patterns/Idempotency"
WHERE file.name != this.file.name
SORT file.name ASC
```
