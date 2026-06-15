---
title: Inference Engineering
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/inference
  - type/concept
  - status/permanent
aliases:
  - Inference
  - AI Inference
  - Inference Engineering
reading:
  total_words: 597
  read_words: 0
  pct: 0
  last_read: ""
---

# Inference Engineering

> [!note] Definición
> **Inference engineering** es la disciplina de servir modelos de lenguaje en producción de forma rápida, barata y confiable. No se trata de entrenar el modelo, sino de ejecutarlo (correr el forward pass) para responder peticiones reales, optimizando latencia, throughput y costo bajo carga.

## La tesis estructural

La idea central que organiza todo el campo: la inferencia de un LLM no es **una** operación, sino **dos operaciones distintas en secuencia con cuellos de botella opuestos** — *prefill* y *decode*.

- Ver [[Prefill-Decode Split]] para el marco completo: una fase es **compute-bound** (prefill, métrica TTFT) y la otra **memory-bandwidth-bound** (decode, métrica TPS).
- Casi toda técnica de inferencia se entiende como "acelerar el prefill", "acelerar el decode" o "rebalancear" entre ambos.
- Esta separación es la razón de que la ingeniería de inferencia sea una especialidad: optimizar una fase no optimiza la otra.

## Por qué se volvió especialidad (2024+)

- A partir de **2024** dejó de ser una tarea secundaria del equipo de ML y pasó a ser una **especialidad amplia** con rol propio.
- El driver: a escala, el costo y la latencia de servir el modelo dominan la economía del producto, no el costo de entrenarlo.
- El campo madura alrededor del split prefill/decode (ver [[Prefill-Decode Split]]) y de un catálogo de técnicas (ver [[Técnicas de Inferencia]]).

## Self-hosting vs APIs cerradas

Hospedar uno mismo modelos abiertos se volvió viable y, en muchos casos, ventajoso frente a las APIs cerradas de proveedores:

- **Confiabilidad**: self-hosting permite apuntar a **cuatro nueves (99,99%)** de disponibilidad, frente a los **dos nueves (99%)** típicos que ofrecen los SLAs de las APIs cerradas.
- **Costo**: caída de costos de inferencia de **~80%** al servir modelos abiertos propios en vez de pagar por token a un proveedor.
- **Latencia**: es **ajustable (tunable)** — al controlar el stack podés optimizar TTFT y TPS para tu caso, cosa imposible detrás de una API cerrada.
- **Disponibilidad de modelos abiertos**: el [[Hugging Face Hub|Hugging Face]] pasó a alojar **más de 2 millones de modelos**, un crecimiento de **~25x**, lo que hizo del open-weights una opción real.
- **Modelos abiertos competitivos**: [[DeepSeek V3]] es un ejemplo de modelo abierto de frontera que hace viable el self-hosting serio.
- **Producto real sobre modelo propio**: [[Cursor Composer 2.0|Composer 2.0]] de Cursor corre sobre infraestructura de inferencia propia.
- **Sectores regulados**: en **healthcare** (y otros con requisitos de privacidad/compliance), self-hosting es muchas veces la única opción aceptable.

## Build vs Buy

La decisión de construir tu propio stack de inferencia o comprar una API cerrada depende de la madurez del producto:

- **Temprano → comprar**: al arrancar, usar **APIs off-the-shelf** (closed APIs) es lo correcto — menos ingeniería, time-to-market rápido.
- **Tres señales de que conviene cambiar a self-hosting**:
  1. El **costo de la API** se volvió una **línea de gasto grande** (a big spend line).
  2. La **latencia** que necesitás **supera** lo que dan las APIs cerradas (latency outpaces closed APIs).
  3. Tus requisitos de **confiabilidad superan los SLAs del proveedor** (reliability exceeds vendor SLAs).
- **Caso ilustrativo**: el **autocomplete sub-segundo** de Cursor **era el producto** en sí — esa latencia no se podía conseguir con una API cerrada, así que self-hosting dejó de ser opcional.

## Referencias

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo
- [DeepSeek-V3](https://arxiv.org/abs/2412.19437)
- Cursor Composer: [Composer](https://cursor.com/blog/composer) · [Composer 2](https://cursor.com/blog/composer-2)
- [Hugging Face Hub](https://huggingface.co/docs/hub/index)

## Related

- [[Prefill-Decode Split]]
- [[Técnicas de Inferencia]]
- [[Tokens]]
- [[KV Cache]]
- [[Quantization]]
- [[DeepSeek V3]]
- [[Cursor Composer 2.0|Composer 2.0]]
- [[Hugging Face Hub|Hugging Face]]
