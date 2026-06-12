---
title: Common Eval Mistakes
source: 'AI Evals For Engineers, PMs & QAs: Complete Study Guide'
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - Common Eval Mistakes
  - Eval Antipatterns
  - Errores Comunes de Evals
---

# Common Eval Mistakes

> [!note] Definición
> Checklist de los **12 antipatrones** más comunes al construir evals. Es una nota de **alto reuso**: revisala antes de cerrar un sistema de evals. Donde un antipatrón ya tiene nota dedicada, se wikilinkea en vez de repetir.

## Los 12 antipatrones

1. **Saltarse el [[Error Analysis]]** — sin él no sabés qué medir. Siempre empezá por ahí.
2. **Validar solo con agreement** — un judge que siempre dice "pass" tiene 90% agreement si las fallas son raras. Calculá **TPR y TNR por separado** (ver [[Judge Metrics]]).
3. **PM/QA delega el error analysis** — los ingenieros no tienen intuición de producto; es product work.
4. **No splitear Train/Dev/Test** — overfitting al test → métricas sin sentido. Usá **15/40/45** y no toques test hasta el final (ver [[Judge Validation]]).
5. **No hacer evals hasta después del launch** — construilas mientras construís el producto.
6. **Demasiados evals** — empezá con **2-3** para los mayores problemas. Regla: si un eval no disparó en 3 meses, removelo.
7. **TNR bajo / ignorar falsos positivos** — un eval con TNR=22% lo vas a terminar ignorando.
8. **No testear los evals mismos** — probalos con casos buenos y malos conocidos (ver [[Code-Based Evaluators]]).
9. **Copy-pastear judge prompts** — escribí evals para TU producto/políticas/usuarios.
10. **No versionar system prompts** — usá prompt management, logueá qué versión con cada trace (ver [[Eval Observability]]).
11. **No corregir el bias del judge** — usá [[judgy]] + confidence intervals.
12. **Over-engineering temprano** — empezá con CSV + Python + cualquier tool de observabilidad.

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Cap 15; Lessons Learned.

## Related

- [[Error Analysis]]
- [[Judge Metrics]]
- [[Judge Validation]]
- [[judgy]]
- [[Eval Lifecycle]]
- [[Evals]]
