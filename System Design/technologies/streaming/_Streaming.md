---
title: Streaming — Mapa del subtema
created: 2026-07-01
tags:
  - system-design/streaming
  - type/moc
  - status/permanent
aliases:
  - Streaming
  - Event Streaming
updated: 2026-07-05
---

# Streaming — Mapa del subtema

> [!note] Cómo usar esta nota
> Índice (MOC) de **tecnologías de event streaming / messaging** dentro de System Design — productos reales (Kafka, y a futuro RabbitMQ, Kinesis…), no patrones abstractos (esos viven en `patterns/`: [[Message Queue]], [[Pub-Sub]], [[Stream Processing]], [[Event Sourcing]]). Cada nota es un deep-dive: arquitectura, garantías de entrega, cuándo usarla en diseño de sistemas.

## 🌊 Streaming / Messaging

- [[Kafka]] — plataforma de event streaming distribuida (message queue O stream): brokers/partitions/topics, orden por partition, replicación leader-follower (acks/ISR/min.insync.replicas), consumer groups pull-based, at-least-once vs exactly-once (idempotent producer + transactions), log compaction, distributed commit log. Enriquecido con la comparación vs RabbitMQ/SQS/Kinesis/Pub-Sub, offset commit modes, cooperative rebalancing, zero-copy, KRaft, tiered storage, outbox/dual-write y el ecosistema (Connect/Streams/Schema Registry).
- [[Flink]] — motor de stream processing distribuido y **stateful** (streaming-first, batch = bounded stream): JobManager/TaskManager/task slots, DataStream API/Flink SQL, event time + watermarks (correctitud bajo desorden), windowing (tumbling/sliding/session/global), state backends (HashMap/RocksDB), checkpointing tipo Chandy-Lamport + exactly-once end-to-end (2PC sinks), backpressure credit-based, CEP. La pareja natural de [[Kafka]] (fuente + buffer) en el pipeline canónico Kafka→Flink→sink. Enriquecido con Kappa vs Lambda, comparación vs Spark/Kafka Streams, casos de uso (Ad Click Aggregator, fraud detection, leaderboards) y escala Uber/Alibaba.

## 🌱 Semillas (tecnologías referenciadas, sin nota propia todavía)

- **RabbitMQ** — broker "inteligente" con routing complejo (exchanges/bindings), dead-lettering y TTL nativos; el contrapunto push-based clásico de Kafka. Candidata a próximo deep-dive · ver [[Message Broker vs Distributed Log]] para el eje broker (route-and-delete) vs log (retain-and-replay).
- **AWS SQS** — cola managed AWS-native con redrive policy / DLQ gratis; el "por qué no SQS" que aparece en toda entrevista de Kafka.
- **AWS Kinesis** — alternativa AWS-native a Kafka (shards en vez de partitions), streaming + analytics en tiempo real.
- **Google Pub/Sub** — pub/sub managed de GCP, push o pull, con dead-letter topics.
- **Debezium** — herramienta de CDC sobre Kafka Connect; referenciada también desde [[Change Data Capture]] y el patrón outbox de [[Kafka]].
- **Kafka Streams / ksqlDB** — capa de stream processing con estado (joins, windowing) sobre Kafka, embebida como librería; la alternativa liviana a [[Flink]] cuando no querés parar un cluster dedicado.
- **Schema Registry** — gestión de contratos de mensajes (Avro/Protobuf) con modos de compatibilidad.

## 🔗 Conexión con el resto del vault

- Volvé al MOC del dominio: [[_System Design|System Design]].
- Tecnologías hermanas: [[_Databases|Databases]] ([[Cassandra]], [[Redis]], [[Elasticsearch]], [[DynamoDB]]) — candidatas de sink de un pipeline [[Flink]].
- Patrones que streaming reusa: [[Message Queue]], [[Pub-Sub]], [[Stream Processing]], [[Event Sourcing]], [[Event-Driven Architecture]], [[Dead Letter Queue]], [[Change Data Capture]], [[Write-Ahead Log]], [[Idempotency]], [[Two-Phase Commit]], [[Saga]].

## 🔍 Todas las notas del subtema (auto)

```dataview
LIST
FROM "System Design/technologies/streaming"
WHERE file.name != this.file.name AND !contains(file.tags, "type/moc")
SORT file.name ASC
```
