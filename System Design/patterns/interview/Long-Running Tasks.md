---
title: Long-Running Tasks
reading:
  total_words: 1820
  read_words: 1820
  pct: 100
  last_read: 2026-07-02
source: HelloInterview — pattern "Managing Long Running Tasks" (adaptado con investigación adicional de AWS/SQS, Sidekiq, Celery/BullMQ, Encore, AbstractAlgorithms, Codelit)
author: Compilado a partir de guías de System Design para entrevistas
created: 2026-07-02
tags:
  - system-design/async
  - type/pattern
  - status/permanent
aliases:
  - Long-Running Tasks
  - Managing Long Running Tasks
  - Tareas de Larga Duración
updated: 2026-07-02
---

> [!note] Definición
> Patrón para operaciones que tardan demasiado para procesarse dentro de un request sincrónico (encoding de video, reportes, envíos masivos, ML). El web server valida y encola el trabajo, devuelve un `job_id` en milisegundos, y workers separados procesan en background mientras el cliente consulta el estado por polling o webhook.

## The Problem

Muchas operaciones tardan **demasiado para procesarse sincrónicamente**: encoding de video, generación de reportes, operaciones bulk, envío masivo de emails, procesamiento de imágenes, entrenamiento de modelos — cualquier tarea que tome más de unos segundos.

Si el usuario tiene que esperar a que termine, la UX es mala y tu web server queda ocupado ejecutando la tarea en vez de atender otras requests. Peor: si el request timeoutea o la conexión se corta, perdés el trabajo.

**La anécdota que resume por qué importa:** en 2018 una plataforma de e-commerce lanzó un flash sale **sin cola** entre el order service y el payment processor. En el pico, 50.000 checkouts simultáneos pegaron a una API de pago licenciada para 5.000 conexiones. El pago devolvía errores, el order service los trataba como fallos, los usuarios volvían a clickear "Comprar", y en 90 segundos el retry storm triplicó la carga. La caída duró 40 minutos y costó ~$800k. Una sola cola entre checkout y pago, con reintentos acotados, habría absorbido el pico y dejado drenar al servicio de pago a su ritmo.

**Señal para reconocer el patrón:** cualquier tarea pesada o lenta que no necesita completarse dentro del request del usuario.

---

## The Solution

El patrón central: cuando el usuario envía una tarea pesada, el web server **valida instantáneamente**, empuja un job a una **cola** (Redis, Kafka, SQS), y devuelve un **job ID en milisegundos**. Procesos **worker** separados van sacando jobs de la cola y ejecutan el trabajo real. El usuario consulta el estado con el job ID o recibe un callback al terminar.

Es "accept the request → enqueue → return job ID → process in background".

---

## Trade-offs

### What you gain

- **Respuesta rápida al usuario:** el request devuelve al instante, sin esperar el trabajo pesado.
- **Escalado independiente:** escalás web servers y workers por separado según su carga.
- **Aislamiento de fallos:** si un worker crashea, el web server sigue atendiendo.
- **Absorción de picos (buffering):** la cola amortigua ráfagas, dejando drenar a los workers a ritmo estable (lo que faltó en el flash sale).

### What you lose

- **Complejidad:** hay que mantener cola, workers, tracking de estado, retries, DLQ.
- **Eventual consistency:** el resultado no está listo al volver el request; el usuario ve "procesando".
- **La trampa nombrada explícitamente:** muchos candidatos meten una cola por reflejo, y **frecuentemente es mala decisión**. Si los jobs son cortos, devolver el estado sincrónicamente con el request simplifica muchísimo la arquitectura y da mejor back-pressure y mejor UX. Solo asincronizás cuando la tarea es genuinamente lenta.

**Frase:** "Async por default para cualquier cosa que el usuario no necesite ver de inmediato; pero si el job corre en menos de un par de segundos, lo dejo síncrono porque simplifica la arquitectura y el back-pressure."

---

## How to Implement

### Message Queue

Un buffer entre productor y consumidor. Opciones y cuándo:
- **Redis (BullMQ, Sidekiq, Celery):** simple, rápido, bueno para colas modestas. Ojo: Redis in-memory sin persistencia (AOF) pierde todos los jobs pendientes si crashea.
- **SQS:** managed, retries y DLQ de fábrica.
- **Kafka:** cuando necesitás retención larga, replay para múltiples consumidores, o throughput ordenado durable a gran escala.

**Regla de tamaño de mensaje (importante):** nunca guardes payloads grandes (>64KB) en el mensaje. Pasá una **referencia** (S3 key, ID de fila) y que el worker traiga el dato. El mensaje describe el trabajo, no lo contiene.

