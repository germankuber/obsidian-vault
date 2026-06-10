---
title: Attention-Based Explanations
source: https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually
author: togoAI Labs
created: 2026-06-10
tags:
  - ai/gnn
  - type/technology
  - status/permanent
aliases:
  - Attention-Based Explanations
  - Attention-Based Explanation
  - Explicaciones basadas en atención
---

# Attention-Based Explanations

> [!note] Definición
> Método de [[GNN Interpretability]] que usa los **pesos de atención** ya presentes en ciertas arquitecturas de [[Graph Neural Network]] — como las **Graph Attention Networks (GAT)** — como explicación. Esos pesos indican cuánto contribuyó cada vecino cuando un nodo agregó información.

## La idea

- En una **GAT**, cuando un nodo actualiza su representación, computa **attention scores** para cada uno de sus vecinos. Un score alto significa que ese vecino tuvo **más influencia**.
- El atractivo: esos pesos **ya se computan durante el forward pass**, así que ¿por qué no usarlos como explicación? Visualizándolos, vemos en qué conexiones el modelo "se enfocó".

## Trade-offs

> [!tip] Strengths
> **Sin cómputo adicional** — las explicaciones vienen **gratis**. Fáciles de implementar y visualizar.

> [!warning] Weaknesses — la fidelidad está en debate
> Hay un debate abierto sobre si los attention weights dan explicaciones **fieles**. Que un modelo **atienda** a algo no significa necesariamente que ese algo fue **importante para la predicción final**: la atención podría capturar algo útil para cómputos intermedios pero no atado directamente al output. **Varios estudios han mostrado que los attention weights pueden ser engañosos**, así que se usan con cautela.

## References

- Fuente: [What Is My Graph Neural Network Actually Looking At?](https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually) — togoAI Labs, 2026-01-27

## Related

- [[GNN Interpretability]]
- [[Graph Attention Network]]
- [[Graph Neural Network]]
