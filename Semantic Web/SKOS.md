---
title: SKOS
source: (SKOS Simple Knowledge Organization System Reference / Primer, W3C Recommendation 2009)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/concept
  - status/done
aliases:
  - SKOS
  - Simple Knowledge Organization System
updated: 2026-08-04
---

# SKOS

> [!note] Qué es
> **SKOS** (*Simple Knowledge Organization System*) es el estándar W3C para representar **taxonomías, thesauri, esquemas de clasificación y vocabularios controlados**. Es deliberadamente **menos expresivo que [[OWL]]** — y esa es exactamente su virtud: mucho más barato de construir, acordar y mantener.

> [!tip] Es el escalón que la mayoría de los proyectos necesita y casi ninguno elige. La literatura empuja hacia OWL por prestigio; el criterio correcto es el [[espectro semántico]]: **si ninguna [[competency questions|competency question]] requiere inferencia lógica, SKOS alcanza**.

## El modelo: conceptos, no clases

La diferencia de fondo con OWL, y la fuente de casi toda la confusión:

| | SKOS | OWL |
|---|---|---|
| **Unidad** | `skos:Concept` — una **idea** en un vocabulario | `owl:Class` — un **conjunto** de individuos |
| **Jerarquía** | `skos:broader` / `narrower` — organización, sin semántica lógica | `rdfs:subClassOf` — **subsunción**: toda instancia de A es instancia de B |
| **Se puede razonar** | Muy poco (transitividad opcional, poco más) | Clasificación, consistencia, inferencia completa |
| **Costo de construcción** | Bajo | Alto |
| **Quién lo construye** | Bibliotecarios, expertos de dominio, editores de contenido | Ingenieros de conocimiento |

> [!warning] **`skos:broader` NO es `rdfs:subClassOf`.** Es la confusión más frecuente. `broader` dice *"este concepto es más específico que aquel"* — una relación de organización del vocabulario. `subClassOf` dice *"toda instancia de esto es también instancia de aquello"* — una afirmación lógica con consecuencias inferenciales. Un razonador OWL no deduce nada útil de `broader`, y está bien: no es su propósito.

## Las construcciones

### Etiquetas — la separación término / concepto

Acá SKOS brilla, y es su aporte más valioso: implementa directamente la distinción entre **término** (etiqueta lingüística) y **concepto** (unidad de significado).

```turtle
@prefix skos: <http://www.w3.org/2004/02/skos/core#> .

:c_notebook  a  skos:Concept ;
    skos:prefLabel   "notebook"@es ,
                     "laptop"@en ;              # UNA preferida POR IDIOMA
    skos:altLabel    "portátil"@es ,
                     "computadora portátil"@es ,
                     "notebook computer"@en ;
    skos:hiddenLabel "notbook"@es ;             # errores frecuentes, para el buscador
    skos:definition  "Computadora personal portátil con pantalla y batería integradas."@es .
```

| Propiedad | Uso |
|---|---|
| `skos:prefLabel` | La etiqueta canónica. **Una sola por idioma** — es la única restricción fuerte de SKOS |
| `skos:altLabel` | Sinónimos, variantes, abreviaturas. Lo que hace funcionar la búsqueda |
| `skos:hiddenLabel` | Errores de tipeo y variantes que no querés mostrar pero sí indexar |

> [!tip] **El multilingüismo es nativo y gratuito.** Un concepto, etiquetas en cada idioma con su tag `@es` / `@en`. Es la razón principal para elegir SKOS cuando trabajás en español sobre estándares en inglés: el concepto es uno solo, las etiquetas conviven.

> [!warning] Cuidado con asumir que la partición conceptual es la misma entre idiomas. El inglés *policy* cubre lo que en español se reparte entre *póliza* y *política*. Cuando los conceptos no se corresponden uno a uno, no fuerces la etiqueta — son conceptos distintos, con `skos:closeMatch` entre ellos si acaso.

### Relaciones semánticas

```turtle
:c_notebook  skos:broader  :c_computadora .    # más general
:c_computadora  skos:narrower  :c_notebook .   # inversa (declarativa, no inferida)
:c_notebook  skos:related  :c_cargador .       # asociativa, no jerárquica
```

- **`skos:broader` / `skos:narrower`** — jerarquía. Son inversas entre sí, pero SKOS **no las infiere**: si querés ambas direcciones, declaralas.
- **`skos:related`** — asociación no jerárquica. Simétrica.
- **`skos:broaderTransitive` / `narrowerTransitive`** — las versiones transitivas, cuando necesitás recorrer la jerarquía completa.

> [!warning] `skos:related` y `skos:broader` son **disjuntas** por definición del estándar: dos conceptos no pueden estar en relación jerárquica y asociativa a la vez. Es la única regla que un validador SKOS chequea seriamente.

