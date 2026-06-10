---
title: GNNExplainer
source: https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually
author: togoAI Labs
created: 2026-06-10
tags:
  - ai/gnn
  - type/technology
  - status/permanent
aliases:
  - GNNExplainer
---

# GNNExplainer

> [!note] Definición
> El método **más conocido** para explicar predicciones de [[Graph Neural Network]]. Introducido en **2019**, se volvió la herramienta de referencia del campo. Es un método de [[GNN Interpretability]].

## La idea

Para una predicción dada, GNNExplainer busca el **subgrafo más chico** y las **features de nodo más importantes** que sean **suficientes** para explicar esa predicción.

## Cómo funciona

- Aprende una **"mask"** sobre los **edges y las features** del grafo de entrada. La mask asigna **scores de importancia**: edges y features con score alto son importantes para la predicción; los de score bajo, no.
- Optimiza la mask preguntando: **"Si solo conservo los edges y features de score alto, ¿sigo obteniendo la misma predicción?"**
- Balancea **dos objetivos**: mantener la explicación **chica** (simple) y mantener la predicción **precisa** (que la explicación capture lo que importa).
- **Output**: un subgrafo que resalta qué nodos y edges fueron más influyentes. Para una molécula, podría resaltar un **grupo funcional** específico que la hace tóxica; para una red social, un cluster de usuarios conectados que llevó a cierta recomendación.

## Trade-offs

> [!tip] Strengths
> **Model-agnostic** (funciona con cualquier arquitectura de GNN) y produce explicaciones **visuales e intuitivas**.

> [!warning] Weaknesses
> Puede ser **lento**: necesita optimizar para **cada predicción individual** (ver [[PGExplainer]], que resuelve justamente esto). Las explicaciones también pueden ser **inestables**: cambios chicos en el input pueden llevar a explicaciones muy distintas.

## References

- Fuente: [What Is My Graph Neural Network Actually Looking At?](https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually) — togoAI Labs, 2026-01-27

## Related

- [[GNN Interpretability]]
- [[PGExplainer]]
- [[Graph Neural Network]]
