---
title: Three-Tier Evaluation Pipeline
source: https://substack.com/home/post/p-198249792
author: Jam with AI
created: 2026-06-10
tags:
  - ai/mlops/evaluation
  - type/pattern
  - status/permanent
aliases:
  - Three-Tier Evaluation Pipeline
  - three-tier-evaluation
  - Pipeline de Evaluación de Tres Niveles
  - Tiered Evaluation
---

# Three-Tier Evaluation Pipeline

> [!note] Definición
> La pieza **más importante** de [[AutoMLOps]]: separar la experimentación offline barata de la validación de producción cara, en **tres niveles** con distinto nivel de evidencia. El agente opera solo en tier 1; humanos revisan tier 2; los usuarios reales recién entran en tier 3. **El objetivo NO es que el agente shippee modelos directo a usuarios.**

## Los tres tiers

- **Tier 1 — composite scalar del agente**
  - Métrica offline **rápida y determinística** sobre un slice held-out fijo.
  - Debe correr en **minutos, no horas**. El **ratchet** ([[AutoResearch]]) usa este score para mantener/revertir.
  - No es validation loss pura, sino **blend de calidad ML + business proxy** (ver [[Offline vs Business Metrics]]). Ej: search = nDCG + conversion estimada; fraude = AUC solo si fraud loss prevented no cae bajo un piso; recommender = ranking quality + diversity / engagement de largo plazo.
  - *"It is not supposed to be perfect. It is supposed to be cheap, stable, and directionally useful."*

- **Tier 2 — business proxy con human gate**
  - Los mejores candidatos del run nocturno se evalúan con más cuidado: **replay window más largo**, **counterfactual evaluation**, **off-policy estimators** (IPS o doubly-robust), o **shadow scoring** contra tráfico vivo.
  - Señal de negocio más fuerte que en tier 1. Un senior IC decide qué promover.
  - **Acá se cazan muchos de los overfits generados por el agente.**

- **Tier 3 — KPI real de negocio**
  - El **A/B test**. Lento, caro, limitado. No podés mandar cada cambio chico del agente a un A/B test → **solo los sobrevivientes de tier 2** llegan acá.

## El desafío de ingeniería

> [!tip] Que el tier 1 valga la pena
> El reto clave es hacer el **tier 1 lo bastante útil** para que las 8 horas del agente no se desperdicien. Test práctico: mirá tus **últimas 20 A/B tests**, ploteá métrica offline vs delta online de negocio. Si las wins offline rara vez se traducen en wins online, tu métrica **no está lista** para optimización agéntica.
>
> *"Before you automate experimentation, make sure your evaluation actually points in the right direction."*

## References

- Fuente: [Harness Engineering: Evolution of MLOps to AutoMLOps](https://substack.com/home/post/p-198249792) — Jam with AI, 2026-05-21

## Related

- [[AutoMLOps]]
- [[AutoResearch]]
- [[Offline vs Business Metrics]]
- [[MLOps Maturity Stages]]
