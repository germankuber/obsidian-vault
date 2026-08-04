---
title: Coordination — Mapa del subtema
created: 2026-07-01
tags:
  - system-design/coordination
  - type/moc
  - status/permanent
aliases:
  - Coordination
  - Coordinación Distribuida
updated: 2026-07-01
---

# Coordination — Mapa del subtema

> [!note] Cómo usar esta nota
> Índice (MOC) de **tecnologías de coordinación distribuida** dentro de System Design — servicios de consenso, leader election, locks distribuidos, service discovery y config compartida. Son productos reales (ZooKeeper, y a futuro etcd/Consul), no patrones abstractos (esos viven en `patterns/`: [[Distributed Lock]], [[Quorum]], [[Two-Phase Commit]]). Cada nota es un deep-dive: modelo de datos, protocolo de consenso, garantías, cuándo usarla.

## 🧭 Coordinación

- [[ZooKeeper]] — servicio de coordinación distribuido **CP** (no es un data store): árbol de znodes (persistent/ephemeral/sequential), watches one-shot, sesiones con heartbeat. Consenso vía **ZAB** (atomic broadcast propio, no Paxos/Raft; zxid epoch+contador, quórum 2f+1), ensemble leader/follower/observer, escrituras linearizables / lecturas stale (Ordered Sequential Consistency, `sync()`). Recipes: distributed lock (+ herd effect), leader election, service discovery, config, barriers, queue (usá Curator). Comparación vs etcd/Consul, por qué Kafka migró a KRaft, ZK/etcd vs Redis Redlock (fencing token), FLP impossibility. Casos: Kafka-legacy, HDFS, HBase, Solr.

## 🌱 Semillas (tecnologías referenciadas, sin nota propia todavía)

- **etcd** — store KV distribuido basado en **Raft**, backing store de Kubernetes (API gRPC/HTTP, MVCC/revisions); la alternativa cloud-native moderna a ZooKeeper. Candidata a deep-dive.
- **Consul** — service discovery + health checks + DNS + service mesh multi-DC (HashiCorp), Raft + Gossip; más "batteries included" que ZooKeeper.
- **Raft** — el protocolo de consenso "entendible" que domina los sistemas nuevos (etcd, KRaft, Consul); diseñado para ser más claro que Paxos separando leader election de log replication.
- **Paxos** — el protocolo de consenso fundacional (Lamport); base teórica de la familia, sin líder único fuerte.
- **Consensus** — el problema abstracto (acordar un valor entre nodos con fallos); equivalente a total order broadcast y a un registro linearizable de compare-and-set.
- **Leader Election** — la primitiva de elegir un coordinador único; recipe canónica de ZooKeeper (znodes ephemeral+sequential).
- **KRaft** — el consenso Raft embebido que reemplazó a ZooKeeper en Kafka 4.0.
- **Apache Curator** — la librería (Netflix→Apache) que implementa las recipes de ZooKeeper production-ready.

## 🔗 Conexión con el resto del vault

- Volvé al MOC del dominio: [[_System Design|System Design]].
- Subtemas hermanos de tecnologías: [[_Databases|Databases]] ([[Cassandra]], [[Redis]], [[Elasticsearch]], [[DynamoDB]]) y [[_Streaming|Streaming]] ([[Kafka]], [[Flink]]).
- Los servicios de coordinación reusan patrones de `patterns/`: [[Quorum]], [[Distributed Lock]], [[Two-Phase Commit]], [[Write-Ahead Log]], [[Service Discovery]], [[Primary-Replica]], [[Vector Clocks]].

## 🔍 Todas las notas del subtema (auto)

```dataview
LIST
FROM "System Design/technologies/coordination"
WHERE file.name != this.file.name AND !contains(file.tags, "type/moc")
SORT file.name ASC
```
