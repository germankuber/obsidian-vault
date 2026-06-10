---
title: Graph Neural Network
source: https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually
author: togoAI Labs
created: 2026-06-10
tags:
  - ai/gnn
  - type/concept
  - status/permanent
aliases:
  - Graph Neural Network
  - GNN
  - Graph Neural Networks
  - Red Neuronal de Grafos
---

# Graph Neural Network

> [!note] Definición
> Un **GNN** es un modelo de deep learning que aprende patrones de **datos estructurados como grafos**. Un **grafo** es una colección de **nodos** (cosas) conectados por **edges** (relaciones): redes sociales, moléculas, sistemas de tráfico y bases de conocimiento se pueden representar todos como grafos.

## Message passing — el mecanismo central

Los GNNs aprenden mediante **message passing**. La idea, en 4 pasos:

1. **Cada nodo arranca con features.** Para una molécula, el tipo de átomo; para una red social, info del perfil del usuario.
2. **Los nodos "hablan" con sus vecinos.** Cada nodo recolecta información de los nodos a los que está conectado.
3. **Esa información se combina y se transforma.** El nodo actualiza su propia representación según lo que aprendió de sus vecinos.
4. **Se repite.** El proceso ocurre múltiples veces (múltiples **"capas"**), así la información fluye por todo el grafo.

Tras varias rondas de message passing, cada nodo tiene una representación que captura no solo sus propias features, sino también información sobre su **vecindario** y la **estructura más amplia** del grafo.

## La idea clave (y la raíz de su dificultad)

> [!tip] La predicción depende de features Y estructura
> La predicción de un GNN depende **tanto de las features de los nodos individuales COMO de la estructura del grafo**. Eso es lo que los hace **poderosos** — pero también lo que los hace **difíciles de interpretar** (ver [[GNN Interpretability]]).

## Por qué importa interpretarlos

Como la mayoría de los modelos de deep learning, un GNN funciona como una **caja negra**: entran datos, salen predicciones, y lo del medio es complicado. Cuando alguien pregunta "¿por qué el modelo predijo eso?", uno se queda sin respuesta. Eso es el **problema de interpretabilidad** — el tema de [[GNN Interpretability]].

## References

- Fuente: [What Is My Graph Neural Network Actually Looking At?](https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually) — togoAI Labs, 2026-01-27

## Related

- [[GNN Interpretability]]
- [[Graph Attention Network]]
- [[Message Passing]]
