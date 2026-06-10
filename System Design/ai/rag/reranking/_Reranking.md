---
title: Reranking (subtema)
created: 2026-06-08
tags:
  - ai/rag/retrieval
  - moc
aliases:
  - Reranking MOC
  - Reranking Index
---

# Reranking — subtema

> [!note] Cómo usar esta nota
> Índice del subtema *reranking* dentro de [[RAG]]. El hub conceptual es [[Reranking]]; esta nota es el mapa de la carpeta. Empezá arriba y bajá: de los fundamentos a las técnicas, las modalidades y los agentes.

## 🚀 Empezá por acá

- [[Reranking]] — **el hub.** Por qué similitud ≠ relevancia, y el panorama general.

## 🧠 Fundamentos

- [[Classical Reranking]] — las raíces estadísticas: posterior correction, logistic reranking, Platt scaling, RankNet/LambdaRank.
- [[Information Theory of Reranking]] — recuperación = compresión con pérdida; el reranking minimiza entropía / maximiza información mutua. La **capa-λ**.
- [[Reranking Metrics]] — cómo se mide: NDCG, MRR, Precision@k, Recall@k.
- [[Node vs Document Reranking]] — rerankear chunks (nodes) vs documentos enteros.

## 🎯 Las dos familias

- [[Derived vs Hybrid Reranking]] — derived (precisión, reordena el Top-K) vs hybrid (recall, fusiona múltiples retrievers). El eje central del tema.

## 🔬 El espectro de métodos semánticos (rápido/grosero → preciso/caro)

- [[Bi-Encoder]] — rápido, embeddings precomputables, pierde matices.
- [[ColBERT]] — *late interaction* (MaxSim): el punto medio.
- [[Cross-Encoder]] — atención cruzada, máxima precisión, máximo costo.
- [[LLM as Reranker]] — el LLM razona la relevancia; lo más capaz y caro.

## 🔤 Recuperación léxica y fusión

- [[BM25]] — ranking *sparse* por frecuencia de términos.
- [[Reciprocal Rank Fusion]] — RRF: consenso entre varios retrievers.

## 🌐 Más allá del texto

- [[Multimodal Reranking]] — imágenes (SIFT/CLIP/ColPali), video (TimeSformer/VideoCLIP), tabular (TabNet/TaBERT), multi-modal.
- [[Agent Reranking]] — cuando los candidatos son salidas de agentes: pointwise, pairwise, listwise/consensus.

## Conexión

- Subtema de [[RAG]] · se aplica sobre los resultados de [[Hybrid Search]].
