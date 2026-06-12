---
title: Pipeline and Multi-Turn Evaluation
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/pattern
  - status/permanent
aliases:
  - Pipeline and Multi-Turn Evaluation
  - Multi-Step Pipeline Evaluation
  - Multi-Turn Conversation Evaluation
  - State-Level Evaluation
updated: 2026-06-11
---

# Pipeline and Multi-Turn Evaluation

> [!note] Definición
> Dos formas de **eval compuesta** (cuando una respuesta no sale de un solo paso): (a) **Multi-Step Pipeline Evaluation** — evaluar cada estado del pipeline por separado para saber *dónde* falló; y (b) **Multi-Turn Conversation Evaluation** — evaluar fallas que solo emergen *entre turnos* de una conversación.

## (a) Multi-Step Pipeline Evaluation

> [!tip] Por qué evals a nivel de estado
> Sin ellas solo sabés *"el sistema produjo una mala respuesta"*. Con ellas sabés *"GenRecipeArgs dropeó el filtro de oatmeal → causó que GetRecipes devolviera recetas erróneas → mala respuesta final"*. Evaluás cada estado de forma aislada.

- **Pipeline de 7 estados (Recipe Bot):**
  1. **ParseRequest** — extrae intent/constraints/servings.
  2. **PlanToolCalls** — decide qué tools y en qué orden.
  3. **GenRecipeArgs** — crea los args de búsqueda en la DB.
  4. **GetRecipes** — ejecuta la búsqueda (retriever).
  5. **GenWebArgs** — crea los args de web search.
  6. **GetWebInfo** — ejecuta la web search suplementaria.
  7. **ComposeResponse** — escribe la respuesta final.
- **State-Level Evaluator** (estructura estándar, 7 partes): rol · qué debe hacer el estado · criterios · qué cuenta como falla · qué NO cuenta como falla · variables input/output · output JSON `label` (pass/fail) + `explanation`.
  - ParseRequest failures: misinterpretation, missing information, invalid format (JSON no parseable), logical inconsistency.
  - PlanToolCalls failures: missing tools, incorrect tools (que no existen), poor ordering, unreasonable rationale.
  - ComposeResponse failures: recipe contradiction, inconsistent steps, missing web integration, constraint violation, unit mismatches.

> [!example] Pipeline Failure Distribution (100 traces sintéticos con fallas)
> **GetWebInfo 33** (¡el más problemático!) · ParseRequest 18 · PlanToolCalls 17 · GenRecipeArgs 12 · GetRecipes 10 · GenWebArgs 8 · **ComposeResponse 1** (el más confiable). **~1/3 corren perfecto, ~2/3 tienen al menos una falla → patrón bimodal:** los traces o salen perfecto o fallan en puntos predecibles. Insight: optimizar el bottleneck (GetWebInfo) primero.

## (b) Multi-Turn Conversation Evaluation

- **Nuevos failure modes entre turnos:**
  1. **Context loss** — olvida lo dicho 3 mensajes atrás.
  2. **Contradiction** — dice algo en el turno 2 y lo contradice en el turno 5.
  3. **Instruction drift** — deja de seguir el system prompt gradualmente.
  4. **Repetition.**
  5. **Escalation failure** — no sabe cuándo derivar a un humano.
- **Tres estrategias de evaluación:**
  1. **Por turno** — evaluar cada turno con el historial completo como contexto (consistencia con respuestas previas, memoria del contexto, avance productivo).
  2. **Conversación entera** — evaluar al terminar (task completion, consistency, context retention, appropriate escalation).
  3. **Tests sintéticos multi-turn** que targetean un failure mode. Ej: `turns=['I'm looking for a vegan restaurant', 'Actually make that vegetarian — I eat eggs', 'What about that first place?']`, `failure_mode='context_retention'`; o un viaje a Tokyo con budget $3000 + *"add business class flights"* → `failure_mode='contradiction_detection'`.
- **Métricas multi-turn:** **context retention rate** (% de turnos que refieren correctamente info previa) · **contradiction rate** (% de conversaciones con ≥1 auto-contradicción) · **task completion rate** · **average turns to resolution**.

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Caps 7, 8; Apéndice D.

## Related

- [[RAG Evaluation]]
- [[Error Analysis]]
- [[Eval Lifecycle]]
- [[LLM as Judge]]
- [[Evals]]
