---
title: Information Theory of Reranking
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-10
tags:
  - ai/rag/reranking
aliases:
  - Information Theory of Reranking
  - information-theory-of-reranking
  - Teoría de la Información del Reranking
  - Lambda Layer
---

# Information Theory of Reranking

> [!note] Definición
> La lectura del reranking como una **capa de refinamiento de información**: la
> recuperación es un proceso **con pérdida (lossy)**; el reranking **minimiza la
> entropía condicional** de la selección final dada la query.

## El argumento

- **Recuperación = compresión con pérdida**: el retriever comprime todo el corpus
  a un conjunto candidato chico (p. ej. Top-50) con una métrica de similitud
  gruesa. Eso preserva *recall* pero **inyecta entropía**: muchos de esos
  documentos están débil o espuriamente relacionados.
- **Reranking minimiza la entropía condicional** — después de rerankear, baja la
  incertidumbre sobre cuál documento es relevante dada la query.
- Visto de otro ángulo: **maximiza la información mutua** I(Q;D) entre query y
  documentos seleccionados. H(D∣Q) captura el "ruido irrelevante" de documentos
  que no aportan información alineada con la intención.
- Rescorear con interacciones semánticas más profundas (cross-encoders) **aumenta
  I(Q;D)**, tensando el acoplamiento semántico query–evidencia.

> [!example] Analogía de señal
> La recuperación es **broadcasting** (emitir una señal ancha para juntar
> candidatos); el reranking es **decoding** (reconstruir la señal para recuperar
> los bits más significativos). El retriever da *bandwidth*; el reranker restaura
> *clarity*.

## La capa-λ (lambda layer)

> [!tip] Espacio de decisión flexible
> El reranking actúa como una **capa-λ** entre recuperación y generación: un
> espacio donde inyectar sesgos específicos del caso de uso:
> - **Recency bias** — empujar documentos recientes.
> - **Credibility bias** — priorizar fuentes autoritativas.
> - **Persona/tono** — alinear si el contexto de la query lo demanda.

Pasar de recuperación **alta-entropía** (diversa pero difusa) a orden
**baja-entropía** (enfocado y preciso) es lo que conecta la *búsqueda orientada a
recall* con el *razonamiento orientado a precisión*.

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Reranking Metrics]]
- [[Derived vs Hybrid Reranking]]
- [[Classical Reranking]]
