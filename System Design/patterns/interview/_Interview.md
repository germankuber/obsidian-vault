---
title: Interview — Patrones para entrevistas de System Design
created: 2026-07-02
tags:
  - system-design/patterns
  - system-design/interview
  - type/moc
  - status/permanent
aliases:
  - Interview
  - Interview Patterns
  - Patrones de Entrevista
  - System Design Interview
updated: 2026-07-02
---

# Interview — Patrones para entrevistas de System Design

> [!note] Cómo usar esta nota
> Es el índice (MOC) de los **7 patrones estilo HelloInterview** compilados para preparar entrevistas de System Design (en español rioplatense). Cada nota copia el material original completo y le fusiona datos, números y casos reales investigados online. Están ordenadas en una **secuencia de lectura pedagógica**: de cómo escalar lo básico, a coordinar procesos distribuidos. Leelas en orden la primera vez; después usá esta nota como acceso rápido.

## 📚 Orden de lectura sugerido

1. **[[Scaling Reads]]** — el primer cuello de botella de casi todo sistema (ratio read:write 10:1 a 100:1). La escalera: índices/denormalización → read replicas → sharding → cache (Redis, CDN). Deep-dives: thundering herd, hot keys, invalidación.
2. **[[Scaling Writes]]** — el lado difícil de escalar: toda escritura va a la fuente de verdad. Vertical scaling + motor LSM → sharding con buena shard key → colas + load shedding → batching/agregación. Deep-dives: resharding (consistent hashing) y hot partitions.
3. **[[Dealing with Contention]]** — qué pasa cuando muchos clientes tocan el mismo dato a la vez (read-modify-write). Escalera: conditional write → optimistic (OCC) → pessimistic → distributed lock. Deep-dives: deadlocks, problema ABA, hot resource.
4. **[[Real-Time Updates]]** — cómo el servidor empuja datos al cliente (chat, dashboards, colaboración). Dos hops: protocolo (polling → long polling → SSE → WebSocket → WebRTC) y distribución interna (pub/sub vs consistent hashing). Deep-dives: reconexión, celebrity/fan-out, ordering.
5. **[[Long-Running Tasks]]** — operaciones pesadas fuera del request (encoding, reportes, bulk). Encolar → job ID → workers en background → polling/webhook. Deep-dives: idempotencia (at-least-once + dedup), DLQ, backpressure, dependencias.
6. **[[Multi-Step Processes]]** — coordinar procesos que cruzan servicios sin transacción ACID global. Transacción → [[Saga]] (coreografía u orquestación) → durable execution engine (Temporal, Step Functions). Reusa la idempotencia de [[Long-Running Tasks]]. Deep-dives: crash a mitad de camino, versioning, exactly-once.
7. **[[Large Blobs]]** — manejar archivos grandes sin que pasen por tus servers (presigned URLs, multipart upload, CDN). Combina varios patterns: un video usa Large Blobs + [[Long-Running Tasks]] (transcoding) + [[Real-Time Updates]] (progreso).

## 🧭 Mapa rápido por eje

| Si el problema es... | Pattern |
|---|---|
| Muchas lecturas, pocas escrituras | [[Scaling Reads]] |
| Millones de escrituras/seg | [[Scaling Writes]] |
| "El último asiento", contadores, balances | [[Dealing with Contention]] |
| El server necesita empujar al cliente | [[Real-Time Updates]] |
| Tarea lenta que no cabe en el request | [[Long-Running Tasks]] |
| Workflow de pasos que cruzan servicios | [[Multi-Step Processes]] |
| Videos, imágenes, backups, archivos grandes | [[Large Blobs]] |

## 🔗 Conexión con el resto del vault

- Volvé al MOC del dominio: [[_System Design|System Design]] — el mapa completo de patrones.
- Patterns relacionados que aparecen cruzados: [[Saga]], [[Distributed Lock]], [[Consistent Hashing]], [[Sharding]], [[Message Queue]], [[Content Delivery Network]], [[Server-Sent Events]], [[Idempotency]].

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "System Design/patterns/interview"
WHERE file.name != this.file.name AND !contains(file.tags, "type/moc")
SORT file.name ASC
```
