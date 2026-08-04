---
title: Databases — Mapa del subtema
created: 2026-07-01
tags:
  - system-design/databases
  - type/moc
  - status/permanent
aliases:
  - Databases
  - Bases de Datos
updated: 2026-07-01
---

# Databases — Mapa del subtema

> [!note] Cómo usar esta nota
> Índice (MOC) de **tecnologías de bases de datos concretas** dentro de System Design — productos reales (Cassandra, DynamoDB, Redis…), no patrones abstractos (esos viven en `patterns/`). Cada nota es un deep-dive de una DB: modelo de datos, internals, cuándo usarla en diseño de sistemas.

## 🗄️ Bases de datos

- [[Cassandra]] — NoSQL wide-column distribuida (AP): consistent hashing + vnodes, replicación DC/rack, LSM tree, gossip, hinted handoff / anti-entropy repair, modelado query-driven (casos Discord + Ticketmaster), LWT/Paxos, SASI, TTL. Alternativa: ScyllaDB.
- [[Redis]] — data structure store in-memory, single-threaded: durabilidad RDB/AOF (+ WAIT/MemoryDB), cluster de 16.384 hash slots (Sentinel vs Cluster), y 7 capacidades (cache + trío penetration/breakdown/avalanche, distributed lock/Redlock, leaderboards, rate limiting fixed/sliding/token-bucket, proximity/geohash, event sourcing/streams, pub/sub). Hot key + HyperLogLog/Bitmaps/session store.
- [[Elasticsearch]] — motor de búsqueda distribuido sobre Apache Lucene (no es una DB): mapping keyword/text, inverted index + doc values, segmentos inmutables (near-real-time por refresh 1s, translog), sharding/réplicas, coordinating nodes + query planning. Relevancia [[BM25]], búsqueda vectorial/semántica (kNN/HNSW/quantization) e híbrida ([[Hybrid Search]] + [[Reciprocal Rank Fusion]]), aggregations, geo, ILM. Se conecta vía CDC a un store autoritativo.
- [[DynamoDB]] — key-value NoSQL fully-managed de AWS: partition + sort key, hash partitioning + B-tree (metadata service centralizado, no ring como Cassandra), GSI/LSI, eventual/strong consistency per-request, ACID transactions (TransactWriteItems), Multi-Paxos + quorum 2/3, DAX (cache), DynamoDB Streams (CDC). Enriquecido: single-table design, hot partitions/adaptive capacity/write sharding, Global Tables active-active/LWW, comparación vs [[Cassandra]]/Postgres.

## 🌱 Semillas (DBs referenciadas, sin nota propia todavía)

- **PostgreSQL** — la relacional de referencia; el contraste "empezá con Postgres salvo razón DynamoDB-shaped". Referenciada desde [[DynamoDB]], Pagination y RAG. Candidata a deep-dive.
- **ScyllaDB** — reescritura C++ de Cassandra, CQL-compatible; Discord migró ahí.
- **Apache Lucene** — la librería Java de indexación/búsqueda que está debajo de [[Elasticsearch]] (segmentos inmutables, inverted index, doc values); el "motor" que ES distribuye. Candidata a nota propia.
- **Vector Database** — categoría emergente (dense_vector/kNN/HNSW); [[Elasticsearch]] y Redis compiten ahí, más DBs dedicadas (Pinecone, Weaviate, pgvector). Crosscutting con AI/RAG.
- **Debezium** — herramienta de CDC (sobre [[Kafka]]) para replicar de un store autoritativo a [[Elasticsearch]]; referenciada también desde [[Change Data Capture]].

## 🔗 Conexión con el resto del vault

- Volvé al MOC del dominio: [[_System Design|System Design]].
- Subtema hermano de tecnologías: [[_Streaming|Streaming]] ([[Kafka]]) — event streaming, no bases de datos.
- Las DBs reusan patrones de `patterns/`: [[Consistent Hashing]], [[Quorum]], [[Sharding]], [[Write-Ahead Log]], [[Vector Clocks]].

## 🔍 Todas las notas del subtema (auto)

```dataview
LIST
FROM "System Design/technologies/databases"
WHERE file.name != this.file.name AND !contains(file.tags, "type/moc")
SORT file.name ASC
```
