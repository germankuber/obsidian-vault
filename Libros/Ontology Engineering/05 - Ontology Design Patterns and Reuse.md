---
title: 05 - Ontology Design Patterns and Reuse
libro: Ontology Engineering
autor: Elisa F. Kendall / Deborah L. McGuinness
capitulo: 5
created: 2026-08-03
tags:
  - libros/ontology-engineering
  - type/reading-note
  - status/done
aliases:
  - Ontology Design Patterns and Reuse
  - Cap 5 - Patrones de diseño y reuso
updated: 2026-08-04
---

# 05 - Ontology Design Patterns and Reuse

> [!info] Capítulo 5 · *Ontology Engineering* — Elisa F. Kendall & Deborah L. McGuinness · Morgan & Claypool, *Synthesis Lectures on Data, Semantics, and Knowledge* (2019)
> No modelar de cero: los **Ontology Design Patterns (ODPs)** son soluciones probadas a problemas de modelado recurrentes, el equivalente de los design patterns del software. Cubre los patrones de contenido más usados (**n-aria**, **parte-todo**, **rol**, **tiempo**), las **ontologías fundacionales** (upper ontologies), las estrategias de reuso —importar, modularizar, alinear— y los antipatrones que degradan un modelo. Navegá: [[_Ontology Engineering|Ontology Engineering]] · anterior [[04 - Modeling Decisions]] · siguiente [[06 - Evaluation and Testing]].

## Resumen

El capítulo anterior mostró que cada decisión de modelado tiene alternativas. Este capítulo aporta la respuesta práctica: **casi ningún problema de modelado es nuevo**. Los problemas recurrentes —representar una relación con más de dos participantes, modelar partes y todos, capturar que alguien juega un rol durante un período, manejar información que cambia en el tiempo— ya fueron resueltos, discutidos y refinados por la comunidad. Un **Ontology Design Pattern** es esa solución empaquetada: el problema que resuelve, la estructura de modelado que propone, y las consecuencias de adoptarla.

El valor de los ODPs va más allá del ahorro de tiempo. Un patrón trae **decisiones ya validadas** —los casos borde que a vos te faltan encontrar—, **vocabulario compartido** para discutir el diseño con otros ontologistas, e **interoperabilidad**: dos ontologías que usan el mismo patrón para el mismo problema son mucho más fáciles de alinear después.

El capítulo recorre los patrones de contenido más frecuentes y después sube un nivel para tratar las **ontologías fundacionales** —modelos abstractos de categorías generalísimas (objeto, evento, proceso, cualidad) sobre los que anclar el modelo de dominio—. Son útiles porque imponen coherencia y facilitan alineamiento, y son riesgosas porque arrastran compromisos filosóficos y complejidad que muchos proyectos no necesitan. Las autoras mantienen su postura pragmática de siempre: adoptalas si el caso lo justifica, no por prestigio.

Cierra con las **estrategias de reuso** —importación completa, modularización, alineamiento por mapeos— con sus costos respectivos, y con un catálogo de **antipatrones**: los errores estructurales que aparecen una y otra vez y que conviene reconocer antes de cometerlos.

## Qué es un Ontology Design Pattern

Un **ODP** es una solución reusable a un problema de modelado recurrente. La analogía con los design patterns del software es directa y las autoras la usan explícitamente: no es código que copiás, es una **forma de estructurar** que sabés que funciona.

Un patrón bien documentado incluye:

- **El problema** que resuelve, en términos de modelado.
- **La solución** — clases, propiedades y axiomas que propone.
- **Las consecuencias** — qué gana y qué cuesta adoptarlo.
- **Ejemplos** de uso en dominios concretos.

Los tipos principales:

- **Patrones de contenido (Content ODPs)** — los más usados. Resuelven un problema de modelado concreto: relaciones n-arias, parte-todo, roles, tiempo. Es lo que la gente quiere decir cuando dice "ODP".
- **Patrones estructurales / lógicos** — resuelven limitaciones expresivas del lenguaje (por ejemplo, cómo representar en [[OWL]] algo que OWL no soporta directamente).
- **Patrones de nomenclatura** — convenciones de naming e IRIs. Menos glamorosos, enormemente valiosos para la mantenibilidad.

