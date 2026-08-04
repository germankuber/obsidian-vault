---
title: espectro semántico
source: (Kendall & McGuinness 2019; la formulación original circula desde ~2004 en la comunidad de Semantic Web)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/concept
  - status/done
aliases:
  - espectro semántico
  - semantic spectrum
  - Semantic Spectrum
  - continuo de expresividad
updated: 2026-08-04
---

# espectro semántico

> [!note] Definición
> Los artefactos que capturan el vocabulario de un dominio **no son binarios** (ontología / no-ontología): forman un **continuo de expresividad creciente**. En cada escalón se puede decir más sobre el dominio y una máquina puede inferir más — y simultáneamente sube el **costo** de construir, acordar y mantener.

Es la idea que ordena todo el campo, y la que evita el error más caro de estos proyectos: construir el escalón equivocado.

## Los escalones

| Escalón | Qué agrega | Puede inferir la máquina | Estándar típico | Caso de uso representativo |
|---|---|---|---|---|
| **Vocabulario controlado** | Términos acordados | Nada (solo validación de pertenencia) | Listas, code sets, `enum` | Normalizar los valores de un campo |
| **Glosario** | Definiciones en lenguaje natural | Nada | Documento, [[SKOS]] parcial | Alinear el entendimiento humano |
| **Taxonomía** | Jerarquía *es-un-tipo-de* | Herencia por subsunción | [[SKOS]], [[RDFS]] | Navegación y clasificación por facetas |
| **Thesaurus** | Sinónimos, broader/narrower, related | Expansión de consultas | [[SKOS]] | Búsqueda que tolera sinónimos |
| **Modelo conceptual** | Clases, propiedades tipadas, cardinalidades | Validación estructural | [[RDFS]], [[OWL]] básico, [[SHACL]] | Integración de esquemas |
| **Ontología lógica** | Axiomas formales, clases definidas, disjunción | Clasificación, consistencia, hechos nuevos | [[OWL]] 2 | Inferencia y razonamiento automático |

> [!note] **Expresividad e inferencia crecen juntas — y el costo con ellas.** No hay un escalón "mejor" en abstracto. Hay un escalón adecuado para cada pregunta que el sistema tenga que responder.

## El mismo concepto en cada escalón

La tabla anterior compara casos de uso distintos, lo que oculta el salto real. Tomemos **un solo concepto** —*Producto*— y subamos:

**1. Vocabulario controlado** — una lista cerrada de valores permitidos.

```
Producto ∈ { "notebook", "monitor", "teclado", "mouse" }
```

Resuelve que tres equipos no escriban `notebook`, `Notebook` y `laptop` para la misma cosa. No sabe nada más.

**2. Glosario** — cada término con definición en prosa.

```
notebook: computadora portátil con pantalla integrada y batería propia.
monitor:  dispositivo de salida de video sin capacidad de cómputo.
```

Un humano ahora distingue los casos. Una máquina sigue sin poder hacer nada con la definición.

**3. Taxonomía** — aparece la primera estructura explotable.

```turtle
:Notebook  rdfs:subClassOf  :Computadora .
:Monitor   rdfs:subClassOf  :Periferico .
:Teclado   rdfs:subClassOf  :Periferico .
```

Una consulta por `:Periferico` ahora devuelve monitores y teclados sin que nadie los haya enumerado. Eso es inferencia por subsunción.

**4. Thesaurus** — se agregan las relaciones no jerárquicas y las etiquetas alternativas.

```turtle
:Notebook  a               skos:Concept ;
           skos:prefLabel  "notebook"@es ;
           skos:altLabel   "laptop"@es , "portátil"@es ;
           skos:broader    :Computadora ;
           skos:related    :Cargador .
```

Una búsqueda por *"laptop"* ahora recupera notebooks. Es el escalón que resuelve la búsqueda semántica barata.

**5. Modelo conceptual** — propiedades tipadas y cardinalidades.

```turtle
:Producto      a               owl:Class .
:tienePrecio   a               owl:DatatypeProperty ;
               rdfs:domain     :Producto ;
               rdfs:range      xsd:decimal .
:fabricadoPor  a               owl:ObjectProperty ;
               rdfs:domain     :Producto ;
               rdfs:range      :Fabricante .
```

Ahora el modelo sabe qué atributos tiene un producto y de qué tipo. Sigue sin deducir categorías.

**6. Ontología lógica** — clases definidas, y el razonador empieza a trabajar.

```turtle
:ProductoPremium  a  owl:Class ;
    owl:equivalentClass [
        a               owl:Class ;
        owl:intersectionOf (
            :Producto
            [ a owl:Restriction ;
              owl:onProperty     :tienePrecio ;
              owl:someValuesFrom [ a rdfs:Datatype ;
                                   owl:onDatatype xsd:decimal ;
                                   owl:withRestrictions ( [ xsd:minInclusive 2000 ] ) ] ]
        )
    ] .
```

