---
title: LLM as Judge
source: https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7
author: Stephan Beyer
created: 2026-06-10
tags:
  - ai/evals
  - type/pattern
  - status/permanent
aliases:
  - LLM as Judge
  - LLM Judge
  - LLM-as-a-Judge
  - Juez LLM
updated: 2026-06-11
---

# LLM as Judge

> [!note] Definición
> Usar un **LLM para evaluar automáticamente** cada output de tu sistema según un eval definido, comparando su veredicto contra el [[Ground Truth]] humano. Es la pieza que **escala** el [[Error Analysis]] manual a todo el dataset. Forma parte del flujo de [[Evals]].

## 7-Step LLM Judge Workflow

> [!tip] El pipeline completo (Om Bharatiya)
> 1. **Generar traces.**
> 2. **Etiquetar** un subset (150-200) como PASS/FAIL — manual (más preciso) o con un LLM potente.
> 3. **Split Train/Dev/Test** estratificado 15/40/45 (ver [[Judge Validation]]).
> 4. **Desarrollar el judge prompt** con few-shot examples del Train.
> 5. **Validar en Dev** — iterar mirando TPR/TNR (ver [[Judge Metrics]]).
> 6. **Evaluación final en Test** — UNA sola vez, métricas no sesgadas.
> 7. **Correr en todos los traces + corregir el bias** con [[judgy]] (`temperature=0`, `concurrency=20`). Ejemplo: raw pass rate sobre 1000 traces = 84.4%.

## Diseño del judge

> [!tip] Preferí respuestas binarias (TRUE/FALSE), no Likert
> Decisión de diseño clave del artículo: **no** usar una escala Likert ni un score 1–5. Se busca una respuesta **binaria** — ¿este output se libera al cliente o no? TRUE o FALSE. Es más simple y accionable, y te fuerza a decidir claramente (algo difícil con un "3.5/5" ambiguo).
> La alternativa descartada es la [[Likert Scale]].

## Técnicas del judge prompt

- **Explanation-Before-Verdict** — la técnica **más impactante**: si el `label` va primero, la explicación es racionalización post-hoc; si el **razonamiento va primero**, el modelo delibera de verdad. Poné `explanation` antes que `label` en el output JSON.
- **Estructura del prompt (4 partes):** (1) rol + definiciones de dominio (ej: las 16 definiciones dietéticas, ver [[Code-Based Evaluators]]); (2) criterios claros PASS/FAIL (considerar ingredientes Y métodos de cocción); (3) **few-shot examples** del Train; (4) output format JSON.
- **Few-Shot Examples** — incluí **1 Pass claro + 1 Fail claro + 1 borderline**, cada uno con su razonamiento. No los copies de otro producto: escribilos para el tuyo.
- **Binary Scores** (ya cubierto arriba) — refuerzo: límite claro → **menos validación** que una escala. Con 1-5 no sabés la diferencia entre 2 y 3, es 5× más trabajo validar, y las decisiones de negocio son binarias igual.
- **Judge Temperature:** clasificación binaria **0.0** · Likert 1-5 **0.0-0.3** · critiques diversas **0.5-0.7** · brainstorming de failure modes **0.7-1.0**. **Para evaluar siempre 0.0.**

> [!warning] Judge Biases (guardarse contra ellos)
> - **Leniency** (default a Pass) → agregar fail examples; instruir *"when in doubt, FAIL"*.
> - **Verbosity** (favorece respuestas largas) → ejemplos donde una respuesta corta pasa y una larga falla.
> - **Position** (favorece primer/último) → randomizar el orden.
> - **Sycophancy** (acuerda con texto confiado) → ejemplos donde el texto confiado está mal.
> - **Anchoring** (se deja llevar por la primera evidencia) → instruir considerar TODA la evidencia.

## Ejemplo de judge prompt (verbatim del artículo)

```
Judge Prompt: Unfriendly Tone (TRUE/FALSE)
You are a customer support expert who reviews and analyses conversations to
determine if they were Unfriendly. You are tasked to analyze the conversations
stated in column B in this spreadsheet. Return only "TRUE" or "FALSE."

## Definitions Unfriendly (Label TRUE) — any of the below occurred:
1. Responding in an emotional, passive-aggressive, or generally unfriendly tone
2. Informing the customer about policy without showing empathy
3. Not apologizing about the situation, even if caused by the customer
4. Telling the customer to complete a task in a direct tone without empathy

## Output format: Return exactly one token "TRUE" or "FALSE"
```

## Evaluar al judge: agreement no alcanza

- Tras correr el judge, comparás sus veredictos (columna G) contra el [[Ground Truth]] humano (columna F). Los desacuerdos se marcan en rojo.
- En el caso del artículo: **83% de agreement** inicial. Pero **el agreement solo como proxy no es ideal** — no te dice *qué* está mal. Hay que entender la **naturaleza de los desacuerdos** para mejorar el judge.

## Iterar el prompt según los desacuerdos

> [!tip] Los disagreements son el dato más valioso
> Iterá basándote en señales, no en suposiciones. Los desacuerdos entre las labels humanas y el judge te dicen exactamente dónde refinar el prompt.

- **Ejemplo de disagreement** (caso M016): el asistente resolvió un envío faltante de forma eficiente y educada, pero el judge lo marcó TRUE (unfriendly) solo por ser **breve**. Diagnóstico: un **sensitivity issue** — el judge era demasiado estricto, confundía "conciso/neutral" con "unfriendly".
- **Fix**: refinar el prompt para distinguir *unfriendly* de simplemente *conciso/neutral*. En vez de reescribirlo a mano, el autor le pidió a Claude que lo mejorara incorporando los aprendizajes → **prompt v1.1**.
- **Resultado de la iteración 2**: capturó el **100%** de las conversaciones "unfriendly" del dataset, y mejoró el true-negative rate de **80% → 93%**. Mejoró tanto el agreement general como las tasas de true-positive y true-negative.

## Cuándo NO es la herramienta correcta

> [!warning] Punto de partida, no bala de plata
> El LLM judge necesita iteración, refinamiento y **calibración continua**, y depende de qué evals necesites. Según el producto, puede que requieras un enfoque totalmente distinto. Alternativas/complementos a investigar: [[Guardrails]] y [[Code-Based Evaluators|Code Assertion-Based Evals]].

## Conexión en el vault

- El [[Grounded Eval Harness]] es un LLM-as-Judge **automatizado y adversario** aplicado a fact-checking claim por claim; su forma general es el [[Generator-Evaluator Pattern]].

## References

- Fuente: [AI Evals: Getting started with Evals](https://medium.com/@STB_90/getting-started-with-evals-a-step-by-step-guide-leveraging-bottom-up-error-analysis-and-llm-judge-3d9b755824a7) — Stephan Beyer, 2026-03-20

## Related

- [[Evals]]
- [[Ground Truth]]
- [[Error Analysis]]
- [[Grounded Eval Harness]]
- [[Generator-Evaluator Pattern]]
- [[Judge Validation]]
- [[Judge Metrics]]
- [[judgy]]
- [[Code-Based Evaluators]]
- [[Likert Scale]]
