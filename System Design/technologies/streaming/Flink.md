---
title: Flink
created: 2026-07-01
tags:
  - system-design/streaming
  - type/technology
  - status/permanent
aliases:
  - Flink
  - Apache Flink
updated: 2026-07-01
reading:
  total_words: 4841
  read_words: 0
  pct: 0
  last_read: 2026-07-01
---

# Flink

> [!note] Tesis operativa
> Flink es un motor de **cómputo distribuido y stateful** diseñado "streaming-first": trata el **batch como un caso especial de streaming** (un *bounded stream*), al revés de Spark, que nació batch-first y hace streaming vía micro-batches. Lo que lo distingue de [[Kafka]] Streams o de consumers custom son tres piezas que resuelven los tres problemas difíciles del stream processing a la vez: **checkpointing tipo Chandy-Lamport** (fault tolerance sin pausar el pipeline), **watermarks** (correctitud bajo eventos fuera de orden) y **state backends escalables** (RocksDB, terabytes de estado). Por eso es la respuesta por default en entrevistas de "real-time processing": Ad Click Aggregator, fraud detection, leaderboards.

## Marco mental (leé esto primero)

Cuatro prerrequisitos sostienen todo lo demás. Sin ellos, el resto de la nota no cierra:

- **Streaming ≠ batch**: batch procesa un dataset finito y conocido; streaming procesa un flujo **infinito** que nunca "termina". Todo el diseño de Flink sale de asumir que el input no se acaba.
- **Bounded vs Unbounded**: *unbounded* = sin fin (clicks, logs, eventos de sensores); *bounded* = finito (un archivo, una tabla). La tesis fundacional de Flink es que **batch = bounded stream**, no al revés — y eso permite correr el mismo código en modo streaming y en modo batch.
- **Lo difícil es el estado**: casi todo cómputo útil (contar, detectar fraude, sesionizar) necesita **recordar algo entre eventos**. Ese estado vive distribuido, tiene que sobrevivir fallos, y tiene que reconciliarse con el offset exacto del stream. "Difícil y caro de hacer bien" — y es justo lo que Flink resuelve por vos.
- **¿De verdad necesitás real-time?**: muchísimos problemas que suenan "streaming" se resuelven con un batch periódico. Es la pregunta #1 de entrevista **antes** de comprometerte a la complejidad operativa de Flink. Ver [[Stream Processing]].

## La cadena causal (el problema que cada pieza ataca)

1. Querés procesar un stream infinito y **stateful** (contar clicks por minuto, digamos). El primer problema es de **infraestructura**: ¿dónde corre eso, cómo se paraleliza, cómo se reparten los recursos? Lo resuelve la **arquitectura del cluster** (JobManager + TaskManagers + task slots).
2. Con el cómputo distribuido aparece el segundo problema: los eventos **llegan desordenados** (particiones de Kafka, reintentos, clientes offline). Si agregás por el reloj de la máquina, "clicks por minuto" depende del orden de llegada, no de cuándo pasaron. Lo resuelve la **semántica de event time + watermarks**.
3. Con event time necesitás decidir *cuándo* una ventana está cerrada y emitís resultado. Lo resuelve el **windowing** (tumbling/sliding/session/global) disparado por watermarks.
4. Ese estado acumulado tiene que vivir en algún lado que escale a terabytes. Lo resuelve el **state management** (keyed state + state backends: HashMap, RocksDB, ForSt).
5. Un nodo se cae y perdés todo el estado en memoria → o perdés datos o los duplicás. Lo resuelve el **checkpointing tipo Chandy-Lamport**, que saca fotos consistentes del estado distribuido **sin pausar** el pipeline, y habilita **exactly-once**.
6. Un downstream lento (un sink a DB con latencia) tiene que frenar a los upstream sin colapsar. Lo resuelve la **backpressure vía credit-based flow control**.

Cada sección de abajo es un eslabón de esta cadena.

## Arquitectura del cluster

El pipeline de Flink corre sobre cuatro piezas, con una división clara entre quién coordina y quién ejecuta:

