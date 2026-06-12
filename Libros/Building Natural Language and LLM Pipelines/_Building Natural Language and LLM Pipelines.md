---
title: Building Natural Language and LLM Pipelines — Mapa del libro
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/moc
  - status/permanent
aliases:
  - Building Natural Language and LLM Pipelines
  - Building NL & LLM Pipelines MOC
---

# Building Natural Language and LLM Pipelines — Mapa del libro

> [!info] Packt · ISBN 9781835467992
> Mapa de lectura de *Building Natural Language and LLM Pipelines*. Abrí esta nota para la tesis del libro, el índice de capítulos y las ideas que cruzan toda la obra. Empezá por la tesis y bajá.

## 🎯 Tesis del libro

El libro argumenta que el camino a aplicaciones agénticas confiables y production-grade NO pasa por el prompt engineering solo, sino por **la aplicación rigurosa y sistemática del procesamiento clásico de data pipelines**: las prácticas data-centric del pasado son el fundamento indispensable de los sistemas agénticos del futuro. Frente a la **agentic reliability crisis** de 2026 (un agente es solo tan bueno como los datos y tools que recibe), la respuesta es una **arquitectura de dos capas** — *tool layer* (pipelines deterministas con **Haystack 2.0** + **Hayhooks**) y *orchestration layer* (razonamiento stateful con **LangGraph 1.0**) — desplegada, evaluada y observada con rigor MLOps/AgentOps.

La conclusión que el libro demuestra empíricamente (Epílogo): *"reliability is not a property of the model, but of the context in which the model operates"*. Tratar al LLM como un componente no fiable (un **Stochastic Processing Unit**) envuelto en código determinista, guardrails y retry policies es aplicar **SRE a la IA**. Sobre ese fundamento se construye el stack soberano (sovereign agents) interoperable vía protocolos estándar (MCP, A2A) y automejorante (ACE).

## 📖 Capítulos

| # | Capítulo | En una línea |
|---|---|---|
| 01 | [[01 - Introduction to Natural Language Processing Pipelines]] | La agentic reliability crisis y la tesis: data pipelines como fundamento de agentes confiables |
| 02 | [[02 - Diving Deep into Large Language Models]] | Del LLM monolítico (2023) al stack agéntico (SLMs, RLMs, context engineering); LangGraph + Haystack |
| 03 | [[03 - Introduction to Haystack by deepset]] | Haystack 2.0: components, pipelines, SuperComponents, tools, agents sobre un directed graph explícito |
| 04 | [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]] | Pipelines reales: indexación multi-fuente, naive/hybrid RAG, multimodal, async |
| 05 | [[05 - Haystack Pipeline Development with Custom Components]] | Custom components, warm_up(), y generar un dataset ground-truth con knowledge graph + Ragas |
| 06 | [[06 - Building Reproducible and Production-Ready RAG Systems]] | Evaluación cuantitativa (Ragas), observabilidad y FinOps (W&B), trade-off de embeddings |
| 07 | [[07 - Deploying Haystack-Based Applications]] | Deploy productivo: FastAPI (control) vs Hayhooks (velocity), Docker, CI/CD, MCP server |
| 08 | [[08 - Hands-On Projects]] | Tool vs orchestration: NER/sentiment/classification como tools, LangGraph, Yelp Navigator |
| 09 | [[09 - Future Trends and Beyond]] | Protocolos (MCP, A2A), memoria evolutiva (ACE) y el modelo de amenazas AgentSecOps |
| 10 | [[10 - Epilogue - The Architecture of Agentic AI]] | Yelp Navigator V1/V2/V3: la fiabilidad es de la arquitectura, no del modelo (SRE for AI) |

## 🔗 Ideas transversales

Conceptos que cruzan varios capítulos:

