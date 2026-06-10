---
title: Evals
source: (conocimiento general — sin artículo fuente; nota escrita por Claude, verificar antes de citar)
author: —
created: 2026-06-10
tags:
  - ai/fundamentals
aliases:
  - Evals
  - Evaluations
  - Evaluaciones
  - LLM Evals
---

# Evals

> [!warning] Nota sin fuente externa
> Escrita desde conocimiento general, no destilada de un artículo. Sin `source:` citable. Verificá los detalles antes de citarlos; enriquecela al importar un artículo real sobre el tema.

> [!note] Definición
> Tests que miden si un modelo o sistema de IA **se comporta como se espera**. No prueban que el sistema sea perfecto; **reducen la ignorancia** sobre cómo se comporta — y eso ya es una mejora seria.

## Qué se mide

- Accuracy, formato, estilo, calidad de retrieval, uso de tools, grounding, seguridad, latencia, éxito en la tarea.

## Qué hace a un buen eval

- Se parece al **uso real**: preguntas reales, edge cases reales, criterios claros, scoring repetible.
- Corre en cada cambio que pueda afectar el comportamiento: nuevo modelo, nuevo prompt, nuevo retriever, nuevo chunking, nueva tool, nuevo guardrail.

## Conexión en el vault

- El [[Grounded Eval Harness]] es un eval **automatizado y adversario**: un segundo LLM evalúa claim por claim al primero. Patrón general: [[Generator-Evaluator Pattern]].
- En MLOps, los evals son las métricas del [[Three-Tier Evaluation Pipeline]] y el problema de [[Offline vs Business Metrics]] (un eval offline puede no predecir el resultado de negocio).
- Para retrieval, las métricas concretas viven en [[Reranking Metrics]] (NDCG, MRR, Precision@k).

## References

- (Sin fuente externa — completar al importar un artículo sobre evals / LLM evaluation.)

## Related

- [[Grounding]]
- [[Hallucinations]]
- [[Grounded Eval Harness]]
- [[Three-Tier Evaluation Pipeline]]
- [[Reranking Metrics]]
