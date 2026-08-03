---
title: OWL
source: (conocimiento general + especificación W3C OWL 2 — verificar sintaxis y perfiles contra la spec antes de citar)
author: W3C
created: 2026-08-03
tags:
  - semantic-web
  - type/technology
  - status/stub
aliases:
  - OWL
  - OWL 2
  - Web Ontology Language
---

# OWL

> [!note] Definición
> **OWL** (*Web Ontology Language*) es el estándar del W3C para escribir [[ontología|ontologías]] formales: declarar clases, propiedades y **axiomas lógicos** sobre un dominio, de modo que un **razonador** pueda deducir hechos que nadie escribió, clasificar individuos automáticamente y detectar contradicciones.

Está fundado en **[[Description Logic|lógicas descriptivas]]**, una familia de lógicas diseñadas para equilibrar expresividad y decidibilidad. Esa base es lo que separa a OWL de un simple esquema: no describe cómo se guardan los datos, describe qué es **verdadero** en el dominio — y de ahí se computan consecuencias.

La versión vigente es **OWL 2** (2009, revisada en 2012). [[01 - Introduction|Deborah McGuinness]] —coautora de las notas de este vault sobre *Ontology Engineering*— es coautora de la especificación original.

## Qué agrega sobre RDFS

[[RDFS]] ya da clases, jerarquía, `domain` y `range`. OWL agrega el poder expresivo que hace posible el razonamiento real:

| Capacidad | RDFS | OWL |
|---|:---:|:---:|
| Clases y subclases | ✅ | ✅ |
| Jerarquía de propiedades | ✅ | ✅ |
| Disjunción entre clases | ❌ | ✅ |
| Cardinalidades | ❌ | ✅ |
| Restricciones de propiedad | ❌ | ✅ |
| Clases **definidas** (condiciones suficientes) | ❌ | ✅ |
| Propiedades transitivas, simétricas, inversas | ❌ | ✅ |
| Detección de inconsistencias | ❌ | ✅ |

## Clases primitivas vs definidas

Es **la distinción de mayor rendimiento** de todo el lenguaje, y la que separa una ontología que trabaja para vos de una taxonomía dibujada a mano.

- **Clase primitiva** — se describe con **condiciones necesarias**. Todo `Vino` tiene productor y añada, pero cumplir eso no te convierte en Vino. La pertenencia se **afirma** a mano.
- **Clase definida** — se describe con **condiciones necesarias y suficientes**. `VinoTinto` ≡ todo vino cuyo color es tinto. Cualquier individuo que cumpla la condición **es** un VinoTinto, lo haya dicho alguien o no.

```turtle
# Primitiva: condición necesaria (rdfs:subClassOf)
:Vino rdfs:subClassOf [
    a owl:Restriction ;
    owl:onProperty :tieneProductor ;
    owl:someValuesFrom :Bodega
] .

# Definida: necesaria Y suficiente (owl:equivalentClass)
:VinoTinto owl:equivalentClass [
    owl:intersectionOf (
        :Vino
        [ a owl:Restriction ; owl:onProperty :tieneColor ; owl:hasValue :Tinto ]
    )
] .
```

> [!note] **Las clases definidas son la razón para usar OWL.** Declarás las condiciones una vez y el razonador clasifica solo cada individuo y cada subclase que las cumpla. Sin ellas, toda la jerarquía es manual y el razonador apenas verifica consistencia — estás pagando la complejidad de OWL sin cobrar su beneficio.

> [!tip] Patrón habitual: jerarquía **primitiva** con las distinciones esenciales del dominio (las que no dependen de propiedades variables) + clases **definidas** para las categorías derivadas — `ClienteMoroso`, `PacienteDeAltoRiesgo`, `PedidoUrgente`. Esas categorías se recalculan solas cuando cambian los datos.

> [!warning] Marcar como definida una clase cuyas condiciones **no** son realmente suficientes produce clasificaciones erróneas silenciosas: el razonador mete individuos donde no corresponde y nadie lo nota hasta que una consulta devuelve basura. Ante la duda, primitiva — el error de una primitiva es un hueco visible; el de una definida es invisible.

## Restricciones de propiedad

Acotan qué valores puede tomar una propiedad para los miembros de una clase:

| Restricción | Significado |
|---|---|
| `owl:someValuesFrom` | **Existencial**: al menos un valor de ese tipo |
| `owl:allValuesFrom` | **Universal**: *si* hay valores, son de ese tipo |
| `owl:hasValue` | Debe incluir un valor concreto |
| `owl:minCardinality` / `maxCardinality` / `cardinality` | Cuántos valores |

> [!warning] **La restricción universal NO implica existencia.** Declarar que *"todos los ingredientes de una PizzaVegetariana son vegetales"* lo satisface trivialmente una pizza **sin ningún ingrediente**. Es el error clásico del tutorial de la pizza y sigue siendo el más cometido. Si querés exigir que haya al menos un vegetal, necesitás **también** una restricción existencial.

## Características de propiedades

