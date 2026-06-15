---
title: Técnicas de Inferencia
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
tags:
  - ai/inference
  - type/pattern
  - status/permanent
aliases:
  - Inference Techniques
  - Técnicas de Inferencia
  - Batching
  - Prefix Caching
  - Speculative Decoding
  - Tensor Parallelism
  - Expert Parallelism
  - Disaggregation
  - Prefill-Decode Disaggregation
reading:
  total_words: 788
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-06-15
---

# Técnicas de Inferencia

> [!note] Definición
> Catálogo de las técnicas con las que se optimiza la inferencia de un LLM. Todas se entienden a través del [[Prefill-Decode Split]]: cada una **acelera el prefill**, **acelera el decode**, o **rebalancea** el trabajo entre ambas fases. Cada técnica trae su propio trade-off — esa es la parte que importa.

Las tres familias:
- **Rebalancear / throughput**: Batching.
- **Acelerar prefill**: Prefix Caching, Quantization (transversal).
- **Acelerar decode**: Speculative Decoding, Quantization (transversal).
- **Escalar / repartir hardware**: Parallelism, Disaggregation.

## Batching

> [!note]
> Intercala (interleaves) varias peticiones **token a token sobre una misma GPU**, procesándolas juntas en cada paso.

![[batching-interleaving.png|487]]
*Batching*

- **Efecto**: **↑ throughput** (más peticiones servidas por unidad de tiempo y hardware).
- Es **la tensión primaria** de toda la inferencia: throughput vs latencia por usuario.

> [!warning] Trade-off
> El costo es **mayor latencia por usuario individual**: cada petición espera a las otras del batch.
> - **Chat / interactivo** → priorizá **baja latencia** (batches chicos).
> - **Pipelines batch / offline** → priorizá **alto throughput** (batches grandes).

## Prefix Caching (acelera prefill)

> [!note]
> Reusa la [[KV Cache]] cuando varios prompts **comparten un segmento inicial idéntico** (típicamente un system prompt largo y común a todas las peticiones).

![[prefix-caching-shared-prefix.png|434]]
*Prefix Caching*

- Evita recomputar el prefill de la parte compartida → prefill más rápido y barato.
- Los proveedores **cobran menos por los input tokens cacheados**.

> [!warning] Trade-off / catch
> El ahorro vale **solo hasta el primer token que NO coincide**. Si el **primer token ya difiere**, el ahorro es **CERO**.
> - **Implicación de diseño**: poné el **contenido compartido al PRINCIPIO** del prompt y la **entrada variable del usuario al FINAL**, para maximizar el prefijo común cacheable.

Distinto de [[Cache-Aside]]: ese es un patrón de caching de aplicación/datos, otra capa — no confundir ni mergear.

## Quantization

Ver [[Quantization]] — comprimir la precisión de los pesos acelera **prefill y decode** a la vez. Es una técnica **transversal** (cruza inferencia, fundamentals y memoria), por eso su tratamiento completo vive en su propia nota y no se duplica acá.

## Speculative Decoding (acelera decode)

> [!note]
> Explota una asimetría: **generar** tokens es caro, pero **verificar** tokens ya propuestos es barato (analogía del **Sudoku**: resolverlo cuesta, chequear una solución es trivial).

![[speculative-decoding-draft-verify.png|425]]
*Speculative Decoding*

- Un **modelo DRAFT pequeño** predice los próximos varios tokens.
- El **modelo principal los verifica todos en UN solo forward pass**, **acepta** los que coinciden y **rechaza** el resto.
- Resultado: **varios tokens por cada forward pass del modelo principal** → mejora **TPS**, deja **TTFT sin cambios**.

> [!warning] Trade-off
> Funciona mejor con **batch chico**. Con **batches grandes** la GPU ya está saturada, así que la especulación deja de aportar y se **apaga dinámicamente (turns OFF)**.

## Parallelism

> [!note]
> Repartir el modelo entre varias GPUs. Dos variantes según cómo se parte.

![[parallelism-tensor-vs-expert.png|374]]
*Tensor Parallelism vs Expert Parallelism*

- **Tensor parallelism**: parte **cada capa** entre GPUs. Requiere **interconexión de alto ancho de banda** ([[NVLink]]). Es el **default para modelos densos muy grandes**.
- **Expert parallelism**: para modelos **[[Mixture-of-Experts]]** (solo un **subconjunto de parámetros activo por token**). Reparte los **expertos entre GPUs** y **enruta los tokens** al experto que toca. Tiene **menor overhead de comunicación** y sirve **multi-nodo**.

> [!tip] En producción
> **Tensor parallelism DENTRO de un nodo** (interconexión rápida intra-nodo) y **expert parallelism ENTRE nodos** (comunicación más barata, escala multi-nodo).

## Disaggregation (Prefill-Decode Disaggregation)

> [!note]
> Es la técnica más **arquitectónica**: corre el **prefill en un conjunto de GPUs** y el **decode en otro**, enviando la [[KV Cache]] por red entre ambos. Rebalancea el split a nivel de hardware.

![[prefill-decode-disaggregation.png|534]]
*Prefill-Decode Disaggregation*

- Cada conjunto de hardware se **afina (tunes) para su fase** y **escala de forma independiente**.
- **Flujo de 3 pasos**:
  1. El **motor de prefill** produce el **primer token + la KV cache**.
  2. La **cache viaja por una interconexión rápida** hasta el **motor de decode**.
  3. **Conditional disaggregation**: las peticiones **cortas o cacheadas se saltan el handoff** y corren **solo en el motor de decode**, lo que rinde mejor contra tráfico real mixto que un setup mezclado (mixed).

> [!warning] Trade-off
> Es la opción más compleja de operar (mover la KV cache por red, dos pools de hardware), pero se vuelve **casi obligatoria a gran escala**.

## Referencias

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo
- [DistServe](https://arxiv.org/abs/2401.09670) (disaggregation)
- [PagedAttention](https://arxiv.org/abs/2309.06180) (gestión de KV cache)
- [Speculative Decoding](https://arxiv.org/abs/2211.17192)
- [Megatron-LM](https://arxiv.org/abs/1909.08053) (tensor parallelism)
- [Sparsely-Gated MoE](https://arxiv.org/abs/1701.06538) (expert parallelism)
- [NVIDIA NVLink](https://www.nvidia.com/en-us/data-center/nvlink/)
- [NVIDIA TensorRT](https://developer.nvidia.com/tensorrt)
- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) (prefix caching)

## Related

- [[Prefill-Decode Split]]
- [[Inference Engineering]]
- [[KV Cache]]
- [[Quantization]]
- [[Mixture-of-Experts]]
- [[NVLink]]
- [[Cache-Aside]]
