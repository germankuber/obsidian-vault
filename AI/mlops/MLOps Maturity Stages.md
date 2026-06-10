---
title: MLOps Maturity Stages
source: https://substack.com/home/post/p-198249792
author: Jam with AI
created: 2026-06-10
tags:
  - ai/mlops
aliases:
  - MLOps Maturity Stages
  - mlops-maturity-stages
  - Etapas de Madurez de MLOps
  - Notebook to AutoMLOps
---

# MLOps Maturity Stages

> [!note] Definición
> Las **tres etapas** por las que evoluciona un sistema ML, del notebook caótico a la experimentación agéntica. [[AutoMLOps]] **se apoya sobre** la etapa 2; no la reemplaza.

## Stage 1 — Notebook

- Un data scientist entrena en un notebook, exporta `model.pkl`, lo sube a S3, lo comparte con un ingeniero, y alguien intenta cablearlo a producción.
- Funciona para prototipos; duele cuando el modelo le importa al negocio.
- **Problema**: no es **reproducible**. No sabés qué datos produjeron el modelo, qué feature logic, qué versión del notebook, qué seed, qué dependencias, ni qué métrica se confió.

> [!warning] Acá AutoResearch no tiene sentido
> No hay pipeline estable, ni evaluador congelado, ni sandbox limpio, ni métrica confiable para hacer ratchet. *"If the system itself is not reproducible, an agent will only make the chaos faster."*

## Stage 2 — MLOps maduro

Donde están muchos buenos equipos de producción hoy:

- Training pipeline · datos versionados · **experiment tracking** (MLflow, W&B o plataforma interna) · model registry · evaluation checks · deployment automation · monitoring (drift, latencia, calidad).
- Permite reproducir runs, comparar experimentos, deployar más seguro, detectar cambios en producción.

> [!warning] El gap de la etapa 2
> El loop offline se enfoca casi solo en **métricas ML** (nDCG/MRR para rankers, AUC/precision/recall/F1 para clasificadores, RMSE/MAE para regresión, recall@K para retrieval). La métrica de **negocio aparece recién en el A/B test** → los humanos siguen haciendo a mano la traducción ML→negocio. Valioso, pero **no escala**. Acá AutoResearch **empieza** a ser útil, pero falta que el loop offline **se preocupe por outcomes de negocio** ([[Offline vs Business Metrics]]).

## Stage 3 — AutoMLOps

- El cambio grande no es que el agente escriba código, sino que **el sistema está diseñado para que el agente experimente sin tocar las partes que definen correctitud**.
- Detalle completo en [[AutoMLOps]] (las 5 piezas y el [[Three-Tier Evaluation Pipeline]]).

> [!tip] No saltes etapas
> La mayoría **no** debería ir directo a AutoMLOps. Si el pipeline no es reproducible, la evaluación es inestable o el tracking es un desastre → arreglá eso primero. AutoMLOps **se sienta encima** de MLOps.

## References

- Fuente: [Harness Engineering: Evolution of MLOps to AutoMLOps](https://substack.com/home/post/p-198249792) — Jam with AI, 2026-05-21

## Related

- [[AutoMLOps]]
- [[AutoResearch]]
- [[Offline vs Business Metrics]]
- [[Three-Tier Evaluation Pipeline]]
