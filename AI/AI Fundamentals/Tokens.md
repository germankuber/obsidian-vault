---
title: Tokens
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/fundamentals
  - type/concept
  - status/permanent
aliases:
  - Token
  - Tokens
reading:
  total_words: 142
  read_words: 0
  pct: 0
  last_read: ""
---

# Tokens

> [!note] Definición
> Un **token** es la **unidad atómica** que procesa un LLM: aproximadamente **una palabra o un fragmento de palabra**. El modelo no ve caracteres ni palabras enteras, sino tokens.

## Cómo funciona

- Una palabra puede ser **un solo token** o **partirse en varios** según el tokenizador.
- Ejemplos: **"inference"** podría ser **1 token**; **"engineering"** podría partirse en **2 tokens**.
- Las métricas de inferencia se cuentan en esta unidad: por ejemplo **tokens/segundo** (TPS) mide la velocidad de generación en tokens, no en palabras. Ver [[Prefill-Decode Split]].

## Related

- [[Context Window]] — la cantidad de tokens que el modelo puede tener en cuenta a la vez (concepto relacionado, no es lo mismo que un token).
- [[Prefill-Decode Split]]
- [[KV Cache]]
- [[Inference Engineering]]

## Referencias

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo
