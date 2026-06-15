---
title: Distributed KV Cache
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/inference
  - type/concept
  - status/permanent
aliases:
  - Distributed KV Cache
  - NVIDIA Dynamo
  - Mooncake
reading:
  total_words: 566
  read_words: 0
  pct: 0
  last_read: ""
---

# Distributed KV Cache

> [!note] Definición
> **Distributed KV Cache** es la **Era 4** de la evolución del [[KV Cache]]: cuando el caché no entra en una sola GPU, o conviene moverlo y reusarlo entre nodos, se lo **distribuye**. Tres patrones complementarios — separar fases, enrutar según el caché, y escalonar el caché por niveles de memoria.

## Patrón (a) Disaggregation (DistServe)

Parte de recordar que prefill y decode tienen **perfiles opuestos** (ver [[Prefill-Decode Split]]): prefill es compute-bound, decode es memory-bandwidth-bound. *Por eso* tiene sentido correrlos en **pools de GPU separados**, cada uno afinado para su fase, enviando la [[KV Cache]] del pool de prefill al de decode.

- El caché viaja por interconexiones rápidas: **InfiniBand / RoCE / NIXL**.
- Ganancia medida en **DistServe**: **4.48× más requests** o **10.2× más ajuste de SLO** (margen para cumplir el objetivo de latencia).
- **Variante para VLMs — encoder disaggregation**: separar el encoder de imagen como su propio pool da **2-2.5× goodput** en modelos visión-lenguaje.
- Es la cara distribuida de lo que la nota de técnicas describe como disaggregation — ver [[Técnicas de Inferencia#Disaggregation (Prefill-Decode Disaggregation)]].

## Patrón (b) KV-aware Routing (NVIDIA Dynamo)

En vez de mover el caché, **enrutar la petición hacia donde el caché ya está**.

- Mantiene una **vista global del caché**: qué nodo tiene qué prefijos cacheados.
- Cuando llega una petición, la rutea al **nodo que ya tiene el prefijo**, evitando recomputar el prefill y evitando mover el caché por red.
- Es el enfoque de **NVIDIA Dynamo**: el routing se vuelve consciente del contenido del caché, no solo del balanceo de carga.

## Patrón (c) KV jerárquico (Mooncake / Kimi)

Trata la memoria como una jerarquía y deja que el caché **rebalse (spillover)** a niveles más lentos pero más grandes:

- **HBM → DRAM (~10× más lenta) → SSD (~100× más lenta)**: el caché caliente vive en HBM y el frío baja a DRAM y SSD en vez de descartarse.
- El truco para que la latencia no mate la idea es el **solapamiento**: cargar el caché de la capa N desde el nivel lento mientras se computa la capa N-1. En un modelo de **80 capas** eso da **80 oportunidades** de esconder la latencia de carga detrás del cómputo.
- Es el enfoque de **Mooncake**: **+525% throughput** en contexto largo, y en **Kimi**, **+75% requests**.

> [!question] 🎯 ¿Por qué el caché jerárquico es esencial a 100K+ tokens pero inútil a 2K?
> Porque el término `n_tokens` de la fórmula del [[KV Cache]] hace que el caché crezca lineal con el contexto: a 100K+ no entra en HBM y rebalsar a DRAM/SSD (con solapamiento) es la única forma de servirlo, mientras que a 2K el caché entero entra holgado en HBM y agregar niveles solo suma latencia sin resolver ningún problema.

## Cuándo NO alcanza / trade-offs

> [!warning]
> Límite honesto de esta era: **muchas optimizaciones de la Era 3 todavía no son compatibles con el caché distribuido**. Mover, enrutar y escalonar caché heterogéneo (SWA + full, Mamba, cuantizado) no está resuelto de forma combinada — y esa incompatibilidad es justo lo que ataca la Era 5 → [[Unified Hybrid KV Cache]].

## References

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo
- [DistServe](https://arxiv.org/abs/2401.09670) (disaggregation)

## Related

- [[KV Cache]]
- [[Prefill-Decode Split]]
- [[Heterogeneous KV Cache]]
- [[Unified Hybrid KV Cache]]
- [[Técnicas de Inferencia]]
