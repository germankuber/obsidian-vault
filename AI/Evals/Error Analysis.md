---
title: Error Analysis
source: https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7
author: Stephan Beyer
created: 2026-06-10
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - Error Analysis
  - Análisis de Errores
---

# Error Analysis

> [!note] Definición
> Leer manualmente el **trace log** (las conversaciones reales usuario↔sistema) y anotar los errores observados, para descubrir **qué evals necesitás de verdad**. Es el corazón del enfoque **bottom-up** de los [[Evals]]: en vez de adivinar qué evaluar, los patrones de error reales te lo dicen.

## El proceso (dos etapas de coding)

Tomado de qualitative research, el análisis pasa por dos fases:

1. **[[Open Coding]]** — *"¿qué está pasando acá?"* Etiquetás **todo** lo que observás, en una columna dedicada al lado de cada conversación logueada.
2. **[[Axial Coding]]** — *"¿cómo se relacionan estas cosas?"* Agrupás y conectás esas etiquetas en patrones/categorías.

De ahí sale un **heatmap de errores** (frecuencia × severidad) que guía la priorización: muestra visualmente dónde se concentran los issues más frecuentes e impactantes. En el caso del artículo, los issues se agruparon en 3 categorías: *Unfriendly Response*, *Missing Human Handoff*, *Not helpful*.

## Por qué lo tiene que hacer un humano (no un LLM)

> [!warning] No automatices este paso con un LLM demasiado pronto
> Si salteás leer las conversaciones y se lo pasás a un LLM muy temprano, estás **volando a ciegas** sobre qué funciona y qué no. Como PM, este análisis es tu **[[Ground Truth]]** y determina si tu usuario tuvo una experiencia significativa con el producto. Leer la data a mano es como construís la intuición.

> [!tip] El LLM sí ayuda DESPUÉS
> Una vez hecho el open coding a mano, sí podés usar un LLM (ej. Claude) para leer todas tus anotaciones y **sugerir las categorías conjuntas** (axial codes), y para mapear esas categorías de vuelta a cada conversación. El humano hace el trabajo de observación; el LLM acelera la agrupación.

## References

- Fuente: [AI Evals: Getting started with Evals](https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7) — Stephan Beyer, 2026-03-20

## Related

- [[Evals]]
- [[Open Coding]]
- [[Axial Coding]]
- [[Ground Truth]]
- [[LLM as Judge]]
