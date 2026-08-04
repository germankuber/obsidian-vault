---
title: Patterns — Mapa del tema
created: 2026-07-05
updated: 2026-07-05
tags:
  - system-design
  - type/moc
  - status/permanent
aliases:
  - Patterns
  - System Design Patterns
  - Patrones de System Design MOC
---

# Patterns — Mapa del tema

> [!note] Cómo usar esta nota
> Índice (MOC) de `System Design/patterns`: los patrones fundamentales de diseño de sistemas, agrupados por el problema que resuelven. Empezá por la categoría que te interesa. Abrí esta nota, no la carpeta.

## 🌐 API Design

- [[REST API]] — contrato digital entre programas: qué requests se pueden hacer y qué respuestas esperar.
- [[REST Constraints]] — los cinco constraints que hacen a un sistema verdaderamente RESTful.
- [[HTTP Methods]] — las acciones estandarizadas (GET/POST/PUT/DELETE) que mapean a operaciones de datos.
- [[HTTP Status Codes]] — los números de tres dígitos que indican el resultado de un request.
- [[API Gateway]] — punto de entrada único que rutea, autentica, limita y transforma requests.
- [[API Versioning]] — mantener varias versiones a la vez para no romper clientes viejos.
- [[Backend for Frontend]] — una capa de API separada por tipo de cliente (móvil vs web).
- [[Rate Limiting]] — restringir cuántos requests hace un cliente por ventana (token bucket, sliding window).
- **Paginación**: [[Pagination]] — cómo una API devuelve listas grandes en pedazos · [[Offset Pagination]] (page+size) · [[Cursor Pagination]] (cursor opaco) · [[Keyset Pagination]] (seek a nivel DB).

## 🛡️ Resiliencia y tolerancia a fallos

- [[Circuit Breaker]] — corta las llamadas a una dependencia que falla repetidamente y devuelve fallback.
- [[Retry with Backoff]] — reintentar con demoras crecientes + jitter en vez de inmediatamente.
- [[Timeout]] — duración máxima para una llamada externa; si se pasa, aborta con fallback.
- [[Bulkhead]] — aislar cargas en pools de recursos separados para que una no hunda a las demás.
- [[Graceful Degradation]] — ante fallas no críticas, servir una experiencia degradada pero funcional.
- [[Health Check]] — endpoint `/health` que balanceadores y orquestadores consultan para rutear.
- [[Dead Letter Queue]] — mensaje que no se pudo procesar tras reintentos se mueve a una cola aparte.
- [[Idempotency]] — ejecutar N veces produce el mismo resultado que una vez (base de los retries seguros).
  - [[Idempotency Key]] — el string único que identifica requests duplicados.
  - [[Idempotency Architecture]] — el diseño server-side que hace seguros los retries automáticos.

## 📈 Escalado

- [[Horizontal Scaling]] — más máquinas (scale out); 10 servidores aguantan ~10×.
- [[Vertical Scaling]] — una máquina más potente (scale up): más CPU/RAM/disco.
- [[Auto-Scaling]] — sumar/quitar instancias automáticamente según métrica (CPU, tráfico).
- [[Load Balancing]] — distribuir requests entre servidores (round-robin, least connections, IP hash).
- [[Consistent Hashing]] — agregar/quitar un nodo solo remapea una fracción de las claves.
- [[Sharding]] — partir los datos entre servidores; una shard key decide dónde vive cada registro.
- [[Content Delivery Network]] — servir contenido estático desde el edge más cercano al usuario.
- [[Reverse Proxy]] — se sienta entre clientes y backends (SSL, compresión, caché, ruteo).
- **Casos de escalado**: [[Scaling Reads]] · [[Scaling Writes]] · [[Dealing with Contention]] · [[Large Blobs]] · [[Long-Running Tasks]] · [[Multi-Step Processes]] · [[Real-Time Updates]].

## 💾 Datos, consistencia y replicación

- [[Primary-Replica]] — el primario toma escrituras; las réplicas copian y sirven lecturas.
- [[Quorum]] — con N réplicas, exigir W + R > N para garantizar consistencia.
- [[Vector Clocks]] — rastrear causalidad con un vector de timestamps lógicos por nodo.
- [[Two-Phase Commit]] — transacción atómica entre participantes (prepare → commit).
- [[Saga]] — secuencia de transacciones locales con compensaciones si un paso falla.
- [[Distributed Lock]] — garantiza que solo un thread modifique una key a la vez.
- [[Write-Ahead Log]] — escribir al log secuencial antes de aplicar, para recuperar tras un crash.
- [[CQRS]] — separar el modelo de escritura del de lectura.
- [[Event Sourcing]] — guardar la secuencia de eventos en vez del estado actual.