- **Client**: NO es parte del runtime. Arma el *dataflow graph* (el JobGraph) a partir de tu código y se lo manda al **JobManager**. Puede desconectarse después (modo *detached*) o quedarse escuchando resultados (*attached*).
- **JobManager** (el coordinador): hace scheduling y — lo más importante — **coordina los checkpoints y la recuperación de fallos**. Internamente son tres subcomponentes: el **ResourceManager** (gestiona los task slots), el **Dispatcher** (expone la REST API + la Web UI, y lanza un JobMaster por cada job), y el **JobMaster** (ejecuta un único JobGraph).
- **TaskManagers** (los workers): ejecutan las tasks y **bufferean e intercambian streams** entre sí. Cada uno es un proceso JVM.
- **Task slots**: la unidad mínima de recursos (memoria administrada) dentro de un TaskManager. Ojo con un detalle clásico de entrevista: los slots aíslan **memoria, no CPU** — no hay aislamiento de CPU entre slots del mismo TaskManager.

### Slot sharing y operator chaining

Dos optimizaciones que bajan overhead y suben throughput:

- **Slot sharing** (activado por default): subtasks del **mismo job** comparten un slot. La consecuencia práctica es que el cluster necesita tantos slots como el **paralelismo máximo** de un operador del job (no la suma de todos) → mejor utilización de recursos.
- **Operator chaining**: encadena operadores consecutivos que tienen el mismo paralelismo y no requieren shuffle dentro de **una sola task de un solo thread**. Menos serialización, menos handoffs entre threads → más throughput y menos latencia.

### Deployment modes

- **Session Cluster**: long-running, corre múltiples jobs sobre el mismo cluster. Poco aislamiento entre jobs (un job problemático puede afectar a los otros), pero barato de operar si tenés muchos jobs chicos.
- **Application Cluster**: dedicado a **una** app; el `main()` corre en el cluster mismo. Mejor aislamiento — es el modo recomendado para producción.
- **Job Cluster**: deprecado.

Corre sobre YARN, Kubernetes o standalone.

## Modelo de dataflow: 3 APIs

Flink expone tres niveles de abstracción, de más control a más declarativo:

- **ProcessFunction**: el nivel más bajo. Trabajás con eventos individuales, keyed state y timers a mano. Máximo control, más verboso.
- **DataStream API**: la principal. `map`/`filter`/`flatMap`, `keyBy`, `window`, agregaciones, joins, `process`. Es donde vive el 90% del código real.
- **Table API / Flink SQL**: declarativa, SQL sobre streams. La que "democratiza" Flink para quien no quiere escribir operadores.

La operación central es **`keyBy`**: convierte un stream plano en un **keyed stream**, particionando por key de modo que **todos los elementos de una misma key van siempre a la misma task**. Eso es lo que permite paralelizar el estado (cada key es independiente) manteniendo la localidad. El programa se compila a un **dataflow graph dirigido**: sources → operadores → sinks, con un paralelismo configurable por operador.

## Semántica de tiempo (la mitad de los bugs)

La noción de "tiempo" que elijas define la correctitud de todo el pipeline:

- **Processing time**: el reloj de la máquina que procesa. Simple y de baja latencia, pero **no reproducible** y sin noción de orden — el mismo input reprocesado da resultados distintos.
- **Event time**: el timestamp que el evento **dice tener** (cuándo ocurrió realmente). Reproducible y correcto, pero te obliga a lidiar con el desorden.
- **Ingestion time**: cuándo el evento entró a Flink. Un punto medio poco usado.

> [!note] ¿Por qué event time y no processing time?
> En un sistema distribuido los eventos llegan desordenados: distintas particiones de Kafka, reintentos, clientes que estuvieron offline y descargan de golpe. Si agregás "clicks por minuto" por **processing time**, el resultado depende del orden de llegada, no de cuándo pasaron realmente los clicks → rompe la correctitud. Event time te da el resultado *correcto* aunque los datos lleguen tarde y desordenados.

### Watermarks — cadena causal

El problema que los watermarks resuelven es concreto: con event time, Flink necesita saber **cuándo cerrar una ventana** (cuándo dejaron de llegar eventos con timestamp anterior a X). En un stream infinito nunca hay garantía dura de eso. La solución se construye paso a paso:

1. Un **watermark** es una marca que fluye por el stream y dice: *"razonablemente seguro de que ya no llegan eventos más viejos que W"*. Es una **heurística, no una garantía** — asumís un límite de desorden.
2. La estrategia más común es **bounded-out-of-orderness**: asumís que ningún evento llega más de N milisegundos tarde, y el watermark queda en `max_timestamp - N`.

