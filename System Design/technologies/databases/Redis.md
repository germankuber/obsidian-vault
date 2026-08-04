---
title: Redis
source: https://www.hellointerview.com/learn/system-design/deep-dives/redis
author: Hello Interview
created: 2026-07-01
tags:
  - system-design/databases
  - type/technology
  - status/permanent
aliases:
  - Redis
  - Redis Cluster
reading:
  total_words: 3363
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-07-01
---

# Redis

> [!note] Tesis operativa
> Redis es rápido y versátil porque toma una decisión radical: **todo vive en memoria y se ejecuta en un solo hilo, comando por comando**. Esa misma decisión explica TODAS sus limitaciones (durabilidad débil, sin joins, hot keys) — no son defectos accidentales, son el **precio de la velocidad**. Se describe a sí mismo como *"data structure store"* (no "base de datos"), y su versatilidad viene de que sus estructuras se parecen a las que ya usás al programar (hashes, sets, sorted sets, streams).

## Marco mental (leé esto primero)

- **"Data structure store"**: escrito en C, todo en memoria, **single-threaded** — sin locks, porque no hacen falta con un comando a la vez. Versiones nuevas delegan I/O a otros hilos, pero el modelo mental es: **un comando a la vez, en orden**.
- **Key-value como base**: cada objeto es un valor guardado en una clave string, y ese valor ES la estructura de datos. Cuando creás un sorted set, lo guardás como el valor en la clave que elegiste.
- **La elección de claves = organización + escalado**: en un cluster, las claves viven en nodos distintos según cómo las nombres. Cómo organizás las keys ES cómo escalás Redis.

![[redis-01-logical-model.svg]]
*Redis logical model*

## Durabilidad: el trade-off fundacional

Redis NO da la garantía "commit está en disco" que da una relacional por default. Dos modos de persistencia:
- **RDB**: snapshots periódicos → un crash pierde todo desde el último snapshot.
- **AOF**: loguea cada escritura, pero hace **fsync 1×/segundo** por default → un crash pierde hasta **1 segundo** de escrituras ya confirmadas al cliente. Se puede configurar fsync-en-cada-escritura, pero casi nadie lo hace (mata la velocidad por la que se eligió Redis).

Es una decisión **intencional** del equipo de Redis: velocidad sobre garantía de disco. Si necesitás durabilidad real, **AWS MemoryDB** sacrifica algo de velocidad a cambio de durabilidad basada en disco. Un punto medio, sin cambiar de motor: el comando **`WAIT numreplicas timeout`** bloquea la escritura hasta que N réplicas la confirmen (o venza el timeout), cerrando parcialmente la ventana de pérdida de la replicación asíncrona. No es consenso (no da linealizabilidad como Raft) y tiene caveats, pero en una entrevista es mejor respuesta que "fsync en cada escritura" cuando preguntan cómo hacer más durable un Redis open-source.

## Estructuras de datos

**Strings · Hashes** (objetos/diccionarios) **· Lists · Sets · Sorted Sets** (priority queues) **· Streams** (append-only logs) **· Geospatial Indexes**. Redis 8 trae en el core estructuras probabilísticas como **Bloom filters**, **JSON** y time series (en versiones viejas venían de los módulos de Redis Stack). Además de estructuras, soporta patrones de comunicación como **Pub/Sub** y **Streams**, que reemplazan parcialmente sistemas más complejos como Apache **Kafka** o AWS SNS/SQS.

Tres estructuras/casos de uso adicionales que aparecen seguido en entrevistas:
- **HyperLogLog** — conteo **aproximado** de cardinalidad (usuarios únicos, IPs distintas) con ~**12 KB fijos** por contador sin importar cuántos elementos (error ~0.81%). Comandos `PFADD`/`PFCOUNT`. Para "contar visitantes únicos" sin guardar cada ID.
- **Bitmaps** — bits sobre un string (`SETBIT`/`GETBIT`/`BITCOUNT`). Ultra-eficientes para flags booleanos a escala: *daily active users* (un bit por usuario por día), feature flags, presencia.
- **Session store** — guardar sesiones con **TTL** (política `volatile-ttl` para evictar solo las expiradas), permitiendo escalar app servers *stateless* (la sesión vive en Redis, no en el server).

