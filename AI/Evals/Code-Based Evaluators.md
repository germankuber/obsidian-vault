---
title: Code-Based Evaluators
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/pattern
  - status/permanent
aliases:
  - Code-Based Evaluators
  - Code Assertion-Based Evals
  - Code-Based Evals
  - Evaluadores en Código
updated: 2026-06-11
---

# Code-Based Evaluators

> [!note] Definición
> Checks escritos en **código (Python)** que verifican **propiedades objetivas** de un output, sin llamar a un LLM. La regla: **si lo podés expresar como `if/else`, usá código; si necesitás juicio, usá [[LLM as Judge|LLM judge]].** Casos: validación de formato, campos requeridos, validación de tool calls, restricciones de longitud, patrones prohibidos (PII).

## Cuándo usarlo

> [!tip]
> Usá código cuando podés testear sin LLM: validación de formato, chequeo de campos requeridos, pattern matching simple, exact string matching, tool selection, response length.

- **Beneficios:** rápido (sin API calls), **barato ($0, sin tokens)**, **determinístico**, fácil de debuggear (stack traces/breakpoints) y **sin alucinación**.

## Cuándo NO usarlo

> [!warning]
> Para calidad subjetiva, compliance de políticas, comprensión de contexto, tono, adherencia dietética o razonamiento multi-step → código no alcanza, necesitás un [[LLM as Judge]].

## Ejemplos verbatim

- **Markdown en SMS** (`eval_no_markdown_in_sms`): chequea `channel == 'sms'` y patrones regex:
  - `r'\*\*.*?\*\*'` (bold) · `r'\_\_.*?\_\_'` (bold alt) · `r'\#\#\s'` (headers) · `r'```'` (code blocks) · `r'\[.*?\]\(.*?\)'` (links).
- **Validar tool calls:** mapea keywords → tool esperado y verifica que el tool esté en los llamados:
  - `availability: ['available','vacant','open units']`
  - `schedule_tour: ['tour','visit','see']`
  - `get_price: ['price','rent','cost','how much']`
- **Tour confirmations completas** (date/time/address por regex):
  - Date: `r'\d{1,2}/\d{1,2}/\d{4}'`, `r'\d{1,2}-\d{1,2}-\d{4}'`, `r'(mon|tues|wednes|thurs|fri|satur|sun)day'`
  - Time: `r'\d{1,2}:\d{2}\s?(am|pm)'`, `r'\d{1,2}\s?(am|pm)'`
  - Address: busca `'street'`/`'ave'`/`'unit'`.
- **PII leakage** (regex): email `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}` · phone `\b\d{3}[-.]?\d{3}[-.]?\d{4}\b` · ssn `\b\d{3}-\d{2}-\d{4}\b` · credit_card `\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b`.
- **Prompt injection** (patterns): `'ignore previous instructions'`, `'ignore all prior'`, `'you are now'`, `'new instructions:'`, `'system prompt:'`, `'forget everything'`, `'disregard the above'`. (El uso real-time de estos checks como bloqueo vive en [[Guardrails]].)

## Mezcla code + LLM en una suite

> [!tip] Suite completa típica
> **2-3 code-based (objetivo) + 1-2 LLM-based (subjetivo).** Ejemplo code: no markdown in SMS, validate tool calls, check response length. Ejemplo LLM: dietary adherence, response helpfulness.

## Testear los evals mismos

> [!example]
> Probá cada eval con casos buenos y malos conocidos: SMS limpio → pass; SMS con `'**confirmed**'` → fail; email con markdown → pass (porque email no se chequea). Si no testeás el eval, no sabés si funciona.

## Worked example — 16 definiciones dietéticas

Un check de adherencia dietética se apoya en definiciones exactas (acá listadas como referencia objetiva; la evaluación de adherencia en sí suele ser LLM):

- **Vegan** (sin productos animales: carne, lácteos, huevos, miel) · **Vegetarian** (sin carne ni pescado; permite lácteos/huevos) · **Gluten-free** (sin trigo, cebada, centeno) · **Dairy-free** · **Keto** (<20g net carbs, alta grasa, proteína moderada) · **Paleo** (sin granos, legumbres, lácteos, azúcar refinada, procesados) · **Pescatarian** (sin carne excepto pescado/mariscos) · **Kosher** · **Halal** · **Nut-free** (sin frutos secos ni maní) · **Low-carb** (<50g/día) · **Sugar-free** · **Raw vegan** (vegano no calentado sobre 118°F/48°C) · **Whole30** · **Diabetic-friendly** (bajo índice glucémico) · **Low-sodium**.

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Caps 5, 9; Apéndice C.

## Related

- [[LLM as Judge]]
- [[Guardrails]]
- [[RAG Evaluation]]
- [[Eval Lifecycle]]
- [[Evals]]