### Workers

Procesos que sacan jobs de la cola y los ejecutan. Escalás **horizontalmente** corriendo más workers en más máquinas (Redis coordina). Consideraciones:
- **Concurrency:** para tareas CPU-bound, matcheá el número de cores; para I/O-bound, podés ir más alto.
- **ACK después de procesar, no antes:** si el worker hace ACK antes de completar y crashea, perdés el job. Configurá "ack late" y "re-queue on worker lost".
- **Visibility timeout:** mientras un worker procesa un mensaje, queda invisible para los demás; el timeout debe ser mayor que el tiempo de procesamiento, para no re-entregarlo prematuramente ni demorarlo de más si el worker muere. Cuando el consumer es Lambda sobre SQS, AWS recomienda configurar el visibility timeout de la cola en al menos **6 veces** el timeout de la función, para dar margen a reintentos internos ante throttling antes de que el mensaje vuelva a quedar visible ([AWS Docs — Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)).
- **Graceful shutdown:** terminar los jobs en curso antes de parar el worker (deploys).

### Putting It Together

Flujo completo: cliente → `POST /job` → web server valida, encola con idempotency key, devuelve `job_id` → worker consume, procesa (trayendo el payload por referencia), actualiza estado → cliente hace polling de `GET /job/{id}` o recibe webhook al terminar. El mejor UX combina **polling + webhooks**.

---

## When to Use in Interviews

### For Example

Encoding/transcoding de video (YouTube), generación de reportes, envío masivo de emails/notificaciones, procesamiento de imágenes (thumbnails, filtros), operaciones bulk, indexado para search, billing reconciliation, cualquier pipeline pesado. Suele combinarse: un video usa Large Blobs (upload) + Long Running Tasks (transcoding) + Real-time Updates (progreso).

**Cuándo NO:** una acción user-facing que necesita completarse de forma inmediata y determinística, donde no tolerás eventual consistency (ej. confirmar el resultado exacto de una operación al instante). Y jobs cortos, donde síncrono es más simple. El default maduro: **híbrido** — síncrono para la confirmación crítica que el usuario necesita, asíncrono para el fan-out y los efectos secundarios no críticos.

---

## Common Deep Dives

### Handling Failures

- **Retries con exponential backoff + jitter:** `delay = min(base * 2^intento, max_delay)`, con jitter aleatorio para no crear un thundering herd de reintentos sincronizados. La variante probada por AWS es "Full Jitter": en vez de sumar jitter al delay exponencial, el delay final se elige *uniformemente al azar entre 0 y el backoff exponencial completo* (`sleep = random_between(0, min(cap, base * 2^intento))`). Su benchmark muestra que este approach reduce el trabajo total del cliente y la carga en el servidor más que "Equal Jitter" o "Decorrelated Jitter", a costa de un pelín más de tiempo total de reintento ([AWS Architecture Blog — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)).
- **Retries durables, no en memoria:** el anti-patrón es un `setTimeout` para reintentar — se pierde si el proceso crashea, no se distribuye, acumula memoria y no da observabilidad. Usá el delay nativo de la cola (SQS DelaySeconds) o re-encolá con delay calculado.
- **Cap de reintentos:** siempre acotá el máximo; reintentos ilimitados causan loops infinitos y agotamiento de recursos.

### Handling Repeated Failures