```java
WatermarkStrategy
    .<Tuple2<Long, String>>forBoundedOutOfOrderness(Duration.ofSeconds(20))
    .withTimestampAssigner((event, timestamp) -> event.f0);
```

3. Ahí aparece el **trade-off latencia-vs-completitud**: N grande = más latencia (esperás más antes de cerrar la ventana) pero perdés menos eventos; N chico = respondés más rápido pero corrés más riesgo de descartar datos válidos que llegaron tarde.
4. Con **múltiples fuentes**, el watermark combinado es el **mínimo** entre todos los inputs — porque la garantía "nada más viejo que W" solo vale si **TODAS** las fuentes la cumplen; la fuente más lenta frena a todas. Eso genera dos problemas operativos:
   - **Idle sources**: una partición sin eventos deja su watermark quieto → como el combinado es el mínimo, frena a todas las demás. Se resuelve con `.withIdleness(Duration.ofMinutes(1))`, que la marca como inactiva y la saca del cálculo del mínimo.
   - **Watermark skew**: una fuente mucho más rápida hace que los operadores downstream bufferen sin control esperando a la lenta. Se resuelve con **watermark alignment** (`.withWatermarkAlignment(...)`), que *pausa* la fuente adelantada hasta que las otras la alcanzan.
5. **Late events**: los que llegan **después** de que pasó el watermark. Por default se **descartan silenciosamente**. Dos válvulas: `.allowedLateness(<time>)` mantiene la ventana viva un rato más (cada late event dispara un *late firing* = recómputo del resultado), y `.sideOutputLateData(tag)` captura los descartados en un stream aparte, estilo dead-letter (ver [[Dead Letter Queue]]).

## Windowing

Una vez que tenés event time y watermarks, el windowing decide **cómo agrupar** los eventos infinitos en pedazos finitos que podés agregar:

| Tipo | Definición | Overlap | Uso típico |
|---|---|---|---|
| **Tumbling** | Tamaño fijo, sin solape | No | Conteos por intervalo (clicks/minuto) |
| **Sliding** | Tamaño fijo + un *slide* | Sí (si slide < size) | Promedios móviles, leaderboards |
| **Session** | Cierra tras un gap de inactividad | No (mergea las cercanas) | Sesionización de usuario |
| **Global** | Todos los eventos de una key en una única ventana infinita | — | Requiere un Trigger custom |

```java
input.keyBy(<key>)
    .window(TumblingEventTimeWindows.of(Duration.ofSeconds(5)))
    .<transformation>(<function>);
```

### Window functions — el eje que decide el tamaño del estado

Cómo agregás dentro de la ventana importa muchísimo para cuánto estado guardás:

- **ReduceFunction**: combina de a pares, **incremental** — solo guardás el acumulado, no los elementos.
- **AggregateFunction**: acumulador con tipos de entrada/acumulador/salida separados, también **incremental**. Ej: un promedio que mantiene `(suma, count)` sin guardar todos los valores.
- **ProcessWindowFunction**: recibe **TODOS** los elementos de la ventana más metadata (la ventana, timestamps). Es la más flexible y la **más cara** en estado, porque tiene que retener todo. Se puede combinar con un aggregate incremental (agregás eagerly y usás la ProcessWindow solo para la metadata final).

**Triggers** deciden *cuándo* dispara una ventana (`EventTimeTrigger` es el default: dispara cuando el watermark pasa el fin de la ventana). **Evictors** remueven elementos de la ventana, pero tienen un costo escondido: **impiden la pre-agregación incremental**, o sea que te obligan a retener todos los elementos → más estado.

> [!warning] Sliding window = explosión de estado
> Una **sliding window** crea **múltiples copias** de cada elemento: una por cada ventana solapada en la que cae. Una sliding de 1 día con slide de 1 segundo = 86.400 copias de cada evento = catástrofe de estado. La mitigación real es usar **ReduceFunction/AggregateFunction** para agregar *eagerly* (guardás el acumulado por ventana, no los elementos), lo que baja el estado drásticamente. Con `ProcessWindowFunction` sola en una sliding chica de slide, te comés todo.

## State management (el feature diferencial)

