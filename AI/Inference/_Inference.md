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
updated: 2026-06-15
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

## 🧬 Evolución del manejo de memoria del KV cache

El hub de esta línea es **[[KV Cache]]** (que ahora carga la columna vertebral de las 6 eras, alias *KV Cache Evolution*). Cada era tiene su nota:
- **[[PagedAttention]]** — Era 2: paginar el caché como memoria virtual de un SO (fragmentación 60-80%→<4%, utilización 20-38%→96%+, 2-4× throughput). Reúne el motor **vLLM**.
- **[[Heterogeneous KV Cache]]** — Era 3: por qué los bloques fijos se rompen (VLMs, KV cuantizado, SWA, Mamba/SSM, híbridos como Gemma 2/3, Jamba, Llama 4); el allocator homogéneo desperdicia hasta 79.6%.
- **[[Distributed KV Cache]]** — Era 4: disaggregation (DistServe 4.48×/10.2×), KV-aware routing (NVIDIA Dynamo) y caché jerárquico HBM→DRAM→SSD (Mooncake/Kimi).
- **[[Unified Hybrid KV Cache]]** — Era 5: que todo componga (Jenga, SGLang CUDA VMM); la frontera abierta.

## 🧠 Conceptos base

Viven en `AI/AI Fundamentals/` porque son transversales:
- **[[Tokens]]** — la unidad atómica que procesa el modelo.
- **[[KV Cache]]** — valores de atención (prefill→decode) y **hub de la evolución del manejo de memoria** del serving de LLMs (6 eras).
- **[[Quantization]]** — comprimir la precisión de los pesos para acelerar ambas fases.

## 🌱 Semillas

Conceptos enlazados que todavía no tienen nota propia:
- [[Mixture-of-Experts]] · [[NVLink]] · [[DeepSeek V3]] · [[Cursor Composer 2.0]] · [[Hugging Face Hub]] · [[Prompt Caching]]

## 🔍 Todas las notas de este sub-dominio (auto)

```dataview
LIST
FROM #ai/inference
WHERE file.name != this.file.name AND !contains(file.tags, "type/moc")
SORT file.path ASC
```
