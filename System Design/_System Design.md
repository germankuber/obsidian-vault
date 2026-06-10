---
title: System Design — Mapa de patrones
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design
  - moc
aliases:
  - System Design
  - System Design Patterns
  - Patrones de System Design
---

# System Design — Mapa de patrones

> [!note] Cómo usar esta nota
> Es el índice (MOC) raíz del vault de System Design. Cataloga **50 patrones**
> agrupados en 10 categorías. Cada patrón tiene su **nota completa** con
> definición, cómo funciona, cuándo usarlo y —lo más valioso— **cuándo NO usarlo
> y sus trade-offs**. Clickeá cualquier `[[patrón]]` para abrir su nota. Empezá
> por el modelo mental y la lista de alta frecuencia.
>
> Las notas viven en `patterns/`. Las frases acá son solo el resumen de una línea.

> [!tip] Modelo mental para decidir qué patrón aplicar
> - **¿Cómo fluyen los datos?** → patrones de Comunicación
> - **¿Cómo se almacenan?** → patrones de Almacenamiento
> - **¿Cómo se accede rápido?** → patrones de Caching
> - **¿Cómo sobrevive a fallas?** → patrones de Confiabilidad
> - **¿Cómo escala?** → patrones de Escalado

## ⭐ Alta frecuencia (los 15 que más caen)

Los más pedidos en entrevistas, según la fuente: [[Primary-Replica]],
[[Sharding]], [[Consistent Hashing]], [[Cache-Aside]],
[[Cache Stampede Prevention]], [[Message Queue]], [[Pub-Sub|Pub/Sub]],
[[Circuit Breaker]], [[Retry with Backoff]], [[Idempotency]],
[[Horizontal Scaling]], [[Load Balancing]], [[Auto-Scaling]], [[API Gateway]],
[[Rate Limiting]].

## 💾 Almacenamiento

- [[Primary-Replica]] — un primario toma todas las escrituras; las réplicas
  copian y sirven lecturas. Si cae el primario, una réplica lo reemplaza.
- [[Sharding]] — los datos se reparten entre varios servidores; una *shard key*
  decide qué servidor guarda qué.
- [[Consistent Hashing]] — agregar/quitar un servidor solo remapea una fracción
  de las claves; servidores y claves viven en un anillo virtual.
- [[Write-Ahead Log]] — toda operación se escribe primero en un log secuencial;
  ante un crash, se reproduce el log para recuperar.
- [[Event Sourcing]] — en vez del estado actual, se guarda la secuencia de
  eventos; el estado se deriva reproduciéndolos.
- [[CQRS]] — separar el modelo de escritura (consistencia) del de lectura
  (consultas rápidas), incluso en bases distintas.

## ⚡ Caching

- [[Cache-Aside]] — mirar la cache primero; si falla, leer de la base, devolver y
  poblar la cache.
- [[Write-Through]] — cada escritura va a cache y base a la vez; la cache siempre
  está al día.
- [[Write-Behind]] — escribir a la cache primero; la cache vuelca a la base en
  lotes asíncronos (escrituras rápidas).
- [[Read-Through]] — la cache misma carga de la base en un miss; la app solo
  habla con la cache.
- [[Cache Stampede Prevention]] — evitar que, al expirar una entrada popular,
  miles de requests peguen a la base a la vez (coalescing, expiración
  probabilística, locks).

## 📡 Comunicación

- [[Request-Response]] — el cliente pide y espera la respuesta; el patrón más
  simple (REST, gRPC).
- [[Message Queue]] — el productor encola un mensaje; el consumidor lo procesa a
  su ritmo sin que el productor espere.
- [[Pub-Sub|Pub/Sub]] — el publisher emite a un *topic*; varios subscribers reciben cada
  mensaje (a diferencia de una cola, donde cada mensaje va a uno solo).
- [[Event-Driven Architecture]] — los servicios se comunican emitiendo y
  reaccionando a eventos, no con llamadas directas.
- [[Webhooks]] — en vez de hacer *polling*, el servidor empuja eventos a una URL
  que el cliente proveyó.
- [[Server-Sent Events]] — push unidireccional servidor→cliente sobre una
  conexión HTTP larga.
- [[Bidirectional Streaming]] — conexión persistente donde ambos lados mandan
  mensajes cuando quieren (WebSockets, gRPC bidireccional).

## 🛡️ Confiabilidad

- [[Circuit Breaker]] — si un servicio downstream falla repetido, dejar de
  llamarlo y devolver un fallback; reintentar recuperación cada tanto.
- [[Retry with Backoff]] — ante una falla, reintentar con demoras crecientes
  (1s, 2s, 4s…) + *jitter* para evitar estampidas.
- [[Bulkhead]] — aislar cargas en pools de recursos separados, para que una no
  agote los recursos de las otras.
- [[Timeout]] — fijar una duración máxima para llamadas externas; abortar y
  devolver error/fallback si se excede.
