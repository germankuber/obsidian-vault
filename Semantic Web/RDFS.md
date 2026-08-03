---
title: RDFS
source: (conocimiento general + especificación W3C RDF Schema 1.1 — verificar contra la spec antes de citar)
author: W3C
created: 2026-08-03
tags:
  - semantic-web
  - type/technology
  - status/stub
aliases:
  - RDFS
  - RDF Schema
  - RDF Schema 1.1
---

# RDFS

> [!note] Definición
> **RDFS** (*RDF Schema*) es la capa de esquema mínima sobre [[RDF]]: agrega el vocabulario para declarar **clases**, **jerarquías** y **propiedades** con dominio y rango. Es el primer escalón donde una máquina puede inferir algo — herencia por subsunción — sin la complejidad de [[OWL]].

Es deliberadamente pequeño: media docena de construcciones. Esa modestia es su virtud — cubre el caso de uso más común (una taxonomía navegable con tipos) a un costo mínimo de modelado y de razonamiento.

## Las construcciones

| Construcción | Qué declara |
|---|---|
| `rdfs:Class` | Que algo es una clase |
| `rdfs:subClassOf` | Jerarquía de clases (*es un tipo de*) |
| `rdf:Property` | Que algo es una propiedad |
| `rdfs:subPropertyOf` | Jerarquía de propiedades |
| `rdfs:domain` | De qué clase es el **sujeto** de una propiedad |
| `rdfs:range` | De qué clase (o tipo) es el **objeto** |
| `rdfs:label` | Etiqueta legible para humanos |
| `rdfs:comment` | Descripción en lenguaje natural |

```turtle
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix :     <http://ejemplo.org/> .

:Persona   a rdfs:Class ;
           rdfs:label   "Persona"@es ;
           rdfs:comment "Un ser humano individual."@es .

:Empleado  a rdfs:Class ;
           rdfs:subClassOf :Persona .

:trabajaEn a rdf:Property ;
           rdfs:domain :Empleado ;
           rdfs:range  :Empresa .
```

> [!tip] `rdfs:label` y `rdfs:comment` parecen decorativos y no lo son: son lo que hace navegable una ontología para cualquiera que no la escribió. Ponelos siempre, con idioma (`@es`, `@en`).

## Qué infiere RDFS

Tres reglas, y con eso ya hay razonamiento útil:

**1. Herencia por subsunción** — si `:Empleado rdfs:subClassOf :Persona` y `:juan a :Empleado`, entonces se deduce `:juan a :Persona`.

**2. Transitividad de la jerarquía** — si A es subclase de B y B de C, entonces A es subclase de C. A cualquier profundidad.

**3. Tipado por domain y range** — si `:trabajaEn` tiene dominio `:Empleado` y existe `:juan :trabajaEn :acme`, se deduce `:juan a :Empleado`.

> [!warning] **`rdfs:domain` y `rdfs:range` NO validan: infieren.** Es el malentendido más costoso del stack. No dicen *"el sujeto debe ser un Empleado"* sino *"todo sujeto de esta propiedad **es** un Empleado"*. Si afirmás que una silla `:trabajaEn` una oficina, RDFS **deduce que la silla es un Empleado**. No hay error, hay una inferencia nueva. Si querés que eso falle, la herramienta es [[SHACL]].

> [!tip] Corolario práctico: declarar `domain`/`range` demasiado específicos es una fuente de inferencias basura. Ante la duda, dejalos sin declarar o usá una clase más general — el razonador va a "ensuciar" el grafo con tipos deducidos que nadie quería.

## RDFS vs OWL: cuándo alcanza

| | RDFS | [[OWL]] |
|---|:---:|:---:|
| Clases y jerarquía | ✅ | ✅ |
| `domain` / `range` | ✅ | ✅ |
| Disjunción entre clases | ❌ | ✅ |
| Cardinalidades | ❌ | ✅ |
| Restricciones de propiedad | ❌ | ✅ |
| Clases **definidas** | ❌ | ✅ |
| Transitiva / simétrica / inversa | ❌ | ✅ |
| Detectar inconsistencias | ❌ | ✅ |
| **Costo de razonamiento** | Muy bajo | Alto (según perfil) |

