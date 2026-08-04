---
title: Scaling Writes
reading:
  total_words: 1550
  read_words: 1550
  pct: 100
  last_read: 2026-07-02
source: HelloInterview — pattern "Scaling Writes" (guía compilada a partir de DesignGurus, DevX, Elysiate, Medium, Google Cloud docs)
author: Guía compilada para entrevistas de System Design
created: 2026-07-02
tags:
  - system-design/scalability
  - type/pattern
  - status/permanent
aliases:
  - Scaling Writes
  - Escalar Escrituras
updated: 2026-07-02
---

> [!note] Definición
> Scaling Writes es el patrón de system design para sistemas donde el cuello de botella son las escrituras (no las lecturas): a diferencia de las lecturas, que se escalan con réplicas y cache, toda escritura debe llegar a la fuente de verdad. La escalera de soluciones va de vertical scaling y motor de storage adecuado (LSM), a [[Sharding|sharding horizontal]] con buena shard key, colas + load shedding para picos, y batching/agregación para reducir volumen.

## The Challenge

A diferencia de las lecturas (que escalás con réplicas y cache), las **escrituras son más difíciles de escalar** porque toda escritura debe ir a la fuente de verdad. Una réplica no absorbe escrituras — todas van al primary. Cuando pasás de cientos a millones de escrituras por segundo, un servidor individual choca contra límites duros: throughput de disco, IOPS, tamaño del dataset.

Escenarios donde las escrituras son el cuello: ingesta de métricas/telemetría, ad click aggregator, logging de eventos, likes/engagement a escala, IoT, feeds de trading.