El estado es lo que hace difícil el stream processing, por cuatro razones a la vez: (a) vive **distribuido**, sharded por key; (b) tiene que **sobrevivir fallos**; (c) tiene que **escalar a terabytes**; (d) tiene que **reconciliarse exactamente con el offset** del stream tras un fallo. Flink te da esas cuatro cosas de fábrica.

**Tipos de estado**:
- **Keyed state**: sharded por key, solo accesible dentro de un keyed stream. Las primitivas son `ValueState`, `ListState`, `MapState`, `ReducingState`, `AggregatingState`.
- **Operator state**: no particionado por key; lo típico es guardar los **offsets de Kafka** que consume una source.

### State backends

Dónde vive el "working state" (el que se lee/escribe en caliente) y cómo se snapshotea define el trade-off memoria-vs-escala:

| Backend | Working state | Snapshot | Cuándo usarlo |
|---|---|---|---|
| **HashMapStateBackend** | JVM Heap | Full | Acceso directo, rápido, pero sujeto a **GC**. Para estado moderado que entra en memoria. |
| **EmbeddedRocksDBStateBackend** | Disco local (RocksDB) | Full o **Incremental** | Estado más grande que la memoria; ~**10x más lento** (serializa/deserializa en cada acceso). Es el **único con checkpoints incrementales**. |
| **ForStStateBackend** (experimental) | Filesystem remoto (S3/HDFS) | Async incremental | Estado más grande que el disco local; cloud-native. **No production-ready** todavía. |

**Checkpoint storage** es una cosa distinta del backend (dónde se **guarda el snapshot** durable): `JobManagerCheckpointStorage` (en el heap del JobManager, solo para testing) vs `FileSystemCheckpointStorage` (S3/HDFS, el de producción). Aparte, el **changelog state backend** sube los cambios de estado continuamente en vez de esperar al checkpoint → checkpoints más cortos y predecibles, a costa de más CPU/IO en régimen normal.

## Fault tolerance — checkpointing, Chandy-Lamport, exactly-once (el tema central)

Este es EL tema por el que se elige Flink. La cadena causal:

1. Un fallo de nodo pierde el estado en memoria → sin recuperación, o **perdés datos** o los **duplicás**.
2. Flink saca **snapshots consistentes y periódicos** de TODO el estado, en storage durable (S3/HDFS). Ante un fallo: restaura el snapshot **y reanuda el stream desde el offset grabado en él**. Estado y posición del stream se recuperan juntos.
3. El desafío es sacar una foto **consistente** de estado **distribuido** **sin pausar** el pipeline.

### Chandy-Lamport (asynchronous barrier snapshotting)

El algoritmo que hace la foto sin frenar:

- El JobManager inyecta **barriers numerados** en los sources. Los barriers fluyen por el grafo **junto con los datos**, como un evento especial.
- Cuando un operador recibe el barrier N en **todos** sus canales de entrada, snapshotea su estado y reenvía el barrier hacia adelante.
- El **checkpoint N** queda definido como: el estado tras consumir todo lo que venía **antes** del barrier N y **nada** de lo que venía después. Consistente por construcción.
- **Barrier alignment**: un operador con dos inputs (un join) tiene que esperar el barrier en **ambos** canales antes de snapshotear. Eso es costoso bajo backpressure — mientras espera el canal lento, tiene que **bufferear** el canal rápido.
- Los state backends usan **copy-on-write**: el procesamiento sigue mientras el snapshot se escribe async, sin bloquear.
- **Unaligned checkpoints**: en vez de esperar el alignment, los barriers **adelantan** a los datos en tránsito (que se incluyen en el checkpoint) → el tiempo de checkpoint se vuelve **independiente del throughput**. Solo aplica con `EXACTLY_ONCE` + `maxConcurrentCheckpoints=1`.

### Garantías de entrega

- **At-most-once**: sin esfuerzo de recuperación; se puede perder data.
- **At-least-once**: nada se pierde, pero se puede duplicar (sin barrier alignment → más performance).
- **Exactly-once**: nada se pierde ni se duplica.

