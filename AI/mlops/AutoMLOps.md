---
title: AutoMLOps
source: https://substack.com/home/post/p-198249792
author: Jam with AI
created: 2026-06-10
tags:
  - ai/mlops/automlops
aliases:
  - AutoMLOps
  - automlops
  - Harness Engineering for MLOps
  - ML Harness Engineering
---

# AutoMLOps

> [!note] Definición
> Una **capa nueva sobre MLOps**: un loop de experimentación **end-to-end** donde el agente corre **desatendido pero solo dentro de límites diseñados por humanos**. Es **harness engineering para MLOps**. (Término no oficial — un modelo mental útil.)

## Qué es y qué no es

- **El objetivo NO es reemplazar ML engineers.** Es dejar que los agentes exploren con seguridad ideas chicas de implementación/optimización mientras los humanos diseñan el problema, la métrica, los gates de evaluación y los constraints de producción.
- **NO es** "agregá un agente a tu workflow ML". Los equipos ya usan agentes para literature review, code generation, feature engineering, PR review y experiment planning. *"That is useful, but it is only the beginning."* AutoMLOps es lo que viene **después**.
- Es la **Stage 3** de las [[MLOps Maturity Stages]], que nace del [[AutoResearch]] de Karpathy llevado a producción.

> [!quote] El insight central
> *"The agent itself is the easy part. The system underneath decides whether the overnight run produces something useful or just an expensive overfit."* El bottleneck no es la capacidad del agente, sino **la calidad de la métrica y la madurez del MLOps** alrededor → ver [[Offline vs Business Metrics]].

## Las cinco piezas nuevas

1. **Frozen evaluation contract** — la versión de producción del evaluador inmutable de AutoResearch. Define la métrica escalar offline, splits held-out, leakage checks, fairness gates y definiciones de business proxy. Lo posee un senior IC o un grupo chico de reviewers de confianza. El agente lo **lee, no lo modifica**.
2. **Formal agent sandbox** — define qué puede cambiar el agente (arquitectura, feature transforms dentro de un set aprobado, hiperparámetros, partes del training recipe). **No** debe editar el evaluador, la lógica de split, las definiciones de métrica de negocio ni los safety checks. Mismo concepto que [[Sandboxing]] en [[Agent Harness|harnesses de agentes]].
3. **Compute guardrails** — AutoResearch sobre un modelo chico es barato; sobre un ranker grande o un dataset de billones de filas se vuelve caro rápido. Necesita **wall-clock cap, per-experiment timeout, total budget y kill switch**.
4. **Agent experiment log** — si el agente corre 50 experimentos de noche, el reviewer de la mañana no debería inspeccionar 50 git diffs crudos. El log resume qué cambió, qué métrica mejoró, qué sub-métricas ML se movieron, qué business proxy se movió y si pasó algo sospechoso.
5. **[[Three-Tier Evaluation Pipeline|Pipeline de evaluación de 3 niveles]]** — separa la experimentación offline barata (tier 1, agente) de la validación cara (tier 2 human-gated, tier 3 A/B test). **La pieza más importante.**

## Qué queda humano

- **Problem formulation** — qué debe mejorar el modelo y qué trade-offs son aceptables. "Aumentar revenue sin dañar fairness" no se le tira a un agente como instrucción vaga; un humano lo traduce a métricas, constraints y gates.
- **Composite metric design** — una de las skills de mayor leverage en producción ML: entender cómo se comportan las métricas offline, cómo se mueven las de negocio, dónde están sesgados los logs históricos.
- **Architecture decisions** — el agente puede implementar DCNv2, un two-tower, un feature interaction block; el humano decide si la dirección tiene sentido para producto, latencia, mantenibilidad y roadmap.
- **Decisiones contraintuitivas**: quitar un feature con leakage baja el offline pero mejora generalización; un fairness constraint baja AUC pero da seguridad; quitar un legacy path frena esta semana pero reduce deuda técnica. **Requieren juicio, no metric-optimization.**

> [!tip] El rol del ML engineer no desaparece, se corre
> De *"probá este cambio de modelo a mano"* a *"definí el sandbox, la métrica, los gates y los constraints para que un agente explore seguro."* Premia a quien entiende sistemas, evaluación, métricas de producto y failure modes — no solo modelos.

## Las cuatro inversiones (Stage 2 → Stage 3)

No es un rewrite, es un set de inversiones enfocadas (útiles **aunque nunca uses un agente**):

1. **Diseñar una métrica offline compuesta** y medir cuánto correlaciona con el KPI de negocio (test de las últimas 20 A/B tests).
2. **Apretar la superficie inmutable** — mover evaluation harness, leakage tests, fairness gates y definiciones de métrica a un área protegida del repo; requerir senior review; enforcearlo en CI.
3. **Agregar compute guardrails** — wall-clock cap, timeout, budget, kill switch.
4. **Hacer el pipeline determinístico** — mismo código + datos + seed → mismo resultado (o suficientemente cercano para que las mejoras chicas signifiquen algo). Si cada run tiene mucho ruido, **el agente optimizará ruido tan fácil como señal**.

## El stack en una frase

> [!quote]
> *"The agent itself is cheap. Pointing Claude Code or Codex at a well-prepared repo is not the hard part. The hard part is preparing the repo, the metric, the evaluator, the sandbox, and the business-aligned scoring system."* Para nuevos ML engineers: **no aprendas solo a usar agentes; aprendé a construir los sistemas que los hacen confiables.**

## References

- Fuente: [Harness Engineering: Evolution of MLOps to AutoMLOps](https://substack.com/home/post/p-198249792) — Jam with AI, 2026-05-21

## Related

- [[AutoResearch]]
- [[MLOps Maturity Stages]]
- [[Offline vs Business Metrics]]
- [[Three-Tier Evaluation Pipeline]]
- [[Agent Harness]]
- [[Sandboxing]]
- [[Generator-Evaluator Pattern]]
