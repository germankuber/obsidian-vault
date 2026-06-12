---
title: Judge Metrics
source: "AI Evals For Engineers, PMs & QAs: Complete Study Guide"
author: Om Bharatiya
created: 2026-06-11
tags:
  - ai/evals
  - type/concept
  - status/permanent
aliases:
  - Judge Metrics
  - Métricas del Judge
  - Confusion Matrix
  - TPR
  - TNR
  - Precision
  - F1
  - Recall
  - Specificity
updated: 2026-06-11
---

# Judge Metrics

> [!note] Definición
> La familia de métricas de **clasificación** con las que se mide qué tan bueno es un [[LLM as Judge]] contra el [[Ground Truth]] humano. Se derivan de la **confusion matrix** y son las que importan de verdad — **no** alcanza con el "agreement". (Las métricas de **retrieval** —Recall@K, MRR, Precision@k, NDCG— viven en [[Reranking Metrics]] y NO se redefinen acá.)

## Confusion Matrix (2×2)

Para un judge binario PASS/FAIL, comparado contra la verdad humana:

| | Verdad = PASS | Verdad = FAIL |
|---|---|---|
| **Judge dice PASS** | TP (true positive) | **FP** (judge "Pass" pero real "Fail" — **defecto perdido**) |
| **Judge dice FAIL** | **FN** (judge "Fail" pero real "Pass" — **falsa alarma**) | TN (true negative) |

```python
def eval_tp(*, output, expected, **kwargs):  # judge PASS y truth PASS
    return 1.0 if output.get('label','').upper()=='PASS' and expected.get('label','').upper()=='PASS' else 0.0
# eval_tn: FAIL/FAIL ; eval_fp: judge PASS pero truth FAIL ; eval_fn: judge FAIL pero truth PASS
```

## Por qué el Agreement engaña

> [!warning] El agreement solo es una trampa cuando las fallas son raras
> `Agreement = (judge concuerda conmigo) / total`. Si las fallas ocurren solo el **10%** del tiempo, un judge que **siempre dice "pass"** tiene **90% de accuracy** siendo completamente inútil — nunca detecta una falla. Por eso hay que calcular **TPR y TNR por separado**.

## Las métricas que importan

- **TPR (True Positive Rate / Recall / Sensitivity):** *"cuando realmente hay un PASS, ¿cuántas veces el judge dice PASS?"*
  - `TPR = TP / (TP + FN)`
  - Target **>80%** (good), **>90%** (great).
- **TNR (True Negative Rate / Specificity):** *"cuando realmente hay un FAIL, ¿cuántas veces el judge dice FAIL?"*
  - `TNR = TN / (TN + FP)`
  - Target **>80%** (good), **>90%** (great).
- **Precision:** `Precision = TP / (TP + FP)`.
- **F1:** media armónica de precision y recall → `F1 = 2 · (Precision · Recall) / (Precision + Recall)`.

> [!warning] AMBOS deben ser altos
> Un judge con **TPR=95% pero TNR=40% es inútil**. Y un **TNR bajo es activamente dañino**: ejemplo real del primer intento → TPR 90.1% pero **TNR 22.2%** (Accuracy 84.0%). Significa que cuando una receta violaba una restricción dietética, el judge solo lo detectaba el **22%** de las veces → le dirías a un diabético que una receta es segura cuando no lo es. Tras iterar: TPR 95.7%, TNR 100.0%, Balanced Accuracy 97.8%, Overall Accuracy 97.0% (32/33 correctas).

## Cómo usarlas para iterar

> [!tip]
> - **TPR bajo** → el judge pierde fallas reales (FN altos): agregá más Fail examples y criterios de fallo explícitos.
> - **TNR bajo** → muchas falsas alarmas (FP altos): agregá una sección "qué NO cuenta como falla" y Pass examples para edge cases.
> - **Ambos bajos** → reescribir el prompt desde cero.

El raw pass rate del judge sigue sesgado aunque TPR/TNR sean altos → se corrige con [[judgy]].

## References

- Fuente: [AI Evals For Engineers, PMs & QAs: Complete Study Guide](https://github.com/ombharatiya/ai-system-design-guide/blob/main/ai_evals_comprehensive_study_guide.md) — Om Bharatiya. Cap 4; Apéndices A, B.

## Related

- [[LLM as Judge]]
- [[Judge Validation]]
- [[judgy]]
- [[Reranking Metrics]]
- [[Ground Truth]]
- [[Evals]]
