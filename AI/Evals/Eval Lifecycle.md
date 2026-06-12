---
title: Eval Lifecycle
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/pattern
  - status/permanent
aliases:
  - Eval Lifecycle
  - Closing the Loop
  - Regression Testing
  - Offline vs Online Evals
  - Tiered Evaluation
updated: 2026-06-11
---

# Eval Lifecycle

> [!note] Definición
> El **ciclo operacional** de las evals: de medir → entender → arreglar → re-medir, y la infraestructura de costo/escala alrededor. *"Medir sin actuar"* es la falla más común — las evals solo valen si **generan acción**.

## Offline vs Online Evals

| | Offline | Online |
|---|---|---|
| Cuándo | Post-hoc, tras recolectar traces | Real-time, antes/durante la respuesta |
| Latencia | minutos-horas | milisegundos-segundos |
| Mide | tendencias de calidad (ej TPR/TNR en test set) | previene malas respuestas (ej [[Guardrails]], content filters) |

## Closing the Loop

> [!tip] El ciclo de mejora
> 1. Correr evals → identificar el **top failure mode**.
> 2. **Root-cause** (¿prompt? retrieval? tool? data?).
> 3. Implementar el fix.
> 4. Correr de nuevo → confirmar mejora **+ chequear regresiones**.
> 5. Repetir.

## Root-Causing Failures

| Ubicación | Síntomas | Fix |
|---|---|---|
| System prompt | tono equivocado, capacidades faltantes, violaciones de política | editar prompt, agregar ejemplos/constraints |
| Retrieval | docs equivocados, contexto faltante | chunking, reranking, query expansion |
| Tool calls | tool/params equivocados | mejorar descripciones, validación |
| Generation | hallucination, formato equivocado, ignora contexto | few-shot, structured output, tuning de temperature |
| Post-processing | truncation, encoding, formato | fix de parsing |

## Regression Testing

> [!tip]
> Un `RegressionSuite` **guarda los known_cases** (casos que fallaron y se arreglaron) y se corre **antes de cada cambio de prompt o switch de modelo**. Garantiza que un fix no rompa lo que ya andaba.

## Model Comparison con Evals

- Loop sobre modelos (`['gpt-4o', 'claude-sonnet-4-5-20250929', 'gemini-2.0-flash']`) comparando **TPR, TNR, costo, latency_p50**.

## Costo y escala

- **Cheaper Models for Judges** (costo por 1K traces):

| Modelo | Costo/1K | Uso |
|---|---|---|
| GPT-4o / Claude Opus | ~$5-15 | juicios subjetivos complejos, safety-critical |
| GPT-4o-mini / Claude Haiku | ~$0.50-1.50 | criterios claros, rubrics bien definidos |
| Code-based | $0 | format checks, pattern matching |

> [!tip] Empezá caro, después optimizá
> Validá el prompt con un modelo fuerte; después testeá si uno barato da TPR/TNR similar (a menudo sí).

- **Trace Sampling** — en vez de evaluar exhaustivo: `sample_traces(sample_rate=0.1, min_sample=100)`. 10% de 50.000 traces/día = 5.000 evals; la confianza estadística sigue alta. **Sampling le gana a evaluación exhaustiva.**
- **Tiered Evaluation:** Tier 1 = code-based en **TODOS** (free) → Tier 2 = LLM barato (~$0.50/1K, gpt-4o-mini) en los que pasaron Tier 1 → Tier 3 = LLM caro (~$5/1K, gpt-4o) en un sample de 500.

> [!info] No confundir con [[Three-Tier Evaluation Pipeline]] (MLOps)
> El *Tiered Evaluation* de acá filtra **traces por costo de evaluador** (code→LLM barato→LLM caro). El [[Three-Tier Evaluation Pipeline]] de MLOps es otra cosa: tier 1 agente / tier 2 human-gated business proxy / tier 3 A/B test, dentro del loop de [[AutoMLOps]]. Mismo número, eje distinto.

> [!info] Distinto de [[Offline vs Business Metrics]] (MLOps)
> Esa nota trata el desacople **métrica ML offline ↔ métrica de negocio**. Acá "offline vs online" es **cuándo** corre la eval (post-hoc vs real-time), no qué tipo de métrica es.

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Caps 9, 11, 13.

## Related

- [[Guardrails]]
- [[judgy]]
- [[Judge Metrics]]
- [[Common Eval Mistakes]]
- [[Three-Tier Evaluation Pipeline]]
- [[Offline vs Business Metrics]]
- [[Evals]]