- **[[_Idempotency|Idempotency]]** 📁 — subtema con carpeta propia (`idempotency/`): diseñar operaciones para que ejecutarlas N veces dé el mismo resultado que ejecutarlas una. El concepto es [[Idempotency]]; la carpeta tiene la [[Idempotency Key|key]] y la [[Idempotency Architecture|arquitectura del servidor]]. Abrí su MOC.
- [[Dead Letter Queue]] — si un mensaje no se puede procesar tras varios
  reintentos, moverlo a una cola aparte en vez de bloquear la principal.
- [[Graceful Degradation]] — si fallan componentes no críticos, seguir sirviendo
  una experiencia degradada pero funcional.

## 📈 Escalado

- [[Horizontal Scaling]] — agregar más máquinas; 10 servidores aguantan ~10× el
  tráfico de uno.
- [[Vertical Scaling]] — pasar a una máquina más potente (más CPU/RAM/disco).
- [[Load Balancing]] — repartir requests entre servidores (round-robin, least
  connections, weighted, IP hash).
- [[Auto-Scaling]] — agregar/quitar instancias según el tráfico (sube si la CPU
  pasa un umbral, baja si cae).
- [[Connection Pooling]] — reutilizar un pool de conexiones a la base en vez de
  abrir una por request.

## 🔄 Procesamiento de datos

- [[MapReduce]] — partir un dataset grande entre máquinas (map), procesar
  independiente, combinar (reduce). La base del batch.
- [[Stream Processing]] — procesar evento por evento al llegar, con latencia
  sub-segundo (Kafka Streams, Flink, Spark Streaming).
- [[Lambda Architecture]] — correr batch (preciso, lento) y stream (aproximado,
  rápido) en paralelo; una capa de serving fusiona ambos.
- [[Change Data Capture]] — capturar cambios de la base (insert/update/delete)
  como stream de eventos al que otros se suscriben.

## 🔌 API

- [[API Gateway]] — punto de entrada único que rutea, autentica, limita y
  transforma requests antes de pasarlos al backend.
- [[Backend for Frontend]] — una capa de API por tipo de cliente (móvil liviano,
  web más rico).
- [[Rate Limiting]] — limitar cuántos requests hace un cliente por ventana de
  tiempo (token bucket, fixed/sliding window).
- [[Cursor Pagination]] — paginar con un cursor opaco que apunta al último ítem;
  el cliente lo pasa para la página siguiente.
- [[API Versioning]] — mantener varias versiones a la vez para que los clientes
  viejos sigan funcionando.

## 🏗️ Infraestructura

- [[Content Delivery Network]] — distribuir contenido estático a *edge servers*
  globales; sirve desde el más cercano, bajando latencia.
- [[Reverse Proxy]] — servidor entre clientes y backends que hace SSL
  termination, compresión, cache y ruteo.
- [[Service Mesh]] — capa dedicada para comunicación servicio-a-servicio donde un
  *sidecar* maneja LB, retries, circuit breaking, mTLS y observabilidad.
- [[Sidecar]] — desplegar un proceso ayudante junto al servicio principal para
  *cross-cutting concerns* sin tocar el código del servicio.

## ✅ Consistencia

- [[Two-Phase Commit]] — protocolo de transacción atómica entre participantes:
  fase 1 pregunta "¿podés commitear?"; fase 2 commitea si todos dijeron que sí.
- [[Saga]] — secuencia de transacciones locales; si un paso falla, transacciones
  *compensatorias* deshacen los anteriores.
- [[Quorum]] — con N réplicas, exigir W para escribir y R para leer con W + R > N,
  garantizando que las lecturas vean la última escritura.
- [[Vector Clocks]] — rastrear causalidad con un vector de timestamps lógicos por
  nodo, para ordenar eventos o detectar concurrencia.
- [[Distributed Lock]] — garantizar que solo un thread manipule un recurso a la vez (unique constraint); resuelve race conditions. Clave en la [[Idempotency Architecture|arquitectura idempotente]].

## 👁️ Observabilidad

- [[Health Check]] — cada servicio expone `/health`; LBs y orquestadores lo
  consultan para saber si está vivo y listo.
- [[Distributed Tracing]] — seguir un request por varios servicios; cada uno
  agrega un *span* con timestamp, mostrando el camino y los cuellos de botella.
- [[Canary Deployment]] — desplegar la versión nueva a un subconjunto chico,
  rutear 1-5% del tráfico, monitorear y aumentar gradual o hacer rollback.

## 🔗 Conexión con otros dominios

Estos patrones se cruzan con el mapa de [[_RAG|RAG]]:

- [[Change Data Capture]] — su nota vive en `ai/rag/` (donde apareció primero),
  cubre tanto la ingesta de RAG como el patrón general de datos.
- [[Server-Sent Events]] — el streaming de la respuesta del [[Enterprise RAG Assistant]].

Todos los 50 patrones tienen nota completa con trade-offs. La próxima vez que un
artículo aporte más profundidad sobre uno, su nota se enriquece (no se duplica).
