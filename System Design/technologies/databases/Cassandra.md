---
title: Cassandra
source: https://www.hellointerview.com/learn/system-design/deep-dives/cassandra
author: Hello Interview
created: 2026-07-01
tags:
  - system-design/databases
  - type/technology
  - status/permanent
aliases:
  - Cassandra
  - Apache Cassandra
reading:
  total_words: 2960
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-07-05
---

# Cassandra

> [!note] Tesis operativa
> Cassandra es una base **NoSQL wide-column**, distribuida y open-source, que combina ideas de **Dynamo** (ver **Amazon DynamoDB**) y **Bigtable**. Prioriza **disponibilidad y throughput de escritura** sobre consistencia estricta (elige **AP** en el **CAP Theorem**), y lo logra con tres decisiones que se encadenan: particiona con [[Consistent Hashing]] para escalar horizontal, replica de forma configurable para tolerar fallos, y almacena con un **LSM tree** optimizado para escritura en vez de un B-tree. Fue creada en Facebook para su inbox search; hoy la usan Netflix, Apple, Bloomberg, Discord y muchos más.

## Marco mental (leé esto primero)

Tres pilares sostienen todo lo demás:

1. **Es AP, no CP.** Ante una partición de red, Cassandra elige seguir disponible y reconciliar después (**eventual consistency**). Todo lo demás —niveles de consistencia sintonizables, replicación, reparación— es consecuencia de esta elección.
2. **El modelado es *query-driven*, no *entity-relationship*.** No hay JOINs ni foreign keys: se diseña la tabla alrededor del *access pattern*, y se **denormaliza** (duplica) lo que haga falta.
3. **Escribir es barato, leer es el costo.** El storage layer (LSM tree) es *append-only*: cada escritura es una entrada nueva. Esa es la razón de su altísimo write throughput, y también de por qué las lecturas tienen que reconciliar varios archivos.

## Data Model — Keyspace / Table / Row / Column

- **Keyspace**: ≈ una "base de datos" relacional (Postgres/MySQL) — define estrategias de replicación y es dueño de los **UDTs** (user-defined types).
- **Table**: vive dentro del keyspace, organiza filas, con schema de columnas + primary key.
- **Row**: registro único identificado por primary key.
- **Column** (wide-column / sparse): las columnas especificadas **pueden variar fila a fila** (no exige una entrada NULL por columna como una relacional). Cada columna tiene **timestamp** propio; en conflicto de escritura entre réplicas → **"last write wins"**.

![[cassandra-01-data-model.svg|550]]
*Cassandra Data Model*

Soporta muchos tipos, incluyendo **UDTs** y valores **JSON** (datos planos y anidados). A nivel básico, se puede pensar la estructura como un JSON grande:

```json
{
  "keyspace1": {
    "table1": {
      "row1": { "col1": 1, "col2": "2" },
      "row2": { "col1": 10, "col3": 3.0 },
      "row3": { "col4": { "company": "Hello Interview", "city": "Seattle", "state": "WA" } }
    }
  }
}
```

## Primary Key: partition key + clustering key

Cada fila se representa de forma única por una **primary key** = una o más **partition keys** + (opcional) **clustering keys**.

- **Partition Key**: columna(s) que determinan **en qué partición** vive la fila.
- **Clustering Key**: columna(s) (cero o más) que determinan **el orden** de las filas dentro de la partición.

```sql
-- Primary key with partition key a, no clustering keys
CREATE TABLE t (a text, b text, c text, PRIMARY KEY (a));

-- Primary key with partition key a, clustering key b ascending
CREATE TABLE t (a text, b text, c text PRIMARY KEY ((a), b))
WITH CLUSTERING ORDER BY (b ASC);

-- Primary key with composite partition key a + b, clustering key c
CREATE TABLE t (a text, b text, c text, d text, PRIMARY KEY ((a, b), c));

-- Primary key with partition key a, clustering keys b + c
CREATE TABLE t (a text, b text, c text, d text, PRIMARY KEY ((a), b, c));

-- Primary key with partition key a, clustering keys b + c (alternative syntax)
CREATE TABLE t (a text, b text, c text, d text, PRIMARY KEY (a, b, c));
```

> [!note] El concepto de primary key es prácticamente **1:1 con el de Amazon DynamoDB** (partition key + sort/clustering key).

## Partitioning — de hashing ingenuo a consistent hashing

