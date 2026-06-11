---
title: System Design — Mapa del tema
created: 2026-06-11
tags:
  - system-design/patterns
  - type/moc
  - status/permanent
aliases:
  - System Design
  - System Design MOC
  - Patrones de Diseño de Sistemas
---

# System Design — Mapa del tema

> [!note] Cómo usar esta nota
> Es el índice (MOC) del dominio **System Design**: patrones de diseño de sistemas agrupados por tema. Empezá por la sección que te interese y bajá. Los **subtemas con carpeta propia** (📁) tienen su propio MOC — abrilo para el detalle. Abrí esta nota, no la carpeta.

## 🌐 Comunicación y API

- **[[_API Design|API Design]]** 📁 — subtema con carpeta propia (`API Design/`): cómo diseñar las interfaces que exponen los servicios. REST, HTTP methods/status, versioning, BFF, rate limiting + [[_Pagination|Pagination]]. Abrí su MOC.
- **[[_Service Mesh|Service Mesh]]** 📁 — subtema con carpeta propia (`Service Mesh/`): cómo se gestiona el tráfico *este-oeste* (servicio↔servicio) en microservicios. Fundamento [[North-South vs East-West Traffic]], mecanismos ([[Sidecar]], [[Data Plane vs Control Plane]]), capacidades ([[Mutual TLS]], [[Service Discovery]], [[Traffic Management]]) e implementaciones ([[Istio]]/[[Linkerd]]/[[Cilium]]/[[Envoy]]). Abrí su MOC.
- [[API Gateway]] — punto de entrada único (tráfico norte-sur): rutea, autentica, limita y transforma.
- [[Reverse Proxy]] — intermediario que recibe requests y los reenvía a backends.
- [[Message Queue]] — desacoplar productores y consumidores con una cola intermedia.
- [[Pub-Sub]] — publicar eventos a múltiples suscriptores sin acoplarlos.
- [[Request-Response]] — el modelo síncrono clásico cliente↔servidor.
- [[Webhooks]] — notificaciones HTTP salientes ante un evento.
- [[Server-Sent Events]] — stream unidireccional servidor→cliente sobre HTTP.
- [[Bidirectional Streaming]] — canal full-duplex (ej. gRPC/WebSocket) entre cliente y servidor.

## 📨 Eventos y procesamiento de datos

- [[Event-Driven Architecture]] — sistemas que reaccionan a eventos en vez de llamadas directas.
- [[Event Sourcing]] — guardar el estado como una secuencia de eventos, no como snapshot.
- [[CQRS]] — separar el modelo de escritura del de lectura.
- [[Stream Processing]] — procesar datos en flujo continuo (no por lotes).
- [[Lambda Architecture]] — combinar capa batch + capa de streaming.
- [[MapReduce]] — procesar datasets grandes en map + reduce paralelizable.
- [[Saga]] — transacciones distribuidas como pasos compensables.
- [[Two-Phase Commit]] — coordinar un commit atómico entre varios nodos.
- [[Dead Letter Queue]] — adónde van los mensajes que no se pudieron procesar.

## 💾 Almacenamiento y datos

- [[Sharding]] — partir los datos horizontalmente entre varios nodos.
- [[Consistent Hashing]] — repartir claves entre nodos minimizando el reshuffle al escalar.
- [[Primary-Replica]] — un primario para escrituras, réplicas para lecturas.
- [[Quorum]] — leer/escribir con mayoría para consistencia en sistemas distribuidos.
- [[Vector Clocks]] — ordenar eventos y detectar conflictos en sistemas distribuidos.
- [[Write-Ahead Log]] — registrar la intención antes de aplicar el cambio (durabilidad).
- [[Distributed Lock]] — coordinar acceso exclusivo a un recurso entre procesos.

## ⚡ Caching

- [[Cache-Aside]] — la app consulta caché y, si falla, va a la DB y la puebla.
- [[Read-Through]] — la caché misma carga el dato de la DB en un miss.
- [[Write-Through]] — escribir a caché y DB en sincronía.
- [[Write-Behind]] — escribir a caché y diferir la escritura a la DB.
- [[Cache Stampede Prevention]] — evitar que muchas requests reconstruyan la misma entrada a la vez.
- [[Content Delivery Network]] — cachear contenido cerca del usuario geográficamente.
- [[Connection Pooling]] — reutilizar conexiones a la DB en vez de abrir una por request.

## 📈 Escalado

- [[Horizontal Scaling]] — agregar más instancias.
- [[Vertical Scaling]] — agrandar la instancia.
- [[Auto-Scaling]] — ajustar la capacidad automáticamente según la carga.
- [[Load Balancing]] — repartir el tráfico entre instancias.
- **[[_Serverless|Serverless]]** 📁 — subtema con carpeta propia (`Serverless/`): scaling-to-zero, cold/warm start, AWS Lambda, IaC. Abrí su MOC.

## 🛡️ Confiabilidad y resiliencia

- [[Circuit Breaker]] — dejar de llamar a un servicio no-sano para evitar fallos en cascada.
- [[Retry with Backoff]] — reintentar con espera creciente.
- [[Timeout]] — cortar una operación que tarda demasiado.
- [[Bulkhead]] — aislar recursos para que una falla no hunda todo el sistema.
- [[Graceful Degradation]] — seguir funcionando con capacidades reducidas ante una falla.
- [[Health Check]] — sondear si una instancia está viva/sana.
- [[Canary Deployment]] — liberar a un % chico antes del rollout completo.
- **[[_Idempotency|Idempotency]]** 📁 — subtema con carpeta propia (`Idempotency/`): operaciones seguras de repetir (idempotency key, arquitectura). Abrí su MOC.

## 🔭 Observabilidad

- [[Distributed Tracing]] — seguir un request a través de múltiples servicios.

## 🔗 Conexión con el resto del vault

- Volvé al [[_Home|Home]] para los otros dominios y los dashboards vivos.
- Cruza con **IA** ([[_AI|AI]]): muchos sistemas LLM (RAG, agentes, MLOps) reusan estos patrones de comunicación, caching, escalado y resiliencia.

## 🔍 Todas las notas del dominio (auto)

```dataview
LIST
FROM "System Design"
WHERE file.name != this.file.name AND !contains(file.tags, "type/moc")
SORT file.path ASC
```
