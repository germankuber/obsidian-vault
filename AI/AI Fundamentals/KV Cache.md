---
title: KV Cache
source: https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering
author: ByteByteGo
created: 2026-06-15
updated: 2026-06-15
tags:
  - ai/fundamentals
  - type/concept
  - status/permanent
aliases:
  - KV Cache
  - Key-Value Cache
  - KV Cache Evolution
reading:
  total_words: 1401
  read_words: 184
  pct: 13
  last_read: 2026-06-15
---

# KV Cache

> [!note] Definición
> La **KV Cache** (key-value cache) almacena los **valores intermedios de atención** (los tensores Key y Value) de los tokens ya procesados, para **reusarlos al generar los tokens siguientes** en vez de recomputarlos. Es lo que hace tratable la generación autoregresiva — pero también es el **cuello de botella de memoria** que define toda la evolución del serving de LLMs.

> [!note] Tesis operativa (esta nota es el hub de la evolución)
> El KV cache convierte el cómputo de la generación de O(N³) a O(N), pero a cambio de una memoria que crece lineal con el contexto. Esa memoria es el recurso escaso, y **la historia entera del serving de LLMs es la historia de cómo administrarla mejor** — seis eras, cada una resolviendo la limitación que dejó la anterior.

## Su rol a lo largo de la inferencia

- **Se produce** durante el [[Prefill-Decode Split|prefill]] (junto con el primer token).
- **Se consume** durante el **decode**: cada token nuevo lee la KV cache de todos los anteriores.
- **Se envía por red** en la disaggregation (del motor de prefill al de decode). Ver [[Técnicas de Inferencia]].
- **Se reusa** en el **prefix caching** cuando los prompts comparten un prefijo idéntico. Ver [[Técnicas de Inferencia]].
- Es **sensible a la precisión**: en [[Quantization]], la KV cache es uno de los componentes que más se degrada al comprimir, así que suele mantenerse en precisión más alta.

## Por qué existe: autoregresión sin caché → O(N³)

Para entender por qué el caché es ineludible hay que ver el costo de NO tenerlo. La generación es **autoregresiva**: el modelo produce un token por vez, y cada token nuevo **atiende a todos los anteriores** (atención = Q·Kᵀ·V sobre toda la secuencia). Sin caché, en cada paso recomputás los tensores K y V de *toda* la secuencia desde cero.

- Recalcular K y V de la secuencia completa cuesta O(N²) por paso (cada token contra todos).
- Y lo hacés una vez por cada uno de los N tokens generados → **O(N³)** acumulado sobre la respuesta entera.
- *Por eso* el caché no es una optimización opcional sino la condición que hace tratable la autoregresión: guardás los K y V ya calculados y solo computás los del token nuevo, bajando el costo por paso a O(N) y el total a O(N²). El precio de ese cambio es nuevo: **memoria que hay que guardar y que crece con el contexto** — y administrar esa memoria es todo lo que sigue.

## Fórmula de tamaño del KV cache

El peso del caché se lee factor por factor (de ahí sale por qué cada técnica de optimización ataca uno u otro término):

```
mem_kv = 2 · n_layers · n_tokens · num_kv_heads · head_dim · bytes_per_element
```

- **2** → guardás *dos* tensores, K y V. Por eso el caché pesa el doble de lo que intuirías mirando solo las keys.
- **n_layers** → una copia de K/V por cada capa del modelo. En modelos de 80 capas esto domina, y *por eso* el caché jerárquico (era 4) puede solapar la carga de la capa N con el cómputo de la capa N-1 capa por capa.
- **n_tokens** → crece lineal con el contexto. Este es el término que vuelve crítico al caché en contextos de 100K+ tokens y trivial en prompts de 2K.
- **num_kv_heads** → cuántas cabezas tienen K/V propios. *Por eso* **GQA (Grouped-Query Attention)** reduce el caché: agrupa varias cabezas de query para que compartan un mismo K/V, bajando este factor sin tocar casi el cómputo de atención. *(no es del artículo: relación con Grouped-Query Attention).*
- **head_dim** → la dimensión de cada cabeza.
- **bytes_per_element** → 2 en fp16, 1 en int8. *Por eso* cuantizar el caché lo parte casi a la mitad — ver [[Quantization]], donde la KV cache aparece como uno de los componentes sensibles que conviene comprimir con cuidado.

**El ejemplo que prueba la tesis.** Meté los números de **Llama-3-70B en FP16** y mirá cómo la memoria te aplasta:

```
2 × 80 capas × 8 KV heads × 128 head_dim × 2 bytes = 327,680 bytes ≈ 320 KB por token
```