> [!warning] Qué significa exactly-once en Flink (el matiz senior)
> Exactly-once **NO** significa que cada evento se procese una sola vez — tras un fallo, los eventos **sí se reprocesan**. Significa que cada evento **afecta el estado exactamente una vez**: al restaurar, el estado se resetea a un punto que todavía no vio esos eventos, así que reprocesarlos deja el estado igual que si se hubieran procesado una sola vez. Es *effectively-once sobre el estado*, no *at-most-once sobre el procesamiento*.

### Exactly-once end-to-end (senior vs mid)

Que el **estado interno** sea exactly-once NO garantiza que los efectos **externos** (los sinks) lo sean. Para exactly-once real de punta a punta hacen falta **tres patas**:

1. **Fuente reproducible** (offsets de [[Kafka]]) — poder rebobinar y releer.
2. **Estado checkpointeado** — capturar estado + offset **atómicamente** (lo que ya vimos).
3. **Sink transaccional o idempotente** — el **Two-Phase Commit Sink** (`TwoPhaseCommitSinkFunction`): abre una transacción, escribe en estado "uncommitted", y **confirma solo cuando el checkpoint se completa**. Ejemplo: un sink Kafka transaccional. La alternativa más barata es un **sink idempotente** (un UPSERT con primary key), que logra el mismo efecto sin 2PC. Ver [[Two-Phase Commit]] e [[Idempotency]].

### Savepoints vs checkpoints

Suenan parecido y son distintos — confundirlos es un error clásico:

- **Checkpoint**: **automático**, para recuperación de fallos, puede ser **incremental**, optimizado para restaurarse rápido. Es interno de Flink.
- **Savepoint**: **manual**, siempre **completo**, para **operaciones**: upgrade de la app con estado, redeploy, y sobre todo **rescaling** (cambiar el paralelismo). Optimizado para flexibilidad, no para velocidad.

```java
env.enableCheckpointing(1000); // cada 1000 ms
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE); // default
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);
env.getCheckpointConfig().setCheckpointTimeout(60000); // 1 min
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
env.getCheckpointConfig().enableUnalignedCheckpoints();
```

Defaults relevantes: timeout 10 min · mode `EXACTLY_ONCE` · `num-retained` 1 · `tolerable-failed-checkpoints` 0.

## Backpressure — credit-based flow control

El problema: un operador downstream lento (típico: un sink a una DB con latencia) tiene que **frenar** a los upstream, o los buffers explotan. Históricamente Flink se apoyaba en el **flow control de TCP**, y eso tenía un defecto grave: una task lenta bloqueaba el **socket compartido** y frenaba a **TODAS** las tasks que usaban esa conexión — además de impedir que el barrier de checkpoint se adelantara.

La solución (Flink 1.5+) es **credit-based flow control** a nivel de aplicación: el downstream le manda **créditos** al upstream según cuántos buffers tiene libres; el upstream **solo envía si tiene crédito**. Si el downstream se pone lento, deja de dar créditos → backpressure **inmediato y localizado**, sin bloquear el socket compartido. Ventajas: detección más rápida, menos latencia, y aislamiento entre tasks de la misma conexión. La Web UI muestra las tasks en rojo ("busy") cuando hay backpressure. Como bonus, [[Kafka]] upstream **absorbe** ese backpressure: en vez de colapsar, el pipeline simplemente lee más lento del log durable.

## CEP (Complex Event Processing) — fraud detection

Flink incluye una librería de **CEP** para detectar **patrones/secuencias** de eventos, no un evento anómalo aislado. El caso canónico es fraude: no es "una transacción grande" sino **una secuencia** — varias transacciones chicas seguidas de una grande, un "viaje imposible" geográfico (dos compras en ciudades lejanas en minutos), un account takeover. La `Pattern` API define el patrón (singleton o *looping* con cuantificadores), el motor evalúa cada evento contra los patrones activos y emite **solo el match completo**. Se combina con `keyBy` (por cuenta/usuario) + el estado de la secuencia. Es el mecanismo detrás de "real-time fraud detection".

## Flink vs Spark Structured Streaming vs Kafka Streams

La comparación #1 de entrevista. La diferencia de fondo es el **modelo de ejecución**:

