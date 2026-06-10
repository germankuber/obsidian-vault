---
title: Grounding
source: (conocimiento general — sin artículo fuente; nota escrita por Claude, verificar antes de citar)
author: —
created: 2026-06-10
tags:
  - ai/fundamentals
aliases:
  - Grounding
  - Grounded
  - Anclaje en Evidencia
---

# Grounding

> [!warning] Nota sin fuente externa
> Escrita desde conocimiento general, no destilada de un artículo. Sin `source:` citable. Verificá los detalles antes de citarlos; enriquecela al importar un artículo real sobre el tema.

> [!note] Definición
> Conectar la respuesta de un modelo a **evidencia verificable**: documentos, bases de datos, búsqueda web, salidas de herramientas, logs, citas, cálculos. Permite preguntar "¿de dónde salió esto? ¿lo puedo verificar?".

## Para qué sirve

- Habilita **trazabilidad**: cada afirmación debería poder rastrearse a su fuente.
- Es la principal defensa contra [[Hallucinations]]: si la respuesta está atada a evidencia real, hay menos espacio para confabular.

## Más amplio que RAG

- **[[RAG]]** hace grounding vía texto recuperado.
- Las **herramientas** ([[Tool Calling]]) groundean vía datos en vivo.
- Las **bases de datos** vía registros; las **calculadoras** vía cómputo exacto.

> [!warning] El grounding solo ayuda si la evidencia es real
> "Una cita que no respalda la respuesta no es grounding, es decoración." El grounding mejora la trazabilidad, no vuelve perfecto al sistema: la evidencia tiene que ser **real, relevante y visible**.

## Conexión en el vault

- El [[Grounded Eval Harness]] es un caso concreto: un evaluador verifica **claim por claim** que cada afirmación esté respaldada por la fuente.
- En RAG, el grounding se rompe cuando el [[Reranking|retrieval]] trae el chunk equivocado.

## References

- (Sin fuente externa — completar al importar un artículo sobre grounding.)

## Related

- [[Hallucinations]]
- [[Grounded Eval Harness]]
- [[RAG]]
- [[Evals]]
