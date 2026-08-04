---
title: Kafka
created: 2026-07-01
tags:
  - system-design/streaming
  - type/technology
  - status/permanent
aliases:
  - Kafka
  - Apache Kafka
updated: 2026-07-05
reading:
  total_words: 4516
  read_words: 0
  pct: 0
  last_read: 2026-07-01
---

# Kafka

> [!note] Tesis operativa
> Kafka es una plataforma de **event streaming** distribuida que se puede usar como **message queue** O como **stream** — la diferencia no es la tecnología, es el patrón de consumo. Su regla de oro es *"always available, sometimes consistent"*: por default prioriza disponibilidad sobre consistencia estricta, salvo que vos mismo actives las garantías (`acks=all` + `min.insync.replicas`) que te la dan. La usa el 80% del Fortune 100.

## Marco mental (leé esto primero)

Cuatro conceptos sostienen todo lo demás:

- **Broker**: un servidor (físico o virtual) del cluster. Guarda datos y sirve a los clientes.
- **Partition**: un log ordenado, inmutable, *append-only*. Es la unidad de paralelismo.
- **Topic**: la agrupación lógica de partitions. Multi-productor.
- **Producer / Consumer**: quien escribe y quien lee. Kafka no sabe (ni le importa) qué es el dato — es un *byte array* con estructura opaca para el broker.

**Topic vs Partition**, la distinción que hay que tener clara desde el arranque: el topic es la agrupación **lógica** (cómo organizás), la partition es la unidad **física** (cómo escalás). Un topic tiene múltiples partitions, y cada una vive en un broker distinto — de ahí sale el paralelismo.

## El ejemplo motivador (Mundial) — cadena causal

1. Pensá un sitio de estadísticas en tiempo real de un Mundial de fútbol. La arquitectura más simple: evento → cola → producer llena la cola → consumer la lee y actualiza el sitio.
2. Escalá eso a 1000 partidos simultáneos y la cola en un solo servidor no aguanta el volumen. La solución obvia es distribuir la cola entre varios servidores.
3. Pero repartir los eventos al azar entre esos servidores **rompe el orden**: un gol podría procesarse antes de que el partido arrancó. La solución es distribuir por **partido** (la key) — así todos los eventos de un mismo partido caen en la misma cola, en orden. Esta es la idea fundamental de Kafka: los mensajes se distribuyen entre partitions según una **partitioning strategy**.

![[kafka-01-motivating-producer-consumer.svg|702]]

![[kafka-02-scaling-queue.svg|634]]

![[kafka-03-partition-by-key.svg|679]]

4. Ahora un solo consumer se ve abrumado por el volumen total. Entra el **consumer group**: cada partition se asigna a exactamente un consumer del grupo, nunca a dos a la vez. (Esto no impide que un mensaje se reprocese bajo *at-least-once* — pero jamás se divide un mismo mensaje entre dos consumers.)

![[kafka-04-consumer-group.svg|635]]

5. Agregás más deportes al sitio: cada tipo de evento va a su propio **topic**, y los consumers se suscriben al que les interesa.

![[kafka-05-topics.svg|718]]

## Anatomía del mensaje

Un mensaje (**record**) tiene cuatro partes: **value** (el payload real), **key** (determina a qué partition va), **timestamp** (cuándo se creó el mensaje — pero ojo, el **orden** dentro de una partition lo define el **offset**, no el timestamp) y **headers** (metadata estilo cabeceras HTTP).

![[kafka-06-message-anatomy.svg]]

Si no mandás key, Kafka no reparte round-robin mensaje por mensaje — usa el **sticky partitioner**: agrupa varios mensajes en una misma partition y recién ahí rota a la siguiente, para poder batchear.

CLI producer:

```bash
kafka-console-producer --bootstrap-server localhost:9092 --topic my_topic --property "parse.key=true" --property "key.separator=:"
> key1: Hello, Kafka with key!
> key2: Another message with a different key
```

