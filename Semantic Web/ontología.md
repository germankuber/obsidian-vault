---
title: ontología
source: (Gruber 1993; Studer et al. 1998; Kendall & McGuinness 2019)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/concept
  - status/done
aliases:
  - ontología
  - ontologia
  - ontology
  - Ontología
  - ontologías
updated: 2026-08-04
---

# ontología

> [!note] Definición
> Una **ontología** es una *especificación **explícita**, **formal** y **compartida** de una **conceptualización** de un dominio*. Cada palabra carga peso, y desarmarla es la mejor forma de entender qué es y qué no es.

La formulación desciende de **Tom Gruber (1993)** —*"an explicit specification of a conceptualization"*— a la que **Studer, Benjamins & Fensel (1998)** agregaron *formal* y *shared*, que es la versión que la comunidad usa hoy.

## Desarmando la definición

- **Explícita** — los supuestos sobre el dominio están **escritos y son inspeccionables**. Lo contrario es el conocimiento tácito que vive en la cabeza de los expertos o enterrado en el código de una aplicación.
- **Formal** — expresada en un lenguaje con **semántica definida**, procesable por máquina sin ambigüedad. Un documento Word que describe conceptos en prosa no es una ontología, por más riguroso que sea.
- **Compartida** — refleja el **consenso de una comunidad de práctica**. Una ontología que solo su autor entiende falló en su propósito: el valor está justamente en el acuerdo.
- **Conceptualización** — modela **el dominio, no una implementación**. Es la diferencia entre modelar *qué es un paciente* y modelar *la tabla `patients` de este sistema*.

> [!note] **Una ontología modela el dominio, no la aplicación.** Este es el criterio que la separa de un esquema de base de datos. El esquema responde a las necesidades de un sistema concreto (performance, normalización, la query que hay que servir); la ontología describe cómo es el mundo del dominio, con independencia de qué sistema lo consuma. Por eso puede **sobrevivir a la aplicación que la originó** y ser reusada por otras.

## El equívoco de origen: filosofía vs ingeniería

| | Filosofía | Ingeniería del conocimiento |
|---|---|---|
| **Forma** | *La* ontología (singular, con artículo) | *Una* ontología, *tres* ontologías (contable) |
| **Qué es** | Rama de la metafísica: el estudio del ser y de qué existe | Un artefacto concreto que alguien construye, versiona y despliega |
| **Producto** | Teoría filosófica | Archivo en [[OWL]] / [[RDFS]] bajo control de versiones |

> [!warning] La confusión más frecuente en la práctica **no** es con la filosofía, sino con el **diagrama entidad-relación**. No es lo mismo: el modelo E-R describe estructuras de almacenamiento; la ontología describe **significado** — y ese significado tiene consecuencias lógicas que un razonador puede computar.

## Qué NO es una ontología

- **Un esquema de base de datos** — modela la aplicación, no el dominio; y no tiene semántica formal para razonar.
- **Un diagrama UML o E-R** — captura estructura, no significado computable.
- **Un glosario o un diccionario de datos** — es *formal* lo que le falta; está un par de escalones abajo en el [[espectro semántico]].
- **Una taxonomía** — es un escalón del espectro, pero solo tiene jerarquía. Toda taxonomía es (casi) una ontología mínima; no toda ontología es una taxonomía.
- **Un knowledge graph** — el [[Knowledge graph]] son los **datos**; la ontología es el **esquema semántico** que les da estructura. Ver la nota para el contraste completo.

> [!note] La frontera con la taxonomía es difusa a propósito: el [[espectro semántico]] muestra que estos artefactos forman un **continuo**, no categorías disjuntas. La pregunta útil no es *"¿esto es una ontología?"* sino *"¿en qué escalón de expresividad está y me alcanza?"*.

## La tesis operativa: es un artefacto de software

El aporte central de [[_Ontology Engineering|Kendall & McGuinness]] es tratarla como lo que es en la práctica:

- **Requisitos** — capturados como [[competency questions]].
- **Diseño** — decisiones de modelado, reuso, elección de nivel de expresividad, patrones.
- **Implementación** — codificación en [[OWL]]/[[RDFS]], con convenciones de nomenclatura e IRIs.
- **Testing** — [[competency questions]] ejecutadas como [[SPARQL]] + razonador para consistencia.
- **Versionado y release** — `owl:versionIRI`, política de compatibilidad, deprecación.
- **Governance** — quién puede cambiarla, cómo se aprueban cambios, cómo se resuelven desacuerdos.
- **Mantenimiento evolutivo** — el dominio cambia y la ontología con él.

