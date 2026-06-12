---
title: Judge Validation
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/pattern
  - status/permanent
aliases:
  - Judge Validation
  - Validación del Judge
  - Train Dev Test Split
  - Stratified Split
  - Cohen's Kappa
  - Inter-Annotator Agreement
updated: 2026-06-11
---

# Judge Validation

> [!note] Definición
> El proceso de **validar que un [[LLM as Judge]] mide lo que decís que mide** antes de confiar en sus veredictos a escala. Se basa en dos pilares: un **split Train/Dev/Test** disciplinado (para no overfittear las métricas) y la **calibración del ground truth humano** (inter-annotator agreement + Cohen's Kappa). Sin validar, un judge puede tener 90% de accuracy y aun así ser inútil o peligroso (ver [[Judge Metrics]]).

## Train/Dev/Test Split estratificado

- Para desarrollar y medir un judge se etiquetan **150-200 traces** como PASS/FAIL (manual = más preciso, o un LLM potente) y se parten en tres conjuntos con proporciones **15 / 40 / 45**:
  - **Train ~15%** — de acá salen los **few-shot examples** del judge prompt.
  - **Dev ~40%** — se itera el prompt acá: calcular TPR/TNR, mirar errores, ajustar, repetir.
  - **Test ~45%** — se toca **UNA sola vez**, al final, para métricas no sesgadas. Tocarlo antes = overfitting → métricas sin sentido.

```python
from sklearn.model_selection import train_test_split
train_dev, test = train_test_split(labeled_data, test_size=0.45, stratify=labeled_data['label'], random_state=42)
train, dev = train_test_split(train_dev, test_size=0.73, stratify=train_dev['label'], random_state=42)  # 0.73 → dev=40% del original
```

## Stratified Split — por qué estratificar

> [!tip]
> Necesitás **PASS y FAIL en cada split**. Sin `stratify`, el Dev podría quedar todo PASS → inútil para detectar fallas (no podés calcular TNR si no hay FAILs). El `stratify=label` garantiza que cada conjunto mantenga la proporción de clases del original.

## Inter-Annotator Agreement

- **Si dos humanos discrepan al etiquetar, tu criterio no es claro.** El acuerdo entre anotadores mide la nitidez de la definición, no la habilidad de la gente.
- Proceso: **2-3 personas** etiquetan independientemente los **mismos 50 traces** → se calcula el agreement.
- **Umbral: si el agreement es <80%, el criterio necesita más especificidad** → discutir, actualizar la guía, re-etiquetar. Recién con criterio claro tiene sentido construir el judge.
- *"Los anotadores humanos discrepan más de lo que creés"* — por eso se mide formalmente con Cohen's Kappa, no a ojo.

## Cohen's Kappa

> [!note] Definición
> Métrica de acuerdo entre anotadores que **corrige por el acuerdo esperado al azar** (un agreement crudo del 90% puede ser casi todo suerte si una clase domina).

```
kappa = (p_observed - p_expected) / (1 - p_expected)
```
donde `p_expected` = acuerdo esperado por azar.

- **Interpretación:**
  - `kappa > 0.8` — **excelente** (criterio claro, casi perfecto).
  - `0.6 - 0.8` — **bueno / sustancial** (clarificaciones menores).
  - `< 0.6` — **pobre** (reescribir el criterio desde cero).
  - Escala fina (Apéndice A): `<0.2` pobre · `0.4-0.6` moderado · `0.6-0.8` sustancial · `>0.8` casi perfecto.

## Label Quality > Quantity

> [!tip]
> **50 labels de alta calidad le ganan a 500 ruidosos.** Invertí en guías escritas con ejemplos, documentación de edge cases y sesiones de calibración, no en volumen.

- **Cuándo labels manuales > LLM:** casos ambiguos (los expertos discrepan), dominios high-stakes (medical/legal/financial), failure modes nuevos, y calibración del ground truth (validar un sample manual aunque uses LLM a escala).
- **PMs/QAs suelen ser mejores labelers** que ingenieros: conocen la buena UX, las políticas/constraints del producto y piensan desde la perspectiva del usuario.

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Caps 4, 12; Apéndices A, B.
- Basada en el curso Maven de Hamel Husain & Shreya Shankar.

## Related

- [[LLM as Judge]]
- [[Judge Metrics]]
- [[judgy]]
- [[Ground Truth]]
- [[Error Analysis]]
- [[Evals]]