> [!note] **RDFS no puede contradecirse.** No tiene disjunción ni cardinalidades, así que no hay forma de escribir un conjunto de axiomas RDFS inconsistente. Eso es simultáneamente su límite (no detecta errores) y su virtud (el razonamiento es barato y predecible).

> [!tip] La pregunta para decidir: **¿necesitás que el sistema detecte contradicciones o clasifique individuos por sus propiedades?** Si no —si solo querés jerarquía navegable y tipos heredados— RDFS alcanza y te ahorra la complejidad entera de OWL. Es el [[espectro semántico]] aplicado: el escalón mínimo que resuelve el caso.

## RDFS y SPARQL

La combinación más práctica del stack, y la que resuelve el 80% de los casos reales.

Sin razonador, [[SPARQL]] no ve la herencia: consultás por `:Persona` y no traés los empleados, porque esa tripleta se deduce, no está escrita. Los property paths lo resuelven a mano:

```sparql
# Trae empleados aunque solo estén declarados como :Empleado
SELECT ?p WHERE {
  ?p a/rdfs:subClassOf* :Persona .
}
```

> [!tip] `a/rdfs:subClassOf*` es el idiom que hace útil una taxonomía RDFS **sin necesidad de razonador**: recorre la jerarquía a cualquier profundidad en tiempo de consulta. Para muchísimos proyectos, esto es todo el "razonamiento" que hace falta.

## Dónde se usa

RDFS es la capa de la que dependen casi todos los vocabularios publicados en la web — **schema.org**, **FOAF**, **Dublin Core** usan RDFS (a veces con algo de OWL) para declarar sus clases y propiedades. Cuando reusás uno de esos vocabularios, estás consumiendo RDFS.

> [!note] Los triple stores traen inferencia RDFS **de fábrica**, y suele venir activada. Vale saber cuál nivel está corriendo: la misma consulta devuelve resultados distintos con y sin inferencia, y la diferencia no se ve en el texto de la consulta.

## Errores comunes

- **Esperar que `domain`/`range` validen** — infieren, y encima pueden meter tipos que no querías.
- **`domain`/`range` demasiado específicos** — generan inferencias basura difíciles de rastrear.
- **Confundir `rdfs:subClassOf` con parte-todo** — *rueda* subclase de *auto* hace que el razonador infiera que toda rueda es un auto. Es subsunción, no meronimia.
- **Saltar a [[OWL]] sin necesitarlo** — sobre-formalización, una causa documentada de fracaso de proyectos.
- **Omitir `rdfs:label` y `rdfs:comment`** — ontología ilegible para cualquiera que no la escribió.
- **Consultar sin razonador ni property paths** — resultados incompletos que parecen correctos.

## Conexión en el vault

- Es la capa intermedia del stack: se apoya en [[RDF]] y es la base sobre la que [[OWL]] agrega lógica formal.
- Comparte con OWL la trampa de `domain`/`range`: ambos infieren, ninguno valida. Eso es [[SHACL]].
- Para taxonomías y thesauri, [[SKOS]] suele ser mejor opción que RDFS puro: está hecho a medida para eso.
- La decisión RDFS vs OWL es una aplicación directa del [[espectro semántico]] y se resuelve con las [[competency questions]].

## References

- [RDF Schema 1.1](https://www.w3.org/TR/rdf-schema/) — W3C Recommendation
- [RDF 1.1 Semantics](https://www.w3.org/TR/rdf11-mt/) — las reglas de inferencia formales

## Related

- [[RDF]]
- [[OWL]]
- [[SKOS]]
- [[SPARQL]]
- [[SHACL]]
- [[espectro semántico]]
- [[_Ontology Engineering|Ontology Engineering]]