### Comandos — protocolo RESP

Redis habla un protocolo de cable simple llamado **RESP**: los comandos viajan casi tal cual los tipearías, por eso el CLI se siente tan directo.

```
SET foo 1  
GET foo     # Returns 1
INCR foo    # Returns 2
XADD mystream * name Sara surname OConnor # Adds an item to a stream
```

Ejemplo con Sets: `SADD` (agregar), `SCARD` (cardinalidad), `SMEMBERS` (listar), `SISMEMBER` (existencia) — análogos directos a un Set de cualquier lenguaje.

## Infraestructura: de nodo único a cluster

Redis corre como **nodo único**, con una **réplica HA**, o como **cluster**.

1. **Nodo único** — simple, sin alta disponibilidad ni escala horizontal.
2. **Réplica HA** — agrega un standby, pero la replicación es **asíncrona**: el primario confirma tu escritura ANTES de que la réplica la haya visto. Si el primario muere justo después, esas últimas escrituras confirmadas desaparecen al promover la réplica. **Esta es la razón más profunda por la que Redis no es un system of record** (y vuelve a importar con los locks).
3. **Cluster** — cada clave hashea a uno de **16.384 hash slots**, y cada slot se asigna a un nodo. Esto ES el [[Sharding]]: cada nodo es dueño de una porción de los slots. Los clientes cachean el mapa slot→nodo, calculan el slot desde la clave y conectan directo al nodo correcto. Si el mapa quedó viejo (rebalanceo/failover), el servidor responde **MOVED** apuntando al correcto, y el cliente refresca (ej. `CLUSTER SHARDS`). Agregar un nodo migra slots.

![[redis-02-infrastructure.svg]]
*Redis infrastructure configurations*

Los nodos comparten estado vía **protocolo gossip** → todos conocen el mapa completo de slots. Si le pedís una clave al nodo equivocado, responde MOVED — **no reenvía** el pedido por vos. Los clusters de Redis son **deliberadamente básicos**: te dan primitivas para resolver la escalabilidad vos mismo, no la resuelven por vos. Con pocas excepciones, todos los datos de un pedido deben estar en **un solo nodo**. Cuando dos claves realmente necesitan vivir juntas, existen los **hash tags**: solo lo que está entre `{llaves}` se hashea, así `{user:123}:posts` y `{user:123}:likes` caen en el mismo slot, listas para una transacción `MULTI` que las toque a ambas.

### Sentinel vs Cluster (alta disponibilidad)

Hay dos modos de HA y la diferencia es una pregunta clásica:
- **Redis Sentinel** — HA sobre una topología **master-replica clásica** (NO shardea). Procesos Sentinel separados monitorean master + réplicas; ante la caída del master alcanzan un **quórum** y promueven una réplica automáticamente. Resuelve **failover**, no escala de datos.
- **Redis Cluster** — shardea los datos en 16.384 slots entre varios masters **Y** da HA (cada master con réplicas, failover por elección entre nodos). Resuelve **escala horizontal + HA**.

**Decisión:** si el cuello de botella es tamaño de datos / throughput → **Cluster** (sharding). Si solo necesitás failover robusto sobre un dataset moderado que entra en un nodo → **Sentinel** (más simple).

## Performance

Un solo nodo Redis maneja del orden de **~100k writes/segundo**. El comando ejecuta en microsegundos; sobre la red vas a ver lecturas **sub-milisegundo**. Esta velocidad hace factibles anti-patrones ruinosos en otras bases: el patrón **N+1** (una query por ítem) hunde a una base SQL, pero con Redis cada comando cuesta microsegundos y podés hacer `pipeline` (o usar `MGET`) para pagar **un** round-trip en vez de cien. Seguiría siendo mejor evitarlo, pero no hunde el diseño. Todo esto es función de que vive en memoria.

