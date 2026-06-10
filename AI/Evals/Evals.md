---
title: Evals
source: https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7
author: Stephan Beyer
created: 2026-06-10
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - Evals
  - Evaluations
  - Evaluaciones
  - LLM Evals
  - LLM Evaluation
updated: 2026-06-10
---

# Evals

> [!note] Definición
> Tests que miden **sistemáticamente si un sistema de IA/LLM se comporta como se espera** y dónde se queda corto. No prueban que el sistema sea perfecto; **reducen la ignorancia** sobre cómo se comporta — y eso ya es una mejora seria.

## Por qué hacen falta: determinístico vs probabilístico

- El software tradicional es **determinístico**: siempre produce el mismo output según lógica predefinida.
- Un LLM es **probabilístico**: produce outputs según probabilidades aprendidas, no reglas fijas. Genera lo que considera "más probable".
- Consecuencia: con LLMs **no se puede especificar el resultado con 100% de certeza**, así que se vuelve esencial medir el modelo contra un conjunto de criterios de evaluación — los **evals**.

> [!tip] Por qué importa para PMs
> Dominar evals será una skill crítica para cualquier Product Manager que trabaje en un producto de IA. El barrier de entrada no es técnico ni de tooling — es la voluntad de **sentarse, leer la data y formarse un punto de vista propio**.

## Bottom-up vs top-down (el enfoque de Hamel Husain)

- El insight central (de [[Hamel Husain]]): usar un enfoque **bottom-up** para determinar los evals correctos, partiendo de un **error log** y un [[LLM as Judge|LLM judge]]. Fuerza a responder la pregunta esencial: *"¿qué evals necesitás realmente, y por qué?"*
- **Bottom-up, no top-down**: arrancar con una herramienta automática y sofisticada NO resuelve el problema de entender *qué evals necesitás y por qué*. Empezás leyendo el error log y cavando en las conversaciones reales — así construís intuición de qué funciona y qué no.
- **No estás adivinando** qué evaluar: dejás que los **patrones de error reales** te lo digan.

## El método paso a paso

El flujo completo (detalle en cada nota atómica):

1. **Preparar la data** — acceder al *trace log* (las conversaciones usuario↔sistema), típicamente exportado a CSV para mapear anotaciones fácil.
2. **[[Error Analysis]]** — leer las conversaciones y anotar errores. Combina [[Open Coding]] (etiquetar lo observado) y [[Axial Coding]] (agrupar en patrones/categorías). **Lo hace un humano, no un LLM** — es tu ground truth.
3. **Derivar los evals** — del heatmap de errores (frecuencia × severidad) salen los evals concretos a construir.
4. **[[LLM as Judge]]** — configurar un LLM que evalúe cada conversación según los evals definidos, comparando contra el [[Ground Truth]] humano e **iterando el prompt** según los desacuerdos.

> [!example] Resultado del caso del artículo
> El ejercicio completo tomó **~1 hora**. Sin tooling sofisticado: solo un Google Sheet, un LLM y curiosidad. De un error log real salieron 2 evals ("unfriendly response" y "human handover") y un LLM judge que, tras 2 iteraciones de prompt, pasó de **83% de agreement** a **100% de recall** de casos "unfriendly" y true-negative rate de **80%→93%**.

## Qué se mide (general)

- Accuracy, formato, estilo, calidad de retrieval, uso de tools, grounding, seguridad, latencia, éxito en la tarea.

## Qué hace a un buen eval

- Se parece al **uso real**: preguntas reales, edge cases reales, criterios claros, scoring repetible.
- **Simplicidad**: en el ejemplo, evals binarios (TRUE/FALSE) resultaron más accionables que escalas Likert matizadas (ej. 3.5/5).
- **No necesitás un setup perfecto para empezar.** No hace falta tooling sofisticado; empezá con lo que tenés.

## Más allá del LLM judge

> [!warning] El LLM judge es un punto de partida, no una bala de plata
> Según el producto, podés necesitar un set de evals totalmente distinto donde un LLM judge **no sea** el enfoque correcto. El artículo recomienda leer sobre [[Guardrails]] y [[Code Assertion-Based Evals]] como alternativas/complementos.

## Conexión en el vault

- El [[Grounded Eval Harness]] es un eval **automatizado y adversario** (un caso de [[LLM as Judge]]): un segundo LLM evalúa claim por claim al primero. Patrón general: [[Generator-Evaluator Pattern]].
- Cruza con [[Hallucinations]] y [[Grounding]] (qué se evalúa) en [[_AI Fundamentals|AI Fundamentals]].
- En MLOps, los evals son las métricas del [[Three-Tier Evaluation Pipeline]] y el problema de [[Offline vs Business Metrics]] (un eval offline puede no predecir el resultado de negocio).
- Para retrieval, las métricas concretas viven en [[Reranking Metrics]] (NDCG, MRR, Precision@k).

## References

- Fuente: [AI Evals: Getting started with Evals — A practical guide leveraging bottom-up error analysis and an LLM Judge](https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7) — Stephan Beyer, 2026-03-20
- Referencia citada por el autor: [[Hamel Husain]] — hamel.dev

## Related

- [[Error Analysis]]
- [[Open Coding]]
- [[Axial Coding]]
- [[LLM as Judge]]
- [[Ground Truth]]
- [[Grounded Eval Harness]]
- [[Generator-Evaluator Pattern]]
- [[Grounding]]
- [[Hallucinations]]
- [[Three-Tier Evaluation Pipeline]]
- [[Reranking Metrics]]
