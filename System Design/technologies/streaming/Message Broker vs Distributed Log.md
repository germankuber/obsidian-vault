---
title: Message Broker vs Distributed Log
source: https://designgurus.substack.com/p/redis-streams-vs-kafka-vs-rabbitmq
author: Arslan Ahmad
created: 2026-07-05
tags:
  - system-design/streaming
  - type/case-study
  - status/permanent
aliases:
  - Kafka vs RabbitMQ vs Redis Streams
  - RabbitMQ vs Kafka vs Redis Streams
  - Broker vs Log
provenance:
  source: from-source
  enriched_sections: []
reading:
  total_words: 1929
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-07-05
---

# Message Broker vs Distributed Log

> [!note] Tesis operativa
> Elegir entre Kafka, RabbitMQ y Redis Streams no arranca por features ni por benchmarks: arranca por una sola distinción de modelo. *That single distinction, broker versus log, is the axis the entire comparison turns on.* Un **broker de mensajes** rutea y descarta; un **log distribuido** registra y retiene. Todo lo demás — throughput, orden, replay — cae de qué lado de ese eje estás.

## Marco mental (leé esto primero)

Antes de comparar herramientas hay que tener claras dos categorías, porque la herramienta correcta es *consecuencia* de en cuál caés, no un punto de partida.

- **Message broker (router)**: recibe un mensaje del productor y lo *entrega* a un consumidor siguiendo reglas de ruteo. Una vez entregado y confirmado, el mensaje **desaparece**. El broker no guarda historia — su trabajo es que el mensaje llegue a quien tiene que llegar y después olvidarlo. Ver [[Message Queue]] y [[Pub-Sub]] como los patrones que un broker implementa.
- **Distributed log (recorder)**: guarda cada evento en un registro *append-only*, ordenado y **retenido**. Los consumidores leen desde una posición y el evento sigue ahí para que otro lo lea después, o el mismo lo relea. Es la base del [[Event Sourcing]] y de la [[Event-Driven Architecture]].

La pregunta que decide todo: **¿necesitás rutear una tarea a un worker, o necesitás registrar un stream de eventos que se pueda releer?** Esa es la bifurcación broker-vs-log.

## La cadena causal (por qué cada herramienta existe)

1. **RabbitMQ** nace del lado *broker*: resuelve cómo entregar tareas discretas a workers con ruteo flexible y confirmación por mensaje. *Pero* al entregar-y-descartar, no puede releer ni retener → si tu problema es un stream reproducible, RabbitMQ es la herramienta equivocada.
2. **Kafka** ataca justo eso desde el lado *log*: registra a escala masiva y deja releer. *Pero* ese poder cuesta peso operativo → no es gratis montar y operar Kafka.
3. **Redis Streams** ataca el costo de Kafka: te da el *modelo* de log (append-only, ordenado, retenido, consumer groups) sin montar una plataforma aparte. *Pero* Redis es in-memory → la retención está acotada por RAM de un modo que el log en disco de Kafka no sufre.

Cada paso resuelve la limitación del anterior y paga un precio nuevo. Por eso, como cierra el artículo: **each tool fails when pushed toward another's model.**

## RabbitMQ — el broker

> [!note] Definición
> Un **message broker** que rutea mensajes del productor al consumidor mediante *exchanges* y reglas de ruteo. Una vez que el mensaje se entrega y se confirma (*ack*), se elimina. **It is the router, not the recorder.**

**Cómo funciona** — el productor publica a un *exchange*, que según reglas de ruteo decide a qué cola va; el consumidor lee de la cola y, al terminar, confirma: *the consumer confirms it has successfully processed the message so the broker knows it can safely delete it*. Ese *ack* por mensaje es el corazón del modelo — habilita **redelivery** (si el consumidor falla sin confirmar, el mensaje se reentrega) y ruteo flexible por reglas.

> [!tip] Cuándo usarlo
> Cuando tu problema es **rutear tareas discretas a workers**: job queues, procesamiento de comandos, tareas puntuales. Lo que hace fuerte a RabbitMQ es exactamente el trío *ack por mensaje + redelivery + ruteo flexible*.

