---
title: Ontología y LLMs
source: (Survey: LLM-empowered knowledge graph construction, arXiv 2510.20345; Ontology-grounded KG construction under Wikidata schema, arXiv 2412.20942)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - ai
  - type/concept
  - status/done
aliases:
  - Ontología y LLMs
  - ontologías y LLMs
  - LLM ontology
  - ontology grounding
updated: 2026-08-04
---

# Ontología y LLMs

> [!info] El cruce que redefinió el campo
> Hasta 2022 la ingeniería de ontologías era un campo maduro y de nicho. Los LLM cambiaron las dos direcciones de la relación a la vez: **la ontología pasó a ser infraestructura de grounding** para sistemas generativos, y **el LLM resolvió el cuello de botella histórico** de la construcción — la extracción manual de conocimiento. Ninguna de las dos cosas está en la literatura clásica del campo, incluido [[_Ontology Engineering|Kendall & McGuinness (2019)]].

## Las dos direcciones

| | Ontología → LLM | LLM → Ontología |
|---|---|---|
| **Qué aporta** | Grounding factual, trazabilidad, restricción del espacio de salida | Extracción a escala, elicitación asistida, propuesta de esquema |
| **Problema que ataca** | Alucinación, falta de verificabilidad | El cuello de botella de la construcción manual |
| **Madurez** | Producción (Graph RAG, agentes con herramientas) | Investigación activa; requiere verificación humana |

## Dirección 1 — La ontología como grounding

### El problema

Un LLM genera desde memoria paramétrica: no distingue entre lo que sabe y lo que interpola, y no puede citar la fuente de una afirmación. En dominios donde importa la corrección —salud, finanzas, legal, agro— eso es descalificante.

### Qué aporta un esquema formal

- **Trazabilidad** — la respuesta se ancla a tripletas concretas, con procedencia. No es "el modelo dijo", es "esta afirmación viene de esta fuente".
- **Restricción del espacio de salida** — si el modelo debe emitir entidades de un vocabulario cerrado, las entidades inventadas se detectan mecánicamente: el IRI no existe.
- **Verificación posterior** — chequear afirmaciones generadas contra el grafo, y con [[SHACL]] validar que la salida estructurada cumple la forma esperada.
- **Razonamiento que el modelo no hace bien** — transitividad, jerarquías profundas, consistencia. Un razonador computa lo que un LLM aproxima con error.

> [!tip] **La división del trabajo que funciona**: el LLM hace lenguaje —entender la pregunta, extraer del texto, redactar la respuesta—; el grafo y el razonador hacen **verdad y estructura**. Cuando se le pide al LLM que haga de base de conocimiento, falla; cuando se le pide que sea la interfaz de lenguaje sobre una base de conocimiento, funciona.

### El patrón operativo

1. La pregunta en lenguaje natural se traduce a [[SPARQL]] (text-to-SPARQL), con la ontología en contexto como esquema.
2. La consulta se ejecuta contra el [[Knowledge graph]].
3. Los resultados —tripletas con procedencia— se pasan al modelo como contexto.
4. El modelo redacta la respuesta **citando** las tripletas usadas.

> [!warning] El paso 1 es el frágil. Text-to-SPARQL es más difícil que text-to-SQL: el espacio de esquemas es más abierto, los IRIs son largos y arbitrarios, y un property path mal construido devuelve vacío sin error. Mitigaciones que funcionan: exponer al modelo solo el fragmento relevante del esquema, validar la consulta contra el vocabulario antes de ejecutarla, y usar `ASK` para verificar supuestos antes de la consulta principal.

### Por qué las competency questions vuelven a importar

Las [[competency questions]] son, literalmente, la especificación de qué preguntas el sistema debe poder responder. En un sistema con LLM cumplen una cuarta función además de las tres clásicas: **son el eval set**. La misma lista que definió el alcance de la ontología define los casos de prueba del asistente.

> [!note] Es una convergencia notable: el instrumento central de la ingeniería de ontologías de los años 90 resulta ser el eval harness natural de un sistema generativo de 2026. Ver [[Evals]] y [[Ground Truth]].

## Dirección 2 — El LLM como constructor

### Lo que se desbloqueó

La construcción de ontologías siempre estuvo limitada por la elicitación manual: entrevistas con expertos, lectura de documentación, normalización de terminología. Los LLM atacan directamente esa etapa.

- **Extracción de terminología** — primera pasada sobre un corpus para proponer candidatos a concepto, con sus variantes. Reemplaza el trabajo que antes se hacía con TF-IDF y C-value.
- **Detección de sinonimia y homonimia** — agrupar términos que refieren al mismo concepto y detectar el mismo término usado en sentidos distintos. Ver [[03 - Terminology and Domain Analysis]].
- **Propuesta de jerarquía** — sugerir relaciones de subsunción candidatas.
- **Extracción de tripletas** — poblar el grafo desde texto no estructurado, que es donde vive la mayor parte del conocimiento organizacional.
- **Generación de competency questions** — a partir de documentación de dominio, como punto de partida para revisar con el experto.

