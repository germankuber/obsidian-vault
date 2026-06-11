---
title: Enterprise RAG Assistant
source: https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design
author: Avani Chaskar
created: 2026-06-08
tags:
  - ai/rag
  - system-design/architecture
  - type/case-study
  - status/permanent
aliases:
  - Enterprise RAG Assistant
  - enterprise-rag-assistant
  - RAG Assistant Design
updated: 2026-06-11
---

# Enterprise RAG Assistant

> [!note] Definition
> Un asistente RAG (Retrieval-Augmented Generation) empresarial responde preguntas en lenguaje natural sobre el conocimiento interno de una empresa (Notion, Slack, Jira, Drive…), citando las fuentes y **respetando los permisos de cada usuario**. El reto central no es el LLM: es un problema de sistemas distribuidos — ingesta event-driven, control de acceso y un índice robusto.

## Requisitos que moldean el diseño

**Funcionales:**
- Ingesta multi-fuente (Notion, Slack, Jira, Drive…).
- Control de acceso estricto: un usuario **nunca** debe ver una respuesta derivada de un documento que no tiene permiso de leer. Ver [[ACL Filtering en RAG]].
- Actualización casi en tiempo real ante ediciones/borrados.
- Q&A conversacional con citas inline.

**No funcionales:**
- Baja latencia: *time-to-first-token* (TTFT) < 500 ms.
- Escala: 100k+ DAU, decenas de millones de documentos.
- Disponibilidad: 99.9%.

> [!tip]
> Los requisitos no funcionales (TTFT, escala) son los que justifican casi cada decisión de abajo: streaming en vez de respuesta completa, filtrado en una sola etapa en vez de post-filtrado, e ingesta asíncrona desacoplada.

## Arquitectura de alto nivel: dos pipelines

El sistema se parte en dos caminos independientes:

- **Pipeline de ingesta (asíncrono)** — capta, trocea e indexa los datos. No está en el camino crítico de la respuesta.
- **Pipeline de consulta (síncrono)** — recupera contexto y genera la respuesta. Acá vive el presupuesto de latencia.

## Pipeline de ingesta

1. **Captura de cambios** — webhooks de las fuentes disparan eventos hacia un bus (Kafka), desacoplando los workers de los rate limits de cada API. Patrón: [[Change Data Capture]].
2. **Procesado y troceo** — se generan chunks pequeños para búsqueda y grandes para contexto del LLM. Técnica: [[Hierarchical Chunking]].
3. **Embedding e indexado** — cada chunk se convierte en vector (p. ej. con un modelo `text-embedding-3-large`) y se indexa. Para bases de código complejas, un grafo de propiedades (Neo4j) puede mapear dependencias junto a los vectores — el enfoque [[GraphRAG]].

## Pipeline de consulta

1. **Pre-proceso de la query** — un LLM liviano reescribe la pregunta usando el historial del chat para desambiguarla ([[Query Rewriting]]).
2. **Búsqueda híbrida** — la query reescrita + la identidad/ACL del usuario van a la base vectorial, que combina búsqueda densa (vectores) y dispersa (BM25). Ver [[Hybrid Search]].
3. **Reranking** — un cross-encoder reordena el top-50 y se queda con los ~5 chunks más relevantes. Ver [[Reranking]].
4. **Generación y streaming** — el prompt obliga a "responder SOLO con el contexto provisto y citar las fuentes"; la respuesta se transmite vía [[Server-Sent Events]] para bajar la latencia percibida.

## La regla de oro: control de acceso

El permiso se aplica **en la base de datos, en una sola etapa de filtrado**, no después de recuperar. Es el corazón de la seguridad del sistema y tiene su propia nota: ver [[ACL Filtering en RAG]].

## Cuellos de botella y trade-offs

> [!warning] El problema del "chunk fantasma"
> Cuando se borra un documento, sus chunks deben desaparecer del índice vectorial de inmediato; si no, el sistema cita información que ya no existe. Se mantiene un mapeo relacional `document_id -> [chunk_ids]` en PostgreSQL para borrar en lote. Ver [[Ghost Chunk Problem]].

> [!tip] El documento "All Hands"
> Queries muy frecuentes se sirven desde un caché (Redis) cuya clave combina el hash de la query **y** la firma ACL del usuario — así dos usuarios con permisos distintos no comparten una respuesta indebida. Ver [[Redis Cache]].

## References

- Fuente: [System Design Deep Dive: How to Design an Enterprise RAG Assistant](https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design) — Avani Chaskar, 2026-05-17

## Related

- [[ACL Filtering en RAG]]
- [[Hybrid Search]]
- [[Reranking]]
- [[Hierarchical Chunking]]
- [[Change Data Capture]]