kafkajs producer:

```javascript
// Initialize the Kafka client
const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092']
})

// Initialize the producer
const producer = kafka.producer()

const run = async () => {
  // Connecting the producer
  await producer.connect()

  // Sending messages to the topic 'my_topic' with keys
  await producer.send({
    topic: 'my_topic',
    messages: [
      { key: 'key1', value: 'Hello, Kafka with key!' },
      { key: 'key2', value: 'Another message with a different key' }
    ],
  })
}
```

## Selección de partition (2 pasos)

1. **Partition Determination**: `hash(key) % num_partitions` con **murmur2** como default. La misma key siempre hashea a la misma partition → así se preserva el orden por key. Sin key, round-robin vía sticky partitioner.
2. **Broker Assignment**: el mapeo partition→broker no lo decide el producer — lo mantiene la metadata del cluster, gestionada por el **Kafka controller**.

## Partition = distributed commit log

Que la partition sea *append-only* la convierte en un **distributed commit log**, y de ahí salen tres beneficios:

- **Immutability**: nada se modifica in-place → simplifica muchísimo la replicación, el recovery y la consistencia (no hay que razonar sobre "¿qué versión del dato es esta?").
- **Efficiency**: escribir siempre al final minimiza los *disk seeks*.
- **Scalability**: el patrón append-only facilita el particionado horizontal.

Cada mensaje tiene un **offset** único dentro de su partition. Los consumers commitean su offset para poder resumir donde quedaron. El default es **at-least-once**: si el consumer crashea después de procesar el mensaje pero antes de commitear el offset, al volver reprocesa ese mensaje. **Exactly-once** también es posible — ver la sección de delivery semantics más abajo.

![[kafka-07-partition-offsets.svg|690]]

### Ordering — solo dentro de la partition

El orden que Kafka garantiza es **únicamente dentro de una partition**, nunca cruzando partitions. Bajo reintentos entra en juego **`max.in.flight.requests.per.connection`**: con **idempotencia** activada, podés tener hasta 5 requests en vuelo simultáneas sin perder el orden (el broker usa números de secuencia para reordenar/deduplicar); sin idempotencia, más de 1 request en vuelo arriesga que un retry reordene mensajes.

No confundas dos cosas que suenan parecido y son un error clásico en entrevista: el **sticky partitioner** (producer-side, agrupa mensajes sin key en batches) no tiene nada que ver con el **CooperativeStickyAssignor** (consumer-side, cómo se reparten partitions en un rebalance — ver más abajo). Ojo también: el sticky partitioner del producer sí es el comportamiento por default desde 2.4; el CooperativeStickyAssignor del consumer NO — hay que activarlo explícitamente (ver rebalancing).

## Replicación leader-follower

1. **Leader replica** (una por partition): maneja todos los writes y, desde la versión 2.4, también puede servir reads si así se configura. La asigna el controller.
2. **Follower replication**: los followers viven en brokers distintos al leader y replican pasivamente — son el backup.
3. **Sync/consistency**: si el leader cae, se promueve un follower que esté *in-sync* (ISR).
4. **Controller**: gestiona la replicación, monitorea el estado de los brokers y reasigna el leader cuando hace falta.

### Rack awareness

`broker.rack` le dice a Kafka en qué AZ/failure-domain vive cada broker, y la asignación de replicas usa ese dato para repartir las réplicas de una partition entre racks/AZs distintas — así una caída de AZ completa no se lleva puestas todas las réplicas. Sin **rack awareness**, un replication factor de 3 no protege nada si las 3 réplicas terminan en la misma AZ.

## Pull-based

Los consumers hacen **poll** activo — Kafka nunca empuja (*push*) datos. Esto da: control de la tasa de consumo, manejo simple de fallos (el consumer decide cuándo pedir más), evita saturar a un consumer lento, y habilita batching natural.

CLI consumer:

```bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic my_topic --from-beginning --property print.key=true --property key.separator=": "

# Output
key1: Hello, Kafka with key!
key2: Another message with a different key
```