Cassandra escala horizontal particionando datos entre nodos con [[Consistent Hashing]]. El hashing tradicional `hash(value) % num_nodes` tiene **2 problemas**:

1. Cambiar el nº de nodos (agregar/quitar) **re-mapea muchísimos valores** → mover datos en exceso.
2. Un hashing desafortunado genera **carga desigual** entre nodos.

**Consistent hashing** hashea el valor a un **ring** y camina **clockwise** hasta el primer nodo → un nodo que entra/sale solo afecta a **un adyacente**.

![[cassandra-02-consistent-hashing-ring.svg]]
*Ring de consistent hashing: se hashea al ring y se camina en sentido horario hasta el primer nodo*

Para la carga desigual: **vnodes** (virtual nodes) — varios nodos del ring mapeados a un mismo **nodo físico**; los valores del ring son **tokens** (t1, t2…), y las máquinas más grandes toman más vnodes.

![[cassandra-03-vnodes-ring.svg]]
*Ring con vnodes y tokens (t1, t2…); el color indica el nodo físico dueño*

## Replication

Escaneo **clockwise** desde el vnode hasheado para hallar N réplicas, **saltando vnodes del mismo nodo físico** ya incluido (para que no caigan varias réplicas juntas cuando un físico se cae).

![[cassandra-04-replication.svg]]
*Replication — escaneo clockwise para elegir réplicas, saltando vnodes del mismo nodo físico*

Dos estrategias:
- **NetworkTopologyStrategy** (producción): consciente de **DC/rack**, réplicas repartidas en varios data centers y racks.
- **SimpleStrategy** (test): solo escaneo clockwise.

```sql
-- 3 replicas
ALTER KEYSPACE hello_interview WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 3 };

-- 3 replicas in data center 1, 2 replicas in data center 2
ALTER KEYSPACE hello_interview WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'dc1' : 3, 'dc2' : 2};
```

## Consistency

Sujeta al **CAP Theorem**, con consistencia **sintonizable**. **No ofrece transacciones ni garantías ACID** — solo escrituras **atómicas y aisladas a nivel de fila** dentro de una partición.

Niveles de consistencia (nº de réplicas que deben responder), de **ONE** (una réplica) a **ALL** (todas). El más usado es **QUORUM = n/2+1**: aplicado a lecturas *y* escrituras garantiza **overlap** → la lectura siempre ve la última escritura.

![[cassandra-05-quorum.svg]]
*Ejemplo de QUORUM con 3 nodos: 3/2+1 = 2, así 2 de 3 deben escribirse y leerse; al menos 1 nodo se solapa*

Objetivo general: **eventual consistency** (dado suficiente tiempo, todas las réplicas convergen).

### Lightweight Transactions (LWT) — la escotilla de consistencia fuerte

Aunque no hay ACID general, Cassandra SÍ ofrece **escrituras condicionales** con consistencia **linealizable** (compare-and-set) para casos puntuales como unicidad o reservas:

```sql
INSERT INTO users (username, email) VALUES ('german', 'g@x.com') IF NOT EXISTS;
UPDATE tickets SET status = 'sold' WHERE seat_id = 42 IF status = 'available';
```

Usan el protocolo **Paxos** con **4 fases** (prepare → promise → propose → commit) = **4 round-trips** coordinador↔réplicas → **latencia altísima**. Se usan con moderación, solo cuando el workload lo pide. Tienen un nivel de consistencia propio para la fase Paxos: **`SERIAL`** / **`LOCAL_SERIAL`**. Caso típico: en Ticketmaster (más abajo), el browsing va sin consistencia, pero el **checkout real de un asiento** se resolvería con un LWT (`IF status = 'available'`) para evitar la doble venta.

## Query Routing

Cualquier nodo puede ser **coordinador**: conoce los nodos vivos vía gossip, ubica los datos con consistent hashing, y enruta la query a las réplicas correspondientes.

![[cassandra-06-query-routing.svg]]
*Query Routing (cliente → coordinador → réplicas)*

## Storage Model (LSM tree, no B-tree)

El storage layer es la clave del write throughput: usa un **Log Structured Merge Tree (LSM tree)** en vez de un B-tree. Prioriza escritura: cada create/update/delete es una **entrada nueva** (*append-only*); el **orden** de las entradas determina el estado de la fila; los deletes escriben un **tombstone**. Tres constructos:

