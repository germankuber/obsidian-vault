---
title: PagedAttention
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/inference
  - type/concept
  - status/permanent
aliases:
  - PagedAttention
  - vLLM
reading:
  total_words: 561
  read_words: 0
  pct: 0
  last_read: ""
---

# PagedAttention

> [!note] Definición
> **PagedAttention** es la **Era 2** de la evolución del [[KV Cache]]: aplica la idea de la **memoria virtual de un sistema operativo** al caché de atención. En vez de reservar un bloque contiguo gigante por petición, parte el caché en **bloques chicos repartidos por la HBM** y los asigna **por demanda**, eliminando casi toda la fragmentación.

## El problema que resuelve

La era naive del [[KV Cache]] reservaba memoria **contigua** del tamaño del contexto máximo por petición. Como casi ninguna petición llega a ese máximo, la mayor parte de la reserva quedaba vacía: la **fragmentación rondaba el 60-80%** y la **utilización real era de apenas 20-38%**. *Por eso* hacía falta dejar de reservar contiguo y empezar a paginar.

## Cómo funciona (analogía de memoria virtual)

La clave es la misma indirección que usa un SO para que un proceso vea memoria "contigua" que físicamente está dispersa:

- **Block table (logical → physical)**: cada secuencia ve sus tokens como un espacio lógico continuo, pero una tabla de bloques los mapea a páginas físicas dispersas en la HBM.
- **Bloques de ~16 tokens**: el caché se parte en bloques chicos de tamaño fijo, no en un segmento gigante.
- **Demand allocation**: se asigna un bloque nuevo solo cuando la secuencia realmente lo necesita, no por anticipado.
- **HBM dispersa (scattered)**: los bloques de una misma secuencia no tienen que ser contiguos; la tabla resuelve la indirección.
- **COW → prefix sharing**: con copy-on-write, varias secuencias que comparten un prefijo apuntan a los **mismos** bloques físicos hasta que una diverge; recién ahí se copia el bloque. Esto es lo que habilita el prefix caching a nivel de bloque — ver [[Técnicas de Inferencia#Prefix Caching (acelera prefill)]].

## Resultados

- Fragmentación de **60-80% → <4%**.
- Utilización de memoria de **20-38% → 96%+**.
- **2-4× throughput** frente a FasterTransformer / Orca.

## vLLM (motor que la introdujo)

- PagedAttention se introdujo en **vLLM**, el motor de serving que la popularizó; por eso "vLLM" y "PagedAttention" suelen aparecer juntos, aunque uno es el motor y el otro la técnica de gestión de memoria que lo define.
- Sirve de baseline para casi todo lo que vino después: las eras siguientes se describen muchas veces como "qué le falta a vanilla vLLM".

## Cuándo NO alcanza / trade-offs

> [!warning]
> PagedAttention asume **bloques de tamaño fijo y homogéneo** — todas las páginas del mismo tamaño, todas guardando el mismo tipo de K/V. Esa premisa se rompe cuando conviven capas con formas de caché distintas (sliding-window vs full attention, caché cuantizado, estados recurrentes de Mamba). Ahí el allocator de bloques fijos vuelve a desperdiciar memoria, lo que motiva la Era 3 → [[Heterogeneous KV Cache]].

> [!question] 🎯 ¿Por qué los bloques de tamaño fijo de PagedAttention se rompen cuando un modelo mezcla Sliding-Window y Full Attention?
> Porque las capas SWA solo necesitan retener una ventana acotada de tokens mientras las capas Full retienen toda la secuencia: con un único tamaño de bloque homogéneo, o sobre-reservás para las SWA o sub-dimensionás para las Full. La heterogeneidad de necesidades por capa es justo lo que ataca [[Heterogeneous KV Cache]].

## References

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo
- [PagedAttention / vLLM](https://arxiv.org/abs/2309.06180)

## Related

- [[KV Cache]]
- [[Heterogeneous KV Cache]]
- [[Prefill-Decode Split]]
- [[Técnicas de Inferencia]]
