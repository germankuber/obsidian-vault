---
title: Evals — Mapa del tema
source: https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7
author: Stephan Beyer
created: 2026-06-10
tags:
  - ai/evals
  - type/moc
  - status/permanent
aliases:
  - Evals MOC
  - Evals Index
  - LLM Evaluation MOC
---

# Evals — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del sub-dominio *Evals* dentro de **IA**: cómo medir sistemáticamente si un sistema de IA/LLM se comporta como se espera. Empezá por [[Evals]] y bajá. Abrí esta nota, no la carpeta.

## 🧭 Fundamentos

- [[Evals]] — qué son y por qué (determinístico vs probabilístico), el enfoque **bottom-up** de Hamel Husain, y el método paso a paso. El punto de partida.

## 🔬 El método bottom-up (error analysis)

- [[Error Analysis]] — leer el trace log a mano y anotar errores; el corazón del enfoque. Lo hace un humano.
- [[Open Coding]] — etiquetar todo lo observado ("¿qué pasa acá?"). Primera etapa.
- [[Axial Coding]] — agrupar las etiquetas en patrones/categorías ("¿cómo se relacionan?"). Segunda etapa → heatmap de errores.

## ⚖️ El LLM judge

- [[LLM as Judge]] — usar un LLM para evaluar cada output automáticamente; preferir binario (TRUE/FALSE), e **iterar el prompt según los desacuerdos** con el ground truth.
- [[Ground Truth]] — las labels humanas contra las que se compara el judge. Define qué es "correcto".

## 🔗 Conexión con el resto del grafo

- Sub-dominio de IA, hermano de [[_RAG|RAG]], [[_AI Agents|AI Agents]], [[_MLOps|MLOps]], [[_GNN|GNN]] y [[_AI Fundamentals|AI Fundamentals]].
- El [[Grounded Eval Harness]] (en AI Agents) es un caso de [[LLM as Judge]] adversario; su forma general es el [[Generator-Evaluator Pattern]].
- Cruza con [[Hallucinations]]/[[Grounding]] (qué se evalúa) y con [[Three-Tier Evaluation Pipeline]]/[[Offline vs Business Metrics]] (evals en MLOps) y [[Reranking Metrics]] (evals de retrieval).

## 🌱 Por escribir (semillas del grafo)

- [[Guardrails]] · [[Code Assertion-Based Evals]] — alternativas/complementos al LLM judge, recomendados explícitamente por el artículo.
- [[Likert Scale]] — la alternativa de scoring que el artículo descarta a favor de binario.
- [[Hamel Husain]] — la fuente de referencia sobre evals (hamel.dev).

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "AI/Evals"
WHERE file.name != this.file.name
SORT file.name ASC
```