- **Tool layer vs orchestration layer** — el patrón arquitectónico central del libro: introducido en [[01 - Introduction to Natural Language Processing Pipelines]], refinado en [[08 - Hands-On Projects]] y demostrado cuantitativamente en [[10 - Epilogue - The Architecture of Agentic AI]]. Haystack construye tools, LangGraph orquesta.
- **Agentic reliability crisis** — la tesis fundante ([[01 - Introduction to Natural Language Processing Pipelines]]), resuelta a lo largo del libro y cerrada en el Epílogo con "SRE for AI".
- **[[RAG]]** — naive y hybrid (sparse BM25 + dense + reranking): construido en [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]], evaluado en [[06 - Building Reproducible and Production-Ready RAG Systems]], desplegado en [[07 - Deploying Haystack-Based Applications]], usado como tool en [[08 - Hands-On Projects]].
- **Regla del vector space** — el mismo embedding model en indexing y querying: [[01 - Introduction to Natural Language Processing Pipelines]], [[02 - Diving Deep into Large Language Models]] y [[06 - Building Reproducible and Production-Ready RAG Systems]] ("map of Paris to navigate Tokyo").
- **[[Context Engineering]]** — write/select/compress/isolate: introducido en [[02 - Diving Deep into Large Language Models]], automatizado como ACE en [[09 - Future Trends and Beyond]], mapeado a V3 en [[10 - Epilogue - The Architecture of Agentic AI]].
- **[[Hayhooks]]** — exponer pipelines como microservicios REST/MCP: [[03 - Introduction to Haystack by deepset]], central en [[07 - Deploying Haystack-Based Applications]] y [[08 - Hands-On Projects]].
- **Evaluación cuantitativa ([[Ragas]] + [[Weights and Biases]])** — generar ground-truth ([[05 - Haystack Pipeline Development with Custom Components]]) y medir faithfulness/cost ([[06 - Building Reproducible and Production-Ready RAG Systems]]).
- **Costo: inference > training / TCO / FinOps** — [[02 - Diving Deep into Large Language Models]], [[06 - Building Reproducible and Production-Ready RAG Systems]] y el sovereign stack del [[10 - Epilogue - The Architecture of Agentic AI]].
- **Interoperabilidad ([[Model Context Protocol (MCP)]], [[A2A]])** — Haystack como MCP server: [[07 - Deploying Haystack-Based Applications]] y [[09 - Future Trends and Beyond]].
- **Modelos open-weight ([[DeepSeek-R1]], [[GRPO]], Qwen3, GPT-OSS)** — [[02 - Diving Deep into Large Language Models]], [[09 - Future Trends and Beyond]] y stress-test del Epílogo.

## 🧩 Síntesis

El libro se lee como un **viaje de ingeniería en cuatro actos**. (1) *Fundamento conceptual* (Caps 1-2): por qué los data pipelines clásicos son la base de los agentes confiables, y el panorama 2025 de LLMs/SLMs/RLMs que motiva la arquitectura híbrida LangGraph+Haystack. (2) *Construcción de la tool layer* (Caps 3-6): dominar Haystack 2.0, construir RAG naive y hybrid, extenderlo con custom components, generar datos ground-truth y probar cuantitativamente reliability y costo. (3) *Producción y orquestación* (Caps 7-8): desplegar el pipeline como microservicio (FastAPI/Hayhooks) y graduarlo a tool de un orquestador agéntico LangGraph (NER, sentiment, classification, Yelp Navigator). (4) *Futuro y demostración* (Cap 9 + Epílogo): los protocolos (MCP/A2A), la memoria evolutiva (ACE) y la seguridad agéntica (AgentSecOps) que estandarizan el ecosistema, y finalmente la prueba empírica — vía tres versiones del Yelp Navigator — de que **la fiabilidad es propiedad de la arquitectura, no del modelo**.

El hilo conductor es uno solo: el LLM es un componente probabilístico poderoso pero no fiable; la confiabilidad production-grade emerge de **rodearlo de ingeniería determinista** — pipelines tipados, evaluación cuantitativa, deployment escalable, guardrails, retries y persistencia. Es la aplicación de las disciplinas maduras (data engineering, MLOps, SRE) al nuevo dominio agéntico.

## 🌱 Conceptos para enlazar / escribir

- [[Haystack 2.0]] · [[Hayhooks]] · [[SuperComponent]] · [[Custom Components (Haystack)]] — el framework de la tool layer.
- [[LangGraph]] · [[State machine]] · [[Supervisor-worker pattern]] · [[Circuit breaker]] — la orchestration layer.
- [[RAG]] · [[Hybrid Search]] · [[BM25]] · [[Reciprocal Rank Fusion (RRF)]] · [[cross-encoder]] · [[Embeddings]] — retrieval.
- [[Context Engineering]] · [[Agentic Context Engineering (ACE)]] — la disciplina núcleo.
- [[Model Context Protocol (MCP)]] · [[A2A]] · [[AgentSecOps]] — interoperabilidad y seguridad.
- [[Ragas]] · [[Weights and Biases]] · [[FinOps]] · [[Knowledge graph]] · [[Graph RAG]] — evaluación y datos.
- [[DeepSeek-R1]] · [[GRPO]] · [[Transformer]] · [[TCO (Total Cost of Ownership)]] · [[Sovereign agent]] — modelos y economía.
- [[Named-entity recognition (NER)]] · [[Sentiment analysis]] · [[Zero-shot classification]] — tools NLP clásicas.
- [[Site Reliability Engineering (SRE)]] — el lente del Epílogo aplicado a la IA.

## 🔍 Todos los capítulos (auto)

```dataview
TABLE autor AS "Autor", capitulo AS "Cap."
FROM "Libros/Building Natural Language and LLM Pipelines"
WHERE capitulo
SORT capitulo ASC
```
