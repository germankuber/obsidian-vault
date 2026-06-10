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
---

# LLM as Judge

> [!note] Definición
> Usar un **LLM para evaluar automáticamente** cada output de tu sistema según un eval definido, comparando su veredicto contra el [[Ground Truth]] humano. Es la pieza que **escala** el [[Error Analysis]] manual a todo el dataset. Forma parte del flujo de [[Evals]].

## Diseño del judge

> [!tip] Preferí respuestas binarias (TRUE/FALSE), no Likert
> Decisión de diseño clave del artículo: **no** usar una escala Likert ni un score 1–5. Se busca una respuesta **binaria** — ¿este output se libera al cliente o no? TRUE o FALSE. Es más simple y accionable, y te fuerza a decidir claramente (algo difícil con un "3.5/5" ambiguo).

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
> El LLM judge necesita iteración, refinamiento y **calibración continua**, y depende de qué evals necesites. Según el producto, puede que requieras un enfoque totalmente distinto. Alternativas/complementos a investigar: [[Guardrails]] y [[Code Assertion-Based Evals]].

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
