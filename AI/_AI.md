---
title: AI — Mapa del tema
created: 2026-06-11
tags:
  - type/moc
  - status/permanent
aliases:
  - AI
  - AI MOC
  - Inteligencia Artificial
---

# AI — Mapa del tema

> [!note] Cómo usar esta nota
> Es el índice (MOC) del dominio **IA**. La IA está organizada en **sub-dominios, cada uno con su propia carpeta y MOC** (📁). Empezá por el sub-dominio que te interese y abrí su MOC para el detalle. Abrí esta nota, no la carpeta.

## 🧩 Sub-dominios

- **[[_AI Agents|AI Agents]]** 📁 (`AI/AI Agents/`) — harnesses y agentes: cómo se construye el andamiaje que ejecuta un LLM con herramientas, sandboxing, permisos y orquestación. Incluye casos como [[Grounded Eval Harness]].
- **[[_RAG|RAG]]** 📁 (`AI/RAG/`) — retrieval-augmented generation: ingesta, recuperación, reranking y el diseño de sistema completo. Con los subtemas [[_Chunking|Chunking]] y [[_Reranking|Reranking]].
- **[[_MLOps|MLOps]]** 📁 (`AI/MLOps/`) — operacionalización de ML y la evolución hacia AutoMLOps: madurez, métricas, pipelines de evaluación.
- **[[_GNN|GNN]]** 📁 (`AI/GNN/`) — graph neural networks e interpretabilidad: qué mira realmente una GNN, métodos de explicación.
- **[[_Evals|Evals]]** 📁 (`AI/Evals/`) — evaluación de sistemas LLM: error analysis, open/axial coding, LLM-as-judge, ground truth.
- **[[_AI Fundamentals|AI Fundamentals]]** 📁 (`AI/AI Fundamentals/`) — los conceptos transversales que cruzan todos los sub-dominios.

## 🔗 Conexión con el resto del vault

- Volvé al [[_Home|Home]] para todos los dominios y los dashboards vivos.
- Cruza con **[[_System Design|System Design]]**: los sistemas LLM reusan patrones de comunicación, caching, escalado y resiliencia (ej. [[_Service Mesh|Service Mesh]], [[Message Queue]], [[Circuit Breaker]]).

## 🔍 Todas las notas del dominio IA (auto)

```dataview
LIST
FROM "AI"
WHERE file.name != this.file.name AND !contains(file.tags, "type/moc")
SORT file.path ASC
```
