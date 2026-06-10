---
title: Reranking
source: https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design
sources:
  - https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design
  - https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval
author: Avani Chaskar; Siddhant Rai
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/retrieval
  - type/concept
  - status/permanent
aliases:
  - Reranking
  - reranking
  - Re-ranking
  - Second-Stage Retrieval
---

# Reranking

> [!note] Definition
> Un segundo paso que toma los candidatos de la recuperación inicial (rápida pero imprecisa) y los **reordena** con un criterio más fino y caro, para quedarse con los realmente relevantes. No cambia *qué* se recuperó; cambia *en qué orden* — es "reordenar por significado", no por similitud superficial.

## El problema que resuelve: similitud ≠ relevancia

Los retrievers ordenan por cercanía semántica (distancia de embeddings, overlap TF-IDF), no por alineación con la intención. Ejemplo clásico: a la pregunta *"¿cuáles son las mejores razas de perro para familias?"* un retriever puede traer un documento que compara perros y gatos — semánticamente cercano, pero contextualmente equivocado. El gap se agranda con corpus densos, especializados o cross-domain, justo donde la precisión más importa.

El reranking es el "segundo pensamiento": una capa de refinamiento entre recuperar y generar.

## Formulación general

Cada documento recuperado obtiene un score reordenado:

```
r_i = T(s(d_i, q), d_i, q)
```

- `d_i`: documento recuperado · `q`: query
- `s(d_i, q)`: score de similitud inicial del retriever
- `T`: función de transformación que ajusta el score con contexto más rico
- `r_i`: score reordenado

El diseño de `T` y la métrica de similitud definen la naturaleza del reranker.

> [!note] Lente de teoría de la información
> La recuperación es *lossy*: comprime todo el corpus en un conjunto chico de candidatos con métricas groseras. El reranking minimiza la entropía condicional de la selección final dada la query — después de rerankear, baja la incertidumbre sobre cuál documento es relevante.

## Por qué importa (la capa-λ)

Los sistemas usan cortes Top-K, pero los scores no siempre están calibrados y la relevancia no es lineal: documentos muy relevantes pueden quedar afuera del Top-5 por compresión de embeddings o overlap parcial de keywords. El reranking es una **capa-λ** flexible entre recuperación y generación donde se pueden inyectar sesgos por caso de uso (recency, credibilidad, alineación con persona).

> [!tip]
> El patrón general es "retrieve cheap & wide, then rerank expensive & narrow": recall barato primero, precisión cara después sobre un conjunto chico.

## Métricas de calidad

- **NDCG** (Normalized Discounted Cumulative Gain): premia ubicar lo relevante más arriba.
- **MRR** (Mean Reciprocal Rank): qué tan pronto aparece el primer relevante.
- **Precision@k / Recall@k**: relevancia convencional en el top-k.

## Node vs Document

En RAG la info se guarda en **nodes** (chunks que entran en el contexto del modelo), no documentos enteros. El reranking puede ser:

- **Node-level**: relevancia semántica fina, local y rápida — bueno para respuestas autocontenidas.
- **Document-level**: coherencia de la fuente y contexto global, señales de rango más largo.

## Las dos familias

> [!example] Derived vs Hybrid
> ``` Derived:  Query → Retriever → top-K nodes → Reranker → Generator Hybrid:   Query → varios retrievers → Fusión/Reranker → Generator ```

- **[[Derived vs Hybrid Reranking|Derived Reranking]]** — rerankea *después* de recuperar, solo entre el top-K. Apunta a **precisión**: refina lo ya pescado, minimiza falsos positivos. "No agranda la red; inspecciona lo que ya cayó." Usa métodos deterministas ([[BM25]], TF-IDF) o semánticos ([[Bi-Encoder]], [[Cross-Encoder]]).
- **[[Derived vs Hybrid Reranking|Hybrid Reranking]]** — rerankea *durante* la recuperación, fusionando señales de varios retrievers (sparse + dense + dominio). Apunta a **recall**: expande el pool, reduce falsos negativos. Su método de consenso típico es [[Reciprocal Rank Fusion]].

## El espectro de métodos semánticos

De más rápido/grosero a más preciso/caro:

1. **[[Bi-Encoder]]** — codifica query y nodo por separado, compara por coseno. Rapidísimo y precomputable, pero pierde señales relacionales.
2. **[[ColBERT]]** — *late interaction*: embeddings a nivel token y MaxSim. El punto medio entre velocidad y precisión.
3. **[[Cross-Encoder]]** — query y nodo juntos en un solo modelo, atención cruzada. El más preciso, el más caro, no precomputable.
4. **[[LLM as Reranker]]** — el LLM razona sobre la relevancia (multi-hop, temporalidad, consistencia factual). Máxima capacidad interpretativa, máxima latencia/costo.

## Más allá del texto

El reranking se extiende a [[Multimodal Reranking]] (imágenes, video, datos tabulares) y a [[Agent Reranking]] (cuando los "documentos" son salidas de agentes independientes que hay que comparar, fusionar o desempatar).

## References

- Fuente: [System Design Deep Dive: How to Design an Enterprise RAG Assistant](https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design) — Avani Chaskar, 2026-05-17
- Fuente: [A Primer on Re-Ranking for Retrieval Systems](https://vizuara.substack.com/p/a-primer-on-re-ranking-for-retrieval) — Siddhant Rai, 2025-10-22

## Related

- [[Enterprise RAG Assistant]]
- [[Hybrid Search]]
- [[Bi-Encoder]]
- [[Cross-Encoder]]
- [[ColBERT]]
- [[Reciprocal Rank Fusion]]
- [[LLM as Reranker]]
