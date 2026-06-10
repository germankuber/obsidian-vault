---
title: Concept-Based Explanations
source: https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually
author: togoAI Labs
created: 2026-06-10
tags:
  - ai/gnn
  - type/technology
  - status/permanent
aliases:
  - Concept-Based Explanations
  - Concept-Based Explanation
  - Concept Whitening
  - Explicaciones basadas en conceptos
---

# Concept-Based Explanations

> [!note] Definición
> Un enfoque **más nuevo** de [[GNN Interpretability]] que explica las predicciones en términos de **conceptos entendibles por humanos**, en vez de features crudas o estructuras de grafo.

## La idea

- Definir un conjunto de **conceptos significativos en tu dominio**. Para moléculas, cosas como "contiene un anillo de benceno" o "tiene un grupo hidroxilo".
- Luego, analizar **cómo las representaciones internas del GNN se relacionan con esos conceptos**.

## Concept Whitening

- Un método llamado **Concept Whitening** **modifica la arquitectura del GNN** para que ciertas **dimensiones del espacio latente se alineen con conceptos predefinidos**.
- Esto permite afirmaciones como: *"el modelo predijo que esta molécula es tóxica **porque detectó la presencia de un grupo nitro**"*.

## Trade-offs

> [!tip] Strengths
> Produce explicaciones en **términos que los expertos del dominio entienden y les importan**. Puede revelar qué **patrones de alto nivel** aprendió el modelo.

> [!warning] Weaknesses
> Requiere **definir los conceptos por adelantado**, lo que necesita **expertise de dominio**. Los conceptos que definís **podrían no coincidir** con lo que el modelo realmente usa internamente.

## References

- Fuente: [What Is My Graph Neural Network Actually Looking At?](https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually) — togoAI Labs, 2026-01-27

## Related

- [[GNN Interpretability]]
- [[Graph Neural Network]]