kafkajs consumer:

```javascript
// Initialize the Kafka client
const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092']
})

// Initialize the consumer
const consumer = kafka.consumer({ groupId: 'my-group' })

const run = async () => {
  // Connecting the consumer
  await consumer.connect()

  // Subscribing to the topic 'my_topic'
  await consumer.subscribe({ topic: 'my_topic' })

  // Consuming messages
  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      console.log({
        value: message.value?.toString(),
        key: message.key?.toString()
      })
    },
  })
}
```

![[kafka-08-tying-together.svg|950]]

### Offset commit modes — de dónde sale exactamente la pérdida o el duplicado

`enable.auto.commit=true` es el default: cada `auto.commit.interval.ms` (5 segundos) el cliente commitea automáticamente el offset más grande que ya se pollió, **sin importar si terminaste de procesarlo**. La alternativa manual: **commitSync** (bloquea hasta confirmar, más seguro) o **commitAsync** (no bloquea, pero puede commitear fuera de orden si hay un retry en curso).

El orden poll→process→commit es exactamente el mecanismo detrás de at-least-once:
- **Commit antes de procesar** → crashea durante el procesamiento → el mensaje quedó marcado como consumido pero nunca se procesó → **pérdida de mensaje**.
- **Commit después de procesar** → crashea entre procesar y commitear → al volver, reprocesa → **duplicado**.

Kafka elige el segundo camino por default (at-least-once): prefiere duplicar antes que perder.

## Message Queue vs Stream

Los dos patrones trackean progreso con **offset commits** — la diferencia está en la retención y en cuántos consumers leen el mismo dato:

- **Message queue**: cada mensaje lo procesa un consumer y queda "consumido" desde la perspectiva de la app (aunque Kafka lo sigue reteniendo igual — no lo borra al consumirlo).
- **Stream**: el log queda retenido y es *replayable*; múltiples consumer groups independientes pueden leer el mismo topic desde distintos puntos, habilitando procesamiento continuo.

**Cuándo usar message queue**: procesamiento asíncrono (ej. YouTube — servís el video en SD ya y transcodeás el resto después, con un puntero al job), necesitás orden (ej. la waiting queue de Ticketmaster), o simplemente querés desacoplar el escalado de productor y consumidor.

**Cuándo usar stream**: necesitás tiempo real (ej. un Ad Click Aggregator) o necesitás que múltiples consumers independientes lean el mismo stream (ej. los comentarios en vivo de un FB Live, donde cada viewer es un consumer distinto vía pub/sub).

## Kafka vs otras colas/streams

La pregunta que más aparece en entrevista, una vez que ya elegiste Kafka: *"¿por qué no una cola más simple?"*. El modelo mental clave es dónde vive la inteligencia: Kafka es **pull-based, replayable, con orden por partition**, y el broker es **tonto** (no sabe qué consumers hay, ni sus reintentos, ni sus DLQs — todo eso es responsabilidad del **consumer**, vía offset). [[Message Queue]] tradicionales como RabbitMQ o AWS SQS son lo opuesto: el broker es **inteligente** (trackea qué mensaje fue entregado a quién, ofrece redrive policies y TTL nativos) y el consumer es tonto.

| | Ordering | Throughput | Retention / Replay | Retry / DLQ | Push / Pull | Mejor para |
|---|---|---|---|---|---|---|
| **Kafka** | Por partition (key) | Muy alto (1M+ msgs/s) | Larga, replayable, múltiples consumer groups | No nativo — hay que construirlo (retry topic + DLQ manual) | Pull | Streaming, múltiples consumers independientes, event sourcing |
| **RabbitMQ** | Por cola (FIFO) | Medio | Corta, no replayable (se borra al consumir) | Nativo (dead-lettering, TTL) | Push | Routing complejo, colas de trabajo clásicas |
| **AWS SQS** | Best-effort (FIFO opcional) | Medio-alto | Corta (hasta 14 días), no multi-consumer real | Nativo (redrive policy, DLQ gratis) | Pull (long polling) | Desacoplar servicios, jobs asíncronos, gestión mínima |
| **AWS Kinesis** | Por shard | Alto | Configurable (hasta 365 días), replayable | No nativo | Pull | Streaming AWS-native, analytics en tiempo real |
| **Google Pub/Sub** | Best-effort (por ordering key opcional) | Alto | Configurable, replayable (seek) | Nativo (dead-letter topics) | Push o pull | Pub/sub a escala, integración GCP |

