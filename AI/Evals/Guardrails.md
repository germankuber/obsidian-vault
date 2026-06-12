---
title: Guardrails
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/pattern
  - status/permanent
aliases:
  - Guardrails
  - Real-Time Guardrails
  - Safety Guardrails
updated: 2026-06-11
---

# Guardrails

> [!note] Definición
> **Evals de seguridad que corren en tiempo real (online), ANTES de que la respuesta llegue al usuario**, y la **bloquean** si falla algún check, sustituyéndola por un fallback. Son la contracara *online* de los evals *offline* (ver [[Eval Lifecycle]]): no miden tendencias post-hoc, **previenen** malas respuestas en el momento.

> [!info] No confundir con los compute-guardrails de [[AutoMLOps]]
> En AutoMLOps, "guardrails" significa **caps de wall-clock / timeout / budget / kill switch** sobre el loop de optimización — un concepto distinto. Acá "guardrails" = **safety checks sobre el output del LLM** en producción.

## Cómo funciona

- Un `GuardrailPipeline` corre una serie de checks (PII, prompt injection, off-topic/harmful) sobre la respuesta candidata:
  - Si **alguno falla** → `action = 'block'` con fallback: *"I'm sorry, I can't help with that. Let me connect you with a human agent."*
  - Si **todos pasan** → `action = 'allow'`.
- Los checks concretos (regex de PII, patterns de prompt injection, LLM judge de contenido dañino) viven en [[Code-Based Evaluators]] y en LLM judges de safety.

## Guardrail Latency Budget

> [!warning] No todo check sirve para real-time
> Un guardrail solo es viable online si entra en el presupuesto de latencia:

| Tipo de check | Latencia | ¿Real-time? |
|---|---|---|
| Regex / código | <1ms | **Sí** |
| Embedding similarity | 10-50ms | **Sí** |
| Small LLM (Haiku-class) | 200-500ms | **Marginal** (delay notable) |
| Large LLM (GPT-4o-class) | 1-3s | **No** (solo offline) |

## Production Monitoring & Alerting

- `daily_eval_report` corre sobre un **sample** de los traces de ayer y cuenta `safety_failures` / `quality_failures` / `injection_attempts`.
- **Alerta si la safety failure rate supera el 1% (0.01).**
- Las safety evals **no son opcionales** antes de shippear.

## Eval Caching

> [!tip]
> Cacheá evals duplicadas usando un **hash MD5 de `input + output`** como key — evita re-evaluar respuestas idénticas y baja costo/latencia.

## Cuándo usarlo / cuándo no

> [!tip] Usalo cuando
> Hay riesgo de daño real-time (PII leakage, prompt injection, consejo médico/financiero sin disclaimers, contenido off-topic) que NO podés permitir que llegue al usuario.

> [!warning] Trade-off
> Cada guardrail agrega latencia al path crítico. Usá los baratos (regex, embeddings) inline y dejá los LLM-judge caros para evaluación **offline**, no como bloqueo real-time.

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Caps 9, 13.

## Related

- [[Code-Based Evaluators]]
- [[Eval Lifecycle]]
- [[LLM as Judge]]
- [[AutoMLOps]]
- [[Evals]]