> **Regla de oro:** el error más grande es shardear demasiado temprano. Una sola instancia de Postgres aguanta ~10-50k escrituras/seg y varios TB. No shardees a 100GB o 10k TPS; hacé la cuenta de capacidad primero y justificá por qué una base no alcanza. Ojo con citar ese rango como un límite físico exacto: depende demasiado del hardware, el schema y el patrón de I/O como para dar un número único y confiable — es una heurística de entrevista, no un benchmark verificado. En la mesa conviene decirlo así: "depende del hardware, pero como orden de magnitud...". ([Tiger Data](https://www.tigerdata.com/learn/postgresql-vs-cassandra-decision-framework-time-series-write-heavy-workloads))

---

## The Solution

### Vertical Scaling and Write Optimization

#### Vertical Scaling

La primera opción y la más simple: máquina más grande (más CPU, RAM, SSD/NVMe más rápido, más IOPS). El hardware moderno es potente y esto compra mucho tiempo sin la complejidad de un sistema distribuido. Techo: en algún punto no hay máquina más grande, o el costo se dispara.

#### Database Choices

La elección de motor importa para writes. Bases con **modelo de almacenamiento append-only / LSM-tree** (Cassandra, RocksDB) son mucho mejores para write-heavy que las basadas en B-tree, porque las escrituras son secuenciales (append) en vez de updates in-place. Cassandra es la referencia clásica para "write-heavy con alta disponibilidad". Trade-off: mejor throughput de escritura a costa de lecturas más complejas (compaction, read amplification).

El matiz importante acá es que el trade-off no es unidireccional: Cassandra puede tragar millones de writes/seg en discos que ahogarían a Postgres, pero si el workload lee 10x más de lo que escribe, un B-tree le gana a un LSM tree en el mismo hardware casi siempre — la elección depende del ratio read/write real, no solo del volumen de escritura. ([Tiger Data](https://www.tigerdata.com/learn/postgresql-vs-cassandra-decision-framework-time-series-write-heavy-workloads)) La razón por la que el LSM escribe tan rápido: Cassandra escribe al mismo tiempo a un memtable en memoria y a un commit log en disco (secuencial), y recién después flushea a SSTables inmutables en background — evita los page splits y el I/O random que un B-tree sufre bajo escritura intensiva. ([DEV Community — Cassandra Internals](https://dev.to/priteshsurana/cassandra-internals-lsm-tree-sstables-and-compaction-2ai8))

**Frase:** "Si el workload es write-heavy, considero una base con storage LSM como Cassandra en vez de un RDBMS con B-tree, porque las escrituras secuenciales escalan mucho mejor."

### Sharding and Partitioning

#### Horizontal Sharding

Partís los datos en shards distribuidos en distintos servidores; cada shard maneja un subconjunto de los datos **y sus escrituras**, así el load total se reparte. Distribuyendo las escrituras entre N shards, el sistema maneja N veces más throughput. Bonus: mayor disponibilidad (si un shard cae, los otros siguen) y escalabilidad incremental (agregás shards).

**La decisión central — la shard key:** determina cómo se distribuyen los datos y define todo lo demás. Debe (a) distribuir la carga **parejo** y (b) mantener juntos los datos que se consultan juntos. **Hash-based** es el default (distribuye parejo); **range-based** crea hot spots fácilmente.

**Trampas a nombrar:**
- Queries y transacciones **cross-shard** se vuelven difíciles y caras.
- Shard key mal elegida → **hot shards** (desbalance severo que anula el beneficio).
- Overhead operativo (monitoreo, migraciones, backups) se multiplica.

#### Vertical Partitioning

Separás distintos **tipos** de datos en distintos stores (ej. la metadata de un post en una base, el blob del contenido en otra; las columnas hot en una tabla, las cold en otra). Reducís el tamaño de cada fila/tabla y aislás el load. Es complementario al sharding horizontal.

### Handling Bursts with Queues and Load Shedding

#### Write Queues for Burst Handling

Aún un sistema bien shardeado puede caer ante spikes. En vez de escribir directo a la base, encolás cada escritura (Kafka, SQS, RabbitMQ) y workers de fondo drenan la cola a un ritmo **estable**. La cola **desacopla** la tasa de entrada de la tasa de procesamiento, absorbiendo el pico (buffering).

Ejemplos: Strava encola eventos de workout antes de escribirlos, suavizando los picos de la mañana/tarde; YouTube encola uploads para transcoding y writes de metadata.

**Trade-off:** agrega latencia (la escritura ahora es asíncrona → eventual consistency) y complejidad (mantener la cola, no perder datos). El usuario ve "procesando" en vez de confirmación inmediata.

#### Load Shedding Strategies

Si el diluvio es tan grande que la cola sigue creciendo sin parar, el sistema hace **load shedding**: rechaza o descarta algunas escrituras para protegerse. Solo aceptable para datos **no críticos** (ej. saltear un update de métrica no esencial durante un surge). Complementás con **backpressure** (avisar río arriba que frene), **backoff con jitter** en los reintentos, y circuit breakers. Priorizás: las escrituras importantes pasan, las descartables se tiran.

### Batching and Hierarchical Aggregation

#### Batching

En vez de N escrituras separadas, agrupás muchas en una sola operación grande. El costo fijo por escritura (setup de transacción, round-trip de red) se paga **una vez** en vez de N. Ejemplo: en vez de 100 inserts individuales, un batch insert de 100. Amortiza el costo fijo y dispara el throughput.

#### Hierarchical Aggregation

En vez de escribir a la base por cada evento chico, mantenés un tally en memoria y escribís el agregado periódicamente (ej. un contador de vistas: sumás en memoria y escribís el total una vez por minuto). Reducís drásticamente la cantidad de escrituras. Para escala masiva, agregás en **niveles** (local por servidor → regional → global), como en un ad click aggregator o métricas. Trade-off: pierdes granularidad y frescura (el contador está atrasado unos segundos/minutos).

**Frase:** "Para un contador de alta frecuencia, no escribo por evento — agrego en memoria y hago flush del sumado cada N segundos, cambiando precisión en tiempo real por una reducción enorme de escrituras."

Este patrón de hierarchical aggregation / hot key splitting tiene un antecedente clásico en Google App Engine: sus entity groups tenían un límite de ~5 writes/seg, así que Google recomendó "sharded counters" — repartir los incrementos entre N shards de contador elegidos al azar y sumar todos al leer, manteniendo consistencia fuerte sin sacrificar throughput. Es el mismo principio que "Split All Keys" aplicado específicamente a contadores. ([Google App Engine — Sharding counters](https://download.huihoo.com/google/gdgdevkit/DVD1/developers.google.com/appengine/articles/sharding_counters.html))

---

## When to Use in Interviews

### Common Interview Scenarios

Ad click aggregator, métricas/monitoring, YouTube top-K, Strava (ingesta de actividades), rate limiter (contadores), FB post search (indexado), logging de eventos, IoT, cualquier sistema de ingesta de alto volumen.

### When NOT to Use in Interviews

- Si el volumen de escritura es moderado (miles por segundo, no millones), una base sola con vertical scaling alcanza. **No shardees prematuramente** — es la trampa número uno.
- Si el cuello es de **lectura**, este patrón no aplica (andá a Scaling Reads).
- No metas colas si los jobs son cortos y la escritura síncrona simplifica la arquitectura y da mejor back-pressure y UX. "Muchos candidatos meten una cola por reflejo; frecuentemente es mala decisión."

---

## Common Deep Dives

### "¿Cómo manejás el resharding cuando necesitás agregar más shards?"

El problema: las decisiones de particionado que funcionaban a 10k usuarios fallan a 10M, y resharding es caro — puede requerir mover todo el dataset.

- **Consistent hashing:** en vez de `hash(key) % N` (que remapea ~90% de las claves al cambiar N), usás un hash ring donde agregar/quitar un nodo solo mueve ~10% de las claves. Esta es la respuesta canónica.
- **Virtual nodes:** cada servidor físico ocupa muchas posiciones en el ring, para que la distribución sea pareja y el rebalanceo sea suave. En números concretos, DynamoDB (siguiendo el paper de Dynamo) típicamente asigna ~256 virtual nodes por nodo físico, distribuidos de forma no contigua en el ring para que ningún par de vnodes vecinos caiga en el mismo servidor físico — así la falla de un nodo físico se reparte entre muchos vecinos en vez de sobrecargar a uno solo. ([Medium — Consistent Hashing: Amazon DynamoDB](https://medium.com/@adityashete009/consistent-hashing-amazon-dynamodb-part-1-f5719aff7681))
- **Pre-sharding:** creás muchos más shards lógicos de los que necesitás (ej. 1024) y los mapeás a pocos servidores físicos; al escalar, movés shards lógicos enteros sin re-hashear datos.
- **Migración online (patrón tipo Vitess):** copiás datos al nuevo shard con replicación filtrada mientras servís tráfico, y hacés el switch de reads/writes al final (split clone → replicación → switch). Validás con queries de paridad pre/post. El detalle operativo es que el reshard de Vitess no es un evento único — se hace en fases (`rdonly` → `replica` → `master`/primary), un cell a la vez, y el corte de tráfico de escritura es el último paso, después de validar paridad de datos entre shard viejo y nuevo. Esto reduce el blast radius de un resharding mal calculado. ([Vitess Docs — What is resharding?](https://vitess.io/docs/faq/sharding/overview/what-is-resharding-how-does-it-work/))
- **Ojo en Kafka:** aumentar particiones **cambia la semántica de ordering** — hay que planificarlo.

### "¿Qué pasa cuando tenés una hot key demasiado popular para un solo shard?" (hot partition)

El problema: aunque shardees bien, una sola clave (el usuario Taylor Swift, un contador viral) recibe tanto tráfico de escritura que satura su shard. El principio: si todas las escrituras caen "al final" del índice (ej. timestamps), ese final se calienta.

**Split All Keys**
Aumentás la granularidad de **todas** las claves agregando un sufijo. Ej: `user123` se convierte en `user123-{0..9}` — las escrituras de un mismo usuario se distribuyen en 10 shards lógicos. Las lecturas ahora requieren consultar los 10 y agregar (scatter-gather). Cambiás complejidad de lectura por escalabilidad de escritura. Común en sistemas de logging de eventos de alto throughput. Costo: penalizás **todas** las lecturas, incluso las de claves que no son hot.

**Split Hot Keys Dynamically**
En vez de saltear todo, detectás las claves calientes en tiempo real (vía top-K / conteo de accesos) y **solo esas** las splitteás dinámicamente. El resto queda simple. Más eficiente porque no penalizás las lecturas de las claves normales, pero más complejo (necesitás detección de hotkeys y ruteo adaptativo). Es lo que hacen sistemas como DynamoDB con su adaptive capacity.

**La distinción para la mesa:** split-all es simple pero castiga todas las lecturas; split-dynamic es eficiente pero necesita maquinaria de detección. Elegís según cuán concentrado y predecible sea el hotspot.

---

## Conclusion

El patrón es una escalera del más simple al más complejo: vertical scaling + motor adecuado (LSM para write-heavy) → sharding horizontal con buena shard key → colas + load shedding para picos → batching/agregación para reducir el volumen. La decisión central es siempre la **shard key** (hash para distribuir, mantener juntos los datos co-consultados), y los dos deep-dives que caen son resharding (consistent hashing, pre-sharding, migración online) y hot partitions (split-all vs split-dynamic).

La frase que resume la actitud: diseñá con la shard key primero y construí el músculo operativo — pero nunca shardees antes de justificar, con números, que una sola base no alcanza. La sobre-ingeniería temprana es lo que más se penaliza acá.

## References

- [PostgreSQL vs. Cassandra: The Decision Framework for Time-Series and Write-Heavy Workloads — Tiger Data](https://www.tigerdata.com/learn/postgresql-vs-cassandra-decision-framework-time-series-write-heavy-workloads)
- [Cassandra Internals: LSM Tree, SSTables, and Compaction — DEV Community](https://dev.to/priteshsurana/cassandra-internals-lsm-tree-sstables-and-compaction-2ai8)
- [Consistent Hashing: Amazon DynamoDB (Part 1) — Medium](https://medium.com/@adityashete009/consistent-hashing-amazon-dynamodb-part-1-f5719aff7681)
- [Consistent Hashing for System Design Interviews — Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing)
- [What is resharding? How does it work? — Vitess Docs](https://vitess.io/docs/faq/sharding/overview/what-is-resharding-how-does-it-work/)
- [Sharding counters — Google App Engine Docs](https://download.huihoo.com/google/gdgdevkit/DVD1/developers.google.com/appengine/articles/sharding_counters.html)

## Related

- [[Sharding]]
- [[Consistent Hashing]]
- [[Write-Ahead Log]]
- [[Write-Behind]]
- [[CQRS]]
- [[Message Queue]]
- [[Horizontal Scaling]]
- [[Vertical Scaling]]
