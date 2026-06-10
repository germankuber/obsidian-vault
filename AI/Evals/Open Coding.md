---
title: Open Coding
source: https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7
author: Stephan Beyer
created: 2026-06-10
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - Open Coding
  - Open Codes
  - Codificación Abierta
---

# Open Coding

> [!note] Definición
> La **primera etapa** de analizar data cualitativa: responde *"¿qué está pasando acá?"*. Etiquetás **todo lo que observás** sin categorías predefinidas. En el contexto de [[Evals]], es anotar cada error que ves al leer el trace log, en una columna dedicada al lado de cada conversación.

## En el flujo de evals

- Es el primer paso del [[Error Analysis]]: leés las conversaciones usuario↔sistema y vas dejando notas sobre los errores observados.
- Las etiquetas son **libres y granulares** — no agrupás todavía, solo registrás lo que ves. La agrupación viene después con [[Axial Coding]].
- Término tomado de la **qualitative research** (investigación cualitativa).

## Por qué importa

- Hacerlo a mano (un humano, no un LLM) es lo que genera la intuición sobre qué funciona y qué no en el producto. Es la materia prima del [[Ground Truth]].

## References

- Fuente: [AI Evals: Getting started with Evals](https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7) — Stephan Beyer, 2026-03-20

## Related

- [[Axial Coding]]
- [[Error Analysis]]
- [[Evals]]