| Dimensión | Flink | Spark Structured Streaming | Kafka Streams |
|---|---|---|---|
| Modelo | Streaming-first, *event-at-a-time* | **Micro-batch** (batches chicos) | Streaming-first, embebido |
| Latencia | Muy baja (ms) | Más alta (atada al micro-batch) | Baja |
| Deployment | Cluster dedicado (JM+TM), YARN/K8s | Cluster Spark (driver+executors) | **Librería embebida en tu app** — sin cluster |
| Infra extra | Sí | Sí | **No** |
| State | Rico (keyed/operator, RocksDB, checkpointing barrier-based) | Soportado, menos granular | Local (RocksDB por instancia), más simple |
| Windowing | Completo (tumbling/sliding/session/global, triggers/evictors) | Menos granular | Simple (hopping/tumbling/session) |
| Exactly-once | Chandy-Lamport + 2PC sinks (end-to-end) | WAL + idempotent sinks | Nativo vía transacciones Kafka (Kafka-a-Kafka) |
| Lenguajes | Java/Scala/Python/SQL | Java/Scala/Python/SQL (más amplio) | Solo Java/Scala |
| Mejor para | Control fino, baja latencia, estado grande, CEP | Equipos Spark, batch+streaming SQL unificado | Microservicios event-driven ligeros sin cluster |

> [!tip] Regla rápida de elección
> - Microservicio event-driven ligero, **sin querer parar un cluster nuevo** → **Kafka Streams** (es una librería, vive dentro de tu app).
> - **Estado grande / latencia mínima / CEP** → **Flink**.
> - Ya vivís en el ecosistema Spark y querés batch+streaming con SQL unificado → **Spark Structured Streaming**.

Dos comparaciones más: **Flink vs Storm** — Storm es el predecesor histórico del stream processing, ya desplazado por Flink. **Flink vs consumers custom** — un consumer artesanal es viable para casos *stateless* simples; pero en el momento en que necesitás event-time windowing, joins de streams o exactly-once tolerante a fallos, reimplementar eso a mano es **reimplementar mal un subconjunto de Flink**.

## Kappa vs Lambda + dónde vive Flink

El **pipeline canónico** de real-time con Flink es siempre el mismo:

`Kafka (ingesta / buffer / replay) → Flink (procesa / agrega / enriquece / CEP) → Sink`

[[Kafka]] es la fuente durable y reproducible que el checkpointing **necesita** (sin poder rebobinar no hay exactly-once), y de paso **absorbe la backpressure** (buffer en vez de colapso). El sink se elige por patrón de acceso: [[Redis]] (hot serving, leaderboards), [[Cassandra]]/[[DynamoDB]] (durable, alto write, TTL), [[Elasticsearch]] u OLAP (Druid/ClickHouse, analítica).

Sobre esa base, las dos arquitecturas:

- **Lambda**: una *batch layer* + una *speed layer* + una *serving layer*. Su costo es **dos code bases** con la misma lógica → riesgo de que diverjan. Ver [[Lambda Architecture]].
- **Kappa**: una **sola** capa de streaming; reprocesar el histórico = **replay del log de Kafka** con el mismo job. Flink la habilita naturalmente porque **batch = bounded stream** (el mismo código corre en modo BATCH). Es la contracara de la evolución que ya conocés en [[MapReduce]]: en vez de dos motores, uno solo que trata batch como un stream acotado.
- **La decisión real**: ¿tu lógica batch y tu lógica streaming son **la misma**? → Kappa/Flink. ¿Son **genuinamente distintas** (ej. un ML pesado offline)? → Lambda puede seguir siendo lo correcto.

> [!warning] Dato de entrevista: incluso Flink se empareja con batch
> El Ad Click Aggregator de HelloInterview usa **Lambda, no Kappa puro**: Flink para el *speed layer* + un **batch diario de Spark que reconcilia contra los eventos crudos guardados en S3** para la correctitud del billing. La moraleja senior: donde Flink domina, igual se lo aparea con un batch de reconciliación cuando la **exactitud financiera** importa más que la elegancia de tener un solo codebase.

## Casos de uso canónicos (diseño Flink-específico)

