---
title: Cross-Encoder
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/retrieval
aliases:
  - Cross-Encoder
  - cross-encoder
  - Cross-Encoder Reranker
---

# Cross-Encoder

> [!note] Definition
> Un modelo que recibe la query y el nodo **juntos**, como una sola secuencia, y deja que la atención fluya entre los tokens de ambos. Produce un score de relevancia mucho más fino que un [[Bi-Encoder]], a cambio de un costo computacional mucho mayor.

## Formulación

```
score(q, d) = f_θ([q; d])
```

La query `q` y el documento `d` se concatenan y pasan por el modelo `f_θ`. La atención cruzada entre los tokens de ambos permite razonamiento relacional.

## Fortalezas y debilidades

> [!tip] Fortalezas
> Captura distinciones sutiles: negación, calificadores, dependencias multi-hop. Produce scores de relevancia precisos sobre conjuntos chicos de candidatos.

> [!warning] Debilidades
> Computacionalmente caro; los embeddings **no** se pueden precalcular (cada par query-documento se procesa en vivo). No es viable rerankear millones de nodos directamente — por eso se aplica solo al top-K que dejó la recuperación inicial.

## Dónde encaja

Es el extremo "preciso y caro" del espectro. Se lo usa como segundo paso después de un [[Bi-Encoder]] rápido. [[ColBERT]] ofrece un punto medio que conserva algo de su poder relacional sin pagar la atención cruzada completa.

> [!example] Implementación típica (LlamaIndex)
> ```python from llama_index.retrievers import VectorIndexRetriever from llama_index.postprocessor import SentenceTransformerReranker
>
> retriever = VectorIndexRetriever(index=my_index, similarity_top_k=20)
>
> reranker = SentenceTransformerReranker( model="cross-encoder/ms-marco-MiniLM-L-6-v2" )
>
> nodes = retriever.retrieve("What are self-reflective models?") reranked = reranker.postprocess_nodes(nodes) ```

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[Bi-Encoder]]
- [[ColBERT]]
