---
title: Ontology Design Patterns (ODP)
source: (ontologydesignpatterns.org; Gangemi & Presutti, Ontology Design Patterns; Kendall & McGuinness 2019 cap. 5)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/concept
  - status/done
aliases:
  - Ontology Design Patterns (ODP)
  - Ontology Design Patterns
  - ODP
  - ODPs
  - patrones de diseño de ontologías
updated: 2026-08-04
---

# Ontology Design Patterns (ODP)

> [!note] Definición
> Un **Ontology Design Pattern** es una solución reusable a un problema de modelado recurrente: el problema que resuelve, la estructura que propone y las consecuencias de adoptarla. La analogía con los design patterns del software es directa — no es código que copiás, es una **forma de estructurar** que ya sobrevivió los casos borde.

## Por qué usarlos

Más allá del ahorro de tiempo: **decisiones ya validadas** (los casos borde que todavía no encontraste), **vocabulario compartido** para discutir diseño con otros, e **interoperabilidad** — dos ontologías que usan el mismo patrón para el mismo problema son mucho más fáciles de alinear después.

## Dónde buscarlos

| Recurso | Qué tiene |
|---|---|
| [ontologydesignpatterns.org](http://ontologydesignpatterns.org) | El portal de la comunidad; catálogo estructurado por tipo |
| [W3C Best Practices — n-ary relations](https://www.w3.org/TR/swbp-n-aryRelations/) | Nota oficial del W3C sobre el patrón n-ario |
| [W3C Best Practices — value partitions](https://www.w3.org/TR/swbp-specified-values/) | Value partition vs value set |
| [PROV-O](https://www.w3.org/TR/prov-o/) | Vocabulario estándar de procedencia |
| [OBO Foundry](https://obofoundry.org) | Patrones de facto en biomedicina; DOSDP templates |

> [!tip] La probabilidad de que tu problema de modelado sea nuevo es baja. Buscá el patrón **antes** de resolverlo a mano.

## El movimiento de fondo

Los cuatro patrones clásicos comparten una misma estrategia, y reconocerla hace que se aprendan casi como uno solo:

> [!note] **Cuando una relación binaria no alcanza, convertí el hecho en entidad.** [[OWL]] solo tiene predicados binarios ([[Description Logic]] es una lógica de dos variables); todo lo que necesite más participantes o atributos propios exige reificar.

---

## Patrón: relación n-aria

**Problema.** OWL solo tiene relaciones binarias, pero el mundo tiene hechos con más participantes o con atributos propios: *"Juan trabajó en Acme como desarrollador desde 2019 hasta 2022 con salario X"*.

**Solución.** Reificar la relación: el hecho se vuelve una entidad de pleno derecho.

```turtle
@prefix :    <https://ejemplo.org/onto/> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

:Empleo_Juan_Acme  a  :Empleo ;
    :empleado        :Juan ;
    :empleador       :Acme ;
    :rolDesempeñado  :Desarrollador ;
    :fechaInicio     "2019-03-01"^^xsd:date ;
    :fechaFin        "2022-11-30"^^xsd:date ;
    :salario         [ :monto 85000 ; :moneda :USD ] .
```

La clase `:Empleo` no existía en el vocabulario del dominio — la introduce el modelo para poder decir todo lo que hay que decir.

> [!warning] **Costo real**: las consultas atraviesan un nodo intermedio y el modelo pierde legibilidad. No reifiques por defecto. Si solo necesitás *"Juan trabaja en Acme"*, una propiedad binaria alcanza.

> [!tip] **Alternativa moderna**: RDF-star permite anotar la tripleta directamente, sin clase intermedia — `<< :juan :trabajaEn :acme >> :desde 2019 .` Más liviano para metadatos simples (procedencia, confianza, tiempo); la reificación clásica sigue siendo mejor cuando el hecho es una entidad del dominio con identidad propia. Ver [[Knowledge graph]].

---

## Patrón: parte-todo (meronimia)

**Problema.** La relación parte-todo **no es subsunción**, y modelarla como tal rompe la herencia: el razonador infiere que toda rueda es un auto.

**Solución.** Familia de propiedades con subtipos, y transitividad declarada **solo donde vale**.

```turtle
:partOf       a  owl:ObjectProperty .          # genérica, NO transitiva
:hasPart      a  owl:ObjectProperty ;
              owl:inverseOf :partOf .

:componentOf  a  owl:ObjectProperty , owl:TransitiveProperty ;
              rdfs:subPropertyOf :partOf .     # tornillo→motor→auto: sí encadena

:memberOf     a  owl:ObjectProperty ;          # NO transitiva
              rdfs:subPropertyOf :partOf .     # jugador→equipo→liga: no encadena
```

Las distinciones que el patrón obliga a tomar:

- **¿Transitiva?** Un tornillo es parte del motor, el motor del auto → el tornillo es parte del auto. Pero un jugador es miembro del equipo y el equipo de la liga → el jugador **no** es miembro de la liga.
- **¿Separable?** Una rueda se quita del auto; un ángulo no se quita del triángulo.
- **¿Componente o porción de materia?** Un motor es componente; una porción de agua es materia. Se comportan distinto ante la división.

> [!warning] Declarar `partOf` transitiva **globalmente** parece inocente y produce inferencias absurdas apenas el modelo crece. La transitividad depende del **tipo** de relación parte-todo.

---

## Patrón: rol

**Problema.** *Empleado* como subclase de *Persona* es casi siempre un error: los roles son múltiples, temporales y tienen atributos propios, mientras que la subsunción es única en su criterio, permanente y sin atributos.

**Solución.** Separar la entidad de su rol; el rol es una entidad con contexto y período.

```turtle
:Persona     a  owl:Class .
:Rol         a  owl:Class .
:RolEmpleado rdfs:subClassOf :Rol .

:desempeña   a  owl:ObjectProperty ;
             rdfs:domain :Persona ;
             rdfs:range  :Rol .

:juan  a  :Persona ;
       :desempeña  [ a          :RolEmpleado ;
                     :en        :Acme ;
                     :cargo     :Desarrollador ;
                     :desde     "2019-03-01"^^xsd:date ] ,
                   [ a          :RolEstudiante ;
                     :en        :Universidad ;
                     :desde     "2021-08-01"^^xsd:date ] .
```

Juan tiene dos roles simultáneos, cada uno con su contexto y sus fechas. Si deja de ser empleado, se cierra el rol — **no se cambia la clase de Juan**.

> [!tip] **El criterio de decisión**: ¿la clasificación es **esencial** o **contingente**? Si algo puede dejar de serlo sin dejar de ser lo que es, es un rol. Un Golden Retriever no puede dejar de ser perro (esencial → subclase); un empleado sí puede dejar de ser empleado (contingente → rol).

---

## Patrón: tiempo y cambio

**Problema.** OWL no tiene noción nativa de tiempo. Modelar información que cambia es de lo más costoso del campo.

Los tres enfoques:

| Enfoque | Cómo | Cuándo |
|---|---|---|
| **Snapshot / versionado** | La ontología representa el estado actual y se versiona entera | Simple; pierde la historia |
| **Fluents / reificación temporal** | Cada hecho que cambia se reifica con su intervalo de validez | Completo y pesado: multiplica entidades |
| **Time-indexed relations** | Variante del n-ario: el hecho lleva un intervalo asociado | El punto medio habitual |

```turtle
# Time-indexed: el hecho con su intervalo
:Asignacion_001  a  :AsignacionDeResponsable ;
    :proyecto     :ProyectoX ;
    :responsable  :Ana ;
    :intervalo    [ a :Intervalo ;
                    :inicio "2021-01-01"^^xsd:date ;
                    :fin    "2022-06-30"^^xsd:date ] .
```

> [!warning] **Modelar tiempo multiplica la complejidad de todo el modelo.** Si tus [[competency questions]] son todas sobre el estado actual, no lo modeles. Si alguna pregunta por historia, acotá el alcance temporal **a las entidades que lo necesitan**, no a todo el modelo.

---

## Patrón: value partition

**Problema.** Valores cualitativos —severidad alta/media/baja, tamaño chico/mediano/grande— modelados como literales string, que no permiten razonar ni garantizan vocabulario cerrado.

**Solución.** Partición explícita: un conjunto de valores exhaustivo, disjunto y enumerado.

```turtle
:Severidad  a  owl:Class ;
    owl:equivalentClass [ a owl:Class ;
                          owl:oneOf ( :Alta :Media :Baja ) ] .   # exhaustivo

[] a owl:AllDifferent ;
   owl:distinctMembers ( :Alta :Media :Baja ) .                  # disjunto

:Alta  a :Severidad .  :Media a :Severidad .  :Baja a :Severidad .

:tieneSeveridad  a  owl:ObjectProperty ;
                 rdfs:range  :Severidad .
```

> [!tip] Con la partición cerrada, el razonador puede usarla: una clase definida como *"incidente con severidad Alta"* clasifica sola. Con un literal `"alta"` no hay nada que razonar — y nada que impida que alguien escriba `"ALTA"` o `"high"`.

---

## Patrón: procedencia

**Problema.** ¿Quién afirmó esto, cuándo y con base en qué? Crítico cuando el grafo se puebla desde múltiples fuentes o desde extracción automática.

**Solución.** Reusar **PROV-O**, el vocabulario estándar del W3C, en vez de inventar uno propio.

```turtle
@prefix prov: <http://www.w3.org/ns/prov#> .

:afirmacion_123  a               prov:Entity ;
                 prov:wasDerivedFrom      :documento_45 ;
                 prov:wasAttributedTo     :pipelineExtraccion ;
                 prov:generatedAtTime     "2026-08-04T10:30:00Z"^^xsd:dateTime .
```

Las tres alternativas para adjuntar procedencia a una afirmación:

| Mecanismo | Cómo | Trade-off |
|---|---|---|
| **Named graphs** | Cada fuente en su propio grafo | Simple y soportado en todas partes; granularidad gruesa |
| **Reificación RDF clásica** | `rdf:Statement` con subject/predicate/object | Verboso: 4 tripletas por hecho |
| **RDF-star** | `<< :a :b :c >> prov:wasAttributedTo :x` | Conciso y nativo; verificá soporte del motor |

> [!note] Para pipelines de extracción con LLM, la procedencia deja de ser opcional: es lo que hace la diferencia entre un grafo auditable y uno que nadie puede verificar. Ver [[Ontología y LLMs]].

---

## Patrón: listas y secuencias

**Problema.** RDF no tiene listas ordenadas usables. `rdf:List` existe pero es una lista enlazada con blank nodes: horrible de consultar en [[SPARQL]] y peor de mantener.

**Soluciones prácticas:**

```turtle
# Opción A — índice explícito (la más consultable)
:receta  :tienePaso  [ :orden 1 ; :accion "Precalentar horno" ] ,
                     [ :orden 2 ; :accion "Mezclar ingredientes" ] .

# Opción B — enlace al siguiente (bueno para recorrer, malo para saltar)
:paso1  :siguiente  :paso2 .
```

> [!tip] La opción A es casi siempre la correcta: `ORDER BY ?orden` en SPARQL resuelve todo, y agregar un paso en el medio no exige reescribir enlaces. Reservá `rdf:List` para cuando un vocabulario externo ya lo impone.

---

## Antipatrones

Reconocerlos vale tanto como conocer los patrones. Cada uno con el síntoma que produce:

| Antipatrón | Síntoma observable |
|---|---|
| **Subsunción para todo** (meronimia, roles, estados como subclases) | El razonador infiere que toda rueda es un auto; jerarquías inferidas absurdas |
| **Jerarquía profunda sin justificación** | Niveles que no introducen propiedades ni responden preguntas; puro mantenimiento |
| **Clase con una sola instancia posible** | Debería ser un individuo; señal de confundir el nivel de tipos con el de individuos |
| **Propiedad genérica `tieneAtributo`** | Reproduce entidad-atributo-valor dentro de OWL: **anula toda capacidad de razonamiento**, porque el razonador no puede saber qué significa un atributo genérico |
| **Disjunción por reflejo** | [[clases insatisfacibles]] apenas el modelo crece |
| **Modelar la aplicación en vez del dominio** | Clases que replican tablas (`:ClienteTmp`, `:FlagActivo`) |

> [!tip] Los antipatrones se detectan mucho mejor por sus **consecuencias inferenciales** —clases insatisfacibles, jerarquías inferidas raras— que por inspección visual. Corré el razonador después de cada bloque de modelado, no al final.

## Ontologías fundacionales

Un escalón de abstracción por encima de los patrones: modelos de categorías generalísimas sobre los que anclar el dominio.

| | BFO | DOLCE | SUMO |
|---|---|---|---|
| **Postura** | Realista: describe la realidad tal cual es | Cognitivista: refleja categorías del lenguaje y el sentido común | Ecléctica, muy amplia |
| **Distinción central** | *continuant* (persiste) vs *occurrent* (ocurre) | Endurants, perdurants, qualities, abstracts | Jerarquía extensa multi-dominio |
| **Estatus** | **ISO/IEC 21838-2**; estándar de facto en biomedicina (OBO Foundry) | Académica, influyente en NLP | La más grande, la menos adoptada |
| **Elegila si** | Trabajás en ciencias de la vida o necesitás un estándar reconocido | Modelás lenguaje natural o conceptos cognitivos | Rara vez es la respuesta |

> [!warning] Anclar en una fundacional **por prestigio** es un error clásico. Si tu caso es integrar tres sistemas internos, BFO es sobrecarga pura. Si construís para un ecosistema científico donde ya es estándar de facto, la decisión se invierte y **no** anclar es el error. Es el criterio del [[espectro semántico]] un nivel más arriba.

## Estrategias de reuso

| Estrategia | Mecanismo | Cuándo |
|---|---|---|
| **Importación completa** | `owl:imports` | La ontología externa es chica y la necesitás casi entera |
| **Modularización** | Extracción de módulo (ROBOT `extract`) | Necesitás una fracción de algo grande — el caso más común |
| **Alineamiento por mapeos** | `owl:equivalentClass`, `skos:exactMatch` | Querés mantener tu vocabulario y desacoplarte |
| **Referencia sin importación** | Usar los IRIs externos sin traer axiomas | Interoperabilidad de identificadores sin heredar semántica |

**Dónde buscar qué reusar** — el catálogo que falta en casi toda la literatura:

| Recurso | Qué tiene |
|---|---|
| [LOV](https://lov.linkeddata.es/dataset/lov) | Linked Open Vocabularies: buscador de vocabularios RDF por término |
| [BioPortal](https://bioportal.bioontology.org) | ~1.000 ontologías biomédicas |
| [OBO Foundry](https://obofoundry.org) | Ontologías biomédicas coordinadas, con principios comunes |
| [schema.org](https://schema.org) | El vocabulario más usado de la web; e-commerce, eventos, personas, organizaciones |
| [Dublin Core](https://www.dublincore.org) | Metadatos documentales — casi siempre parte de la respuesta |
| [FOAF](http://xmlns.com/foaf/spec/) | Personas y relaciones sociales |
| [PROV-O](https://www.w3.org/TR/prov-o/) | Procedencia |
| [SKOS](https://www.w3.org/TR/skos-reference/) | Taxonomías y thesauri |
| [QUDT](https://qudt.org) | Cantidades, unidades y dimensiones |
| [Wikidata](https://www.wikidata.org) | Entidades del mundo real como IRIs reusables |

> [!warning] **Reusar mal es peor que no reusar.** Importar una ontología grande "por las dudas" arrastra cientos de clases irrelevantes, sus axiomas y su carga de razonamiento. Si necesitás cinco clases de un vocabulario de mil, extraé el módulo o declará mapeos.

> [!tip] Evaluá cada candidata contra tus propias [[competency questions]]: si no ayuda a responder ninguna, no la reuses por prestigio. Y **fijá la versión** de toda ontología importada — es una dependencia versionada, igual que una librería.

## Conexión en el vault

- [[05 - Ontology Design Patterns and Reuse]] — el capítulo del libro que introduce estos patrones.
- [[04 - Modeling Decisions]] — las decisiones que estos patrones empaquetan.
- [[OWL]] · [[Description Logic]] — las limitaciones expresivas que varios patrones existen para sortear.
- [[clases insatisfacibles]] — el síntoma por el que se detectan los antipatrones.
- [[espectro semántico]] — el criterio del escalón mínimo, aplicado también a fundacionales.
- [[Knowledge graph]] — RDF-star como alternativa moderna a la reificación.

## References

- [Ontology Design Patterns portal](http://ontologydesignpatterns.org)
- Gangemi, A. & Presutti, V. (2009) — *Ontology Design Patterns*, en Handbook on Ontologies.
- [Defining N-ary Relations on the Semantic Web](https://www.w3.org/TR/swbp-n-aryRelations/) — W3C Working Group Note.
- ISO/IEC 21838-2:2021 — Basic Formal Ontology (BFO).
- Kendall, E. & McGuinness, D. (2019) — *Ontology Engineering*, cap. 5.

## Related

- [[OWL]]
- [[Description Logic]]
- [[clases insatisfacibles]]
- [[espectro semántico]]
- [[competency questions]]
- [[Knowledge graph]]
- [[05 - Ontology Design Patterns and Reuse]]