## Capacidades

### Redis como Cache

El despliegue más común. Cache keys = Redis keys, valores cacheados = Redis values, distribuidos trivialmente entre los nodos del cluster. Ejemplo: cachear `product:123` como blob JSON o Redis Hash con campos `name`, `price`, `inventoryCount`. Normalmente configurás un **TTL** por clave — Redis garantiza que **nunca** leerás un valor después de que expiró.

![[redis-03-cache.svg]]
*Redis as a cache*

⚠️ La expiración maneja *staleness*, NO presión de memoria: por default Redis **rechaza escrituras** cuando la memoria se llena. Para un cache configurás una política de eviction como **allkeys-lru** (Redis aproxima LRU **muestreando** claves, suficiente para un cache).

**Estrategias de escritura** (Redis es la implementación concreta de todas):
- [[Cache-Aside]] — la app lee cache, en miss va a la DB y puebla. El default.
- [[Read-Through]] — la cache misma carga de la DB en un miss.
- [[Write-Through]] — escribe a cache y DB en sincronía (consistencia de lectura fuerte).
- [[Write-Behind]] — escribe a cache y difiere la DB (alta frecuencia de escritura, tolera pérdida).

**El trío de problemas de cache** que un entrevistador siempre pregunta:
- **Cache Penetration** — queries de datos que NO existen ni en cache ni en DB → cada request pega a la DB. Solución: cachear el nulo (TTL corto), o un **Bloom filter** de las keys conocidas que rechaza lo inexistente antes de tocar la DB.
- **Cache Breakdown / Stampede** (thundering herd) — una key **caliente** expira y N requests concurrentes reconstruyen el cache a la vez → avalancha sobre la DB. Solución: **distributed lock** (`SET NX`) para que UNO solo reconstruya mientras el resto espera, *logical expiration*, o recomputo probabilístico anticipado. Ver [[Cache Stampede Prevention]].
- **Cache Avalanche** — MUCHAS keys expiran al mismo tiempo (o Redis se cae) → la DB se inunda. Solución: **TTLs con jitter** (aleatorizar el vencimiento) + réplicas/HA para que Redis no sea un SPOF.

Redis no resuelve por sí solo el **hot key problem** (una clave absorbe tráfico desproporcionado; también le pasa a Memcached y **DynamoDB**) — remediaciones en Shortcomings.

### Redis como Distributed Lock

Otro uso común (ver [[Distributed Lock]]). El lock es solo una clave acordada, ej. `lock:concert:343`. Adquirir:

```
SET lock:concert:343 my-token NX EX 30
```

- **NX** = el SET solo tiene éxito si la clave no existe (si funcionó, tenés el lock; si no, esperás y reintentás).
- **EX 30** = TTL de 30s (un proceso crasheado no lo retiene para siempre).
- **my-token** = valor random único tuyo.

Liberar: **nunca hagas un `DEL` a ciegas** — tu lock pudo expirar y ser tomado por otro, y borrarías el lock ajeno. Chequeás que la clave todavía tenga tu token y recién ahí borrás, como **script Lua atómico** (Redis lo ejecuta como un comando en su único hilo):

```
if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) end
```

> [!info] Esto es locking **pesimista** (tomar el lock antes de trabajar). La alternativa **optimista**: `WATCH` una clave, correr la transacción con `MULTI`/`EXEC`, y aborta si la clave observada cambió.