### Esquemas de concepto

```turtle
:vocabProductos  a  skos:ConceptScheme ;
    dct:title  "Vocabulario de productos"@es ;
    skos:hasTopConcept  :c_computadora , :c_periferico .

:c_notebook  skos:inScheme  :vocabProductos .
```

Un `ConceptScheme` agrupa conceptos en un vocabulario nombrado. Permite que un mismo concepto participe de varios esquemas — común cuando distintas áreas organizan el mismo dominio de forma distinta.

### Mapeos entre vocabularios

Lo que hace a SKOS especialmente bueno para integración:

| Propiedad | Significado |
|---|---|
| `skos:exactMatch` | Intercambiables en todo contexto |
| `skos:closeMatch` | Suficientemente parecidos para la mayoría de los usos |
| `skos:broadMatch` / `narrowMatch` | Uno es más general que el otro, en otro vocabulario |
| `skos:relatedMatch` | Asociados, sin jerarquía |

> [!tip] Los mapeos SKOS son la forma más barata de alinear tu vocabulario con uno externo **sin importarlo**. Mantenés el tuyo, declarás las correspondencias, y ganás interoperabilidad sin heredar la complejidad ajena. Ver estrategias de reuso en [[Ontology Design Patterns (ODP)]].

## Cuándo SKOS y cuándo OWL

| Necesitás | Usá |
|---|---|
| Vocabulario acordado, navegación, búsqueda con sinónimos | **SKOS** |
| Taxonomía para clasificar contenido o facetar | **SKOS** |
| Multilingüismo con conceptos compartidos | **SKOS** |
| Alinear varios vocabularios existentes | **SKOS** (mapeos) |
| Que el sistema **deduzca** categorías desde propiedades | **OWL** (clases definidas) |
| Detectar contradicciones lógicas en el modelo | **OWL** (razonador) |
| Restricciones, cardinalidades, disjunción | **OWL** |
| Validar que los datos cumplen una forma | **[[SHACL]]** |

> [!note] **No son excluyentes.** Un patrón muy común y muy sano: SKOS para el vocabulario de dominio —los conceptos que la gente nombra— y OWL para el modelo estructural que los relaciona. Se conectan asociando conceptos SKOS a individuos o clases OWL según el caso.

## El error de sobre-formalizar

> [!warning] Construir OWL con razonamiento donde alcanzaba SKOS es **una de las causas documentadas de fracaso** de proyectos de ontología. Pagás complejidad de modelado, curva de aprendizaje del equipo, tiempo de razonamiento y costo de mantenimiento — por capacidad que nadie va a usar.

La pregunta que lo decide: **¿alguna de tus competency questions requiere que el sistema deduzca algo que nadie declaró?** Si todas se responden recuperando y navegando lo que ya está afirmado, SKOS es la respuesta correcta y OWL es sobre-ingeniería.

## Vocabularios SKOS relevantes

- **AGROVOC** (FAO) — agricultura, alimentación, pesca. ~40.000 conceptos, multilingüe. Relevante para dominio agro.
- **EuroVoc** — vocabulario multilingüe de la Unión Europea.
- **Library of Congress Subject Headings** — publicado en SKOS.
- **UNESCO Thesaurus** — educación, ciencia, cultura.
- **GEMET** — terminología ambiental europea.

> [!tip] Buscá en [LOV](https://lov.linkeddata.es/dataset/lov) antes de construir: para dominios establecidos suele existir ya un vocabulario SKOS mantenido, y adoptarlo te da interoperabilidad gratis.

## Conexión en el vault

- [[espectro semántico]] — SKOS ocupa los escalones de taxonomía y thesaurus; entender el espectro es entender por qué SKOS y OWL no compiten.
- [[03 - Terminology and Domain Analysis]] — la distinción término/concepto que SKOS implementa directamente con `prefLabel`/`altLabel`.
- [[OWL]] — el escalón de arriba: cuándo vale la pena subir.
- [[competency questions]] — el criterio que decide entre uno y otro.
- [[Knowledge graph]] — muchos knowledge graphs industriales usan SKOS para su capa de vocabulario.

## References

- [SKOS Simple Knowledge Organization System Reference](https://www.w3.org/TR/skos-reference/) — W3C Recommendation, 2009.
- [SKOS Primer](https://www.w3.org/TR/skos-primer/) — W3C Working Group Note.
- [AGROVOC](https://agrovoc.fao.org/) — FAO, thesaurus agrícola multilingüe.

## Related

- [[espectro semántico]]
- [[OWL]]
- [[RDFS]]
- [[competency questions]]
- [[03 - Terminology and Domain Analysis]]
- [[Ontology Design Patterns (ODP)]]