El follow-up clásico: *"elegiste Kafka, pero SQS te da DLQ gratis, ¿por qué no SQS?"* — porque necesitás **múltiples consumer groups independientes** replayando el mismo stream (SQS consume un mensaje una vez y listo), **orden garantizado por key** dentro de una partition, y throughput sostenido de 1M+ mensajes por segundo que un broker "inteligente" como SQS no está diseñado para sostener.

## Delivery semantics

### Exactly-once, en profundidad

Hay dos mecanismos que juntos dan exactly-once, y es importante entender qué cubre cada uno:

- **Idempotent producer**: el broker le asigna al producer un **Producer ID** (PID) y mantiene un número de secuencia monótono por partition. Si el producer reintenta un batch (por timeout, por ejemplo), el broker detecta la secuencia repetida y la deduplica. Esto es *producer-side* y funciona **dentro de una sola partition**.
- **Transactions**: usan un `transactional.id` fijo y un **transaction coordinator** en el broker. El consumer que quiere ver solo datos comprometidos configura `isolation.level=read_committed`, que respeta el **Last Stable Offset (LSO)** — el punto hasta donde todas las transacciones ya cerraron. Esto habilita escrituras atómicas a **múltiples partitions/topics a la vez**, el patrón *read-process-write* exactly-once (leer de un topic, transformar, escribir a otro, todo o nada).

El `transactional.id` también sirve para **fencing de zombie producers**: si un producer viejo (que se colgó pero sigue vivo) intenta escribir después de que un nuevo proceso tomó su `transactional.id`, el coordinator usa un número de época (*epoch*) creciente para rechazar al zombie.

> [!warning] EOS es angosto
> Exactly-once cubre escrituras **atómicas dentro de Kafka** + deduplicación — NO cubre efectos secundarios externos (una llamada HTTP, un insert en otra DB). Si tu pipeline toca algo fuera de Kafka, necesitás el patrón **outbox** (ver más abajo).

## Fault tolerance

La fault tolerance descansa en la replicación. `acks=all` le dice al producer que espere el ack hasta que todas las réplicas **ISR** (in-sync replicas) hayan recibido el mensaje. Con replication factor 3 (1 leader + 2 followers) el sistema tolera la caída de un broker sin perder datos — resumido en la frase *"Kafka is always available, sometimes consistent"*.

### min.insync.replicas — el knob real de durabilidad

`acks=all` por sí solo solo garantiza "esperá a la ISR **actual**" — si la ISR se achicó hasta quedar solo el leader (los followers cayeron), la condición se cumple trivialmente con un único broker. **`min.insync.replicas`** es el knob que realmente te protege: con replication factor 3 y `min.insync.replicas=2` + `acks=all`, la escritura necesita que **al menos 2** réplicas confirmen; si la ISR bajó por debajo de ese mínimo, el producer recibe `NotEnoughReplicasException` y la escritura **falla** en vez de degradar la durabilidad en silencio. Con RF=3 / `min.insync.replicas=2` tolerás exactamente 1 broker caído sin perder durabilidad ni disponibilidad.

Cuando el que cae es un **consumer** (no Kafka): entra el offset management (commitea después de procesar) y el **rebalancing** (redistribuye las partitions del consumer caído entre el resto del grupo). El trade-off de cuándo commitear aparece en casos como un Web Crawler: no commiteás hasta haber guardado el HTML descargado, con el consumer dividido en dos fases (download, parse) para que un crash a mitad de camino no pierda ni duplique trabajo de forma silenciosa.

