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
updated: 2026-06-11
---

# Evals — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del sub-dominio *Evals* dentro de **IA**: cómo medir sistemáticamente si un sistema de IA/LLM se comporta como se espera. Empezá por [[Evals]] y bajá, de fundamentos a operación. Abrí esta nota, no la carpeta.
>
> Cubre **dos fuentes**: el artículo introductorio de **Stephan Beyer** (enfoque bottom-up) y la guía completa de **Om Bharatiya** (*AI Evals For Engineers, PMs & QAs*), que aporta validación estadística del judge, evals en código/safety, evals compuestas y la capa de operación/observabilidad.

## 🧭 Fundamentos

- [[Evals]] — qué son y por qué (determinístico vs probabilístico), las **tres verdades centrales**, el enfoque bottom-up de Hamel Husain y el ciclo = método científico. El punto de partida.

## 🔬 Error Analysis (el método bottom-up)

- [[Error Analysis]] — leer el trace log a mano y anotar errores; el paso más importante. Incluye dimensional sampling (375 combos), failure modes, frecuencia × severidad y theoretical saturation.
- [[Open Coding]] — etiquetar todo lo observado ("¿qué pasa acá?"), ~30s/trace freeform. Primera etapa.
- [[Axial Coding]] — agrupar en 4-6 failure modes nombrados y contar frecuencia ("¿cómo se relacionan?"). Segunda etapa → heatmap.

## ⚖️ LLM Judge

- [[LLM as Judge]] — usar un LLM para evaluar a escala; binario, explanation-before-verdict, temperature 0.0, biases. El **7-step workflow**.
- [[Judge Validation]] — split Train/Dev/Test 15/40/45 estratificado, inter-annotator agreement (<80% = criterio no claro), Cohen's Kappa, calidad > cantidad.
- [[Judge Metrics]] — confusion matrix, TPR/TNR/Precision/F1, por qué el agreement engaña. (Métricas de retrieval → [[Reranking Metrics]].)
- [[judgy]] — corrige el bias del judge con TPR/TNR y devuelve success rate + confidence intervals.
- [[Ground Truth]] — las labels humanas contra las que se compara el judge.

## 🛠️ Code & Safety

- [[Code-Based Evaluators]] — checks en código para propiedades objetivas (formato, tool calls, PII, longitud). *Si es `if/else`, código; si es juicio, LLM.* (Alias: *Code Assertion-Based Evals*.)
- [[Guardrails]] — evals de safety **real-time** que bloquean antes de llegar al usuario; latency budget, monitoring & alerting, caching.

## 🧩 Evals compuestas

- [[RAG Evaluation]] — los dos failure modes de RAG (retrieval vs generation), diagnosis, synthetic queries, tokenizer domain-specific.
- [[Pipeline and Multi-Turn Evaluation]] — eval por estado de un pipeline (7 estados, distribución bimodal) y eval de conversaciones multi-turno (context loss, contradiction, drift).

## 🔁 Operación

- [[Eval Lifecycle]] — offline vs online, closing the loop, root-causing, regression testing, model comparison, sampling, tiered evaluation, modelos baratos para judges.
- [[Eval Observability]] — traces y spans, plataformas (Phoenix/Langfuse/Braintrust/LangSmith), prompt versioning, eval frameworks landscape.
- [[Common Eval Mistakes]] — checklist de los 12 antipatrones.

## 🔗 Conexión con el resto del grafo

- Sub-dominio de IA, hermano de [[_RAG|RAG]], [[_AI Agents|AI Agents]], [[_MLOps|MLOps]], [[_GNN|GNN]] y [[_AI Fundamentals|AI Fundamentals]].
- El [[Grounded Eval Harness]] (en AI Agents) es un caso de [[LLM as Judge]] adversario; su forma general es el [[Generator-Evaluator Pattern]].
- Cruza con [[Hallucinations]]/[[Grounding]] (qué se evalúa); con [[Three-Tier Evaluation Pipeline]]/[[Offline vs Business Metrics]] (evals en MLOps — **homónimos a no confundir**, ver [[Eval Lifecycle]]); con [[Reranking Metrics]]/[[BM25]] (evals de retrieval); y con [[AgentOps]] (observabilidad).

## 📅 Plan de aprendizaje (30 días)

- **Semana 1 — Fundaciones** (días 1-7): elegir plataforma, instrumentar tracing, dataset con dimensional sampling, primer experiment, open coding 50 traces, axial coding, matriz frecuencia × severidad.
- **Semana 2 — Code-based evals** (días 8-14).
- **Semana 3 — LLM Judge** (días 15-21): etiquetar 100-150, split, prompt, validar Dev TPR/TNR, iterar >80%, test, correr todo + [[judgy]].
- **Semana 4 — Advanced & Production** (días 22-30): RAG, multi-step, multi-turn, safety, regression suite, inter-annotator calibration, cost optimization, dashboard, documentar + presentar.

## 🌱 Por escribir (semillas del grafo)

- [[Likert Scale]] — la alternativa de scoring que ambas fuentes descartan a favor de binario.
- [[Hamel Husain]] — fuente de referencia sobre evals (hamel.dev).
- Plataformas/frameworks de [[Eval Observability]]: [[Opik]] · [[RAGAS]] · [[Galileo]] · [[METR]] · [[DeepEval]] · [[Braintrust]] · [[LangSmith]].

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "AI/Evals"
WHERE file.name != this.file.name
SORT file.name ASC
```
