---
title: Gradient-Based Methods
source: https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually
author: togoAI Labs
created: 2026-06-10
tags:
  - ai/gnn
  - type/technology
  - status/permanent
aliases:
  - Gradient-Based Methods
  - Gradient-Based Explanations
  - Métodos basados en gradientes
---

# Gradient-Based Methods

> [!note] Definición
> Métodos de [[GNN Interpretability]] **tomados prestados del toolkit general de interpretabilidad de deep learning**. La idea: computar **gradientes del output respecto al input** y usarlos como scores de importancia.

## La idea

- Si cambiar levemente una feature de entrada cambiaría significativamente el output, esa feature debe ser **importante**. El gradiente mide exactamente esa **sensibilidad**.
- Para un [[Graph Neural Network]], se pueden computar gradientes respecto a **features de nodo**, **pesos de edges** o **node embeddings**. Nodos o features con **gradientes de gran magnitud** se consideran importantes.

## Variantes comunes

- **Vanilla gradients** — computar el gradiente y usar su magnitud como importancia, sin más.
- **Integrated Gradients** — acumular gradientes a lo largo de un **camino desde un input baseline hasta el input real**. Tiende a dar atribuciones **más estables y teóricamente fundamentadas**.
- **Grad-CAM adaptations** — originalmente diseñados para CNNs, adaptados a GNNs para resaltar **regiones importantes del grafo**.

## Trade-offs

> [!tip] Strengths
> **Rápidos de computar**, teóricamente fundamentados, y funcionan con **cualquier GNN diferenciable**.

> [!warning] Weaknesses
> Pueden ser **ruidosos** y sensibles a cambios chicos del input. Pueden resaltar features que están **correlacionadas** con las importantes, en vez de las verdaderamente **causales**.

## References

- Fuente: [What Is My Graph Neural Network Actually Looking At?](https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually) — togoAI Labs, 2026-01-27

## Related

- [[GNN Interpretability]]
- [[Integrated Gradients]]
- [[Graph Neural Network]]