- **Ad Click Aggregator**: consume clicks de Kafka → **tumbling window de 1 minuto** (event time + watermarks para el desorden), truncando el timestamp al minuto como key compuesta con `ad_id` → **AggregateFunction** (mantiene 1 acumulador en memoria a la vez, escala) → **dedup** con keyed state upstream (+ chequeos anti-fraude) → sink Kafka "uncommitted"→"committed" vía 2PC. Para el **hot ad / celebrity** (un anuncio que explota en clicks): saltear el `ad_id` con un sufijo random (`AdId:0-N`), strip del sufijo antes de escribir, y un `SUM` en el OLAP recombina — misma idea que la *hot partition* de [[Kafka]]. Sink final: OLAP (Snowflake/ClickHouse) sharded por advertiser.
- **Fraud detection**: **CEP** / pattern matching, reglas stateful cross-event (ver la sección de CEP).
- **Leaderboards / Top-K**: `keyBy` → **sliding window** (no tumbling: necesitás evictar viejos *mientras* agregás nuevos) → Top-K local por partición → merge global → **Redis sorted sets** para el serving. Ver [[Redis]].
- **Streaming ETL / enrichment**: join de un stream contra una dimensión *slowly-changing* con **temporal joins** (`FOR SYSTEM_TIME AS OF`), alimentando la dimensión vía **Change Data Capture** (los Flink CDC connectors) para un join *point-in-time* correcto.
- **Sesionización**: **session windows** (cierran tras un gap de inactividad), por key.
- **Métricas / logs**: windowed aggregation simple → OLAP o time-series DB.

## Cuándo NO usar Flink

> [!warning] Flink es potente pero caro de operar — antes de traerlo, descartá lo simple
> - **¿De verdad es real-time?** Muchísimos casos "streaming" se resuelven con un batch periódico. Es la pregunta #1.
> - **Complejidad operativa**: un cluster que hay que deployar y tunear (checkpoint interval, state backend, monitoreo de backpressure y checkpoint duration) vs una librería como Kafka Streams que no necesita cluster.
> - **Stateless / ETL simple**: si es filter/transform sin estado, usá Kafka Streams, o landeá directo en ClickHouse (insert-time). Regla práctica: **empezá con Kafka→ClickHouse directo**; traé Flink recién cuando pegues contra algo que ClickHouse no puede (joins stateful, windowing complejo, exactly-once).
> - **Escala chica**: no montes Flink para contar 100 eventos/segundo. El overhead operacional no se justifica.

## Escala / staff-level

- **Uber**: pricing en tiempo real, fraud, matching. Trillones de mensajes por día — creció de ~1M a ~12M msg/s en 5 años, con latencia de API sub-5ms. Documentó públicamente una arquitectura **Kappa** sobre Flink + Kafka.
- **Alibaba**: forkeó Flink en **Blink** (después mergeado upstream). En el Double 11 de 2021 procesó **4 billones de records/segundo (~7 TB/s)**.
- **Rescaling**: se hace con **savepoint → restart con nuevo paralelismo** (no en caliente).
- **State size = el límite real de escala**, no el throughput: RocksDB a escala de terabytes es lo que dispara la duración del checkpoint, el I/O y la presión de memoria. Cuando un pipeline de Flink no escala más, casi siempre es por el estado, no por los eventos/segundo.
- **Flink SQL democratiza**: `SESSION()`, `MATCH_RECOGNIZE`, temporal joins — todo sin escribir operadores a mano.
- **Batch/stream unification**: el mismo DataStream API corre en **modo BATCH** explícito. Es la prueba concreta de la tesis: batch es streaming acotado, en código real.

## 🎯 Preguntas de retrieval

> [!question] ¿Por qué el watermark combinado de múltiples inputs es el **mínimo**, no el máximo?
> Porque la garantía de un watermark es "ya no llegan eventos más viejos que W", y esa promesa solo es cierta si **TODAS** las fuentes la cumplen. Basta una fuente atrasada para que existan eventos viejos aún en camino, así que el combinado tiene que ser el mínimo — la fuente más lenta manda. (Por eso una *idle source* frena a todas si no la marcás con `withIdleness`.)

> [!question] ¿Qué significa exactly-once en Flink, exactamente?
> Que cada evento **afecta el estado una sola vez**, NO que se procese una sola vez. Tras un fallo los eventos se reprocesan, pero como el estado se restaura a un checkpoint que todavía no los vio, el efecto neto sobre el estado es el mismo que si se hubieran procesado una vez. Es effectively-once sobre el estado.

> [!question] ¿Qué tres patas hacen falta para exactly-once **end-to-end**?
> (1) Una fuente **reproducible** (offsets de Kafka) para poder rebobinar; (2) **estado checkpointeado** que capture estado + offset atómicamente; (3) un **sink transaccional** (Two-Phase Commit Sink) **o idempotente** (UPSERT con PK). El estado interno exactly-once no alcanza si el sink duplica.