- **8K tokens × 320 KB = 2.56 GB por una sola request.**
- **32 requests concurrentes × 2.56 GB = 81.9 GB** → **no entra en una A100 de 80 GB**… y eso es ANTES de cargar los pesos del modelo.
- *Por eso* el verdadero límite del serving es **la memoria del caché, no el cómputo** — y de ese número nace toda la cadena de eras que sigue.

## Evolución del manejo de memoria (6 eras)

Esta es la columna vertebral del tema: cada era resuelve la limitación que dejó abierta la anterior. Leída de corrido, es la historia de cómo se pasó de desperdiciar el 80% de la VRAM a componer todas las optimizaciones a la vez.

**Era 0 — Reserva contigua naive.** Se reservaba por anticipado un bloque **contiguo** de memoria del tamaño del contexto máximo, por petición. Como casi ninguna petición llega al máximo, **solo el 20-38% de la memoria reservada contenía tokens reales**: el resto era fragmentación reservada "por las dudas". *Por eso* la era siguiente tuvo que paginar la memoria.

**Era 1 — Reducir el caché en origen.** Antes de administrar mejor la memoria, atacar la fórmula: **GQA** baja `num_kv_heads`, **MQA** lo lleva al extremo (una sola cabeza K/V), y **cuantizar el caché** baja `bytes_per_element`. Reduce el problema pero no resuelve la fragmentación. Ver [[Quantization]].

**Era 2 — Paginar la memoria: PagedAttention.** Aplica la idea de la **memoria virtual de un SO** al caché: en vez de un bloque contiguo gigante, parte el caché en bloques chicos (~16 tokens) repartidos por la HBM y asignados por demanda. La fragmentación cae de 60-80% a <4% y la utilización sube a 96%+. Ver [[PagedAttention]].

**Era 3 — El caché deja de ser homogéneo: Heterogeneous KV Cache.** PagedAttention asume bloques de tamaño fijo y homogéneo. Esa premisa se rompe con VLMs, caché cuantizado, sliding-window attention, estados recurrentes de Mamba y modelos híbridos que mezclan tipos de capa. El allocator homogéneo vuelve a desperdiciar memoria (hasta 79.6%). Ver [[Heterogeneous KV Cache]].

**Era 4 — Sacar el caché de una sola GPU: Distributed KV Cache.** Cuando el caché no entra o conviene moverlo, se distribuye: disaggregation (pools separados de prefill y decode), routing consciente del caché, y caché jerárquico HBM→DRAM→SSD. Ver [[Distributed KV Cache]].

**Era 5 — Que todo componga: Unified Hybrid KV Cache.** La frontera actual: cada optimización (VLM + spec decoding + disaggregation + híbrido + prefix caching) debería poder combinarse con las demás en un mismo allocator unificado. Ver [[Unified Hybrid KV Cache]].

## Analogía (mnemónica)

El caché es como la **memoria virtual de un SO**. La era naive reservaba un segmento contiguo gigante por las dudas; **PagedAttention** son las *páginas* que rompen esa reserva en bloques chicos no contiguos. (La analogía rompe en que acá no hay swap a disco dentro de una GPU — aunque el caché jerárquico de la era 4 reintroduce justamente un "swap" hacia DRAM y SSD.)

## ¿Qué aplica a mi caso?

- Contexto corto, una sola GPU, latencia crítica → caché simple + paginado alcanza ([[PagedAttention]]).
- VRAM al límite → cuantizá el caché (ver [[Quantization]]) y/o usá GQA.
- Modelo híbrido (SWA + full, Mamba + full) o VLM → necesitás un allocator heterogéneo ([[Heterogeneous KV Cache]]).
- Contexto largo (100K+) o muchos usuarios concurrentes → caché distribuido/jerárquico ([[Distributed KV Cache]]).
- Querés combinar varias de las anteriores → la frontera es [[Unified Hybrid KV Cache]].

> [!question] 🎯 ¿Por qué GQA reduce el KV cache pero casi no cambia el cómputo de atención?
> Porque ataca el factor `num_kv_heads` de la fórmula de memoria: menos cabezas con K/V propios = menos tensores que guardar. Cada token sigue atendiendo a toda la secuencia, así que el cómputo por paso apenas se mueve; lo que baja es la memoria del caché.

## Related

- [[Prefill-Decode Split]]
- [[Técnicas de Inferencia]]
- [[Quantization]]
- [[Tokens]]
- [[Inference Engineering]]
- [[PagedAttention]]
- [[Heterogeneous KV Cache]]
- [[Distributed KV Cache]]
- [[Unified Hybrid KV Cache]]

## Referencias

- Fuente: [A Guide to AI Inference Engineering](https://blog.bytebytego.com/p/a-guide-to-ai-inference-engineering) — ByteByteGo
- [PagedAttention / vLLM](https://arxiv.org/abs/2309.06180) — gestión eficiente de la KV cache
- Relación con GQA/MQA y cuantización del caché: conocimiento general, no del artículo fuente.