1. **Commit Log** — un [[Write-Ahead Log]] para durabilidad.
2. **Memtable** — estructura en memoria, ordenada por primary key.
3. **SSTable** ("Sorted String Table") — archivo **inmutable** en disco, volcado (flush) desde un Memtable.

**Secuencia de escritura**:
1. Llega la escritura a un nodo.
2. Se escribe al **commit log** (para no perderla si el nodo se cae).
3. Se escribe al **Memtable**.
4. Al superar un umbral de tamaño/tiempo, el Memtable se **flushea a un SSTable inmutable**.
5. Se limpian del commit log los mensajes de ese Memtable (ya son superfluos).

![[cassandra-07-storage-model.svg|775]]
*Storage Model*

**Lectura**: Memtable primero (dato más reciente) → si no está, un **Bloom Filter** decide qué SSTables mirar → lee SSTables **de más nuevo a más viejo** (ordenados por PK). Dos conceptos más:
- **Compaction** — consolida varios SSTables en un set más chico, removiendo tombstones y updates viejos. Eficiente porque todo está ordenado.
- **SSTable Indexing** — mapea key → byte offset (ej. key **12** → offset **984**) para acelerar la lectura en disco, parecido a cómo un B-tree apunta a disco.

## Gossip

Los nodos propagan el estado del cluster con **gossip**, un esquema peer-to-peer. El conocimiento universal elimina puntos únicos de fallo. Por cada nodo conocido se manejan dos números que juntos forman un **vector clock** (ver [[Vector Clocks]]), permitiendo descartar info vieja:
- **generation** — timestamp de cuándo el nodo fue *bootstrapped*.
- **version** — reloj lógico que incrementa cada ~1 segundo.

Los nodos eligen con quién hacer gossip con sesgo hacia los **seed nodes** — "hotspots"/choke points que garantizan que la info llegue a todo el cluster (evitan sub-clusters aislados), descubribles vía service discovery.

## Fault Tolerance

Cassandra detecta fallos durante el gossip con un **Phi Accrual Failure Detector**: cada nodo decide independientemente si otro está vivo. Un nodo que no responde es **"convicted"** (deja de recibir escrituras) pero puede **re-entrar** al volver a emitir heartbeats. Nunca se lo considera "down" de verdad salvo decomisión manual — así se evita el re-balanceo por fallos intermitentes.

![[cassandra-08-fault-tolerance.svg]]
*Fault Tolerance — hinted handoff*

Para reconciliar réplicas hay **3 mecanismos** complementarios:

1. **Hinted handoff** (corto plazo) — cuando el coordinador ve un nodo offline, guarda un **"hint"** con la escritura y se la reenvía al volver. ⚠️ Los hints se **descartan** si el nodo estuvo caído más de **`max_hint_window`** (default **3 horas**).
2. **Read repair** — durante una lectura (con QUORUM+), el coordinador compara réplicas y repara al vuelo las divergentes.
3. **Anti-entropy repair** — reparación activa/programada que usa **Merkle trees** (árboles de hash: las hojas son el hash de cada fila, cada padre es el hash de sus hijos) para detectar qué rangos divergen entre réplicas y streamear solo las diferencias. Es **costoso** (CPU, memoria, disco, red); hay una variante **incremental** (solo SSTables no reparadas) para abaratarlo.

> [!warning] Gotcha operacional — `gc_grace_seconds` + tombstone resurrection
> Un **tombstone** (marca de borrado) no se purga hasta pasados **`gc_grace_seconds`** (default **10 días**). Si un nodo estuvo caído y NO corriste anti-entropy repair antes de ese plazo, el dato borrado puede **"resucitar"** (tombstone resurrection) cuando el nodo vuelve con la versión vieja. Regla: **correr repair periódicamente, con período < `gc_grace_seconds`**.

## Data Modeling — query-driven, no entity-relationship

El modelado relacional es *entity-relationship-driven*: datos normalizados, una copia de cada entidad, relaciones vía foreign keys y JOINs. Cassandra es lo opuesto: **no tiene FK, ni integridad referencial, ni JOINs**, y **no favorece la normalización**. El modelado es **query-driven** — se parte de los *access patterns* de la app y se **denormaliza** (duplica) entre tablas lo que haga falta. Las 4 consideraciones:

