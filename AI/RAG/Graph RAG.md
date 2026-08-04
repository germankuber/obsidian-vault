---
title: Graph RAG
source: (Microsoft Research, From Local to Global: A Graph RAG Approach to Query-Focused Summarization, arXiv 2404.16130)
author: —
created: 2026-08-04
tags:
  - ai
  - rag
  - type/concept
  - status/done
aliases:
  - Graph RAG
  - GraphRAG
  - graph rag
  - RAG sobre grafos
updated: 2026-08-04
---

# Graph RAG

> [!note] Definición
> **Graph RAG** es recuperación aumentada que usa un grafo de entidades y relaciones —en vez de, o además de, un índice vectorial de chunks— como estructura de recuperación. La diferencia operativa: el RAG vectorial recupera **fragmentos parecidos a la pregunta**; Graph RAG recupera **entidades conectadas y los caminos entre ellas**.

## El problema que ataca

El RAG vectorial clásico tiene un límite estructural: cada chunk se recupera de forma **independiente**, por similitud con la consulta. Eso funciona bien cuando la respuesta vive en un pasaje, y falla en tres casos concretos:

- **Preguntas multi-hop** — *"¿qué proveedores de nuestros competidores también nos abastecen?"* exige encadenar hechos que están en documentos distintos. Ningún chunk individual se parece a la pregunta.
- **Preguntas globales o agregativas** — *"¿cuáles son los temas principales de este corpus?"* no tiene respuesta en ningún fragmento: exige una vista del conjunto.
- **Contexto disperso** — una entidad mencionada en cincuenta documentos, cada mención parcial. Top-k recupera cinco chunks y pierde el resto.

> [!note] La causa raíz es la misma en los tres: **la similitud vectorial no captura estructura**. Dos hechos pueden ser críticos para la respuesta y no parecerse ni a la pregunta ni entre sí. La relación entre ellos es lo que importa, y un embedding no la representa.

## Cómo funciona

### Fase de indexado

1. **Extracción de entidades y relaciones** del corpus — hoy típicamente con un LLM, opcionalmente restringido por un esquema. Ver [[Ontología y LLMs]].
2. **Resolución de entidades** — unificar menciones de la misma cosa. Es el paso difícil y el que define la calidad del grafo.
3. **Construcción del grafo** — nodos (entidades), aristas (relaciones), con procedencia hacia el texto de origen.
4. **Detección de comunidades** — clustering jerárquico (Leiden es el habitual) para agrupar subgrafos densamente conectados.
5. **Resúmenes por comunidad** — un resumen generado por cada cluster, en varios niveles de la jerarquía. Es lo que permite responder preguntas globales.

### Fase de consulta

Dos modos, según el tipo de pregunta:

| Modo | Para qué | Cómo recupera |
|---|---|---|
| **Local** | Preguntas sobre entidades concretas | Identifica entidades en la consulta, expande a su vecindad en el grafo, trae los chunks asociados |
| **Global** | Preguntas temáticas o agregativas | Recorre los resúmenes de comunidad, mapea respuestas parciales y las reduce a una respuesta final |

> [!tip] El modo global es lo que ninguna arquitectura vectorial puede replicar: la respuesta se construye desde resúmenes jerárquicos del corpus entero, no desde k fragmentos. El costo es que la indexación es cara — se paga una pasada de LLM sobre todo el corpus.

## El costo real

> [!warning] **Graph RAG es significativamente más caro que RAG vectorial en indexación.** Extraer entidades y relaciones con LLM sobre un corpus grande, resolver entidades y generar resúmenes de comunidad tiene un costo que crece con el tamaño del corpus, y hay que rehacerlo —al menos parcialmente— cuando el corpus cambia. No es un reemplazo por defecto del RAG vectorial: es la herramienta para los casos donde la estructura importa.

Criterio de decisión honesto:

- **Corpus chico o preguntas de un solo hop** → RAG vectorial. Graph RAG es sobre-ingeniería.
- **Preguntas multi-hop, entidades recurrentes, necesidad de agregación global** → Graph RAG rinde.
- **Dominio con vocabulario acordado y necesidad de trazabilidad** → Graph RAG sobre un grafo con esquema formal ([[Knowledge graph]] + [[ontología]]).

## Con esquema o sin esquema

La decisión de diseño más consecuente, y donde se conecta con la ingeniería de ontologías.

