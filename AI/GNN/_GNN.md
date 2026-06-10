---
title: GNN — Mapa del tema
source: https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually
author: togoAI Labs
created: 2026-06-10
tags:
  - ai/gnn
  - type/moc
  - status/permanent
aliases:
  - GNN MOC
  - GNN Index
  - Graph Neural Networks MOC
updated: 2026-06-10
---

# GNN — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del sub-dominio *Graph Neural Networks* dentro de **IA**: modelos que aprenden de datos en grafo, y cómo interpretar sus predicciones. Empezá por [[Graph Neural Network]] y bajá. Abrí esta nota, no la carpeta.

## 🧠 Fundamentos

- [[Graph Neural Network]] — qué es un GNN, el **message passing** (4 pasos), y por qué su predicción depende de features **y** estructura. El punto de partida.

## 🔍 Interpretabilidad

- [[GNN Interpretability]] — el problema de entender **por qué** un GNN predice lo que predice. Por qué importa (trust, debugging, scientific discovery, fairness, GDPR), qué lo hace difícil, aplicaciones reales y limitaciones. El hub de los métodos.

## 🛠️ Los 5 métodos de explicación

- [[GNNExplainer]] — subgrafo + features mínimos vía mask optimizada por-predicción (2019). El más conocido.
- [[PGExplainer]] — red entrenada que genera explicaciones al instante; resuelve la lentitud de GNNExplainer.
- [[Attention-Based Explanations]] — los attention weights de un GAT como explicación (gratis, pero fidelidad en debate).
- [[Gradient-Based Methods]] — gradientes del output respecto al input (vanilla, Integrated Gradients, Grad-CAM).
- [[Concept-Based Explanations]] — explicar por conceptos del dominio (Concept Whitening).

## 🔗 Conexión con el resto del grafo

- Sub-dominio de IA, hermano de [[_RAG|RAG]], [[_AI Agents|AI Agents]], [[_MLOps|MLOps]] y [[_AI Fundamentals|AI Fundamentals]].
- Cruza con interpretabilidad/explainability de ML en general (los métodos basados en gradientes vienen del toolkit común de deep learning).

## 🌱 Por escribir (semillas del grafo)

- [[Graph Attention Network]] — la arquitectura GAT con atención, base de las explicaciones por atención. Candidata fuerte a promover.
- [[Message Passing]] — el mecanismo central de los GNN; promover si un futuro artículo lo profundiza.
- [[Over-Smoothing]] — el problema de representaciones que convergen en GNNs profundos.
- [[Integrated Gradients]] · [[Grad-CAM]] — técnicas de atribución mencionadas, enlazables desde [[Gradient-Based Methods]].

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "AI/GNN"
WHERE file.name != this.file.name
SORT file.name ASC
```
