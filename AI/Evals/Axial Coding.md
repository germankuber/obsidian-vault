---
title: Axial Coding
source: https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7
author: Stephan Beyer
created: 2026-06-10
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - Axial Coding
  - Axial Codes
  - Codificación Axial
---

# Axial Coding

> [!note] Definición
> La **segunda etapa** de analizar data cualitativa: responde *"¿cómo se relacionan estas cosas?"*. Tomás las etiquetas sueltas del [[Open Coding]] y las **agrupás y conectás en patrones / categorías significativas**. En [[Evals]], convierte una lista de errores individuales en un conjunto manejable de categorías de error.

## En el flujo de evals

- Segundo paso del [[Error Analysis]], después del [[Open Coding]].
- Aquí **sí podés usar un LLM**: en el caso del artículo, se usó Claude para leer todas las anotaciones de open coding y **sugerir un set de categorías conjuntas** (axial codes).
- El resultado alimenta un **heatmap de errores** que muestra dónde se concentran los issues más frecuentes e impactantes → guía qué evals construir primero. En el ejemplo, las categorías derivadas fueron *Unfriendly Response*, *Missing Human Handoff* y *Not helpful*.

## Por qué importa

- Sin agrupar, tenés una lista inmanejable de observaciones. Las categorías axiales son las que se convierten en **evals concretos** (frecuencia × severidad).

## References

- Fuente: [AI Evals: Getting started with Evals](https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7) — Stephan Beyer, 2026-03-20

## Related

- [[Open Coding]]
- [[Error Analysis]]
- [[Evals]]
