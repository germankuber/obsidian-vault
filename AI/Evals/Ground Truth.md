---
title: Ground Truth
source: https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7
author: Stephan Beyer
created: 2026-06-10
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - Ground Truth
  - Verdad de Referencia
---

# Ground Truth

> [!note] Definición
> Las **labels humanas confiables** que representan la respuesta "correcta" de un eval, contra las cuales se compara el veredicto del [[LLM as Judge]]. Es el patrón de referencia que define el éxito o fracaso de cada caso. Sin ground truth, no podés saber si tu judge acierta.

## Cómo se construye

- Surge del [[Error Analysis]] manual: tras el [[Open Coding]] y el [[Axial Coding]], se **mapea cada categoría de error de vuelta a cada conversación**, etiquetando fila por fila. Para esto se puede usar un LLM, pero la base son las observaciones humanas.
- Ese etiquetado **es** el ground truth: el output esperado y validado por humanos, contra el que después se mide el judge.

## Para qué sirve

- En el spreadsheet del caso: la **columna F** tiene el output esperado humano (ground truth); la **columna G** tiene la respuesta del [[LLM as Judge]]. Comparar ambas da el **agreement rate** y, más útil, los **desacuerdos** que revelan dónde falla el judge.

> [!warning] Es tu responsabilidad como PM
> El ground truth refleja si el usuario tuvo una experiencia significativa con el producto. Por eso el [[Error Analysis]] que lo produce **lo tiene que hacer un humano**, no delegarse a un LLM demasiado pronto.

## References

- Fuente: [AI Evals: Getting started with Evals](https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7) — Stephan Beyer, 2026-03-20

## Related

- [[LLM as Judge]]
- [[Error Analysis]]
- [[Evals]]