> [!question] ¿Checkpoint o savepoint?
> Checkpoint: automático, para recuperación de fallos, puede ser incremental, optimizado para restaurar rápido. Savepoint: manual, siempre completo, para operaciones — upgrade con estado, redeploy y sobre todo **rescaling** (cambiar el paralelismo).

> [!question] ¿Por qué Flink y no Spark Structured Streaming?
> Porque Flink es *true streaming* (event-at-a-time) y Spark hace **micro-batch** — la latencia de Spark queda atada al tamaño del micro-batch. Si necesitás latencia de milisegundos real, Flink. Si ya vivís en Spark y querés SQL unificado batch+streaming, Spark alcanza.

> [!question] ¿Por qué Flink y no Kafka Streams?
> Kafka Streams es una **librería embebida** sin cluster, ideal para microservicios event-driven simples. Traés Flink cuando necesitás **estado grande** (RocksDB a TB), **latencia mínima**, **CEP**, o **windowing rico** — cosas para las que una librería embebida en tu app no está pensada.

> [!question] ¿Tumbling o sliding para un leaderboard?
> Sliding. Un leaderboard necesita "top de los últimos N minutos actualizado constantemente", o sea **evictar los viejos mientras agregás los nuevos** — eso es exactamente una sliding window. Una tumbling te daría saltos bruscos al cerrar cada ventana.

> [!question] ¿HashMap o RocksDB como state backend?
> HashMap (heap JVM): rápido por acceso directo, pero limitado por memoria y sujeto a GC → estado moderado. RocksDB (disco local): escala más allá de la memoria y es el único con checkpoints incrementales, pero ~**10x más lento** por serializar/deserializar en cada acceso. Elegís por tamaño de estado.

> [!question] ¿Kappa o Lambda?
> Si tu lógica batch y streaming es **la misma** → Kappa (una capa de streaming, reprocesás con replay del log; Flink lo habilita porque batch = bounded stream). Si son **genuinamente distintas** (ej. ML pesado offline) → Lambda puede seguir siendo lo correcto, asumiendo el costo de dos code bases.

> [!question] ¿Cuándo NO usar Flink?
> Cuando un batch periódico alcanza; cuando es stateless/ETL simple (Kafka Streams o Kafka→ClickHouse directo); cuando la escala es chica (100 eventos/s no justifican un cluster); o cuando no querés cargar con la complejidad operativa de deployar, tunear y monitorear un cluster con state backends y checkpointing.

## References

- Los enriquecimientos de esta nota (internals de arquitectura, watermarks/idle sources/alignment, state backends, Chandy-Lamport/unaligned checkpoints, exactly-once end-to-end y 2PC sinks, credit-based flow control, CEP, comparativas vs Spark/Kafka Streams/Storm, Kappa vs Lambda, casos de uso y escala Uber/Alibaba) provienen de **research de preparación de entrevistas de system design combinando múltiples fuentes**, no de un único artículo: la documentación oficial de Apache Flink (architecture, fault tolerance, checkpointing, watermarks, windows, state backends), el deep-dive de HelloInterview (**paywalled** — solo accesible el TOC/intro y el breakdown del Ad Click Aggregator), comparativas de Onehouse/Redpanda, artículos de Kai Waehner (CEP, Kappa, cuándo NO usar Flink), engineering de Uber y Alibaba (escala) y Confluent/Streamkap (exactly-once, temporal joins).

## 🔗 Conexión con el vault

- Volvé al MOC del subtema: [[_Streaming|Streaming]].
- Otras tecnologías deep-dive: [[Kafka]] (la pareja natural de Flink — fuente + buffer), [[Cassandra]], [[Redis]], [[Elasticsearch]], [[DynamoDB]] (candidatos de sink).
- Patrones y conceptos relacionados: [[Stream Processing]], [[Event-Driven Architecture]], [[Event Sourcing]], [[CQRS]], [[Lambda Architecture]], [[MapReduce]], [[Two-Phase Commit]], [[Idempotency]], [[Dead Letter Queue]], [[Message Queue]], [[Pub-Sub]], [[Saga]].