> [!warning] Cuándo NO usarlo / trade-offs
> No hay **replay**, no hay **retención**, no hay muchos consumidores independientes releyendo lo mismo — el mensaje se va cuando se confirma. El throughput queda por debajo de Kafka para volúmenes masivos, y bajo sobrecarga las **colas profundas degradan** el sistema. La línea que zanja la elección: *if you find yourself wanting replay from RabbitMQ, you have chosen the wrong tool.*

## Kafka — el log distribuido

> [!note] Definición
> Un **distributed log**: un tópico se divide en **particiones**, y eso es lo que le da a la vez escala y orden. **It is the recorder built for scale.**

**Cómo funciona** — el throughput escala **agregando particiones**: más particiones, más paralelismo. Cada consumidor lleva su propio **offset** (la posición hasta donde leyó), y de ahí salen las dos capacidades clave del log:

- **Replay**: reseteás el offset hacia atrás y volvés a leer eventos ya consumidos. El evento sigue en el log, no se borró al leerlo.
- **Consumer groups independientes**: varios grupos leen el mismo tópico sin interferirse — cada uno con su propio offset. Un pipeline de analytics y uno de facturación consumen los mismos eventos sin pisarse.

> [!warning] El orden es por partición, no global
> *Ordering in Kafka is guaranteed within a partition, not across the whole topic.* El orden se logra eligiendo una **clave** (los mensajes con la misma clave caen en la misma partición y quedan ordenados entre sí). Creer que Kafka ordena el tópico entero es una **trampa clásica de entrevista** — no lo hace, y no puede hacerlo sin renunciar al paralelismo.

> [!warning] Cuándo NO usarlo / trade-offs
> El costo de Kafka es **peso operativo**: montarlo y operarlo no es trivial. Elegir Kafka por defecto para todo — cuando el problema no pide ni volumen masivo ni replay — es un error caro.

## Redis Streams — el log dentro de Redis

> [!note] Definición
> Un **log dentro de Redis**: append-only, ordenado, retenido, con consumer groups — el modelo del log **sin montar una plataforma aparte**. *It is the log for teams that want the model without the machinery.*

**Cómo funciona** — trae las mismas piezas conceptuales que Kafka (append-only, orden, retención, consumer groups) pero embebidas en [[Redis]], que el equipo quizás ya tiene corriendo. De ahí su ventaja: **simplicidad y velocidad** para equipos ya sobre Redis, en volúmenes **moderados**.

> [!tip] Cuándo usarlo
> Cuando querés el modelo de log a **escala moderada** y ya tenés Redis en tu stack — evitás sumar y operar una plataforma dedicada solo para esto.

> [!warning] Cuándo NO usarlo / trade-offs
> El límite es la **memoria**: *Redis is primarily an in-memory system, so retaining a very large volume of events is constrained by memory in a way Kafka's disk-based log is not.* A gran volumen, la retención te choca contra la RAM — y ahí la migración natural es hacia Kafka, cuyo log en disco no tiene ese techo.

## Comparando los tres

![[broker-vs-log-comparison.jpg]]
*Kafka, RabbitMQ, or Redis Streams? A Practical Guide to Choosing — comparison table*

- **Throughput** → para volumen **masivo**, Kafka. Escala sumando particiones; los otros dos quedan por debajo en ese régimen.
- **Orden** → nadie te lo da global y gratis. Kafka ordena **por partición**, RabbitMQ **por cola**, Redis **por stream**. Ejemplo canónico: usar el **user-ID como clave** para que todos los eventos de un usuario caigan ordenados en la misma partición. El trade-off de fondo: **alcance del orden vs. paralelismo** — cuanto más amplio querés el orden, menos podés paralelizar.
- **Retención / replay** → la línea más limpia de toda la comparación, y la que **descarta a RabbitMQ** de una: si necesitás retener y releer, necesitás un log (Kafka o Redis Streams), no un broker.

## Cómo elegir (marco de decisión)

La regla que resume todo: *RabbitMQ when you route, Kafka when you record at scale, Redis Streams when you want to record at moderate scale without the overhead.*

