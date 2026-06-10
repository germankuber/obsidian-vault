---
title: Hallucinations
source: (conocimiento general — sin artículo fuente; nota escrita por Claude, verificar antes de citar)
author: —
created: 2026-06-10
tags:
  - ai/fundamentals
aliases:
  - Hallucinations
  - Hallucination
  - Alucinaciones
  - Confabulación
---

# Hallucinations

> [!warning] Nota sin fuente externa
> Escrita desde conocimiento general, no destilada de un artículo. Sin `source:` citable. Verificá los detalles antes de citarlos; enriquecela al importar un artículo real sobre el tema.

> [!note] Definición
> Salida de un LLM que es **incorrecta o no respaldada por la evidencia, pero suena plausible**: fluida, confiada, bien estructurada y a la vez falsa. El problema no es que el modelo se equivoque, sino que lo hace **sin señal de incertidumbre**.

## Por qué pasan

- El modelo genera el **texto más probable**, no el más verdadero. Cuando el contexto es insuficiente, **extrapola** (rellena el hueco con algo verosímil).
- No verifica automáticamente contra una fuente: la fluidez no es evidencia, la confianza no es correctitud.

## Cómo se manifiestan

- Citas inventadas, hechos falsos, cálculos errados, APIs que no existen, resúmenes que tergiversan, interpretación incorrecta de resultados de herramientas.

## Cómo se reducen (no se eliminan)

- **[[Grounding]]** — atar la respuesta a evidencia real (documentos, tools, cálculos).
- **Retrieval** ([[RAG]]) — darle al modelo los documentos correctos.
- **Validación / [[Evals]]** — medir y detectar.
- **Verificación adversaria** — un segundo sistema que audita al primero, como el [[Generator-Evaluator Pattern]] / [[Grounded Eval Harness]].
- **Revisión humana** en los casos críticos.

> [!warning] No se eliminan del todo
> Ninguna técnica las elimina por completo; reducen su frecuencia y las hacen detectables.

## References

- (Sin fuente externa — completar al importar un artículo sobre hallucinations.)

## Related

- [[Grounding]]
- [[Evals]]
- [[Grounded Eval Harness]]
- [[RAG]]