### Cooperative rebalancing

Cuando un consumer se une o sale de un grupo, Kafka reasigna partitions — eso es un **rebalance**. El asignador *eager* clásico ("stop-the-world", el `RangeAssignor`/`RoundRobinAssignor` que sigue siendo el default) revoca **todas** las partitions de **todos** los consumers y las reasigna desde cero, aunque solo haya cambiado uno. El **CooperativeStickyAssignor** (introducido en 2.4, hay que activarlo explícitamente vía `partition.assignment.strategy` — no es el default) hace un rebalance **incremental**: solo reasigna las partitions que realmente necesitan moverse — ~20x más rápido en grupos grandes, y los consumers no afectados por el cambio siguen procesando sin interrupción.

## Scalability

`message.max.bytes` limita el tamaño de un mensaje individual (menos de 1MB por default).

> [!warning] Anti-patrón
> No metas blobs grandes en Kafka — no es una base de datos de objetos. Un video va a S3 y el puntero (la URL) va en el mensaje de Kafka.

Un broker aguanta del orden de ~1TB de datos y ~1M de mensajes por segundo. Hay dos ejes de escalado: horizontal (más brokers — pero necesitás suficientes partitions para aprovecharlos) y la **partitioning strategy**, que es LA decisión que más importa: `hash(key) % num_partitions` con murmur2; una key mal elegida concentra tráfico en pocas partitions y genera una **hot partition**. Lo que se escala son los topics, no el cluster entero. Managed services típicos: Confluent Cloud, AWS MSK.

**Hot partitions** — el caso clásico de entrevista es un Ad Click Aggregator donde un anuncio de Nike con LeBron explota en clicks. Cuatro mitigaciones, cada una con su costo:

1. Sin key — pierde el orden garantizado.
2. Random salting — reparte la carga pero complica la agregación (hay que combinar resultados de varias partitions después).
3. Compound key — combinar `ad_id` + región/segmento para dispersar sin perder toda la localidad.
4. Back pressure — frenar/throttlear el producer cuando la partition está saturada.

### Partition count planning

Las partitions se pueden **aumentar** pero **nunca reducir** — para bajar el número hay que borrar el topic y recrearlo, migrando los datos. Reglas prácticas: arrancá con 6-12 partitions y escalá según throughput real; evitá números primos (distribuyen mal los leaders entre brokers — preferí 12/24/30/48/60); y el número de partitions es un **techo duro** al paralelismo de consumo — consumers de más en el grupo que partitions simplemente quedan idle. Sobre-particionar también tiene costo: más file handles abiertos en cada broker, rebalances más lentos, latencia agregada.

### Tiered storage

**KIP-405** permite offloadear segmentos viejos de una partition a almacenamiento de objetos (S3), desacoplando storage de cómputo. Extiende directamente el límite de ~1TB por broker — con tiered storage la retención puede ser prácticamente infinita y barata sin escalar el disco de cada broker.

## Por qué Kafka es rápido

Tres factores, y ninguno es "vive en memoria" (Kafka NO es in-memory):

- **Sequential disk I/O**: el append es lineal, así que incluso escribiendo a disco (no SSD necesariamente) el throughput es altísimo — un disco secuencial puede superar a uno con acceso random incluso en SSD.
- **Page Cache** del sistema operativo: un consumer que está al día lee directamente de RAM (la page cache del OS), no toca disco.
- **Zero-copy** vía `sendfile()`: los datos van de la page cache directo al socket buffer de la placa de red, sin copiarse nunca al heap de la JVM. Menos copias, menos garbage collection, más throughput.

## Retries

Producer retries:

```javascript
const producer = kafka.producer({
  retry: {
    retries: 5, // Retry up to 5 times
    initialRetryTime: 100, // Wait 100ms between retries
  },
  idempotent: true,
});
```

