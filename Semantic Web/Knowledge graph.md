---
title: Knowledge graph
source: (Hogan et al. 2021, Knowledge Graphs — ACM Computing Surveys; W3C RDF 1.2 / SPARQL 1.2 Working Drafts)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/concept
  - status/done
aliases:
  - Knowledge graph
  - knowledge graph
  - Knowledge Graph
  - knowledge graphs
  - grafo de conocimiento
  - KG
updated: 2026-08-04
---

# Knowledge graph

> [!note] Definición
> Un **knowledge graph** es un grafo de **entidades y relaciones a escala**, poblado con datos reales, cuyo significado está definido por un esquema semántico —típicamente una [[ontología]]—. La distinción operativa: **el knowledge graph son los datos; la ontología es el esquema**.

El término lo popularizó Google en 2012 y desde entonces desplazó casi por completo a "ontología" en el vocabulario industrial — mientras el trabajo subyacente seguía siendo el mismo.

## Ontología vs knowledge graph

| | Ontología | Knowledge graph |
|---|---|---|
| **Qué es** | El esquema semántico | Los datos poblados |
| **En [[Description Logic]]** | TBox | ABox (+ TBox) |
| **Tamaño típico** | Cientos–miles de axiomas | Millones–miles de millones de tripletas |
| **Énfasis** | Rigor lógico, razonamiento | Escala, cobertura, consulta |
| **Cambia** | Lento, con governance | Continuamente, por ingesta |
| **Comunidad** | Academia, estándares W3C, dominios regulados | Industria, buscadores, e-commerce, LLMs |
| **Métrica de éxito** | Consistencia, inferencias correctas | Cobertura, frescura, latencia de consulta |

> [!note] **La diferencia es de énfasis, no de naturaleza.** El mundo de las ontologías pone el peso en el rigor lógico; el de los knowledge graphs, en la escala y la población de datos. Los mejores sistemas toman de ambos: rigor suficiente para que el significado sea explícito, pragmatismo suficiente para que escale.

> [!tip] **Todo lo que la ingeniería de ontologías enseña aplica a un knowledge graph, aunque nadie en el proyecto use la palabra "ontología"**: [[competency questions]] para acotar el alcance, análisis de terminología para el vocabulario, decisiones de modelado explícitas, evaluación contra casos de uso, y governance para que sobreviva. La ausencia del término no es ausencia del problema.

## La división real de la industria: RDF vs property graphs

Es la decisión de arquitectura más consecuente y la que más confusión genera, porque son **modelos de datos incompatibles**, no dos sintaxis del mismo.

### RDF (tripletas)

```turtle
:juan  :trabajaEn  :acme .
```

Todo es `sujeto → predicado → objeto`. Los identificadores son **IRIs globales**, así que dos datasets de organizaciones distintas se fusionan sin negociación previa. Es el modelo del W3C.

### Property graph (LPG — Labeled Property Graph)

```cypher
(juan:Persona {nombre: "Juan"})
  -[:TRABAJA_EN {desde: 2019, rol: "dev"}]->
(acme:Empresa {nombre: "Acme"})
```

Nodos y aristas tienen **etiquetas y pares clave-valor propios**. Es el modelo de Neo4j y de la mayoría de las bases de grafos comerciales.

### La comparación

| | RDF | Property Graph |
|---|---|---|
| **Unidad** | Tripleta | Nodo / arista con propiedades |
| **Identidad** | IRI global, resoluble | ID local a la base |
| **Atributos en aristas** | No nativo (requiere reificación o RDF-star) | **Nativo** — la ventaja principal |
| **Esquema** | Ontología formal, razonamiento DL | Opcional, sin semántica formal |
| **Consulta** | [[SPARQL]] (estándar W3C) | Cypher / Gremlin / **GQL** (ISO 39075:2024) |
| **Federación** | Nativa: `SERVICE` sobre endpoints | Débil; los IDs no son globales |
| **Inferencia** | Razonadores OWL/RDFS | Ninguna (o reglas ad-hoc) |
| **Fuerte en** | Interoperabilidad, datos públicos, dominios regulados | Analítica de grafos, recorridos, performance operativa |

> [!warning] **La incompatibilidad práctica**: en un property graph poner un atributo en una arista es trivial; en RDF exige reificar el hecho como entidad, lo que multiplica tripletas y complica cada consulta. En sentido inverso: fusionar dos property graphs de organizaciones distintas es un proyecto de integración; fusionar dos grafos RDF con IRIs bien elegidos es una unión de conjuntos.

> [!tip] **Criterio de elección.** ¿Los datos tienen que interoperar fuera de tu organización, hay requisito regulatorio, o necesitás inferencia? → RDF. ¿Es un grafo interno, con muchos atributos en las relaciones, y el caso de uso es recorrido y analítica? → property graph. La mayoría de los sistemas empresariales caen en el segundo grupo, y por eso Neo4j domina la mención comercial mientras RDF domina los datos públicos y la biomedicina.

### RDF-star: el puente

**RDF-star** (RDF 1.2, en el track de Recommendation del W3C junto con SPARQL 1.2) agrega afirmaciones **sobre** afirmaciones:

