---
title: Change Data Capture
source: https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design
author: Avani Chaskar
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/data-pipeline
  - type/concept
  - status/permanent
aliases:
  - Change Data Capture
  - change-data-capture
  - CDC
---

# Change Data Capture

> [!note] Definition
> Patrón de ingesta que reacciona a **cambios** en una fuente de datos (creación,
> edición, borrado) en lugar de re-escanearla entera periódicamente. En sistemas
> modernos suele implementarse con webhooks que emiten eventos hacia un bus de
> mensajería.

## Por qué importa en ingesta

Un asistente que indexa Notion/Slack/Jira no puede re-leer todo cada vez: es
lento, caro y choca contra los *rate limits* de cada API. CDC capta solo el
delta: cuando un usuario edita una página, un **webhook** dispara un evento.

> [!tip]
> Meter esos eventos en un bus como **Kafka** desacopla los workers de ingesta de
> las APIs de origen: si llega una ráfaga de cambios o una API está lenta, los
> eventos se encolan y se procesan a su ritmo, sin perder actualizaciones ni
> tumbar la fuente. Es lo que da el requisito de "actualización casi en tiempo
> real" sin sacrificar estabilidad.

## Como patrón general de system design

Más allá de RAG, CDC es un patrón de **procesamiento de datos**: capturar los
cambios de una base (insert/update/delete) como un stream de eventos al que otros
sistemas se suscriben y reaccionan. Se implementa leyendo el log de la base (p.
ej. el WAL de Postgres → Debezium) o vía [[Webhooks]].

> [!warning] Cuándo NO usarlo / trade-offs
> - **Acoplamiento al log interno de la base**: leer el WAL ata tu pipeline a
>   detalles internos del motor; un cambio de versión puede romperlo.
> - **Orden y duplicados**: los eventos pueden llegar desordenados o repetidos →
>   los consumidores deben ser [[Idempotency|idempotentes]].
> - **Carga sobre la fuente**: mal configurado, el capture agrega presión a la
>   base de origen.
> - Para necesidades simples de sincronización periódica, un batch/polling es más
>   simple que montar CDC.

## References

- Fuente: [System Design Deep Dive: How to Design an Enterprise RAG Assistant](https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design) — Avani Chaskar, 2026-05-17
- También catalogado en: [50 System Design Patterns](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[Enterprise RAG Assistant]]
- [[_System Design|System Design]]
- [[Stream Processing]]
- [[Webhooks]]
