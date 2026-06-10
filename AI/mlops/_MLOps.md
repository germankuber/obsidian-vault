---
title: MLOps — Mapa del tema
created: 2026-06-10
tags:
  - ai/mlops
  - moc
aliases:
  - MLOps
  - MLOps MOC
  - AutoMLOps MOC
---

# MLOps — Mapa del tema

> [!note] Cómo usar esta nota
> Índice (MOC) de la carpeta `ai/mlops`. Empezá por arriba y bajá: de cómo madura un sistema ML hasta la experimentación agéntica. Abrí esta nota, no la carpeta.

## 🚀 Empezá por acá

- [[AutoMLOps]] — el concepto central: una capa sobre MLOps donde un agente corre experimentos **desatendido pero dentro de contratos humanos**. *"Harness engineering for MLOps."* Todo lo de abajo son piezas de esta idea.

## 🪜 El camino

- [[MLOps Maturity Stages]] — las 3 etapas: Notebook (no reproducible) → MLOps maduro (pipeline, tracking, registry, monitoring) → AutoMLOps.
- [[AutoResearch]] — el setup de Karpathy que lo inspira: 3 archivos, el **ratchet** (el codebase solo avanza), el contrato. Resultados Red Hat / Shopify.

## 📊 Evaluación — el corazón del problema

- [[Offline vs Business Metrics]] — métricas ML (nDCG/AUC/…) vs de negocio (conversion/revenue/…); por qué se desacoplan; la métrica compuesta.
- [[Three-Tier Evaluation Pipeline]] — tier 1 (agente, scalar barato) → tier 2 (human-gated business proxy) → tier 3 (A/B test). La pieza más importante.

## 🔗 Conexión con otros dominios

- El **agent sandbox** y los guardrails de AutoMLOps son los conceptos de [[Sandboxing]] y [[Agent Harness]] (carpeta `ai/agents/`) aplicados a MLOps.
- El **ratchet** (keep-if-improves) es pariente del [[Generator-Evaluator Pattern]].
- NDCG/MRR como métricas ML → ver [[Reranking Metrics]] (`ai/rag/reranking/`).

## 🌱 Por escribir (semillas del grafo)

Conceptos mencionados al pasar, candidatos a promover cuando un próximo artículo aporte material:

- [[Experiment Tracking]] (MLflow, W&B) · [[Model Registry]] · [[Drift Monitoring]]
- [[A/B Testing]] · [[Off-Policy Evaluation]] (IPS, doubly-robust) · [[Overfitting]]
- [[DCNv2]] · [[Two-Tower Model]] — arquitecturas que el agente puede implementar.

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "AI/mlops"
WHERE file.name != this.file.name
SORT file.name ASC
```
