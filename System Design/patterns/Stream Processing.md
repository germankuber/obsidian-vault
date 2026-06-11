---
title: Stream Processing
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/data-processing
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - Stream Processing
  - stream-processing
updated: 2026-06-11
---

# Stream Processing

> [!note] Definition
> Procesar los datos **evento por evento, a medida que llegan**, con latencia sub-segundo. En vez de juntar todo y procesar en lote, se procesa el flujo continuo. Herramientas: Kafka Streams, Apache Flink, Spark Streaming.

## Cómo funciona

Un pipeline consume de un stream ([[Pub-Sub|Pub/Sub]]/Kafka), aplica transformaciones, agregaciones por ventana (últimos 5 min) y joins sobre el flujo, y emite resultados continuamente. Maneja estado (contadores, ventanas) y garantías de procesamiento (at-least-once / exactly-once según el motor).

## Cuándo usarlo

> [!tip]
> Cuando la **frescura importa**: detección de fraude, métricas en vivo, alertas, dashboards de tiempo real, feature pipelines de ML online. Todo lo que no puede esperar al batch nocturno.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Más complejo que el batch**: manejar estado, ventanas, *late/out-of-order events*, y exactly-once es difícil.
> - **Reprocesar es complicado**: arreglar un bug y recalcular el histórico no es tan simple como re-correr un batch.
> - **Trade-off de [[Lambda Architecture]]**: el stream da resultados *aproximados* rápido; el batch da *exactos* lento. A veces necesitás ambos.
> - Para agregaciones históricas sin urgencia, [[MapReduce]]/batch es más simple y barato.

## Patrones relacionados / alternativas

- [[MapReduce]] — batch: preciso pero lento.
- [[Lambda Architecture]] — corre batch + stream en paralelo.
- [[Change Data Capture]] — fuente típica de eventos para el stream.
- [[Pub-Sub|Pub/Sub]] — el transporte del flujo.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[MapReduce]]
- [[Lambda Architecture]]
- [[Change Data Capture]]
