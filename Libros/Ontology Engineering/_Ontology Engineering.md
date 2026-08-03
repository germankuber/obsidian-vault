---
title: Ontology Engineering — Mapa del libro
libro: Ontology Engineering
autor: Elisa F. Kendall / Deborah L. McGuinness
created: 2026-08-03
tags:
  - libros/ontology-engineering
  - type/moc
  - status/stub
aliases:
  - Ontology Engineering
  - Ontology Engineering MOC
updated: 2026-08-03
---

# Ontology Engineering — Mapa del libro

> [!info] Elisa F. Kendall & Deborah L. McGuinness · Morgan & Claypool, *Synthesis Lectures on Data, Semantics, and Knowledge* (2019)
> Mapa de lectura de *Ontology Engineering*. Abrí esta nota para la tesis del libro, el índice de capítulos y las ideas que cruzan toda la obra. Empezá por la tesis y bajá.

## 🎯 Tesis del libro

El libro sostiene que una **[[ontología]] es un artefacto de software con ciclo de vida completo** — requisitos, diseño, implementación, testing, versionado, governance y mantenimiento — y no un ejercicio filosófico ni un diagrama conceptual que se dibuja una vez y se publica. De ahí el título: *Engineering*, no *Design*. La consecuencia práctica es que las disciplinas maduras de ingeniería de software aplican tal cual al modelado de conocimiento, y que un proyecto de ontología sin governance ni casos de uso está condenado por las mismas razones por las que lo estaría un proyecto de software sin ellos.

El segundo eje es el **[[espectro semántico]]**: los artefactos de vocabulario forman un continuo de expresividad creciente (vocabulario controlado → glosario → taxonomía → thesaurus → modelo conceptual → ontología lógica), donde cada escalón agrega poder inferencial **y costo**. La postura de las autoras es deliberadamente anti-maximalista: el escalón correcto es el mínimo que resuelve tus [[competency questions]], y subir más alto de lo necesario es sobre-ingeniería, no rigor.

Las credenciales importan para leer el enfoque: **McGuinness** es coautora de la especificación [[OWL]] del W3C, y **Kendall** viene del lado de industria y estándares (OMG). El libro se lee como la destilación de esa doble mirada — formalismo real, pero al servicio de proyectos que tienen que sobrevivir en una organización.

## 📖 Capítulos

| # | Capítulo | En una línea |
|---|---|---|
| 01 | [[01 - Introduction]] | Qué es una ontología, el espectro semántico, y la tesis: ontología = artefacto de software |
| 02 | [[02 - Ontology Development Methodology]] | El proceso iterativo: competency questions como requisitos, alcance y tests; elicitación, reuso y evaluación |
| 03 | [[03 - Terminology and Domain Analysis]] | Término vs concepto, sinonimia y homonimia, el glosario acordado, y de vocabulario a estructura |
| 04 | [[04 - Modeling Decisions]] | Clase vs instancia, jerarquías, propiedades y sus características, restricciones, clases primitivas vs definidas |
| 05 | [[05 - Ontology Design Patterns and Reuse]] | ODPs (n-ario, parte-todo, rol, tiempo), ontologías fundacionales, estrategias de reuso y antipatrones |
| 06 | [[06 - Evaluation and Testing]] | Verificación vs validación, el razonador, competency questions como tests SPARQL, integración continua |
| 07 | [[07 - Lifecycle, Versioning and Governance]] | Política de IRIs, tipos de cambio e impacto inferencial, deprecación y quién decide |
| 08 | [[08 - Tools and Practical Considerations]] | Protégé, razonadores, triple stores, por qué fracasan los proyectos, y la relación con knowledge graphs |

## 🔗 Ideas transversales

Conceptos que cruzan varios capítulos (se llena a medida que aparecen):

