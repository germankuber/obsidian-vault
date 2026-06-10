---
title: Multimodal Reranking
source: https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Siddhant Rai
created: 2026-06-10
tags:
  - ai/rag/reranking
  - type/pattern
  - status/permanent
aliases:
  - Multimodal Reranking
  - multimodal-reranking
  - Reranking Multimodal
  - Re-ranking Across Modalities
---

# Multimodal Reranking

> [!note] Definición
> Extender el reranking **más allá del texto** —imágenes, video, datos tabulares, audio— entendiendo la naturaleza de cada tipo de dato y alineando sus señales con la query. El patrón **low-level (recall) → high-level (precisión) → fusión** se repite en cada modalidad, espejando el pipeline textual.

## Imágenes / Visual

- **Low-level**: descriptores **SIFT, SURF, ORB, histogramas de color**. Conceptualmente son **el [[BM25]] de las imágenes**: no entienden contenido semántico, capturan patrones estructurales. Derived reranking reordena por cantidad/calidad de keypoint matches.
- **High-level / semántico**: redes profundas **ResNet, ViT**, embeddings tipo **CLIP o ColPali** (extensión de [[ColBERT]]). Actúan como bi-encoders (query e imágenes embebidas por separado, similitud coseno). Existen análogos cross-encoder (visual cross-attention).
- **Hybrid**: combinar ORB (rápido, recall) + embeddings (precisión) y rankear con [[Reciprocal Rank Fusion|RRF]].
- **Issues**: low-level frágil a ruido/rotación/iluminación; high-level puede confundir imágenes visualmente similares pero semánticamente irrelevantes (gato sobre piano vs piano sin gato).

## Video — eje temporal

- **Low-level**: ORB/SIFT por frame, vectores de movimiento, histogramas por frame.
- **High-level**: encoders de video **TimeSformer, VideoCLIP** → embeddings espacio-temporales (estilo bi-encoder o cross-encoder con atención temporal).
- **Dynamic retrieval**: recall rápido con keyframes ralos → rerank Top-K secuencias con embeddings/atención.
- **Issues**: features por frame ruidosas; high-level pierde detalle temporal fino (un clip importante de 2 s); cross-encoder de video **extremadamente caro**.

## Tabular / Series temporales

- **Low-level**: matches exactos, keywords, proximidad numérica (**BM25 análogo para tablas**). Ej: *"tablas con sales > 1M en 2024"* → matchea columnas ("sales", "year") + filtro numérico. Series: peak detection, medias móviles, autocorrelación, coeficientes de Fourier.
- **High-level**: embeddings row/column (**TabNet, TaBERT**, transformers temporales). Cross-encoder análogo modela la **tabla/serie entera** (**TimesNet, TimeSformer, NBEATS**).
- **Issues**: matches exactos frágiles; embeddings pierden precisión numérica; cross-encoder pesado para tablas grandes.

## Multi-modal (combinando modalidades)

Una query/candidato puede abarcar varias modalidades a la vez (ej: *"clips de gatos tocando piano"* requiere texto + visual + audio). Estrategias combinadas:

1. **Hybrid retrieval por modalidad** → recuperación low-level separada por modalidad, luego fusión.
2. **Derived reranking sobre embeddings** → high-level por modalidad alimenta un reranker.
3. **Cross-modal attention / análogos de ColBERT** → cross-atención entre modalidades (texto atendiendo a frames y audio).
4. **LLM multi-modal** → integra captions, transcripts y metadata para juicios semánticos heterogéneos.

> [!warning] Issues multi-modal
> - Alineación entre modalidades difícil (desalineación temporal, granularidad inconsistente).
> - La computación escala rápido → **precomputar embeddings por modalidad es crucial**.
> - Evaluación difícil: las métricas estándar rara vez capturan relevancia cross-modal.

## References

- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Reranking]]
- [[ColBERT]]
- [[BM25]]
- [[Reciprocal Rank Fusion]]
- [[Agent Reranking]]
