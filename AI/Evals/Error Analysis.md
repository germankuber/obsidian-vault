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
updated: 2026-06-11
---

# Error Analysis

> [!note] Definición
> Leer manualmente el **trace log** (las conversaciones reales usuario↔sistema) y anotar los errores observados, para descubrir **qué evals necesitás de verdad**. Es el corazón del enfoque **bottom-up** de los [[Evals]]: en vez de adivinar qué evaluar, los patrones de error reales te lo dicen.

## El proceso (dos etapas de coding)

Tomado de qualitative research, el análisis pasa por dos fases:

1. **[[Open Coding]]** — *"¿qué está pasando acá?"* Etiquetás **todo** lo que observás, en una columna dedicada al lado de cada conversación logueada.
2. **[[Axial Coding]]** — *"¿cómo se relacionan estas cosas?"* Agrupás y conectás esas etiquetas en patrones/categorías.

De ahí sale un **heatmap de errores** (frecuencia × severidad) que guía la priorización: muestra visualmente dónde se concentran los issues más frecuentes e impactantes. En el caso del artículo, los issues se agruparon en 3 categorías: *Unfriendly Response*, *Missing Human Handoff*, *Not helpful*.

## El proceso sistemático (la versión de Om Bharatiya)

Error analysis = (1) revisar traces, (2) anotar problemas, (3) categorizarlos, (4) contar frecuencia. Es la skill más importante; la mayoría salta a dashboards o judges, y eso está al revés.

### Dimensional Sampling

Para generar queries diversas, definí **3-4 dimensiones** y combinálas. Ejemplo Recipe Bot: `dietary_restriction` (5) × `cuisine_type` (5) × `meal_type` (5) × `skill_level` (3) = **375 combinaciones**.

```python
import random
random.seed(42)
for i in range(25):  # 25 tuplas diversas
    tuple_data = {k: random.choice(DIMENSIONS[k]) for k in DIMENSIONS}
```

Las tuplas se convierten a queries en lenguaje natural con un LLM (`temperature=0.9` para variedad). Ej: `(vegan, Italian, dinner, beginner)` → *"Hey, I'm new to cooking and vegan. Can you suggest an easy Italian dinner?"*.

### Failure Mode

Un **failure mode** es una **forma nombrada de falla** (label corto, ≤2 palabras, patrón distinto, aplicable a varios traces). Ejemplo real: `['Dietary Ignored', 'Formatting Error', 'Complexity Mismatch', 'Meal Type Mismatch', 'Ingredient Omission', 'Skill Level Misalignment']`. Las categorías deben ser tan específicas que otra persona pueda etiquetar con ellas — evitá genéricas como "Temporal issues".

### Frequency × Severity Prioritization

Se prioriza por **frecuencia × severidad**, no solo frecuencia. Ejemplo: violaciones dietéticas 11% pero pueden dañar a usuarios con alergias = **HIGH-SEVERITY**; problemas de formato 11% pero solo molestos = **LOW-SEVERITY** → arreglar adherencia dietética primero. (Conteo real: Complexity/Meal Type/Ingredient Omission 22% c/u; Dietary/Formatting/Skill 11% c/u.)

### Theoretical Saturation

Cuándo **parar** de revisar traces: cuando dejan de aparecer failure modes nuevos. Ejemplo: primeros 50 traces → 10 tipos; siguientes 25 → 2 nuevos; siguientes 25 → 0 nuevos → **STOP**. No necesitás 1000 traces si tras 100 no hay patrones nuevos. (Revisar 100 traces toma ~45 min en total.)

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
- [[Pipeline and Multi-Turn Evaluation]]
- [[Common Eval Mistakes]]
