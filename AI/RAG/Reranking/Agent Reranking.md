---
title: Agent Reranking
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-10
tags:
  - ai/rag/reranking
  - type/pattern
  - status/permanent
aliases:
  - Agent Reranking
  - agent-reranking
  - Reranking Across Agents
  - Reranking entre Agentes
  - Agent-Level Reranking
---

# Agent Reranking

> [!note] Definición
> Reranking cuando **los candidatos son salidas de sistemas/agentes independientes**, no datos estáticos. La pregunta cambia de *"¿qué documento es relevante?"* a *"¿qué salida de agente responde mejor la query?"* y cómo **combinar perspectivas de varios agentes** para una respuesta más rica.

## Qué es un agente acá

Un agente = **LLM + memoria + tools**. Puede ser un LLM fine-tuneado para un dominio, un retriever optimizado para una modalidad, un módulo de razonamiento con heurísticas/tools, o un pipeline multi-paso (resúmenes, traducciones, recomendaciones).

> [!example] La fila de expertos
> Cada agente es un **node independiente** que genera candidatos con su propio sesgo, conocimiento y scoring. Imaginá una **fila de expertos**, cada uno con una carpeta de color (su respuesta): algunas se solapan, algunas se contradicen, algunas se complementan. El reranking es tu mano ordenando: lo más relevante arriba, fusionando notas complementarias, descartando contradicciones.

## Por qué hace falta

Incluso los agentes sofisticados son imperfectos: dos LLMs pueden dar resúmenes en conflicto; agentes multimodales pueden discrepar sobre qué modalidad importa; un agente de precisión puede perder contexto amplio. Permite:

1. **Seleccionar el mejor candidato** del set de salidas.
2. **Fusionar salidas complementarias** (respuesta textual de alta precisión + resumen visual de alto recall).
3. **Resolver contradicciones**, usando meta-conocimiento sobre la confiabilidad de cada agente.

Conceptualmente es **[[ColBERT]] a nivel agente**: en vez de comparar embeddings de token, comparás **salidas de agentes** (que pueden ser multimodales o cadenas de razonamiento multi-paso).

## Enfoques de scoring

1. **Pointwise** — puntuar cada salida **independientemente** vs la query. `f_θ` puede ser un LLM, un cross-encoder o una función entrenada para estimar correctitud/completitud/utilidad. Simple, paralelizable; bueno con salidas autocontenidas.
2. **Pairwise** — comparar salidas **de a pares**. Útil para consistencia y orden relativo; captura matices que el scoring de una sola salida pierde.
3. **Listwise / Consensus** — considerar **todas las salidas juntas** y producir un ranking global. Puede usar **voting**, **agregación ponderada** o **razonamiento de un LLM sobre múltiples salidas**. Alinea con la filosofía hybrid: exploración amplia + reordenamiento preciso.

## Desafíos

> [!warning]
> - **Bias y confiabilidad**: los agentes varían en precisión; el reranker debe tenerlo en cuenta.
> - **Diversidad vs precisión**: las salidas top pueden ser redundantes → hace falta ranking *diversity-aware*.
> - **Fusión cross-modal multi-agente**: la complejidad crece combinatoriamente.
> - **Métricas**: las de texto tradicionales pueden fallar → a menudo hace falta evaluación humana o métricas task-specific.

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[ColBERT]]
- [[LLM as Reranker]]
- [[Multimodal Reranking]]
- [[Agent Harness]]