Consumer retries: Kafka **no** los tiene nativos (a diferencia de SQS). Hay que construir un **retry topic** propio con un consumer separado, y de ahí escalar a una **Dead Letter Queue**. Este es justamente el motivo por el que, en el caso de estudio de un Web Crawler, se elige SQS en vez de Kafka: retry y DLQ vienen gratis.

![[kafka-09-consumer-retries-dlq.svg|569]]

## Performance

Batching:

```javascript
await producer.send({
  topic: 'my_topic',
  messages: [
    { key: 'key1', value: 'message1' },
    { key: 'key2', value: 'message2' },
    { key: 'key3', value: 'message3' },
  ],
});
```

Compression:

```javascript
const { CompressionTypes } = require('kafkajs')

await producer.send({
  topic: 'my_topic',
  compression: CompressionTypes.GZIP,
  messages: [
    { key: 'key1', value: 'Hello, Kafka!' },
  ],
});
```

Los algoritmos disponibles son GZIP, Snappy y LZ4. Pero el mayor impacto en performance no es batching ni compression — vuelve, otra vez, a la elección de la partition key.

## Retention

Se controla con `retention.ms` / `retention.bytes`, con un default de 7 días. Es un trade-off directo contra costo de storage: retener más tiempo cuesta más disco (o más tiered storage).

### Log compaction

Además de borrar por tiempo/tamaño (`cleanup.policy=delete`), Kafka soporta `cleanup.policy=compact`: en vez de purgar mensajes viejos, retiene solamente el **último valor por key**. Un **tombstone** es una entrada con la key y valor `null`, que se purga recién después de `delete.retention.ms`. Las dos políticas son combinables (`compact,delete`).

La compactación es lo que habilita que Kafka funcione como algo más que una cola: potencia el topic interno `__consumer_offsets`, los pipelines de **CDC**/changelog (Debezium escribe eventos keyed por primary key, así el topic siempre refleja el estado *actual* de la tabla — podés reconstruir una vista materializada simplemente replayando el topic desde el inicio), y las **KTables** de Kafka Streams. En el fondo, Kafka no es solo una cola — es también un **key-value changelog distribuido**. Ver [[Change Data Capture]].

## El problema dual-write y el patrón outbox

Escribir a tu base de datos y publicar a Kafka como dos operaciones separadas (**dual-write**) no es atómico: si el proceso crashea entre medio, o el publish falla, terminás con eventos perdidos o fantasma (publicados sin que el write a DB haya sucedido, o viceversa).

El patrón **outbox** resuelve esto sin necesitar [[Two-Phase Commit]]: escribís el evento a una tabla `outbox` en la **misma transacción de DB** que tu escritura de negocio, y un proceso de **Change Data Capture** (Debezium leyendo el write-ahead log de la DB) streamea las filas de esa tabla outbox a Kafka. Esto te da at-least-once sin 2PC — a cambio, tus consumers tienen que ser [[Idempotency|idempotentes]], porque CDC puede reentregar. Esta es una de las razones por las que log compaction + CDC + outbox terminan conectados: el mismo mecanismo que replica tu tabla a Kafka es el que hace que un topic compactado sea un changelog fiel del estado actual. Se conecta también con [[Saga]] cuando la coordinación cruza varios servicios.

## Ecosistema Kafka

- **Kafka Connect**: framework de conectores source/sink — en vez de escribir código a mano para mover datos entre Kafka y una DB/API externa, usás un conector ya hecho (Debezium para CDC es el ejemplo más común).
- **Kafka Streams** / **ksqlDB**: procesamiento de streams con estado — joins stream-stream y stream-table, windowing, agregaciones continuas. ksqlDB expone todo esto con sintaxis SQL. Cuando el estado crece a terabytes, necesitás CEP, o querés exactly-once end-to-end con windowing rico, el salto es a [[Flink]] (un cluster dedicado, no una librería embebida). Ver [[Stream Processing]].
- **Schema Registry**: centraliza los schemas de los mensajes (**Avro**, **Protobuf**, JSON), con modos de compatibilidad `BACKWARD`/`FORWARD`/`FULL` que garantizan que evolucionar un schema no rompa a los consumers existentes.

