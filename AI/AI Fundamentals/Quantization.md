---
title: Quantization
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/fundamentals
  - type/pattern
  - status/permanent
aliases:
  - Quantization
  - Cuantización
reading:
  total_words: 349
  read_words: 0
  pct: 0
  last_read: ""
---

# Quantization

> [!note] Definición
> **Quantization** (cuantización) es almacenar los **pesos del modelo en menor precisión** para que ocupen menos memoria y se muevan más rápido. Es una técnica **transversal** que acelera tanto el prefill como el decode, a cambio de una posible pérdida de calidad.

## Cómo funciona

- La mayoría de los modelos se **entrenan en punto flotante de 16 bits**.
- La cuantización los **comprime a 8 bits o 4 bits** → **pesos más chicos**, **menos memoria** y **menos movimiento de datos**.
- **Acelera el prefill**: la matemática en baja precisión corre más rápido en las unidades de cómputo (math units) de la GPU.
- **Acelera el decode**: hay **menos presión sobre el ancho de banda de memoria** (los pesos pesan menos). Ver [[Prefill-Decode Split]].
- **Ganancia típica**: **~30 a 50%**.

![[quantization-precision-formats.png]]
*Quantization*

## Cuándo usarlo

> [!tip]
> Cuando necesitás reducir memoria, costo o latencia y podés tolerar algo de pérdida de calidad. El grueso de la ingeniería está en **decidir QUÉ comprimir y CON QUÉ agresividad**, no en aplicarlo de forma uniforme.

## Cuándo NO usarlo / trade-offs

> [!warning]
> El costo es una **posible pérdida de calidad**, y la sensibilidad **no es uniforme** entre componentes. De menos a más sensible:
> 1. **Pesos lineales (linear weights)** — los menos sensibles.
> 2. **Activaciones (activations)**.
> 3. **[[KV Cache]]**.
> 4. **Capas de atención (attention layers)** — las **más sensibles**.
>
> Los errores en las capas de atención **se acumulan / se hacen bola de nieve (snowball)** a lo largo de la secuencia de tokens. Por eso la **mayoría de los despliegues en producción mantienen la atención en PRECISIÓN COMPLETA (full precision)** y solo cuantizan lo menos sensible.

## Patrones relacionados / alternativas

- [[Product Quantization]] — comprime **vectores de embedding** para búsqueda ANN; es una técnica **distinta** (otra capa, otro objetivo), no confundir ni mergear.
- [[KV Cache]] — componente sensible que normalmente se preserva.
- [[Técnicas de Inferencia]] · [[Prefill-Decode Split]] · [[Inference Engineering]]

## Referencias

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo
- [NVIDIA TensorRT](https://developer.nvidia.com/tensorrt)