> [!warning] El modo de falla característico del enfoque académico es la **ontología huérfana**: técnicamente elegante, lógicamente impecable, publicada en un paper — y sin ningún sistema que la consuma ni nadie que la mantenga.

## Para qué sirve

- **Integración de datos heterogéneos** — el modelo común contra el cual se mapean fuentes que usan vocabularios distintos, en lugar de mapeos punto-a-punto que crecen cuadráticamente.
- **Búsqueda semántica** — una consulta por *cardiopatía* recupera documentos que dicen *insuficiencia cardíaca*, porque el modelo sabe que uno es tipo del otro.
- **Interoperabilidad** — dos organizaciones intercambian datos sin negociar el significado en cada integración. Motivador dominante en dominios regulados.
- **Inferencia** — derivar hechos no declarados. La única capacidad que no se obtiene de ningún escalón inferior.
- **Documentar el significado compartido** — el beneficio más subestimado: el proceso mismo obliga a la organización a explicitar y acordar qué significan sus términos. **Muchos proyectos cosechan acá su retorno principal, antes de que ningún razonador se ejecute.**
- **Grounding de sistemas LLM** — el uso que creció más rápido desde 2023: la ontología como esquema que restringe y verifica lo que un modelo extrae o afirma. Ver [[Ontología y LLMs]].

## Cuándo NO construir una ontología

La contracara que se omite casi siempre. **No la necesitás si** se cumplen todas estas:

- Un solo sistema consume los datos, y no hay perspectiva de que sean varios.
- Un solo equipo define el vocabulario, y hay acuerdo.
- El vocabulario es estable y chico.
- Ninguna pregunta del negocio requiere inferencia — todo se resuelve con un `JOIN`.
- No hay requisito de interoperabilidad externa ni regulatorio.

> [!tip] En ese escenario, un `enum` en la base de datos y un glosario en la wiki resuelven el problema real, y cualquier cosa más es sobre-ingeniería. La pregunta *"¿necesito una ontología?"* se responde con las mismas [[competency questions]] que la construirían: si ninguna pregunta necesita inferencia ni vocabulario compartido, la respuesta es no.

## Anatomía: de qué está hecha

| Componente | Qué es | Ejemplo |
|---|---|---|
| **Clases** | Tipos, conjuntos de individuos | `:Medicamento`, `:Paciente` |
| **Individuos** | Instancias concretas | `:aspirina_100mg` |
| **Object properties** | Relaciones entre individuos | `:contraindicadoEn` |
| **Datatype properties** | Atributos con valor literal | `:dosisMaxima` |
| **Axiomas** | Afirmaciones lógicas sobre lo anterior | `:PolizaVida owl:disjointWith :PolizaPatrimonial` |
| **Anotaciones** | Metadatos no lógicos | `rdfs:label`, `rdfs:comment`, `owl:deprecated` |

En vocabulario de [[Description Logic]]: las clases, propiedades y axiomas forman la **TBox** (el esquema); los individuos y sus afirmaciones, la **ABox** (los datos).

## Conexión en el vault

- El [[espectro semántico]] ubica a la ontología como el escalón más expresivo de un continuo — y explica por qué la mayoría de los proyectos no lo necesitan.
- Las [[competency questions]] son el instrumento que decide qué modelar y cuándo está terminada.
- [[OWL]] es el lenguaje donde se expresa; [[Description Logic]] su fundamento formal; [[SHACL]] la contraparte de validación que OWL deliberadamente no cubre.
- [[Knowledge graph]] — el pariente industrialmente dominante: mismo trabajo, otro énfasis y otro vocabulario.
- [[_Ontology Engineering|Ontology Engineering]] — el libro que la trata como artefacto de software de punta a punta.
- [[Ubiquitous Language]] — el paralelo en DDD, con una diferencia de fondo: DDD **acepta** que un término signifique cosas distintas en contextos distintos; la ontología tiende a **unificar**.

## References

- Gruber, T. (1993) — *A Translation Approach to Portable Ontology Specifications*. Knowledge Acquisition 5(2).
- Studer, R., Benjamins, R. & Fensel, D. (1998) — *Knowledge Engineering: Principles and Methods*. Agrega *formal* y *shared*.
- Kendall, E. & McGuinness, D. (2019) — *Ontology Engineering*, Morgan & Claypool.
- [OWL 2 Primer](https://www.w3.org/TR/owl2-primer/) — W3C.

## Related

- [[espectro semántico]]
- [[competency questions]]
- [[OWL]]
- [[Description Logic]]
- [[Knowledge graph]]
- [[SHACL]]
- [[_Ontology Engineering|Ontology Engineering]]
