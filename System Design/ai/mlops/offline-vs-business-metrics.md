---
title: Offline vs Business Metrics
source: https://substack.com/home/post/p-198249792
author: Jam with AI
created: 2026-06-10
tags:
  - ai/mlops/evaluation
aliases:
  - Offline vs Business Metrics
  - offline-vs-business-metrics
  - Métricas Offline vs de Negocio
  - Composite Metric
  - Métrica Compuesta
---

# Offline vs Business Metrics

> [!note] Definición
> En producción importan **dos tipos de métrica** que están **relacionadas pero
> no son lo mismo**: la **métrica ML** (offline) y la **métrica de negocio**.
> Optimizar ciegamente la primera no garantiza mover la segunda.

## Los dos tipos

- **Métrica ML (offline)** — nDCG, MRR, AUC, F1, RMSE, precision, recall (ver
  [[Reranking Metrics]] para NDCG/MRR).
- **Métrica de negocio** — conversion rate, revenue per session, fraud loss
  prevented, retention, número de aplicaciones, user satisfaction, support tickets
  reducidos.

## Por qué se desacoplan

> [!warning] La correlación se rompe
> Un ranker puede subir nDCG sin subir conversion; un recommender puede mejorar
> ranking offline sin subir engagement; un modelo de fraude puede subir AUC y aún
> perder los casos que importan financieramente; un search puede verse mejor
> offline y dar un **A/B test plano**.

Causas comunes (search ranker): **feedback loops** en logs históricos, **fresh
inventory** que el modelo nunca vio, **query distribution shift**, **position
bias**, **novelty effects**. Por eso:

- Un **+0.5% de nDCG** puede dar un A/B test plano.
- Un modelo que se ve **levemente peor offline** puede ganar online porque
  **expone ítems que el viejo enterraba** consistentemente.

> [!quote] La lección de producción
> *"Offline metrics are necessary, but they are not the final truth."* El agente
> hace exactamente lo que le pedís: optimiza la métrica. Eso **no** significa que
> optimice el negocio.

## La métrica compuesta — la solución

En [[AutoMLOps]] el agente **no** debe optimizar una métrica ML pura y aislada,
sino una que **mezcle calidad ML con un business proxy**. Puede ser:

- **Score ponderado**, p. ej.:

```
0.6 * nDCG + 0.4 * estimated_revenue_per_session
```

- **Constraint**: maximizar conversion estimada **manteniendo nDCG sobre un
  umbral**; o en fraude, mejorar fraud loss prevented **manteniendo falsos
  positivos bajo un límite aprobado por negocio**.

La fórmula exacta depende del caso (marketplace ranker, job recommender, fraud
classifier, churn model → cada uno necesita su lógica de scoring). El principio:
**el agente debe optimizar lo que realmente querés mejorar, no la métrica ML más
fácil de calcular.**

> [!tip] El bottleneck real
> *"The bottleneck for AutoResearch in production is not only the agent's
> capability. It is the quality of the metric and the maturity of the MLOps
> system around it."*

## Cómo testear si tu métrica está lista

Mirá tus **últimas 20 A/B tests**: ploteá la métrica offline contra el delta de
negocio online. Si las victorias offline rara vez se traducen en victorias
online, **tu métrica no está lista** para optimización agéntica.

## References

- Fuente: [Harness Engineering: Evolution of MLOps to AutoMLOps](https://substack.com/home/post/p-198249792) — Jam with AI, 2026-05-21

## Related

- [[AutoMLOps]]
- [[Three-Tier Evaluation Pipeline]]
- [[Reranking Metrics]]
- [[MLOps Maturity Stages]]