### Los enfoques que se consolidaron

- **Ontology-grounded extraction** — el esquema se le da al modelo como restricción, y la extracción se limita a clases y propiedades existentes. Reduce drásticamente la invención de relaciones espurias.
- **Extract–Define–Canonicalize (EDC)** — pipeline de tres etapas: extracción abierta, definición semántica, normalización contra un esquema existente. Permite alinear esquemas inducidos automáticamente con ontologías ya establecidas.
- **Schema induction bottom-up** — generar el grafo a nivel de instancias desde texto y **después** abstraer conceptos por clustering y generalización. Es la línea que abrieron GraphRAG y OntoRAG.
- **Workflows agénticos con verificación** — el modelo actúa como diseñador de esquema equipado con herramientas que verifican cada identificador contra el grafo antes de usarlo. Es la dirección donde está la investigación más reciente.

> [!warning] **Ningún enfoque elimina la verificación humana, y el motivo es estructural.** Una ontología es una *especificación compartida* — su valor está en el **acuerdo** de una comunidad sobre qué significan los términos. Un modelo puede proponer una estructura plausible; no puede producir consenso organizacional. La distinción es la misma que el campo siempre hizo entre verificación (automatizable) y validación (no). Ver [[06 - Evaluation and Testing]].

> [!tip] El patrón que rinde: **LLM propone, razonador verifica, experto valida.** El modelo genera candidatos a escala; el razonador detecta inconsistencias y [[clases insatisfacibles]]; el experto revisa las **consecuencias inferidas**, no los axiomas. Cada capa filtra errores que la anterior no puede ver.

## El error de fondo: "el LLM ya sabe el dominio"

El argumento aparece en cada discusión: *si el modelo conoce el dominio, ¿para qué modelarlo?*

Confunde dos cosas distintas:

| | Un LLM tiene | Una ontología aporta |
|---|---|---|
| **Naturaleza** | Conocimiento estadístico, implícito, distribuido en pesos | Especificación explícita e inspeccionable |
| **Autoridad** | Lo que estaba en el corpus de entrenamiento | Lo que **esta organización acordó** que significan sus términos |
| **Verificabilidad** | No auditable | Cada axioma es revisable y trazable |
| **Consistencia** | Puede contradecirse entre respuestas | Verificable por razonador |
| **Actualización** | Reentrenar o depender del contexto | Editar un axioma y versionar |

> [!note] El punto decisivo es la **autoridad**: cuando tu organización define *cliente activo* de una manera particular que no coincide con el uso general, ninguna cantidad de entrenamiento le da al modelo esa definición. Esa clase de conocimiento —convencional, local, acordado— es exactamente lo que una ontología existe para capturar, y es el 90% de lo que importa en un sistema empresarial.

## Qué mirar con escepticismo

- **"Ontología generada automáticamente"** — sin validación de expertos es una taxonomía plausible, no una especificación compartida. Le falta la propiedad que define el artefacto.
- **Métricas de extracción sin ground truth de dominio** — precisión medida contra la salida de otro modelo no dice nada. Ver [[Ground Truth]].
- **Grafos sin esquema** — extraer tripletas sin vocabulario controlado produce miles de predicados casi-sinónimos (`trabajaEn`, `empleadoDe`, `esParteDelEquipoDe`) que ninguna consulta puede aprovechar. Es el fracaso más común de los pipelines automáticos.
- **Grounding declarado sin trazabilidad** — si la respuesta no puede señalar las tripletas que la sustentan, el grounding es decorativo.

## Conexión en el vault

- [[Knowledge graph]] — el sustrato de datos; esta nota cubre la capa de LLM sobre él.
- [[Graph RAG]] — la aplicación más directa de la dirección 1.
- [[competency questions]] — requisitos, alcance, tests y ahora eval set.
- [[Grounding]] · [[Hallucinations]] — el problema que la ontología ataca desde el lado de AI del vault.
- [[Evals]] · [[Ground Truth]] · [[LLM as Judge]] — cómo se mide si esto funciona.
- [[Function Calling]] · [[MCP]] — el mecanismo por el que un agente consulta el grafo como herramienta.
- [[03 - Terminology and Domain Analysis]] — la etapa que más se transforma con extracción asistida.
- [[06 - Evaluation and Testing]] — la distinción verificación/validación, que explica por qué la revisión humana no es opcional.

## References

- *LLM-empowered Knowledge Graph Construction: A Survey* (2025) — [arXiv:2510.20345](https://arxiv.org/abs/2510.20345).
- *Ontology-grounded Automatic Knowledge Graph Construction by LLM under Wikidata schema* (2024) — [arXiv:2412.20942](https://arxiv.org/abs/2412.20942).
- *AutoSchemaKG: Autonomous Knowledge Graph Construction through Dynamic Schema Induction* (2025) — [arXiv:2505.23628](https://arxiv.org/abs/2505.23628).

## Related

- [[Knowledge graph]]
- [[Graph RAG]]
- [[ontología]]
- [[competency questions]]
- [[Grounding]]
- [[SHACL]]
- [[SPARQL]]