Nadie declara qué producto es premium: **el razonador lo clasifica solo** a partir del precio, y reclasifica cuando el precio cambia. Esa capacidad —y solo esa— es lo que justifica haber subido hasta acá.

> [!tip] Releé la progresión al revés y vas a ver el costo: cada escalón exige más acuerdo humano, más tiempo de modelado y más mantenimiento. El escalón 6 no es "mejor" que el 4 — es más caro, y se paga solo si necesitás lo que hace.

## La regla de decisión

> [!tip] **Elegí el escalón por el caso de uso, no por ambición.** Si tu problema es que tres equipos llaman distinto a la misma cosa, un vocabulario controlado lo resuelve. Si necesitás que el sistema deduzca que un paciente cumple los criterios de un ensayo clínico sin que nadie lo haya declarado, ahí sí necesitás una ontología lógica.

El instrumento concreto para decidir son las [[competency questions]]: si ninguna pregunta requiere inferencia, no necesitás [[OWL]].

## Los dos errores simétricos

> [!warning] **Sobre-formalizar** — construir OWL con razonamiento donde alcanzaba [[SKOS]]. Pagás complejidad, tiempo de modelado y costo de mantenimiento por capacidad que nadie va a usar. Es una de las causas documentadas de fracaso de proyectos de ontología.

> [!warning] **Sub-formalizar** — quedarse en taxonomía cuando el caso de uso pide inferencia. La lógica faltante termina **hardcodeada en la aplicación**, dispersa y duplicada en cada sistema consumidor. Es más barato al principio y mucho más caro a largo plazo, porque se pierde justamente el beneficio de tener el significado en un lugar declarativo y único.

> [!note] La asimetría importante: **sobre-formalizar se nota enseguida** (el proyecto se atrasa, el razonador tarda, nadie entiende el modelo); **sub-formalizar se nota años después**, cuando la misma regla de negocio está implementada de tres formas distintas en tres sistemas y ninguna coincide.

## Cómo se mueve un proyecto por el espectro

En la práctica casi nadie arranca en el escalón 6. El recorrido típico:

1. **Glosario acordado** — el trabajo de [[03 - Terminology and Domain Analysis]]. Ya entrega valor: la organización explicita qué significan sus términos.
2. **Taxonomía SKOS** — cuando aparece la necesidad de navegar o buscar.
3. **Modelo conceptual** — cuando hay que integrar esquemas de varias fuentes.
4. **Axiomas OWL selectivos** — y acá la clave: **no todo el modelo sube**. Se agregan clases definidas solo donde el razonamiento aporta.

> [!tip] El espectro no obliga a elegir un escalón para toda la ontología. Un modelo real suele ser taxonomía plana en el 80% de sus clases y lógica formal en el 20% donde la inferencia rinde. Subir selectivamente es la estrategia correcta.

## El mismo criterio, un nivel más arriba

La lógica del espectro se repite en la decisión sobre **ontologías fundacionales** (BFO, DOLCE, SUMO): anclar en una upper ontology agrega coherencia y capacidad de alineamiento, y cuesta compromiso filosófico, complejidad y curva de aprendizaje. El criterio es idéntico — el escalón mínimo que resuelve tu caso de uso. Ver [[05 - Ontology Design Patterns and Reuse]] y [[Ontologías fundacionales]].

## Conexión en el vault

- Es la idea vertebral de [[01 - Introduction]], y gobierna la elección de formalismo en [[04 - Modeling Decisions]].
- Su violación —la sobre-formalización— aparece como causa de fracaso documentada en [[08 - Tools and Practical Considerations]].
- Las [[competency questions]] son el instrumento concreto que decide el escalón: sin ellas la decisión se toma por intuición o por ambición.
- [[SKOS]] es el escalón 3-4 y [[OWL]] el 6; entender el espectro es entender por qué ambos existen y no compiten.
- Aplica igual a un [[Knowledge graph]] aunque nadie use la palabra ontología: la mayoría de los knowledge graphs industriales viven en el escalón 5, no en el 6.

## References

- Kendall, E. & McGuinness, D. (2019) — *Ontology Engineering*, Morgan & Claypool. Capítulo 1.
- [OWL 2 Web Ontology Language Primer](https://www.w3.org/TR/owl2-primer/) — W3C.
- [SKOS Simple Knowledge Organization System Primer](https://www.w3.org/TR/skos-primer/) — W3C.

## Related

- [[ontología]]
- [[competency questions]]
- [[SKOS]]
- [[OWL]]
- [[RDFS]]
- [[Knowledge graph]]
- [[_Ontology Engineering|Ontology Engineering]]
