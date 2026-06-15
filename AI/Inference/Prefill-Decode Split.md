---
title: Prefill-Decode Split
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/inference
  - type/concept
  - status/permanent
aliases:
  - Prefill
  - Decode
  - TTFT
  - TPS
  - Time To First Token
  - Tokens Per Second
reading:
  total_words: 364
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-06-15
---

# Prefill-Decode Split

> [!note] Definición
> Una petición a un LLM se procesa en **dos operaciones en secuencia con cuellos de botella opuestos**: **prefill** (procesa todo el prompt de entrada) y **decode** (genera los tokens de salida uno a uno). Es el marco fundamental de la ingeniería de inferencia: como cada fase tiene un bottleneck distinto, se optimizan por separado y con métricas separadas.

![[prefill-decode-split.png|752]]
*Prefill and Decode*

## Fase 1: Prefill (compute-bound, TTFT)

- Pasa **todo el prompt completo por cada capa del modelo en paralelo** (todos los tokens de entrada a la vez).
- **Salida**: produce el **primer token** de la respuesta + la **[[KV Cache]]** (los valores intermedios de atención que se reusarán después).
- **Cuello de botella**: **compute-bound** — está limitada por la capacidad de cómputo de la GPU (las unidades de matemática), no por la memoria.
- **Métrica**: **TTFT** (*Time To First Token*) — cuánto tarda en aparecer el primer token.
- **Frecuencia**: ocurre **una vez por petición**.

## Fase 2: Decode (memory-bandwidth-bound, TPS)

- Genera **cada token siguiente de a uno**, haciendo un **forward pass completo del modelo por cada token**.
- Cada token nuevo **depende de todos los tokens anteriores** → el proceso es **secuencial**, no se puede paralelizar a lo largo de la secuencia.
- **Cuello de botella**: **memory-bandwidth-bound** — está limitada por el ancho de banda de memoria (mover los pesos y la KV cache), no por el cómputo.
- **Métrica**: **TPS** (*Tokens Per Second*) — cuántos tokens por segundo se generan. Se mide en [[Tokens|tokens]].
- **Frecuencia**: ocurre **una vez por cada token** generado.

## Consecuencia del split

- Como los **cuellos de botella son opuestos**, una técnica que acelera una fase **casi no ayuda** a la otra.
- Por eso los benchmarks reportan **TTFT y TPS por separado** — un solo número de "velocidad" oculta la realidad.
- Las técnicas de inferencia se agrupan justamente en tres familias según qué fase tocan: **acelerar prefill**, **acelerar decode**, o **rebalancear** entre ambas. Ver [[Técnicas de Inferencia]].

## Referencias

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo

## Related

- [[Inference Engineering]]
- [[Técnicas de Inferencia]]
- [[KV Cache]]
- [[Tokens]]
- [[Quantization]]