```text
¿Rutear tareas a workers?
  → sí → RabbitMQ (broker)
  → no → necesitás un stream retenido / reproducible → es un LOG:
         ├─ volumen alto / stream central de la organización → Kafka
         └─ volumen moderado + ya usás Redis → Redis Streams
```

> [!tip]
> No son mutuamente excluyentes: un sistema real puede usar RabbitMQ para tareas y Kafka para su stream de eventos a la vez. La pregunta se responde *por problema*, no eligiendo un ganador único.

## Dónde se rompe cada uno

> [!warning]
> - **RabbitMQ** se rompe cuando le pedís lo que es de un log: *if you find yourself wanting replay from RabbitMQ, you have chosen the wrong tool.*
> - **Kafka** se rompe como *default*: elegirlo para todo por costumbre es el error caro — pagás peso operativo sin necesitarlo.
> - **Redis Streams** se rompe por **memoria**: a gran volumen la retención te fuerza a migrar a un log en disco.
>
> El patrón meta: **each tool fails when pushed toward another's model.**

## En la entrevista

El approach ganador es **model-first**: primero nombrás la categoría (broker o log), y recién después justificás la herramienta por el **problema + la escala** — no al revés. *The tool is a consequence of the problem, not a starting point.*

Señales que un buen candidato menciona sin que se las pidan:
- El orden en Kafka es **por partición**, no global.
- RabbitMQ **no tiene replay** — si el enunciado pide releer, RabbitMQ está descartado.
- Redis Streams es **memory-bound** — a gran volumen la retención no escala.

> [!question] 🎯 Te dan un sistema que debe reprocesar los últimos 7 días de eventos ante un bug, y varios equipos consumen el mismo stream. ¿Broker o log? ¿Cuál de los tres, y por qué NO los otros?
> Es un **log**, porque "reprocesar" = replay y "varios equipos el mismo stream" = consumer groups independientes — dos cosas que un broker como RabbitMQ no da (se descarta de una). Entre los logs: si el volumen es alto o es un stream central, **Kafka**; si es moderado y ya hay Redis, **Redis Streams**. La clave de partición (p. ej. user-ID) da el orden que necesites sin renunciar al paralelismo.

## Puntos clave

> [!summary] Puntos clave
> Los 7 takeaways del artículo, verbatim (en inglés, tal como los resume la fuente):
>
> 1. "The systems have different models, with RabbitMQ a broker that routes and deletes, and Kafka and Redis Streams logs that retain and replay, which is the axis the whole comparison turns on."
> 2. "RabbitMQ is the router, best at delivering discrete tasks to workers with sophisticated routing and per-message acknowledgment, but not built for retention or replay."
> 3. "Kafka is the log at scale, best for high-throughput, durable, replayable event streaming with many independent consumers, at the cost of significant operational weight."
> 4. "Redis Streams is the log without the weight, giving the log model with low complexity for teams already using Redis, but bounded by memory at high volume and long retention."
> 5. "Ordering is per partition, queue, or stream, never free and global at scale, and you preserve order for related events by keeping them together, trading ordering scope against parallelism."
> 6. "Choose by problem then scale, RabbitMQ to route tasks, Kafka to record at high scale, Redis Streams to record at moderate scale, and combine them where workloads differ."
> 7. "Each breaks when pushed toward another's model, so the model-first framing is what prevents the common mistakes of forcing replay from RabbitMQ or reaching for Kafka before the scale justifies it."

## Cierre

> [!note]
> *They are answers to different questions. RabbitMQ answers how to route tasks to workers, Kafka answers how to record a massive stream of events for many consumers, and Redis Streams answers how to record a moderate stream without the operational weight of a dedicated platform.*

## References

- Source: [Kafka, RabbitMQ, or Redis Streams? A Practical Guide to Choosing](https://designgurus.substack.com/p/redis-streams-vs-kafka-vs-rabbitmq) — Arslan Ahmad, 2026-07-04. (Substack de pago / paywalled — el contenido completo puede requerir suscripción.)

## Related

- [[Kafka]]
- [[Redis]]
- [[Message Queue]]
- [[Pub-Sub]]
- [[Stream Processing]]
- [[Event Sourcing]]
- [[Event-Driven Architecture]]
