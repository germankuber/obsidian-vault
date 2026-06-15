---
title: KV Cache
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/fundamentals
  - type/concept
  - status/permanent
aliases:
  - KV Cache
  - Key-Value Cache
reading:
  total_words: 184
  read_words: 0
  pct: 0
  last_read: ""
---

# KV Cache

> [!note] Definición
> La **KV Cache** (key-value cache) almacena los **valores intermedios de atención** de los tokens ya procesados, para **reusarlos al generar los tokens siguientes** en vez de recomputarlos. Es lo que hace tratable la generación autoregresiva.

## Su rol a lo largo de la inferencia

- **Se produce** durante el [[Prefill-Decode Split|prefill]] (junto con el primer token).
- **Se consume** durante el **decode**: cada token nuevo lee la KV cache de todos los anteriores.
- **Se envía por red** en la disaggregation (del motor de prefill al de decode). Ver [[Técnicas de Inferencia]].
- **Se reusa** en el **prefix caching** cuando los prompts comparten un prefijo idéntico. Ver [[Técnicas de Inferencia]].
- Es **sensible a la precisión**: en [[Quantization]], la KV cache es uno de los componentes que más se degrada al comprimir, así que suele mantenerse en precisión más alta.

## Related

- [[Prefill-Decode Split]]
- [[Técnicas de Inferencia]]
- [[Quantization]]
- [[Tokens]]
- [[Inference Engineering]]

## Referencias

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo
- [PagedAttention](https://arxiv.org/abs/2309.06180) — gestión eficiente de la KV cache
