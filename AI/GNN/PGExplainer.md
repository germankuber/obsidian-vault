---
title: PGExplainer
source: https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually
author: togoAI Labs
created: 2026-06-10
tags:
  - ai/gnn
  - type/technology
  - status/permanent
aliases:
  - PGExplainer
  - Parameterized GNNExplainer
---

# PGExplainer

> [!note] Definición
> **PGExplainer** (*Parameterized GNNExplainer*) ataca la principal limitación de [[GNNExplainer]]: la **velocidad**. Es un método de [[GNN Interpretability]].

## Cómo funciona

- En vez de optimizar una **mask separada para cada predicción** (como [[GNNExplainer]]), PGExplainer **entrena una red neuronal** que aprende a **generar explicaciones**.
- Una vez entrenada, esa **explainer network** produce explicaciones para predicciones nuevas **al instante**.
- La red toma los **node embeddings** del GNN y predice **scores de importancia de edges**.
- Se entrena sobre un conjunto de ejemplos para producir explicaciones que sean **faithful** (que realmente expliquen la predicción) y **consistent** (predicciones similares reciben explicaciones similares).

## Trade-offs

> [!tip] Strengths
> Mucho **más rápido** que [[GNNExplainer]] para generar explicaciones **a escala**. Tiende a producir explicaciones **más consistentes** entre inputs similares.

> [!warning] Weaknesses
> Requiere **datos de entrenamiento** y **tiempo de entrenamiento adicional** por adelantado. Puede no capturar tan bien los **matices específicos de cada instancia** como los métodos que optimizan por-predicción.

## References

- Fuente: [What Is My Graph Neural Network Actually Looking At?](https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually) — togoAI Labs, 2026-01-27

## Related

- [[GNNExplainer]]
- [[GNN Interpretability]]
- [[Graph Neural Network]]