En OWL las propiedades no son campos: son relaciones con semántica formal, y cada característica **habilita una inferencia**.

| Característica | Qué afirma | Ejemplo | Qué infiere el razonador |
|---|---|---|---|
| **Funcional** | A lo sumo un valor | `:tieneMadreBiológica` | Dos valores distintos ⇒ son el mismo individuo |
| **Inversa funcional** | El valor identifica al sujeto | `:númeroDocumento` | Dos sujetos con igual valor ⇒ son el mismo |
| **Transitiva** | Se encadena | `:esParteDe` | Parte de una parte es parte del todo |
| **Simétrica** | Vale en ambos sentidos | `:esHermanoDe` | A→B implica B→A |
| **Asimétrica** | Nunca al revés | `:esPadreDe` | A→B y B→A ⇒ inconsistencia |
| **Inversa** | Relación opuesta | `:tieneAutor` / `:esAutorDe` | Navegación bidireccional |
| **Reflexiva / Irreflexiva** | Se relaciona consigo mismo, o nunca | `:conoce` / `:esPadreDe` | — |

> [!tip] Declarar transitividad o funcionalidad **cuesta tiempo de razonamiento**. En ontologías grandes, características declaradas "por si acaso" degradan seriamente la clasificación. Declaralas cuando habiliten una inferencia que alguna [[competency questions|competency question]] necesita.

## Las dos trampas semánticas

Son la causa de la mayoría de la frustración de quien llega desde bases de datos.

### Mundo abierto (open world assumption)

> [!warning] **Lo que no está declarado es desconocido, no falso.** Si no cargaste el teléfono de un empleado, OWL no concluye que no tiene teléfono: concluye que **todavía no sabemos** cuál es. Por eso OWL **nunca** reporta un dato faltante como error. Si necesitás validar completitud, la herramienta es [[SHACL]], no OWL.

Corolario práctico: `rdfs:domain` y `rdfs:range` **no validan, infieren**. Declarás que el dominio de `:trabajaEn` es `:Persona`, afirmás que una silla `:trabajaEn` una oficina, y el razonador **deduce que la silla es una Persona**. No hay error.

### Sin nombres únicos (no unique name assumption)

Dos IRIs distintos pueden referirse al **mismo** individuo salvo que se declare lo contrario (`owl:differentFrom`, `owl:AllDifferent`). Combinado con propiedades funcionales, produce inferencias sorprendentes: si `:legajo` es funcional y un empleado tiene `"A-1"` y `"B-2"`, el razonador concluye que **esos dos valores son la misma cosa**.

## Disjunción — el axioma más subestimado

`owl:disjointWith` declara que dos clases no comparten instancias. Sin él, OWL **no asume** que las clases sean distintas: mientras nadie lo prohíba, un individuo podría pertenecer a ambas.

Es indispensable para que el razonador detecte contradicciones reales. Y es peligroso declarado a la ligera: produce **clases insatisfacibles** — clases que por sus axiomas no pueden tener ninguna instancia.

> [!warning] **Una clase insatisfacible es siempre un error de modelado, nunca una decisión de diseño.** El caso trabajado está en [[02 - Ontology Development Methodology]]: tres axiomas que el experto validó uno por uno se combinan para declarar que un producto real es imposible. Nadie afirmó eso — se dedujo.

## Los perfiles de OWL 2

OWL 2 DL completo tiene complejidad de razonamiento **muy alta** en el peor caso. Los perfiles acotan la expresividad a cambio de garantías computacionales:

| Perfil | Optimizado para | Complejidad | Uso típico |
|---|---|---|---|
| **EL** | Ontologías enormes con jerarquías profundas | Polinomial | Biomédicas (SNOMED CT, Gene Ontology) |
| **QL** | Consulta sobre bases de datos grandes | Muy baja (reescritura a SQL) | Integración de datos, OBDA |
| **RL** | Razonamiento por reglas sobre muchos datos | Polinomial | Triple stores, reglas de negocio |
| **DL** | Máxima expresividad decidible | N2EXPTIME (peor caso) | Ontologías chicas o medianas, modelado rico |

> [!note] **Elegir perfil y elegir razonador es la misma decisión desde dos lados.** ELK solo soporta EL, pero es órdenes de magnitud más rápido que HermiT o Pellet — es lo que hace viable clasificar SNOMED CT. Decidí el perfil **temprano** y modelá dentro de él; descubrir a los seis meses que necesitás bajar de DL a EL implica rehacer axiomas.

> [!tip] El rendimiento depende de **qué construcciones usás**, no de cuántas clases tenés. Una ontología de mil clases con axiomas complejos puede ser mucho más lenta que una de cien mil en perfil EL.

## Sintaxis: hay varias

OWL es un **modelo**, no un formato. Se serializa de varias maneras:

- **Turtle** — legible, la mejor para control de versiones y trabajo humano.
- **RDF/XML** — la serialización normativa original. Verbosa y con diffs horribles.
- **Manchester Syntax** — pensada para humanos, sin prefijos ni tripletas: `VinoTinto EquivalentTo Vino and tieneColor value Tinto`. Es la que se ve en [[Protégé]].
- **Functional Syntax**, **OWL/XML**, **JSON-LD** — otras variantes.