## 🗄️ Caching

- [[Cache-Aside]] — la app mira la cache primero; en miss lee la base y la puebla.
- [[Read-Through]] — la cache misma carga el dato en un miss; la app solo habla con la cache.
- [[Write-Through]] — cada escritura va a la vez a cache y base (cache siempre fresca).
- [[Write-Behind]] — la escritura va a la cache y se confirma ya; vuelca a la base asíncrono.
- [[Cache Stampede Prevention]] — evitar que miles de requests golpeen la base cuando expira una key popular.

## 📨 Comunicación y mensajería

- [[Request-Response]] — el cliente manda y espera la respuesta (REST, gRPC).
- [[Message Queue]] — el productor encola; el consumidor procesa a su ritmo; un mensaje = un consumidor.
- [[Pub-Sub]] — el publisher emite a un topic; varios subscribers reciben cada mensaje.
- [[Event-Driven Architecture]] — los servicios emiten y reaccionan a eventos en vez de llamarse directo.
- [[Webhooks]] — el servidor empuja el evento a una URL registrada en vez de que el cliente haga polling.
- [[Server-Sent Events]] — push unidireccional servidor→cliente sobre HTTP de larga duración.
- [[Bidirectional Streaming]] — conexión persistente donde ambos lados mandan en cualquier momento.
- [[Connection Pooling]] — reutilizar conexiones a la base en vez de abrir una por request.

## 🔀 Procesamiento de datos

- [[MapReduce]] — partir un dataset entre máquinas (map), procesar, y combinar (reduce).
- [[Stream Processing]] — procesar evento por evento a medida que llegan, latencia sub-segundo.
- [[Lambda Architecture]] — batch (exacto, lento) + stream (aproximado, rápido) en paralelo.

## 🕸️ Service Mesh

- [[Service Mesh]] — capa de infra dedicada a la comunicación servicio-a-servicio (tráfico este-oeste).
- [[Data Plane vs Control Plane]] — los sidecars (data) vs el cerebro que los configura (control).
- [[Sidecar]] — proxy ligero junto a cada servicio que maneja mTLS, retries, métricas.
- [[Service Discovery]] — cómo un servicio encuentra instancias sanas de otro sin IPs hardcodeadas.
- [[North-South vs East-West Traffic]] — externo↔interno (N-S) vs servicio↔servicio (E-W).
- [[Mutual TLS]] — ambos lados se autentican con certificados (no solo el servidor).
- [[Traffic Management]] — control fino de cómo fluye el tráfico interno (canary, splitting).
- [[Kubernetes Gateway API]] — estándar vendor-neutral para routing N-S y E-W en Kubernetes.
- **Implementaciones**: [[Istio]] (la más adoptada, usa [[Envoy]]) · [[Linkerd]] (liviana) · [[Cilium]] (eBPF, sin sidecar) · [[Envoy]] (el proxy sidecar de facto).

## ☁️ Serverless

- [[Serverless]] — el desarrollador no gestiona servidores; el hardware existe pero es invisible.
- [[AWS Lambda]] — el servicio de cómputo serverless de AWS.
- [[Cold Start]] — la latencia de despertar una función dormida (el trade-off principal).
- [[Warm Start]] — reutilizar un contenedor ya inicializado (respuesta en ms).
- [[Scaling to Zero]] — remover todo el cómputo cuando no hay tráfico.
- [[Stateless]] — no retener memoria local entre ejecuciones (requisito del escalado horizontal).
- [[Infrastructure as Code]] — gestionar infra con archivos de definición legibles por máquina.

## 🚀 Deployment y observabilidad

- [[Canary Deployment]] — desplegar la versión nueva a un % chico de tráfico y monitorear antes de expandir.
- [[Distributed Tracing]] — seguir un request por múltiples servicios con spans en un trace compartido.
- [[Networking Essentials]] — la red como pila de abstracciones; cada capa resuelve un problema.

## 🎯 Interview

- Ver la subcarpeta `interview/` para hojas de estudio y breakdowns de problemas concretos.

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "System Design/patterns"
WHERE file.name != this.file.name AND type != "moc"
SORT file.name ASC
```