```turtle
<< :juan :trabajaEn :acme >> :desde 2019 ; :fuente :rrhh .
```

Cierra la brecha funcional principal con los property graphs —metadatos de arista— sin abandonar el modelo de tripletas. Es el desarrollo más relevante del stack RDF de los últimos años: resuelve de forma nativa lo que antes exigía reificación manual o named graphs.

> [!note] Estado a 2026: RDF 1.2 y SPARQL 1.2 avanzan como Working Drafts del RDF & SPARQL Working Group. El soporte de RDF-star ya está implementado en varias triple stores desde antes de la estandarización formal — verificá el soporte de tu motor antes de comprometerte.

## De dónde salen los datos

El gap que casi ninguna metodología cubre: la ontología existe, ¿y los datos? Es el 70% del esfuerzo de un proyecto real.

- **R2RML / RML** — mapeo declarativo de relacional (y CSV, JSON, XML con RML) a RDF. Estándar W3C.
- **Virtual knowledge graph (OBDA)** — no se materializa nada: la consulta SPARQL se reescribe a SQL contra la base existente. **Ontop** es la implementación de referencia, y el perfil OWL 2 QL existe precisamente para esto. Ideal cuando los datos deben quedarse en su sistema de origen.
- **Extracción desde texto** — de NER + relation extraction clásicos a pipelines con LLM. Ver [[Ontología y LLMs]].
- **Ingesta desde APIs y streams** — con mapeo a la ontología en el punto de entrada.

> [!warning] **Entity resolution es el problema difícil, no el mapeo.** Fusionar fuentes exige decidir cuándo dos registros son la misma entidad. En RDF esto se expresa con `owl:sameAs`, y usarlo a la ligera es destructivo: `sameAs` es transitivo y simétrico, así que un solo enlace erróneo **fusiona dos entidades para siempre** y propaga todas sus propiedades cruzadas.

## Materializar o razonar en consulta

Decisión de arquitectura que define el comportamiento del sistema:

| | Materializar (forward chaining) | Razonar en consulta (backward chaining) |
|---|---|---|
| **Cuándo infiere** | Al cargar/actualizar | Al consultar |
| **Consulta** | Rápida | Más lenta |
| **Escritura** | Cara: reinferir al cambiar | Barata |
| **Espacio** | Crece — a veces mucho | Sin overhead |
| **Conviene si** | Ratio lectura/escritura alto | Datos volátiles |

> [!tip] La mayoría de los knowledge graphs productivos materializan inferencia RDFS o el perfil OWL 2 RL —barato y suficiente— y dejan el razonamiento DL completo para la fase de diseño de la ontología, no para runtime.

## Knowledge graphs y LLMs

El cruce que redefinió el campo desde 2023, y donde el knowledge graph pasó de repositorio estático a **infraestructura de grounding**. Los usos principales:

- **Grounding factual** — el modelo responde consultando el grafo en vez de su memoria paramétrica, con la consecuencia de que la respuesta es trazable a una tripleta concreta.
- **[[Graph RAG]]** — recuperación que aprovecha la estructura del grafo (vecindad, caminos, comunidades) en vez de solo similitud vectorial. Ver la nota para el detalle.
- **Construcción asistida** — el LLM extrae entidades y relaciones del texto; la ontología actúa como esquema que restringe y valida lo extraído.
- **Verificación** — el grafo como fuente contra la cual chequear afirmaciones generadas.

> [!note] La relación es **bidireccional**: el grafo mejora al LLM (grounding, trazabilidad, menos alucinación) y el LLM mejora al grafo (extracción a escala, que era el cuello de botella histórico de la construcción). Ver [[Ontología y LLMs]].

## Conexión en el vault

- [[ontología]] — el esquema que le da significado; el knowledge graph son los datos.
- [[Description Logic]] — la división TBox/ABox es exactamente la división ontología/knowledge graph.
- [[SPARQL]] — el lenguaje de consulta del lado RDF.
- [[SHACL]] — la validación imprescindible cuando los datos entran desde fuentes heterogéneas.
- [[Graph RAG]] · [[Ontología y LLMs]] — el cruce con el bloque de AI del vault.
- [[08 - Tools and Practical Considerations]] — donde el libro plantea la relación entre ambos términos.
- [[Hybrid Search]] — el pariente del lado de recuperación: combinar señal léxica, vectorial y estructural.

## References

- Hogan, A. et al. (2021) — *Knowledge Graphs*. ACM Computing Surveys 54(4). El survey de referencia.
- [RDF & SPARQL Working Group Charter](https://www.w3.org/2025/04/rdf-star-wg-charter.html) — W3C. RDF 1.2 y SPARQL 1.2.
- [SPARQL 1.2 Query Language](https://www.w3.org/TR/sparql12-query/) — W3C Working Draft.
- ISO/IEC 39075:2024 — **GQL**, el estándar de lenguaje de consulta para property graphs.
- [R2RML: RDB to RDF Mapping Language](https://www.w3.org/TR/r2rml/) — W3C Recommendation.

## Related

- [[ontología]]
- [[SPARQL]]
- [[SHACL]]
- [[Description Logic]]
- [[Graph RAG]]
- [[Ontología y LLMs]]
- [[espectro semántico]]