**Modo de falla**: como la replicación es asíncrona, si el primario muere justo después de otorgar tu lock, la réplica promovida no se enteró y le otorga el mismo lock a otro. El algoritmo **Redlock** existe para esto (adquiere en la **mayoría** de nodos independientes), pero es **controversial** — un cliente pausado puede actuar después de que su lock expiró, y la crítica de **Martin Kleppmann** recorre cómo falla. La defensa estándar es un **fencing token** (número creciente; el storage rechaza escrituras con un número viejo), y **Redis no tiene nada built-in** para eso. Tratá un lock de Redis como herramienta de eficiencia que ocasionalmente falla, NO garantía de corrección: si la corrección importa, hacé cumplir el invariante donde viven los datos, usá [[ZooKeeper]]/**etcd**, o un `SELECT ... FOR UPDATE` / `UPDATE ... WHERE version = X` en la DB.

### Redis para Leaderboards

Los **sorted sets** mantienen datos ordenados consultables en tiempo logarítmico — encajan naturalmente en leaderboards, especialmente a escala donde una base SQL empieza a sufrir. Ejemplo (Post Search): top de posts por keyword.

```
ZADD tiger_posts 500 "SomeId1" # Add the Tiger woods post
ZADD tiger_posts 1 "SomeId2" # Add some tweet about zoo tigers
ZREMRANGEBYRANK tiger_posts 0 -6 # Remove all but the top 5 posts
```

`ZADD` setea el score del member (lo reemplaza si ya existía → re-agregar un post con su nuevo conteo de likes mueve su rank). `ZREMRANGEBYRANK` con índices negativos cuenta desde el rank más alto: remover 0 a -6 borra desde el peor score hasta el sexto-desde-arriba, dejando exactamente el **top 5**.

### Redis para Rate Limiting

Ver [[Rate Limiting]]. Algoritmo **fixed-window**: cuando llega un request, `INCR` el contador de la ventana actual; si excede N, rechazás (429 + header `Retry-After`). **Sutileza clave**: seteá el expiry SOLO cuando `INCR` devuelve 1 (el primer request de la ventana) — llamar `EXPIRE` en cada request empuja el reset y con tráfico constante la ventana nunca resetea. Corré `INCR`+`EXPIRE` como **un solo script Lua**.

Para **sliding window**: un sorted set por usuario con el timestamp como score — `ZREMRANGEBYSCORE` (borra lo más viejo que la ventana) + `ZCARD` (cuenta) + `ZADD` (agrega si está bajo N), todo en Lua atómico.

**Token bucket** (el que permite **ráfagas controladas** manteniendo una tasa promedio — el estrella de "Design a Rate Limiter"): un **HASH** con `tokens` (cuántos quedan) y `last_refill` (timestamp), y un script Lua que corre **refill → check → consume**:
1. **Refill**: `elapsed × refill_rate` tokens nuevos desde el último request, capado a la capacidad (`math.min(max_tokens, ...)`) para que no se acumulen infinitos.
2. **Check + consume**: si hay ≥1 token, resta uno y permite; si no, rechaza.
3. Persiste el hash + `EXPIRE` para autolimpiar buckets idle.

Va en Lua atómico por lo mismo que el lock: sin atomicidad, dos requests concurrentes gastan el mismo token (race TOCTOU). Comparación: **fixed-window** = simple pero sufre ráfaga en el borde; **sliding-window log** = exacto pero O(n) memoria; **token bucket** = ráfagas hasta la capacidad con promedio estable; **leaky bucket** = drena a tasa fija, sin ráfagas.

### Redis para Proximity Search

Índices geoespaciales nativos:

```
GEOADD key longitude latitude member # Adds "member" to the index at key "key"
GEOSEARCH key FROMLONLAT longitude latitude BYRADIUS radius unit # Searches the index at key "key" at specified position and radius
```

`GEOSEARCH` corre en **O(N+log(M))** (N = candidatos dentro del bounding box alineado a grilla, M = los que caen en el radio). Por debajo usa **geohashes** (lat/lon codificadas en un único valor ordenable, guardado en un sorted set — de ahí el término log). Dos pases: el primero agarra los N del bounding box, el segundo filtra a los M dentro del radio exacto.

### Redis para Event Sourcing

Los **streams** son append-only logs similares a los topics de **Kafka**, building block de diseños event-sourced (ver [[Event Sourcing]]). Los productores agregan con `XADD`; los **consumer groups** (`XREADGROUP`, `XCLAIM`, `XAUTOCLAIM`) coordinan qué consumer procesa qué. ⚠️ Las entradas son tan durables como tu configuración de persistencia.

![[redis-04-streams.svg]]
*Redis streams and consumer groups*

**Work queue**: agregás ítems con `XADD` y adjuntás un consumer group. Camino feliz: un worker lee vía `XREADGROUP`, procesa, hace ack. Cada pendiente lleva un **idle time**; cuando un worker muere a mitad de tarea, el idle time sube hasta que otro lo reclama con `XCLAIM` (o `XAUTOCLAIM`) y reinicia. Como Redis no distingue un worker lento de uno muerto, un ítem puede procesarse **dos veces** → hacé el procesamiento **idempotente** (ver [[Idempotency]]). ¿Cuándo **Kafka** en vez? Retención larga, replay para muchos consumers independientes, o throughput ordenado y durable a una escala donde perder un mensaje es inaceptable.

### Redis para Pub/Sub

Los streams son para consumers que necesitan ponerse al día; **Pub/Sub** es para entregar a quien esté escuchando **ahora** (ver [[Pub-Sub]]): los mensajes se difunden a múltiples subscribers en tiempo real.

```
PUBLISH channel message   # Sends a message to all subscribers of 'channel'
SUBSCRIBE channel         # Listens for messages on 'channel'
```

![[redis-05-pubsub.svg]]
*Redis Pub/Sub*

> [!tip] Limitación histórica ya NO vigente: el Pub/Sub clásico en cluster difundía cada mensaje a TODOS los nodos (agregar nodos no sumaba capacidad). Desde **Redis 7**, el **sharded Pub/Sub** (`SPUBLISH`/`SSUBSCRIBE`) rutea cada canal al shard dueño de su slot, y la capacidad escala con el cluster.

Un canal es solo un nombre (nada que crear). Entrega **at-most-once**: un subscriber offline cuando se publica se pierde el mensaje. El overhead de conexión es **por nodo, no por canal** — un subscriber mantiene una conexión por nodo y recibe todos sus canales por ella, así que millones de canales NO significan millones de conexiones.

> [!warning] Redis Pub/Sub es simple y rápido pero NO durable (at-most-once). Si necesitás persistencia, garantías de entrega o replay, considerá Redis Streams o un broker dedicado como **Kafka** o RabbitMQ, o combinar Pub/Sub con una cola (SNS→SQS) o un patrón outbox.

**¿Pub/Sub casero?** (anti-patrón) — algunos proponen una clave por topic cuyo valor es el conjunto de direcciones de los subscribers, y publicar mandando directo a esos servidores. Downsides:

```
# Pub/Sub nativo: 2 saltos
1. Client sends message to Pub/Sub node
2. Pub/Sub node dispatches message to all subscribers

# Casero: 3 saltos
1. Client requests subscribers for topic key from Redis
2. Redis responds with servers to contact
3. Client sends message to each server
```

El 3er salto es el más caro (probablemente no hay una conexión TCP ya abierta a cada subscriber; con Pub/Sub nativo esa conexión ya está abierta y mantenida). Además, costo de memoria residente: con Pub/Sub nativo solo se trackean canales con subscribers activos (cuando el último se va, el canal se borra); con el casero hay que enterarse cuando un servidor se cae (heartbeats / TTL), agregando complejidad y chatter. **Si tu caso se parece a Pub/Sub, usá Pub/Sub.**

## Shortcomings y remediaciones

### Hot Key

Si la carga no está distribuida uniformemente, aparece el **hot key**. Ejemplo: cacheás ítems de un ecommerce, escalás a 100 nodos, todo parejo — un día un ítem tiene un surge tan grande que su volumen iguala al del resto combinado. Ahora **un solo servidor** está dramáticamente más cargado y empieza a fallar.

![[redis-06-hot-key.svg]]
*Hot key issue*

| Remediación | Cómo | Trade-off |
|---|---|---|
| **Client-side caching** | cache local de los ítems calientes en cada app server | datos stale hasta el TTL local; mejor con TTLs cortos + pocas claves |
| **Key copies** | mismo dato en `product:123:1`..`product:123:10` (distintos slots/nodos), reader elige sufijo random | readers deben saber qué claves están duplicadas; cada write hace fan out a todas las copias |
| **Read replicas** | multiplica capacidad de lectura | clientes de cluster leen del primario por default (hay que configurarlos); NO ayuda con un hot key de **escritura** (toda escritura va al único primario del slot) |

### Cuándo NO usar Redis

- **No como system of record** — entre la replicación asíncrona y las ventanas de pérdida de persistencia, escrituras confirmadas pueden desaparecer.
- **No si el working set no entra económicamente en RAM** — la memoria es el lugar más caro para guardar datos.
- **Sin flexibilidad de queries** — no hay joins, no hay queries cross-key, y en cluster las operaciones multi-key solo funcionan dentro de un único slot (salvo hash tags).
- **Streams durables/replayables con retención larga para muchos consumers** → eso es trabajo de **Kafka**.

## 🎯 Preguntas de retrieval

> [!question] ¿Por qué Redis puede perder escrituras ya confirmadas al cliente aunque tengas réplicas?
> Porque la replicación es asíncrona: el primario confirma la escritura al cliente ANTES de que la réplica la reciba. Si el primario muere en ese instante, la réplica promovida nunca vio esas escrituras — se pierden. (El comando `WAIT` mitiga esto bloqueando hasta que N réplicas confirmen.)

> [!question] ¿Por qué un DEL directo es peligroso al liberar un distributed lock, y qué lo reemplaza?
> Porque tu lock pudo haber expirado (TTL) y ser tomado por otro proceso — un DEL a ciegas borraría el lock ajeno. Se reemplaza por un script Lua atómico que verifica que el valor sea tu token único y solo entonces borra.

> [!question] ¿Por qué el rate limiter fixed-window debe llamar EXPIRE solo cuando INCR devuelve 1?
> Porque si llamás EXPIRE en cada request, el TTL se empuja hacia adelante constantemente y con tráfico continuo la ventana nunca resetea. Solo seteando el TTL en el primer request (INCR devuelve 1) la ventana tiene un límite fijo real.

> [!question] Read replicas y hot keys: ¿por qué agregar réplicas NO resuelve un hot key de escritura?
> Porque toda escritura sigue yendo al único primario dueño del slot de esa clave — las réplicas escalan lecturas, no escrituras. Y por default los clientes de cluster ni siquiera leen de réplicas salvo que se configuren.

## References

- Source: [Redis](https://www.hellointerview.com/learn/system-design/deep-dives/redis) — Hello Interview
- Las figuras embebidas son SVGs de `files.hellointerview.com`.
- Enriquecimientos fuera del artículo (estrategias de caching + trío penetration/breakdown/avalanche, token/leaky bucket, `WAIT`, Sentinel vs Cluster, HyperLogLog, Bitmaps, session store) verificados contra docs de Redis, Baeldung y fuentes de preparación de entrevistas.

## Related

- [[_Databases|Databases]]
- [[_System Design|System Design]]
- [[Cassandra]]
- [[Distributed Lock]]
- [[Pub-Sub]]
- [[Event Sourcing]]
- [[Cache-Aside]]
- [[Read-Through]]
- [[Write-Through]]
- [[Write-Behind]]
- [[Cache Stampede Prevention]]
- [[Rate Limiting]]
- [[Consistent Hashing]]
- [[Sharding]]
- [[Idempotency]]
- [[Primary-Replica]]
- [[Connection Pooling]]
- [[Message Queue]]