> [!tip] Antes de resolver un problema de modelado que se siente difícil, buscá si ya existe un patrón. La probabilidad de que tu problema sea nuevo es baja, y la solución de la comunidad ya sobrevivió casos borde que vos todavía no viste.

### Dónde buscarlos

El capítulo recomienda buscar patrones sin decir dónde. El catálogo real:

| Recurso | Qué tiene |
|---|---|
| [ontologydesignpatterns.org](http://ontologydesignpatterns.org) | El portal de la comunidad; catálogo estructurado por tipo |
| [W3C — n-ary relations](https://www.w3.org/TR/swbp-n-aryRelations/) | Nota oficial sobre el patrón n-ario |
| [W3C — specified values](https://www.w3.org/TR/swbp-specified-values/) | Value partition vs value set |
| [PROV-O](https://www.w3.org/TR/prov-o/) | El vocabulario estándar de procedencia |
| [OBO Foundry](https://obofoundry.org) | Patrones de facto en biomedicina; plantillas DOSDP |

> [!note] Además de los cuatro patrones que el capítulo desarrolla, hay tres de alta frecuencia que conviene conocer: **value partition** (valores cualitativos como severidad alta/media/baja, sin caer en literales string), **listas y secuencias** (RDF no tiene listas ordenadas usables — `rdf:List` es una lista enlazada horrible de consultar), y **procedencia** (quién afirmó qué y cuándo, crítico si el grafo se puebla automáticamente). Todos con ejemplo en [[Ontology Design Patterns (ODP)]].

## Patrón: relación n-aria

El problema más frecuente de todos. [[OWL]] solo tiene relaciones **binarias** (sujeto-predicado-objeto), pero el mundo está lleno de relaciones con más participantes o con atributos propios.

Ejemplo típico: *"Juan trabajó en Acme como desarrollador desde 2019 hasta 2022 con un salario de X"*. Eso no es una propiedad binaria — hay persona, organización, rol, dos fechas y un monto, todos parte de un mismo hecho.

La solución del patrón: **reificar la relación**, convertirla en una entidad de pleno derecho.

```turtle
:Empleo_Juan_Acme a :Empleo ;
    :empleado        :Juan ;
    :empleador       :Acme ;
    :rolDesempeñado  :Desarrollador ;
    :fechaInicio     "2019-03-01"^^xsd:date ;
    :fechaFin        "2022-11-30"^^xsd:date ;
    :salario         [ :monto 85000 ; :moneda :USD ] .
```

> [!note] La clase *Empleo* no existía en el vocabulario del dominio — la introduce el modelo para poder decir todo lo que hay que decir sobre el hecho. Este es el movimiento central del patrón: **el hecho se vuelve entidad**.

> [!warning] La reificación tiene costo: las consultas se vuelven más largas (hay que atravesar el nodo intermedio) y el modelo menos legible. No reifiques por defecto — hacelo cuando la relación realmente tiene atributos propios o más de dos participantes. Si solo necesitás *"Juan trabaja en Acme"*, una propiedad binaria alcanza.

## Patrón: parte-todo (meronimia)

Ya advertido en [[03 - Terminology and Domain Analysis]] y [[04 - Modeling Decisions]]: la relación parte-todo **no es subsunción** y necesita su propio tratamiento.

Las distinciones que el patrón obliga a tomar:

- **¿Transitiva?** — un tornillo es parte de un motor, el motor es parte del auto: ¿el tornillo es parte del auto? Normalmente sí, pero no siempre (un dedo es parte de una mano, la mano parte del músico, ¿el dedo es parte del músico? sí — pero *la mano es parte de la orquesta* ya no funciona).
- **¿Separable?** — una rueda se puede quitar del auto; un ángulo no se puede quitar de un triángulo.
- **¿Componente funcional o porción de materia?** — un motor es un componente; una porción de agua es materia. Se comportan distinto ante la división.

> [!tip] La familia de propiedades habitual: `partOf` / `hasPart`, con subpropiedades más específicas (`componentOf`, `memberOf`, `substanceOf`) según el tipo de parte. Declarar transitividad **solo** en las subpropiedades donde realmente vale, no en la genérica.

> [!warning] Declarar `partOf` como transitiva de forma global es una decisión que parece inocente y produce inferencias absurdas en cuanto el modelo crece. La transitividad de la meronimia depende del **tipo** de relación parte-todo, y mezclar tipos bajo una sola propiedad transitiva garantiza problemas.

## Patrón: rol

El problema: *Juan* es una persona, pero también es *empleado*, *cliente* y *estudiante* — y esos papeles cambian con el tiempo, mientras que ser persona no.

La tentación es modelar *Empleado* como subclase de *Persona*. El patrón dice que casi siempre es un error:

- Una persona puede tener **varios roles simultáneos**, lo que genera herencia múltiple desordenada.
- Los roles son **temporales**; la clasificación en una jerarquía es permanente. Si Juan deja de ser empleado, habría que cambiarle la clase — lo cual contradice la idea de que la jerarquía captura lo esencial.
- Los roles tienen **atributos propios** (fecha de inicio, contexto, contraparte) que no caben en una relación de subclase.

> [!note] **La solución del patrón**: separar la entidad de su rol. *Persona* es la entidad; *Empleado* es un rol que la persona **desempeña** en un contexto y durante un período. El rol se modela como entidad propia (parientes del patrón n-ario), no como subclase.

> [!tip] El criterio de decisión: **¿la clasificación es esencial o contingente?** Si algo puede dejar de serlo sin dejar de ser lo que es, es un rol, no una subclase. Un Golden Retriever no puede dejar de ser perro; un empleado sí puede dejar de ser empleado.

#### El patrón implementado

```turtle
@prefix owl:  <http://www.w3.org/2002/07/owl#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix :     <https://ejemplo.org/onto#> .

:Persona      a  owl:Class .
:Rol          a  owl:Class .
:RolEmpleado   rdfs:subClassOf  :Rol .
:RolEstudiante rdfs:subClassOf  :Rol .

:desempeña    a  owl:ObjectProperty ;
              rdfs:domain  :Persona ;
              rdfs:range   :Rol .

# Juan es una Persona — eso es esencial y no cambia.
# Sus roles son entidades con contexto y período propios.
:juan  a  :Persona ;
       :desempeña  [ a       :RolEmpleado ;
                     :en     :Acme ;
                     :cargo  :Desarrollador ;
                     :desde  "2019-03-01"^^xsd:date ] ,
                   [ a       :RolEstudiante ;
                     :en     :Universidad ;
                     :desde  "2021-08-01"^^xsd:date ] .
```

Dos roles simultáneos, cada uno con su contexto y sus fechas. Si Juan deja de trabajar en Acme se cierra ese rol — **no se cambia la clase de Juan**, que es lo que habría que hacer con `:Empleado rdfs:subClassOf :Persona`.

> [!note] Compará con la alternativa ingenua: `:juan a :Empleado, :Estudiante` obliga a herencia múltiple, no admite fechas ni contexto, y exige reclasificar al individuo cada vez que su situación cambia. El patrón de rol es una indirección más a cambio de resolver los tres problemas.

## Patrón: tiempo y cambio

Modelar información que cambia es de lo más costoso en [[OWL]], porque el lenguaje no tiene noción nativa de tiempo. Los enfoques habituales:

- **Snapshot / versionado** — la ontología representa el estado actual y se versiona entera. Simple, pero pierde la historia.
- **Reificación temporal (fluents)** — cada hecho que cambia se reifica con su intervalo de validez. Es la solución completa y también la más pesada: multiplica entidades y complica cada consulta.
- **Time-indexed relations** — variante del n-ario donde el hecho lleva un intervalo asociado.

> [!warning] **Modelar tiempo multiplica la complejidad de todo el modelo.** Si tus [[competency questions]] son todas sobre el estado actual, no lo modeles. Si alguna pregunta por historia (*"¿quién era el responsable en 2021?"*), no hay escapatoria — pero acotá el alcance temporal a las entidades que realmente lo necesitan, no a todo el modelo.

### Tabla 5.1 — Patrones de contenido más frecuentes

| Patrón | Problema que resuelve | Solución | Costo principal |
|---|---|---|---|
| **N-ario** | Relación con >2 participantes o con atributos | Reificar el hecho como entidad | Consultas más largas, modelo menos legible |
| **Parte-todo** | Meronimia confundida con subsunción | Familia `partOf`/`hasPart` con subtipos | Decidir transitividad por subtipo |
| **Rol** | Clasificación temporal tratada como esencial | Rol como entidad, no como subclase | Una indirección más en el modelo |
| **Tiempo** | Hechos que cambian | Fluents / relaciones indexadas por tiempo | Multiplica entidades y complica todo |

> [!tip] Los cuatro comparten una misma estrategia de fondo: **cuando una relación binaria no alcanza, convertí el hecho en entidad**. Reconocer ese movimiento común hace que los cuatro patrones se aprendan casi como uno.

## Ontologías fundacionales (upper ontologies)

Un escalón de abstracción por encima: modelos de categorías generalísimas —objeto, evento, proceso, cualidad, rol, participación— pensados para que las ontologías de dominio se anclen en ellos. Los ejemplos citados habitualmente son **BFO**, **DOLCE** y **SUMO**.

Qué aportan:

- **Coherencia** — decisiones estructurales ya tomadas y consistentes entre sí.
- **Alineamiento** — dos ontologías ancladas en la misma fundacional son mucho más fáciles de integrar.
- **Distinciones filosóficas ya resueltas** — continuant vs occurrent, universal vs particular, dependencia existencial. Distinciones que a un modelador le tomaría meses formular bien.

Qué cuestan:

- **Compromiso filosófico** — adoptás la visión del mundo de esa fundacional, incluidas las decisiones que no compartís.
- **Complejidad** — agregan una capa de abstracción que todo el equipo tiene que entender.
- **Curva de aprendizaje** — no trivial, y difícil de justificar ante stakeholders.

> [!warning] Anclar en una ontología fundacional **por prestigio** es un error clásico. Si tu caso de uso es integrar tres sistemas internos, BFO es sobrecarga pura. Si estás construyendo un modelo que debe interoperar en un ecosistema científico donde la fundacional ya es estándar de facto, la decisión se invierte y no anclar es el error.

> [!note] Es el mismo criterio del [[espectro semántico]] aplicado un nivel más arriba: **el escalón correcto es el mínimo que resuelve tu caso de uso**. La ambición conceptual no es una virtud del modelo, es un costo.

## Estrategias de reuso

[[02 - Ontology Development Methodology]] estableció que reusar es la norma. Este capítulo detalla el **cómo**, porque hay más de una forma y no son equivalentes:

- **Importación completa (`owl:imports`)** — traés la ontología entera. Simple, y arrastra todo: clases irrelevantes, axiomas, compromisos de modelado y carga de razonamiento.
- **Modularización** — extraés solo el **módulo** que necesitás. Más trabajo, mucho más liviano. Existen técnicas de extracción de módulos que garantizan preservar las inferencias relevantes.
- **Alineamiento por mapeos** — mantenés tu vocabulario y declarás correspondencias hacia el externo (`owl:equivalentClass`, `rdfs:subClassOf`, `skos:exactMatch`). Desacopla, al costo de mantener el mapeo vivo.
- **Referencia sin importación** — usás los IRIs del vocabulario externo sin importar sus axiomas. Ganás interoperabilidad de identificadores sin heredar la semántica.

> [!tip] **Por defecto, alineamiento antes que importación** cuando la ontología externa es grande y solo necesitás una fracción. La importación completa es cómoda al principio y se vuelve una deuda cuando la dependencia evoluciona en una dirección que no te sirve.

> [!warning] Una importación crea una **dependencia versionada**, exactamente como una librería de software. Si el vocabulario externo cambia, tu modelo hereda el cambio y sus consecuencias inferenciales. Fijá la versión importada y actualizá deliberadamente, nunca por arrastre.

### Dónde buscar qué reusar

El otro catálogo que el libro omite y sin el cual "buscá ontologías existentes" es inaplicable:

| Recurso | Qué tiene |
|---|---|
| [LOV](https://lov.linkeddata.es/dataset/lov) | Linked Open Vocabularies: buscador de vocabularios RDF **por término**. El primer lugar donde mirar |
| [schema.org](https://schema.org) | El vocabulario más usado de la web: productos, eventos, personas, organizaciones |
| [Dublin Core](https://www.dublincore.org) | Metadatos documentales — casi siempre parte de la respuesta |
| [PROV-O](https://www.w3.org/TR/prov-o/) | Procedencia |
| [FOAF](http://xmlns.com/foaf/spec/) | Personas y relaciones |
| [QUDT](https://qudt.org) | Cantidades, unidades y dimensiones |
| [BioPortal](https://bioportal.bioontology.org) | ~1.000 ontologías biomédicas |
| [OBO Foundry](https://obofoundry.org) | Ontologías biomédicas coordinadas con principios comunes |
| [AGROVOC](https://agrovoc.fao.org/) | FAO: agricultura, alimentación, pesca. Multilingüe |
| [Wikidata](https://www.wikidata.org) | Entidades del mundo real como IRIs ya reusables |

> [!tip] Para extraer un módulo mínimo de una ontología grande sin arrastrar todo, `robot extract` lo hace preservando las inferencias relevantes. Ver [[Herramental de ontologías]].

## Antipatrones

El capítulo cierra con los errores estructurales recurrentes — reconocerlos vale tanto como conocer los patrones:

- **Subsunción usada para todo** — meronimia, roles y estados colgados de la jerarquía. El origen del mayor número de inferencias absurdas.
- **Jerarquía profunda sin justificación** — niveles intermedios que no introducen propiedades ni responden preguntas.
- **Clases con una sola instancia posible** — normalmente eso debería ser un individuo, no una clase.
- **Propiedades genéricas tipo `tieneAtributo`** — reproducen un modelo entidad-atributo-valor dentro de OWL y anulan toda capacidad de razonamiento.
- **Disjunción declarada por reflejo** — produce clases insatisfacibles en cuanto el modelo crece.
- **Modelar la aplicación en vez del dominio** — el error raíz que [[01 - Introduction]] ya señalaba, y que reaparece disfrazado en cada nivel.

> [!tip] Corré el razonador después de cada bloque de modelado, no al final. Los antipatrones se detectan mucho mejor por sus **consecuencias inferenciales** —clases insatisfacibles, jerarquías inferidas absurdas— que por inspección visual del modelo.

## Para aplicar

- **Buscá un ODP existente antes de resolver a mano** cualquier problema de modelado que se sienta difícil.
- **Reificá cuando la relación tenga atributos propios o más de dos participantes** — y solo entonces.
- **Separá roles de la jerarquía**: si algo puede dejar de serlo sin dejar de ser lo que es, es rol, no subclase.
- **No modeles tiempo salvo que una competency question lo exija**, y acotá el alcance temporal a las entidades que lo necesitan.
- **Declará transitividad de `partOf` por subtipo**, nunca globalmente.
- **Preferí modularizar o alinear antes que importar completo** cuando la ontología externa es grande.
- **Fijá la versión de toda ontología importada** y actualizá deliberadamente.
- **Revisá el catálogo de antipatrones antes de cada release** — es una checklist barata que evita errores caros.

## Conexiones

- [[_Ontology Engineering|Ontology Engineering]] — el MOC del libro.
- [[04 - Modeling Decisions]] — capítulo anterior: las decisiones que estos patrones empaquetan · [[06 - Evaluation and Testing]] — capítulo siguiente: cómo verificar que el modelo resultante funciona.
- [[Ontology Design Patterns (ODP)]] — **candidato fuerte a nota propia** con el catálogo de patrones.
- [[OWL]] — las limitaciones expresivas que varios de estos patrones existen para sortear.
- [[SKOS]] — alternativa liviana cuando el caso de uso no justifica patrones complejos.
- [[espectro semántico]] — el criterio de "el escalón mínimo que resuelve el caso", acá aplicado a fundacionales.
- [[Design Patterns]] — el paralelo directo en ingeniería de software: mismo concepto, otro dominio.
