---
title: judgy
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/technology
  - status/permanent
aliases:
  - judgy
  - Bias Correction
  - Confidence Interval
  - Statistical Correction
updated: 2026-06-11
---

# judgy

> [!note] Definición
> Librería Python ([github.com/ai-evals-course/judgy](https://github.com/ai-evals-course/judgy)) que **corrige el sesgo de un [[LLM as Judge]]** usando su TPR/TNR conocidos, y devuelve un **success rate corregido con intervalos de confianza**. Es el paso 7 (final) del workflow de judge.

## El problema que resuelve

- Aunque el judge sea bueno, **comete errores**: un TPR=95.7% pierde el 4.3% de los PASS reales; un TNR=100% nunca pierde un FAIL real → el **raw pass rate observado está sesgado**.
- judgy usa los errores medidos del judge ([[Judge Metrics]]) para estimar el rate verdadero.

## Cómo funciona

Toma tres entradas:
1. **test labels** — el ground truth del Test set.
2. **test predictions** — lo que dijo el judge sobre esos traces etiquetados.
3. **unlabeled predictions** — lo que dijo el judge sobre **todos** los traces de producción.

```python
from judgy import estimate_success_rate
results = estimate_success_rate(test_labels=test_labels, test_preds=test_preds, unlabeled_preds=unlabeled_preds)
```

## Confidence Interval (CI)

> [!note] Definición
> Un **intervalo de confianza** expresa el rango plausible del valor real con una probabilidad dada (típicamente 95%). En vez de un punto ("88%") reportás un rango ("entre 84% y 99% con 95% de confianza"), comunicando la incertidumbre estadística honestamente.

## Resultados reales (1000 traces)

- Raw observed success rate **84.4%** → **Corrected success rate 88.2% (+3.8 pp)**, **95% CI [84.4%, 98.5%]**.
- La corrección importa porque el raw **subestima** el rendimiento real (el judge tiene leve tendencia a falsos negativos: TPR=95.7%, no 100%).

> [!tip] Cómo reportar a stakeholders
> *"Sigue restricciones dietéticas el 88% del tiempo, 95% de confianza de que el rate real está entre 84% y 99% → ~12% de recetas pueden violar preferencias; para dietas de alto riesgo (diabetic-friendly, nut-free) recomendamos safeguards adicionales."* Más creíble que "lo testeamos y parece funcionar".

## Notación estadística

- `p_obs` = raw pass rate observado · `θ̂` (theta-hat) = rate corregido.

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Cap 10; Apéndice A.
- Repo: [github.com/ai-evals-course/judgy](https://github.com/ai-evals-course/judgy) (licencia Open).

## Related

- [[LLM as Judge]]
- [[Judge Metrics]]
- [[Judge Validation]]
- [[Eval Lifecycle]]
- [[Evals]]