- **Partition Key** — qué determina en qué partición cae el dato.
- **Partition Size** — qué tan grande puede crecer una partición.
- **Clustering Key** — cómo se ordenan los datos (si es que se ordenan).
- **Data Denormalization** — qué datos hay que duplicar para servir las queries.

> [!warning] Los NÚMEROS de partition size
> Regla práctica: cada partición **≤ 100 MB** y **≤ 100.000 filas** (idealmente bastante menos). Pasado eso → latencias altas, presión de GC/heap, compaction lenta, hotspots. Este límite es **el que motiva** el bucketing de Discord y el particionado por sección de Ticketmaster (abajo).

### Ejemplo: Discord Messages

Los canales de Discord son *group chats* muy activos; los usuarios leen lo reciente en orden cronológico inverso. Schema original:

```sql
CREATE TABLE messages (
  channel_id bigint,
  message_id bigint,
  author_id bigint,
  content text,
  PRIMARY KEY (channel_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

> [!note] ¿Por qué `message_id` y no `created_at`? Discord usa **Snowflake IDs** = UUID ordenable cronológicamente. Colisión imposible (es un UUID), mientras que un timestamp —aun con granularidad de ms— sí puede colisionar.

La partition key `channel_id` sirve un canal desde **una sola partición** (evita el *scatter-gather*). Problema: canales muy activos → **particiones grandes** → mal rendimiento + crecimiento monótono. Solución: agregar un **bucket** a la partition key = **10 días** de datos desde `DISCORD_EPOCH` (1 enero 2015). Acota el tamaño y rota la partición automáticamente con el tiempo:

```sql
CREATE TABLE messages (
  channel_id bigint,
  bucket int,
  message_id bigint,
  author_id bigint,
  content text,
  PRIMARY KEY ((channel_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

### Ejemplo: Ticketmaster (el "Taylor Swift Problem")

La UI de browsing de tickets **no necesita consistencia estricta** (la disponibilidad cambia mientras mirás; el checkout real la valida aparte). El caso simplifica los picos de eventos ultra-populares, apodados **"The Taylor Swift Problem"**. Primer schema (problema: eventos con **10.000+ tickets** obligan a agregaciones caras y muy frecuentes):

```sql
CREATE TABLE tickets (
  event_id bigint,
  seat_id bigint,
  price bigint,
  -- seat_id is added as a clustering key to ensure primary key uniqueness; order
  -- doesn't matter for the app access patterns
  PRIMARY KEY (event_id, seat_id)
);
```

La UX del browsing motiva el rediseño — primero se ve el mapa del venue con secciones:

![[cassandra-09-ticketmaster-venue.png]]
*Ticketmaster venue map with section popovers*

…y al entrar a una sección, los asientos individuales:

![[cassandra-10-ticketmaster-seat.png]]
*Ticketmaster individual seat view*

Eso sugiere agregar `section_id` a la partition key (distribuye el evento en varias particiones, cada una sirve menos datos):

```sql
CREATE TABLE tickets (
  event_id bigint,
  section_id bigint,
  seat_id bigint,
  price bigint,
  PRIMARY KEY ((event_id, section_id), seat_id)
);
```

Y una tabla **denormalizada** de resumen por sección para la vista de venue completa (tolera eventual consistency — Ticketmaster muestra "100+", no el número exacto; los eventos tienen **< 100 secciones** → single partition):

```sql
CREATE TABLE event_sections (
  event_id bigint,
  section_id bigint,
  num_tickets bigint,
  price_floor bigint,
  -- section_id is added as a clustering key to ensure primary key uniqueness; order
  -- doesn't matter for the app access patterns
  PRIMARY KEY (event_id, section_id)
);
```

## Features avanzadas

- **Storage Attached Indexes (SAI)** — índices secundarios *(locales, ver nota abajo)*; más flexibles pero con rendimiento peor que la query por partition key. Evitan denormalizar de más para queries poco frecuentes. Es el índice secundario **recomendado en producción** (Cassandra 5.0+).
- **SASI (SSTable Attached Secondary Index)** — otro índice secundario que habilita queries de texto/rango: `LIKE 'abc%'` (PREFIX, **Trie-based**), `LIKE '%abc%'` (CONTAINS, **Trie-based**) y rangos `>`/`<` (SPARSE, **B+ tree**). Liga el ciclo de vida del índice al del SSTable (compacta junto con los datos). ⚠️ Es **experimental** y tiene bugs en edge cases → **SAI lo reemplaza** en producción.
- **Materialized Views** — Cassandra materializa tablas a partir de una tabla fuente (denormalización automática, sin escribir a múltiples tablas manualmente).
- **Search Indexing** — integra un motor de búsqueda distribuido (ElasticSearch, Apache Solr) vía plugins, ej. el **Stratio Lucene Index**.
- **TTL (Time-To-Live)** — expiración automática de filas/columnas (en segundos): ideal para sesiones, tokens, cache. Al vencer, el dato se vuelve un **tombstone** (interactúa con compaction y `gc_grace_seconds`).

```sql
INSERT INTO sessions (id, data) VALUES ('abc', '...') USING TTL 3600;  -- expira en 1h
```

> [!warning] "Local" vs "global": todos los índices secundarios de Cassandra son LOCALES
> Cassandra los llama a veces "global secondary index", pero es impreciso: **todos** los índices secundarios (2i clásico, SASI, SAI) son **locales por nodo** — el índice vive co-ubicado con los datos. Por eso una query por columna indexada **sin** partition key hace **scatter-gather** a todas las réplicas y el coordinador junta los resultados. El término **"global secondary index" es propio de Amazon DynamoDB**, no de Cassandra.

## ¿Cuándo usar Cassandra? (decisión)

| Usala si… | Evitala si… |
|---|---|
| Disponibilidad > consistencia, alta escalabilidad | Necesitás consistencia estricta |
| Alto **write throughput** (LSM tree) | Queries avanzadas: JOINs multi-tabla, agregaciones ad-hoc |
| **Wide-column** / schema flexible o sparse | |
| **Access patterns claros** (modelado query-driven) | |

### ScyllaDB — la alternativa que preguntan en entrevistas

**ScyllaDB** es una reescritura de Cassandra en **C++** (sin JVM, arquitectura *shard-per-core* / shared-nothing), **API-compatible con CQL** — drop-in en muchos casos. Suele dar **latencias más bajas y consistentes**, mejor manejo de **hot partitions** y menos overhead operacional (nada de tuning de GC de la JVM). Dato relevante: **Discord migró de Cassandra a ScyllaDB** (justo el caso de estudio de arriba) por latencia/GC a escala de billones de mensajes. Trae además **Alternator**, una API compatible con [[DynamoDB]]. En una entrevista suma decir *"usaría Cassandra, o ScyllaDB si necesito exprimir latencia/costo a escala"*.

## 🎯 Preguntas de retrieval

> [!question] ¿Por qué QUORUM garantiza que una lectura vea la última escritura?
> Porque exige n/2+1 réplicas en escritura *y* lectura. Con 3 nodos son 2 de 3, y cualquier par de subconjuntos de 2 sobre 3 necesariamente se solapa (pichonera) → al menos un nodo que participó de la escritura participa de la lectura.

> [!question] ¿Por qué Discord agregó `bucket` en vez de solo `channel_id`?
> Porque `channel_id` solo generaba particiones que crecían sin límite (mala performance sobre ~100 MB / 100k filas). El bucket (ventana fija de 10 días) acota el tamaño y rota la partición automáticamente con el tiempo, sin perder el servir la mayoría de queries desde una sola partición.

> [!question] ¿Por qué Cassandra usa un LSM tree y no un B-tree?
> Porque prioriza velocidad de escritura. El LSM tree permite escrituras casi enteramente append-only (Commit Log → Memtable → flush a SSTable inmutable), evitando las reescrituras in-place costosas de un B-tree, a costa de que la lectura revise varios SSTables (mitigado con bloom filters e indexing).

## References

- Source: [Cassandra](https://www.hellointerview.com/learn/system-design/deep-dives/cassandra) — Hello Interview
- Las figuras embebidas son SVGs/PNGs de `files.hellointerview.com`.
- Enriquecimientos fuera del artículo (LWT/Paxos, partition-size limits, anti-entropy repair + Merkle trees, `gc_grace_seconds`, SASI, TTL, ScyllaDB) verificados contra docs de Apache Cassandra, DataStax, AxonOps, Instaclustr y ScyllaDB.

## Related

- [[_Databases|Databases]]
- [[_System Design|System Design]]
- [[Consistent Hashing]]
- [[Quorum]]
- [[Write-Ahead Log]]
- [[Sharding]]
- [[Vector Clocks]]
- [[Networking Essentials]]