- **[[espectro semántico]]** — el continuo de expresividad que ordena todo el campo: introducido en [[01 - Introduction]], gobierna la elección de formalismo en [[04 - Modeling Decisions]], reaparece un nivel más arriba con las ontologías fundacionales en [[05 - Ontology Design Patterns and Reuse]], y su violación (sobre-formalización) es causa de fracaso en [[08 - Tools and Practical Considerations]].
- **[[competency questions]]** — el instrumento más transversal del libro: requisitos, alcance y tests a la vez. Nacen en [[01 - Introduction]], se vuelven método en [[02 - Ontology Development Methodology]], resuelven cada duda de modelado en [[03 - Terminology and Domain Analysis]] y [[04 - Modeling Decisions]], y se ejecutan como consultas [[SPARQL]] en [[06 - Evaluation and Testing]].
- **Ontología como artefacto de software** — la tesis fundante ([[01 - Introduction]]) que se cobra operativamente en [[07 - Lifecycle, Versioning and Governance]]: ciclo de vida, versionado, deprecación y governance.
- **Reuso antes que construcción** — planteado como norma en [[02 - Ontology Development Methodology]] y detallado con sus estrategias y costos (importar, modularizar, alinear) en [[05 - Ontology Design Patterns and Reuse]].
- **Mundo abierto vs mundo cerrado** — [[OWL]] infiere bajo *open world assumption*, [[SHACL]] valida. La trampa conceptual recurrente para quien viene de bases de datos: aparece en [[01 - Introduction]] y golpea en cada decisión de [[04 - Modeling Decisions]] (`domain`/`range` no validan, la cardinalidad no exige).
- **Validar consecuencias, no afirmaciones** — el experto debe revisar lo que el razonador **infiere**, no los axiomas escritos. Planteado en [[02 - Ontology Development Methodology]] y convertido en procedimiento en [[06 - Evaluation and Testing]].
- **El factor humano** — expertos de dominio, ingenieros de conocimiento y desarrolladores: el problema de traducción ([[01 - Introduction]]) se ataca con técnicas de elicitación ([[02 - Ontology Development Methodology]]), aparece como desacuerdo de definiciones ([[03 - Terminology and Domain Analysis]]) y se institucionaliza como governance ([[07 - Lifecycle, Versioning and Governance]]).
- **El caso de uso manda** — no hay respuesta correcta en abstracto sobre clase vs instancia, qué escalón del espectro usar, o si modelar tiempo. Todas las decisiones se resuelven contra las competency questions. Es el criterio único que atraviesa los ocho capítulos.

## 🧩 Síntesis

El recorrido se lee como un **manual de proyecto en cuatro movimientos**. (1) *Encuadre* ([[01 - Introduction]]): qué es una ontología, dónde ubicarla en el espectro semántico, y la tesis de que es un artefacto de software. (2) *Antes de modelar* ([[02 - Ontology Development Methodology]], [[03 - Terminology and Domain Analysis]]): el proceso iterativo, las competency questions como requisitos-alcance-tests, y la normalización del vocabulario que el dominio ya usa. (3) *Modelar* ([[04 - Modeling Decisions]], [[05 - Ontology Design Patterns and Reuse]]): las decisiones formales recurrentes y los patrones probados que las empaquetan. (4) *Sostener* ([[06 - Evaluation and Testing]], [[07 - Lifecycle, Versioning and Governance]], [[08 - Tools and Practical Considerations]]): evaluación con razonador y tests ejecutables, versionado y governance, y el herramental que lo hace viable.

El hilo conductor es uno solo: **el formalismo está al servicio del caso de uso, nunca al revés**. Las autoras conocen la lógica descriptiva a fondo —McGuinness co-escribió OWL— y precisamente por eso insisten en que la mayoría de los proyectos no necesitan el escalón más alto del espectro. El valor aparece antes: en el acuerdo explícito sobre qué significan los términos de una organización.

La confirmación llega al final, y cierra el círculo: las causas por las que fracasan los proyectos de ontología ([[08 - Tools and Practical Considerations]]) **no son técnicas** — modelar sin caso de uso, sin experto comprometido, sin consumidor real, sin governance. Son exactamente los riesgos que la tesis del capítulo 1 anticipaba. Por eso el libro dedica la mayor parte de sus páginas a metodología y proceso, y solo el final a herramientas.

## 🌱 Conceptos para enlazar / escribir

- [[ontología]] · [[espectro semántico]] · [[competency questions]] — el núcleo conceptual del libro.
- [[RDF]] · [[RDFS]] · [[OWL]] · [[SKOS]] · [[SHACL]] — el stack de estándares del W3C.
- [[Knowledge graph]] — el pariente industrialmente dominante; vale una nota que contraste ambos términos.
- [[Graph RAG]] — donde este conocimiento se cruza con el trabajo de recuperación del vault.
- [[Model Context Protocol (MCP)]] — vocabulario compartido para interoperabilidad, en clave agéntica.

## 🔍 Todos los capítulos (auto)

```dataview
TABLE autor AS "Autor", capitulo AS "Cap."
FROM "Libros/Ontology Engineering"
WHERE capitulo
SORT capitulo ASC
```
