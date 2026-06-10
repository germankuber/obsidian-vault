---
title: Generator-Evaluator Pattern
source: https://substack.com/home/post/p-197984224
author: Avani (alwaysavani)
created: 2026-06-10
tags:
  - ai/agents/architecture
  - ai/agents/safety
aliases:
  - Generator-Evaluator Pattern
  - generator-evaluator-pattern
  - Patrón Generador-Evaluador
  - Adversarial Evaluator Pattern
---

# Generator-Evaluator Pattern

> [!note] El patrón en una frase
> *"When you cannot make an LLM reliably stay within its source material, add a
> second LLM whose only job is to catch the first one leaving it."* Dos agentes
> con **objetivos opuestos** en un loop de feedback cerrado, hasta que el
> evaluador queda satisfecho o se alcanza el techo de reintentos.

## Los dos agentes

- **Generator** — optimiza para **relevancia** (producir la salida que maximiza el
  fit con el objetivo).
- **Evaluator** (grounding/adversarial) — optimiza para **trazabilidad** (que cada
  claim de la salida se respalde explícitamente en la fuente).
- El harness **los hace pelear** y termina solo cuando el evaluador está
  satisfecho o se llega al ceiling de iteraciones.

> [!tip] El evaluador no necesita ser más inteligente
> *"The evaluator doesn't need to be smarter than the generator. It needs a
> different objective: distrust by default, verify explicitly, report
> precisely."* La fuerza viene de la **divergencia de objetivos**, no de más
> capacidad.

## Por qué funciona

- Un LLM trata los documentos recuperados como **contexto**, no como
  **constraints**. Cuando el contexto es insuficiente, **extrapola**. El límite
  entre salida grounded y confabulación es **invisible** en la respuesta final.
- El fix **no es un mejor prompt** — es un **segundo sistema cuyo único trabajo es
  auditar al primero**.
- El feedback del evaluador (los claims ofensivos exactos) se inyecta en la
  siguiente generación → análogo estructural a **RLHF**: señales de corrección en
  *inference time*, sin fine-tuning.

## Aplicaciones directas

- **RAG citation verification** — ¿cada claim de la respuesta rastrea a un
  documento recuperado?
- **Code generation** — ¿la salida satisface la especificación o el modelo
  extrapoló?
- **Legal / medical drafting** — ¿todas las aserciones están presentes en los
  materiales de input?
- **Financial reporting** — ¿todas las cifras son trazables a la fuente?

## Implementación de referencia

- [[Grounded Eval Harness]] — el caso de estudio concreto (resume tailoring) que
  da origen a este patrón, con [[LangGraph]] + Groq/Llama 3.1.
- Es una instancia aplicada de un [[Agent Harness]].

## References

- Fuente: [Grounded Eval Harness: Building an AI That Fact-Checks Itself](https://substack.com/home/post/p-197984224) — Avani, 2026-05-16

## Related

- [[Grounded Eval Harness]]
- [[Agent Harness]]
- [[LangGraph]]
- [[Grounding]]
- [[Hallucinations]]
