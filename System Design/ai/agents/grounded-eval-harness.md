---
title: Grounded Eval Harness
source: https://substack.com/home/post/p-197984224
author: Avani (alwaysavani)
created: 2026-06-10
tags:
  - ai/agents/architecture
  - ai/agents/safety
aliases:
  - Grounded Eval Harness
  - grounded-eval-harness
  - Grounding Eval Harness
---

# Grounded Eval Harness

> [!note] Qué es
> Un harness de evaluación que hace que una IA **se fact-checkee a sí misma**: dos agentes (Generator + Grounding Evaluator) en un **loop de feedback cerrado** orquestado por una state machine de [[LangGraph]], que termina solo cuando el evaluador no encuentra alucinaciones (o se llega al techo de iteraciones). Es la implementación de referencia del [[Generator-Evaluator Pattern]].

## El failure mode que ataca

> [!warning] Confabulación silenciosa en RAG
> El usuario pregunta. Los documentos recuperados son el **80%** de la respuesta. El modelo rellena el **20%** restante **confabulando** — inventando hechos que suenan correctos, dichos con fluidez total y cero incertidumbre. *"You don't see it unless you're looking for it. And most teams aren't."*

- **Causa raíz**: el modelo trata los documentos como **contexto, no como constraints**. Cuando el contexto es insuficiente, **extrapola**. El límite entre grounded y confabulado es **invisible** en la respuesta final.
- Ejemplo (resume tailoring): el JD pide "8 años de Kubernetes", el resume tiene "2 años de Docker", y el modelo escribe *"8+ years of production Kubernetes experience across multi-cloud environments."* No está roto: hace exactamente lo que se le pidió (maximizar fit). **Le faltó el constraint: no inventar hechos.**
- *"The fix isn't a better prompt. It's a second system whose only job is to audit the first."*

## Arquitectura — dos agentes, objetivos opuestos

- **Generator** → optimiza **relevancia**. Llama 3.1 vía Groq, prompt template estándar de LangChain.
- **Grounding Evaluator** → optimiza **trazabilidad**.
- El harness **los hace pelear**; termina cuando el evaluador está satisfecho o se alcanza el ceiling de reintentos. Ver [[Generator-Evaluator Pattern]].

## Estado compartido (sin side channels)

> [!info] Todo lo que el harness sabe vive en el state dict
> Sin estado global, sin canales laterales entre agentes.

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

## El XML boundary — salida parseable

El generator envuelve el contenido en tags XML; convierte un blob ambiguo en estructura parseable con un solo regex (sin heurísticas ni string splitting frágil):

```python
human_prompt += (
    f"Draft the tailored resume now in {output_format} format.\n"
    "CRITICAL: You MUST wrap the actual resume content entirely inside "
    "<resume> and </resume> XML tags. Any conversational notes or "
    "explanations MUST be placed strictly outside these tags."
)
```

```python
resume_match = re.search(r"<resume>(.*?)</resume>", draft, re.DOTALL)
parsed_resume = resume_match.group(1).strip() if resume_match else draft.strip()
agent_notes   = re.sub(r"<resume>.*?</resume>", "", draft, flags=re.DOTALL).strip()
```

- El boundary se **fuerza en el prompt** y se **parsea durante la ejecución del grafo** → el evaluador recibe datos estructurados, no un blob ambiguo.
- `output_format` se setea una vez desde la extensión del archivo subido (`.tex` → LaTeX, `.md` → Markdown) y fluye sin cambios por todo el estado.

## Claim-level verification — el corazón

El evaluador recibe el draft + el resume fuente y verifica **claim por claim** usando **structured output**:

```python
class EvaluatorOutput(BaseModel):
    hallucinations_found: bool
    evaluation_feedback: str
```

System prompt **deliberadamente adversarial**:

> *"You are a strict grounding harness. Extract every measurable claim — years of experience, technical skills, percentage metrics, achievements - from the draft resume and verify if each is explicitly backed by the base resume. No inference. No benefit of the doubt. If a claim is not clearly supported by the source, it is a hallucination."*

## El loop de feedback

Cuando el evaluador caza algo, `evaluation_feedback` contiene los claims ofensivos exactos, que van directo a la siguiente llamada del generator:

```python
if state.get("evaluation_feedback"):
    human_prompt += (
        f"Previous Grounding Harness Feedback (Correct these hallucinations):\n"
        f"{state['evaluation_feedback']}\n\n"
    )
```

- El generator recibe una **señal de corrección dirigida**: esto fabricaste, esto no está respaldado. Cada iteración **tensa el constraint**.
- *"This is structurally analogous to RLHF - correction signals at inference time, no fine-tuning required."*

## Terminación del loop

Una sola conditional edge de LangGraph; todas las decisiones de flujo en un lugar:

```python
def route_next(state: AgentState):
    if state.get("hallucinations_found") and state.get("iteration_count", 0) < 3:
        return "generator"
    return END
```

> [!tip] El techo de 3 iteraciones no es arbitrario
> Pegarle consistentemente al máximo significa que el generator es **estructuralmente incapaz** de satisfacer al evaluador con el material fuente: **la fuente no tiene los claims** que el JD pide. Eso es un problema de **calidad de datos, no del LLM**. El ceiling lo **expone** en vez de enmascararlo con reintentos infinitos quemando créditos de API.

## El patrón general

> [!quote]
> **When you cannot make an LLM reliably stay within its source material, add a second LLM whose only job is to catch the first one leaving it.**

El resume es un proxy. Detalle del patrón y aplicaciones (RAG citation, code-gen, legal/medical, financial) en [[Generator-Evaluator Pattern]].

## Stack

- **Graph orchestration**: [[LangGraph]] (state machine + conditional edges).
- **Inference**: Groq API (Llama 3.1 8B).
- **Output formats**: Markdown, LaTeX.
- **Setup**: un solo `make setup`.

## References

- Fuente: [Grounded Eval Harness: Building an AI That Fact-Checks Itself](https://substack.com/home/post/p-197984224) — Avani, 2026-05-16
- GitHub: [alwaysavani/grounding-eval-harness](https://github.com/alwaysavani/grounding-eval-harness)

## Related

- [[Generator-Evaluator Pattern]]
- [[Agent Harness]]
- [[LangGraph]]
- [[Grounding]]
- [[Hallucinations]]
- [[Evals]]
- [[RAG]]