> [!tip] Turtle para el repositorio, Manchester para leer y discutir con humanos. Evitá RDF/XML salvo que una herramienta lo exija: sus diffs hacen inútil la revisión de cambios en git.

## Razonadores

| Razonador | Perfil | Nota |
|---|---|---|
| **HermiT** | OWL 2 DL | Completo, el más usado en Protégé |
| **Pellet** | OWL 2 DL | Completo, con buenas explicaciones |
| **ELK** | OWL 2 EL | Órdenes de magnitud más rápido; hace viables las ontologías biomédicas |
| Embebidos en triple stores | RDFS / RL | Optimizados para volumen, no para expresividad |

Lo que un razonador hace: **chequeo de consistencia**, **detección de clases insatisfacibles**, **jerarquía inferida** y **clasificación de individuos**.

> [!tip] Comparar la jerarquía **declarada** con la **inferida** es el mejor instrumento de revisión que existe. Si aparecen subclases que no esperabas, o dos clases resultan equivalentes sin que lo quisieras, encontraste un error de modelado que ninguna inspección visual del archivo habría mostrado.

## OWL vs SHACL — no compiten

| | OWL | [[SHACL]] |
|---|---|---|
| **Qué hace** | Infiere hechos nuevos | Valida los hechos que hay |
| **Semántica** | Mundo abierto | Mundo cerrado |
| **Dato ausente** | Desconocido | **Violación** |
| **Pregunta** | *¿Qué se deduce de esto?* | *¿Estos datos sirven?* |

> [!note] El flujo habitual en producción los usa a los dos: razonar primero con OWL para que las clasificaciones inferidas existan en el grafo, validar después con SHACL sobre el grafo enriquecido.

## Cuándo NO usar OWL

Tan importante como saber usarlo:

- **Si nadie va a correr un razonador** — [[SKOS]] resuelve taxonomías y thesauri a una fracción del costo de construcción y mantenimiento.
- **Si tu necesidad es validar datos** — es [[SHACL]], y OWL te va a frustrar por diseño.
- **Si el caso de uso es solo consultar un grafo** — [[RDF]] + [[SPARQL]] con property paths alcanza.

> [!warning] La **sobre-formalización** es una de las causas de fracaso de proyectos de ontología ([[08 - Tools and Practical Considerations]]): subir al escalón OWL del [[espectro semántico]] cuando alcanzaba SKOS significa pagar complejidad, tiempo de modelado y costo de razonamiento por capacidad que nadie va a usar.

## Errores comunes

- **Esperar que OWL valide** — no lo hace, y no por limitación sino por diseño.
- **`allValuesFrom` sin `someValuesFrom`** — el error de la pizza: la restricción universal la satisface el conjunto vacío.
- **Disjunción declarada por reflejo** — genera clases insatisfacibles apenas el modelo crece.
- **Todo primitivo** — jerarquía manual, razonador desaprovechado.
- **Definidas con condiciones insuficientes** — clasificaciones erróneas silenciosas.
- **Meronimia como subsunción** — *rueda* subclase de *auto* hace que el razonador infiera que toda rueda es un auto.
- **Instancias masivas dentro de la ontología** — la ontología modela el esquema; los datos van al triple store.
- **Ignorar el perfil hasta que el razonamiento explota** — para entonces rehacer axiomas es caro.

## Conexión en el vault

- Es el escalón más alto del [[espectro semántico]] y el que habilita razonamiento — la capacidad que justifica su costo.
- Se consulta con [[SPARQL]] (que **no** razona por sí solo) y se complementa con [[SHACL]] para validación.
- Las decisiones de modelado que este lenguaje expresa están desarrolladas en [[04 - Modeling Decisions]]; los patrones que sortean sus limitaciones expresivas, en [[05 - Ontology Design Patterns and Reuse]].
- El razonador es la herramienta central de [[06 - Evaluation and Testing]]: convierte las consecuencias ocultas de tus axiomas en algo que un humano puede refutar.
- `owl:versionIRI`, `owl:deprecated` y `owl:imports` son los mecanismos de versionado que usa [[07 - Lifecycle, Versioning and Governance]].

## References

- [OWL 2 Web Ontology Language — Document Overview](https://www.w3.org/TR/owl2-overview/) — W3C Recommendation
- [OWL 2 Primer](https://www.w3.org/TR/owl2-primer/) — la mejor puerta de entrada
- [OWL 2 Profiles](https://www.w3.org/TR/owl2-profiles/) — EL, QL, RL
- [OWL 2 Manchester Syntax](https://www.w3.org/TR/owl2-manchester-syntax/)

## Related

- [[RDF]]
- [[RDFS]]
- [[SHACL]]
- [[SPARQL]]
- [[SKOS]]
- [[Description Logic]]
- [[espectro semántico]]
- [[Protégé]]
- [[_Ontology Engineering|Ontology Engineering]]
