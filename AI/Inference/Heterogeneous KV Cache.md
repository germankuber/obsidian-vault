---
title: Heterogeneous KV Cache
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/inference
  - type/concept
  - status/permanent
aliases:
  - Heterogeneous KV Cache
reading:
  total_words: 534
  read_words: 0
  pct: 0
  last_read: ""
---

# Heterogeneous KV Cache

> [!note] Definición
> **Heterogeneous KV Cache** es la **Era 3** de la evolución del [[KV Cache]]: el reconocimiento de que la premisa de [[PagedAttention]] — bloques de tamaño fijo, todos del mismo tipo — deja de valer cuando distintas partes del modelo necesitan formas de caché distintas. Un allocator homogéneo aplicado a un modelo heterogéneo vuelve a desperdiciar memoria.

## Por qué se rompe la homogeneidad

[[PagedAttention]] asume que todos los bloques de caché son intercambiables: mismo tamaño, mismo tipo de tensor. Eso funciona mientras todas las capas atienden igual. Pero cuando conviven capas con presupuestos de memoria muy distintos, el allocator homogéneo o **sobre-reserva** para las capas baratas o **sub-dimensiona** para las caras. El desperdicio medido lo deja claro:

- **79.6%** de memoria desperdiciada en Llama 3.2 11B Vision.
- **25%** en Gemma-2.
- **56.25%** en Ministral.

## Fuentes de heterogeneidad

Son cinco orígenes por los que el caché deja de ser uniforme:

- **VLMs (modelos visión-lenguaje)**: los tokens de imagen y de texto producen caché con perfiles distintos, así que un solo tamaño de bloque no encaja bien para ambos.
- **KV cuantizado**: si una parte del caché se guarda en menor precisión (ver [[Quantization]]), esos bloques pesan distinto que los de precisión completa — bloques de distinto tamaño efectivo conviviendo.
- **Sliding Window Attention (SWA)**: las capas SWA solo retienen una **ventana acotada** de tokens, mientras las capas full retienen toda la secuencia. Su footprint de caché es chico y constante, no creciente.
- **Mamba / SSM**: las capas recurrentes mantienen un **estado recurrente de tamaño fijo**, no un KV cache que crece — literalmente no es un caché de K/V, así que no encaja en el mismo allocator.
- **Modelos híbridos**: combinan tipos de capa en un mismo modelo y son el caso más agudo:
  - **Gemma 2/3** — SWA + Full attention.
  - **Jamba / Bamba** — Mamba + Full attention.
  - **Llama 4** — Local Chunked + Full attention.

Una sexta fuente viene del **speculative decoding**, donde el modelo draft y el principal manejan caché de forma distinta — ver [[Técnicas de Inferencia#Speculative Decoding (acelera decode)]].

## Cuándo NO alcanza / trade-offs

> [!warning]
> El arreglo parcial de esta era es usar **allocators aislados** — un pool de memoria separado por cada tipo de caché. Funciona, pero es **frágil**: cada pool se dimensiona por separado, no comparten memoria libre entre sí, y agregar un tipo nuevo de capa rompe el balance. Esa fragilidad es lo que motiva la Era 5 → [[Unified Hybrid KV Cache]], donde la meta es que todos los tipos compartan un mismo allocator unificado.

> [!question] 🎯 ¿Por qué un estado recurrente de Mamba no encaja en el allocator de PagedAttention?
> Porque PagedAttention pagina un KV cache que **crece con la secuencia**; el estado de una capa Mamba/SSM es **de tamaño fijo** y no es un par K/V. Meterlo en bloques pensados para K/V creciente desperdicia memoria y conceptualmente no corresponde — es otra forma de heterogeneidad.

## References

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo

## Related

- [[KV Cache]]
- [[PagedAttention]]
- [[Unified Hybrid KV Cache]]
- [[Quantization]]
- [[Técnicas de Inferencia]]
