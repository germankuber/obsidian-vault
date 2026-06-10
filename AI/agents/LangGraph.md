---
title: LangGraph
source: https://substack.com/home/post/p-197984224
author: Avani (alwaysavani)
created: 2026-06-10
tags:
  - ai/agents/ecosystem
aliases:
  - LangGraph
  - langgraph
---

# LangGraph

> [!note] Definición
> Librería para orquestar agentes como una **máquina de estados** (state machine) con **conditional edges**. De la familia [[AI Framework|LangChain]]. Modela el flujo de control de un harness de agentes de forma explícita: nodos + estado compartido + aristas condicionales.

## Modelo de estado compartido

- Todo lo que el harness sabe vive en un **state dict** tipado — **sin estado global, sin side channels** entre agentes. Ejemplo (`TypedDict`):

```python
class AgentState(TypedDict):
    base_resume_text: str       # source of truth — never modified
    job_description_text: str   # the target
    draft_resume: str           # generator output, overwritten each iteration
    evaluation_feedback: str    # harness findings, injected into next generation
    hallucinations_found: bool  # loop control signal
    iteration_count: int        # ceiling enforcement
    output_format: str          # "Markdown" or "LaTeX" — flows through all nodes
```

- Un campo seteado una vez (ej. `output_format`) **fluye sin cambios** por todos los nodos: una fuente de verdad, sin detección/branching por nodo.

## Conditional edges — el control de flujo

- Las decisiones de routing viven en **un solo lugar** (la conditional edge), no dispersas por las implementaciones de los nodos:

```python
def route_next(state: AgentState):
    if state.get("hallucinations_found") and state.get("iteration_count", 0) < 3:
        return "generator"
    return END
```

> [!tip] Dónde encaja en el ecosistema
> LangGraph es el **[[Orchestrator|orquestador]]** de un [[Agent Harness]]: secuencia los nodos y decide el loop. Es la pieza que el patrón [[Generator-Evaluator Pattern]] usa para hacer pelear a los dos agentes.

## References

- Fuente: [Grounded Eval Harness: Building an AI That Fact-Checks Itself](https://substack.com/home/post/p-197984224) — Avani, 2026-05-16

## Related

- [[Grounded Eval Harness]]
- [[Agent Harness]]
- [[Orchestrator]]
- [[AI Framework]]