**Sin esquema (schema-free)** — el LLM extrae las relaciones que encuentra, con predicados en lenguaje libre.

> [!warning] Produce miles de predicados casi-sinónimos: `trabajaEn`, `empleadoDe`, `formaParteDe`, `esMiembroDelEquipoDe`. El grafo se ve impresionante en una visualización y **ninguna consulta estructurada lo puede aprovechar**, porque no hay vocabulario estable contra el cual consultar. Es el modo de falla más frecuente de los pipelines automáticos.

**Con esquema (ontology-grounded)** — se define de antemano el vocabulario de clases y relaciones, y la extracción se restringe a él.

> [!tip] Es más trabajo al principio y el grafo resultante es consultable, validable con [[SHACL]] y trazable. El esquema no tiene que ser una ontología OWL completa: un vocabulario controlado de 20 clases y 30 relaciones —el escalón bajo del [[espectro semántico]]— ya resuelve el problema de los predicados dispersos.

> [!note] Acá es donde las [[competency questions]] rinden en un contexto de AI: **las preguntas que el sistema debe responder definen qué entidades y relaciones hace falta extraer**. Sin ellas, la extracción intenta capturar todo y el grafo se llena de relaciones que ninguna consulta usa.

## Híbrido: el patrón que domina en producción

En la práctica los sistemas serios combinan las tres señales en vez de elegir una:

- **Vectorial** — similitud semántica, tolerante a paráfrasis. Ver [[Hybrid Search]].
- **Léxica (BM25)** — coincidencia exacta de términos, nombres propios, códigos.
- **Estructural (grafo)** — vecindad, caminos, comunidades.

Y se fusionan los resultados, típicamente con reciprocal rank fusion o reranking. Ver [[Reranking]].

> [!tip] Un patrón que funciona bien y suele pasarse por alto: usar el grafo para **expandir la consulta**, no para recuperar directamente. Se identifican las entidades de la pregunta, se expande a entidades relacionadas en el grafo, y esos términos enriquecen la búsqueda vectorial y léxica. Aprovecha la estructura sin rehacer todo el pipeline de recuperación.

## Evaluación

Los evaluadores de RAG clásico no capturan lo que Graph RAG aporta:

- **Corrección multi-hop** — ¿la respuesta encadenó los hechos correctos, o acertó por casualidad? Requiere casos de prueba con la cadena esperada explícita.
- **Calidad del grafo, separada de la respuesta** — precisión de extracción de entidades y relaciones, y tasa de errores de resolución de entidades. Un grafo malo produce respuestas malas por razones que la evaluación end-to-end no distingue.
- **Trazabilidad** — ¿la respuesta señala las tripletas y documentos que la sustentan?

> [!warning] El error de evaluación más común: medir Graph RAG contra RAG vectorial con un dataset de preguntas de un solo hop. En ese caso Graph RAG no gana nada y cuesta más — pero el dataset no está midiendo aquello para lo que existe. Ver [[RAG Evaluation]] y [[Common Eval Mistakes]].

## Conexión en el vault

- [[Knowledge graph]] — el sustrato: entidades, relaciones y el debate RDF vs property graph.
- [[ontología]] · [[espectro semántico]] — el esquema que evita el grafo de predicados dispersos, y cuánto formalismo hace falta realmente.
- [[Ontología y LLMs]] — extracción asistida, grounding y sus límites.
- [[competency questions]] — qué extraer y qué dejar afuera; también el eval set.
- [[Hybrid Search]] · [[Reranking]] — las otras señales de recuperación con las que se combina.
- [[RAG Evaluation]] · [[Ground Truth]] — cómo medir si realmente aporta.
- [[Enterprise RAG Assistant]] — el caso de aplicación del vault.

## References

- Edge, D. et al. (2024) — *From Local to Global: A Graph RAG Approach to Query-Focused Summarization*. [arXiv:2404.16130](https://arxiv.org/abs/2404.16130). Microsoft Research.
- Hogan, A. et al. (2021) — *Knowledge Graphs*. ACM Computing Surveys 54(4).
- *LLM-empowered Knowledge Graph Construction: A Survey* (2025) — [arXiv:2510.20345](https://arxiv.org/abs/2510.20345).

## Related

- [[Knowledge graph]]
- [[ontología]]
- [[Ontología y LLMs]]
- [[Hybrid Search]]
- [[Reranking]]
- [[RAG Evaluation]]
- [[competency questions]]
