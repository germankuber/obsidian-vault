---
title: Inference
created: 2026-06-15
tags:
  - ai/inference
  - type/moc
  - status/permanent
aliases:
  - Inference
  - Inference Engineering
  - AI Inference
---

# Inference

> [!note] Cómo usar esta nota
> Índice (MOC) del sub-dominio **Inference** (`AI/Inference/`): cómo servir LLMs en producción de forma rápida, barata y confiable. Todo el sub-dominio gira alrededor de **una sola idea estructural** — la inferencia son dos fases con cuellos de botella opuestos. Abrí esta nota, no la carpeta.

## 🚀 Empezá por acá

- **[[Inference Engineering]]** — el **hub** del tema: qué es la disciplina, por qué se volvió especialidad (2024+), self-hosting vs APIs cerradas, y la decisión build vs buy.
- **[[Prefill-Decode Split]]** — **el marco**: las dos operaciones en secuencia con cuellos de botella opuestos. Si entendés esto, entendés todo lo demás.

## ⚙️ Las dos fases

El núcleo conceptual vive en **[[Prefill-Decode Split]]**:
- **Prefill** — compute-bound, métrica **TTFT** (time to first token), una vez por petición.
- **Decode** — memory-bandwidth-bound, métrica **TPS** (tokens per second), una vez por token.

## 🧰 Las seis técnicas

- **[[Técnicas de Inferencia]]** — Batching, Prefix Caching, Quantization, Speculative Decoding, Parallelism (tensor/expert) y Disaggregation, cada una con su trade-off y su lugar en el split (acelerar prefill / acelerar decode / rebalancear).

## 🧠 Conceptos base

Viven en `AI/AI Fundamentals/` porque son transversales:
- **[[Tokens]]** — la unidad atómica que procesa el modelo.
- **[[KV Cache]]** — valores de atención que se producen en prefill y se reusan en decode.
- **[[Quantization]]** — comprimir la precisión de los pesos para acelerar ambas fases.

## 🌱 Semillas

Conceptos enlazados que todavía no tienen nota propia:
- [[Mixture-of-Experts]] · [[NVLink]] · [[DeepSeek V3]] · [[Cursor Composer 2.0]] · [[Hugging Face Hub]] · [[PagedAttention]] · [[DistServe]] · [[Prompt Caching]]

## 🔍 Todas las notas de este sub-dominio (auto)

```dataview
LIST
FROM #ai/inference
WHERE file.name != this.file.name AND !contains(file.tags, "type/moc")
SORT file.path ASC
```