## Kafka moderno: KRaft

Kafka dependía históricamente de [[ZooKeeper]] para guardar la metadata del cluster (qué brokers existen, quién es leader de cada partition). **KRaft** (KIP-500) reemplaza eso con un quorum Raft implementado dentro del propio Kafka — Kafka 4.0 eliminó ZooKeeper por completo. El resultado: failover de controller más simple y menos superficie operacional (un sistema distribuido menos que mantener).

## Extras — quotas, lag, geo-replicación

Tres piezas operativas que completan el cuadro: **Quotas** limitan el byte-rate por cliente, clave en un cluster multi-tenant para que un consumer/producer ruidoso no ahogue al resto. **Consumer Lag** (offset más reciente menos offset commiteado) es la métrica de salud número uno de un consumer group, y el disparador típico de autoscaling. **MirrorMaker 2** replica topics entre clusters distintos para geo-replicación y disaster recovery activo-pasivo, traduciendo offsets entre el cluster origen y destino.

## ¿Cuándo usar Kafka (y en qué modo)?

| Situación | Elección |
|---|---|
| Múltiples consumer groups necesitan leer el mismo stream de forma independiente | Kafka en modo **stream** |
| Necesitás orden garantizado por key a throughput de 1M+ msgs/s | **Kafka** |
| Solo necesitás desacoplar productor/consumidor, con DLQ y retry nativos, sin gestión operativa | **SQS** (o RabbitMQ si necesitás routing complejo) |
| Necesitás replay/rebuild de estado desde el historial completo | Kafka en modo **stream**, con **log compaction** si además es un changelog por key |
| Consistencia de DB + Kafka sin dual-write | **Outbox + CDC** (Debezium), nunca escritura directa a ambos |
| Multi-tenant con clientes ruidosos | Kafka + **Quotas** |
| DR / geo-replicación entre regiones | Kafka + **MirrorMaker 2** |
| Volumen bajo, sin necesidad de replay ni multi-consumer | Considerá algo más simple que Kafka — el overhead operacional no se justifica |

## 🎯 Preguntas de retrieval

> [!question] ¿Por qué el orden dentro de una partition lo define el offset y no el timestamp?
> Porque el timestamp lo pone el producer (o el broker) al momento de la escritura y puede no reflejar el orden real de llegada bajo reintentos o clocks desincronizados; el offset es un contador estrictamente creciente que el broker asigna en el orden real de escritura a esa partition.

> [!question] `acks=all` ¿garantiza cero pérdida de datos?
> No por sí solo. Solo espera a la ISR **actual** — si la ISR se redujo al leader solo, la condición se cumple trivialmente. Necesitás `min.insync.replicas` para forzar un mínimo real de réplicas confirmando, o la escritura falla en vez de degradar silenciosamente.

> [!question] Bajo hot partition, ¿por qué "sacar la key" no es gratis?
> Porque perdés el orden garantizado que la key te daba — la mitigación cambia un problema (concentración de carga) por otro (pérdida de orden), y hay que decidir cuál importa más para el caso de uso.

> [!question] ¿Por qué no meter blobs (videos, imágenes grandes) directamente en Kafka?
> Porque Kafka no es un object store: `message.max.bytes` limita el tamaño de mensaje (~1MB default), y forzar blobs grandes rompe el throughput y la eficiencia del broker. El patrón correcto es guardar el blob en S3 y mandar solo el puntero por Kafka.

> [!question] ¿Por qué Kafka no tiene reintentos de consumer nativos como SQS?
> Es una decisión de diseño: Kafka deja la inteligencia de entrega del lado del consumer (broker "tonto", pull-based, basado en offset) en vez de trackear el estado de cada mensaje por consumer como hace un broker "inteligente" tipo SQS/RabbitMQ. El costo es que retry/DLQ hay que construirlos vos mismo con un retry topic.

> [!question] ¿Cuál es la diferencia real entre usar Kafka como message queue y como stream?
> No es la tecnología, es el patrón de consumo: en modo queue tratás cada mensaje como consumido una vez que un consumer lo procesó (aunque Kafka lo retiene igual); en modo stream aprovechás que el log es replayable y dejás que múltiples consumer groups independientes lo lean completo, cada uno a su propio ritmo.

> [!question] Si un consumer muere, ¿qué se cae — el consumer o Kafka?
> Solo el consumer. Kafka detecta la ausencia de heartbeats, dispara un rebalance (cooperativo si activaste el CooperativeStickyAssignor, si no eager/stop-the-world) y reasigna las partitions del consumer caído al resto del grupo; el cluster de brokers sigue funcionando sin interrupción.

> [!question] ¿Por qué elegirías Kafka sobre SQS si SQS te da DLQ gratis?
> Porque necesitás múltiples consumer groups independientes replayando el mismo stream, orden garantizado por partition key, y throughput sostenido de 1M+ mensajes/segundo — cosas que un broker "inteligente" de bajo throughput como SQS no está pensado para dar.

> [!question] ¿Qué es exactly-once realmente, y qué NO cubre?
> Es la combinación de Producer ID + números de secuencia (dedup dentro de una partition) más transacciones (atomicidad multi-partition/topic con `read_committed`). Cubre escrituras atómicas y deduplicadas **dentro de Kafka**. No cubre efectos secundarios externos (una llamada HTTP, un write a otra DB) — eso necesita el patrón outbox.

> [!question] ¿Cómo mantenés tu DB y tu topic de Kafka consistentes sin dual-write?
> Con el patrón outbox: escribís el evento en una tabla outbox dentro de la misma transacción de DB que tu escritura de negocio, y un proceso de CDC (Debezium) streamea esas filas a Kafka de forma asíncrona pero confiable.

> [!question] ¿Por qué Kafka es rápido escribiendo a disco?
> Por tres factores combinados — I/O secuencial (el append lineal es mucho más rápido que random incluso en SSD), la page cache del SO (un consumer al día lee de RAM, no de disco) y zero-copy vía `sendfile()` (los datos van de la page cache al socket de red sin pasar por el heap de la JVM). No es que Kafka viva en memoria — es que evita tocar disco innecesariamente y evita copias de más.

## References

- Enriquecimientos (Kafka vs RabbitMQ/SQS/Kinesis/Pub-Sub, offset commit modes, exactly-once/transacciones, log compaction, ordering/max.in.flight, cooperative rebalancing, min.insync.replicas, zero-copy/page cache, partition count planning, KRaft, tiered storage, outbox/dual-write, ecosistema Connect/Streams/Schema Registry, rack awareness, quotas/lag/MirrorMaker): conocimiento general de preparación de entrevistas de system design, no de un único artículo fuente.

## 🔗 Conexión con el vault

- Volvé al MOC del subtema: [[_Streaming|Streaming]].
- Otras tecnologías deep-dive: [[Flink]] (la pareja natural de Kafka — el pipeline canónico Kafka→Flink→sink), [[ZooKeeper]] (coordinaba la metadata del cluster hasta que KRaft lo reemplazó), [[Cassandra]], [[Redis]], [[Elasticsearch]], [[DynamoDB]].
- Patrones relacionados: [[Message Queue]], [[Pub-Sub]], [[Stream Processing]], [[Event Sourcing]], [[Event-Driven Architecture]], [[Dead Letter Queue]], [[Change Data Capture]], [[Write-Ahead Log]], [[Idempotency]], [[Two-Phase Commit]], [[Saga]], [[CQRS]], [[Primary-Replica]], [[Sharding]].
