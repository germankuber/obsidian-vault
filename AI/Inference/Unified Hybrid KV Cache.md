---
title: Unified Hybrid KV Cache
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/inference
  - type/concept
  - status/permanent
aliases:
  - Unified Hybrid KV Cache
  - Jenga
reading:
  total_words: 512
  read_words: 0
  pct: 0
  last_read: ""
---

# Unified Hybrid KV Cache

> [!note] Definición
> **Unified Hybrid KV Cache** es la **Era 5** — el cierre de la evolución del [[KV Cache]]. Su tesis: en vez de un allocator distinto y aislado por cada tipo de caché (lo frágil que dejó la Era 3), **toda optimización debería COMPONER** dentro de un mismo allocator unificado. Que VLM, spec decoding, disaggregation, capas híbridas y prefix caching puedan convivir, no elegirse entre sí.

## El problema que cierra

La Era 3 ([[Heterogeneous KV Cache]]) resolvió la heterogeneidad con **allocators aislados**, pero eso es frágil: pools separados que no comparten memoria libre y se rompen al agregar un tipo nuevo. La Era 4 ([[Distributed KV Cache]]) distribuyó el caché pero quedó **incompatible** con buena parte de esas optimizaciones heterogéneas. La Era 5 busca el allocator único donde todo coexiste.

## Enfoques → Jenga (LCM huge-page allocator)

- **Jenga** unifica tipos de caché de distinto tamaño usando un **huge page** cuyo tamaño es el **mínimo común múltiplo** de los tamaños de bloque que conviven. Ejemplo: `LCM(256, 384) = 768` bytes como página grande, que después se **subdivide** para servir bloques de cualquiera de los tipos.
- Usa un **allocator de dos niveles**: el huge page arriba, la subdivisión por tipo abajo.
- Resultado: **+79.6% de utilización de GPU**, y **4.92× pico / 1.80× promedio** frente a vanilla vLLM.

## Enfoques → SGLang (CUDA Virtual Memory)

- **SGLang** ataca lo mismo con las **APIs de CUDA Virtual Memory (VMM)**: remapea la memoria **física** por debajo, exponiendo pools **virtualmente contiguos pero físicamente dispersos**.
- Esto permite **ratios afinados en runtime**: por ejemplo, repartir dinámicamente la memoria entre el pool de estado de **Mamba** y el pool de **KV** según lo que el modelo híbrido necesite en ese momento, sin reservas fijas.

## Composability (la frontera)

> [!note]
> El santo grial es la **composabilidad total**: que en un mismo deployment convivan **VLM + speculative decoding + disaggregation + capas híbridas + prefix caching unificado**, todos compartiendo el mismo allocator sin que uno excluya al otro.

- Es la frontera abierta del tema, no un problema cerrado.
- Aparece en el **roadmap de SGLang para Q1-2026** como objetivo explícito.
- Es el **beat de cierre** de la columna vertebral de las eras: del 80% de VRAM desperdiciada en la Era 0 a un allocator único donde cada optimización compone con las demás.

> [!question] 🎯 ¿Por qué "allocators aislados" (Era 3) no son lo mismo que un caché unificado (Era 5)?
> Porque los aislados dan a cada tipo su propio pool fijo que no comparte memoria libre ni compone con los demás: sirven un tipo de heterogeneidad a la vez. El unificado (Jenga vía LCM huge-page, SGLang vía CUDA VMM) pone todos los tipos sobre un mismo allocator, así que la memoria se reparte dinámicamente y las optimizaciones se combinan en lugar de excluirse.

## References

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo

## Related

- [[KV Cache]]
- [[Heterogeneous KV Cache]]
- [[Distributed KV Cache]]
- [[PagedAttention]]
- [[Técnicas de Inferencia]]
