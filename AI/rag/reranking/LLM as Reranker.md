---
title: LLM as Reranker
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/retrieval
  - type/pattern
  - status/permanent
aliases:
  - LLM as Reranker
  - llm-as-reranker
  - LLM Reranker
---

# LLM as Reranker

> [!note] Definition
> Usar un LLM como reranker: en vez de "matchear" textos por patrones (como [[Bi-Encoder]] o [[Cross-Encoder]]), el LLM **razona** sobre la relevancia — multi-hop, información temporal, consistencia factual. Es un "intérprete narrativo", no un comparador.

## Setup

Dada la query `q` y los candidatos `{d_1..d_K}`, el reranker evalúa cada uno:

```
s_i = LLM(Prompt(q, d_i))
```

Un prompt simple:

```
Question: {q}
Document: {d_i}
Is this document relevant to the question? Answer:
```

El modelo emite una **probabilidad** (LLMs open) o una **etiqueta discreta** ("Yes/No", con LLMs cerrados como Gemini/ChatGPT) convertible en score. Hace al reranking un problema de **prompt-engineering + agregación de scoring**. En la práctica se usa:

- **Logits de completion likelihood**.
- **Probabilidades softmax-normalizadas**.
- **Pairwise preference scoring** (comparar dos docs y ver cuál se prefiere).

> [!tip] Potencia con optimización de prompts
> Combinar esto con un setup de optimización de prompts + feedback como **DSPy** lo vuelve realmente poderoso.

## Desafíos

> [!warning] Por qué no es gratis
> - **Latencia**: evaluar cada documento con un LLM es lento.
> - **Costo**: el billing por tokens vuelve caro rerankear K grande.
> - **Alucinación**: el LLM puede inventar cadenas de razonamiento para justificar una etiqueta equivocada.
> - **Deriva de evaluación**: la noción de "relevancia" es subjetiva y varía con el prompt o el modelo.

## Dónde encaja

Es el extremo más capaz e interpretativo del espectro de [[Reranking]], y el más caro. Conviene reservarlo para K chico o casos donde el razonamiento profundo (temporalidad, contradicción, multi-hop) justifica el costo.

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Cross-Encoder]]