- **Dead Letter Queue (DLQ):** los jobs que agotan todos los reintentos van a una DLQ para inspección, evitando que un "poison message" bloquee la cola principal o se pierda silenciosamente.
- **La DLQ es una alerta, no un archivo:** su profundidad es un indicador líder de que tu SLO de procesamiento está roto. Hay que monitorearla y tener SLAs de triage, o acumulás deuda de confiabilidad silenciosa.
- **Poison pill detection:** mover a DLQ tras N fallos. Ese N no es arbitrario: AWS sugiere un `maxReceiveCount` mínimo de **5 intentos** en la redrive policy antes de mandar un mensaje a la DLQ, para no "quemar" mensajes que fallan por transitorios (throttling, timeouts de red) en vez de por un bug real del payload ([AWS re:Post — Understand the Amazon SQS dead-letter queue](https://repost.aws/knowledge-center/sqs-dead-letter-queue)).

### Preventing Duplicate Work

Este es el deep-dive central. La entrega es **at-least-once** en casi todo broker de producción (SQS, BullMQ, Sidekiq, Celery), porque sobrevive particiones y crashes sin perder trabajo. Consecuencia: **tu job va a correr más de una vez**. Un worker puede crashear después de terminar el trabajo pero antes del ACK, y el broker lo re-entrega (comportamiento correcto — el broker no sabe que el efecto ya pasó).

- **La respuesta correcta NO es perseguir exactly-once en el broker** (es imprácticamente imposible end-to-end: el ACK que confirmaría la entrega única puede perderse). La respuesta es **hacer los consumers idempotentes**: que un re-run no cambie nada.
- **Idempotency key / deduplication:** cada job lleva un ID único; el worker chequea en un dedup store si ya lo procesó antes de ejecutar efectos secundarios (el patrón "dedupe-before-ack", el hueco más común en los postmortems de sistemas con colas). Ejemplo: antes de cobrar, `SELECT id FROM payments WHERE order_id = X`; si existe, ACK y return sin cobrar de nuevo. Stripe formalizó esta idea como contrato de API, no solo como truco interno del worker: el cliente manda un header `Idempotency-Key` (recomiendan UUID v4) en cada POST; Stripe persiste el status code y el body de la *primera* respuesta y devuelve exactamente eso ante reintentos con la misma key — incluso si esa primera respuesta fue un error 500. Las keys expiran (se pueden purgar) después de al menos 24 horas. Es una variante productizada del dedup store ([Stripe — Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency), [Stripe API Reference — Idempotent requests](https://docs.stripe.com/api/idempotent_requests)).
- **Guard de estado:** el handler chequea el estado del recurso antes de actuar (si la orden ya está "processed", no re-procesa).
- **Temporal** logra ejecución exactly-once de la lógica de orquestación construyendo **sobre** un substrato at-least-once, no eliminándolo.

### Managing Queue Backpressure

- **Monitorear queue depth / consumer lag:** si la cola crece más rápido de lo que drena, hay un problema. El lag **por partición** es más útil que el promedio (el promedio esconde hot partitions).
- **Autoscaling de workers:** escalás workers según la profundidad de la cola.
- **Load shedding:** si el diluvio es insostenible, rechazás/descartás jobs no críticos para proteger el sistema.
- **Backpressure natural:** la cola misma actúa de throttle — los productores no esperan, pero el trabajo se acumula de forma visible y controlada en vez de tumbar al downstream (la lección del flash sale).

### Handling Mixed Workloads

- **Colas por prioridad:** jobs críticos en una cola high-priority que los workers procesan más seguido; los no urgentes en otra. (Sidekiq/Celery soportan múltiples colas con routing.)
- **Aislar workloads:** separar jobs cortos de largos en colas/pools distintos para que un job de horas no bloquee a uno de segundos.
- **Fairness:** `prefetch_multiplier=1` para que un worker no acapare muchos jobs y otros queden ociosos.

### Orchestrating Job Dependencies

- Cuando un job depende del resultado de otro (pipeline con etapas), no alcanza una cola simple.
- **Fan-out/fan-in:** un dispatcher reparte una tarea en sub-tareas paralelas (fan-out), y luego se agregan los resultados (fan-in). Ideal para procesamiento paralelo de tareas independientes seguido de consolidación.
- **Workflow engines (Temporal, Step Functions):** para workflows multi-etapa con dependencias, compensación y estado — se solapa con el pattern Multi-step Processes.
- **DAG de tareas:** modelás las dependencias como un grafo dirigido; cada tarea se dispara cuando sus predecesoras completaron.

---

## Conclusion

El patrón se resume en: aceptar el request rápido, encolar, devolver un job ID, y procesar en background con workers escalables. Ganás respuesta rápida, escalado independiente, aislamiento de fallos y absorción de picos; perdés simplicidad e inmediatez.

Los invariantes que atraviesan todo y que hay que nombrar sí o sí: **consumers idempotentes** (at-least-once es la realidad, exactly-once en el broker es un mito — se absorbe con idempotencia), **DLQ como alerta** (no como archivo), **retries con backoff + jitter y cap**, **referencias en vez de payloads grandes**, y **ACK después de procesar**. La frase que resume la actitud: el problema difícil de los background jobs nunca fue el encolar, fue el camino de falla — diseñá para el duplicado, instrumentá el backlog, y la cola deja de ser la parte del sistema que temés durante un incidente. Y no metas una cola para un job que corre en dos segundos.

---

## References

- [AWS Architecture Blog — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Stripe — Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)
- [Stripe API Reference — Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [AWS Docs — Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [AWS re:Post — Understand the Amazon SQS dead-letter queue](https://repost.aws/knowledge-center/sqs-dead-letter-queue)

## Related

- [[Message Queue]]
- [[Dead Letter Queue]]
- [[Retry with Backoff]]
- [[Idempotency]]
- [[Webhooks]]
