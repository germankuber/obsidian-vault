---
title: RDF
source: (conocimiento general + especificación W3C RDF 1.1 — verificar contra la spec antes de citar)
author: W3C
created: 2026-08-03
tags:
  - semantic-web
  - type/technology
  - status/stub
aliases:
  - RDF
  - RDF 1.1
  - Resource Description Framework
  - tripleta
  - tripletas
---

# RDF

> [!note] Definición
> **RDF** (*Resource Description Framework*) es el modelo de datos base de la web semántica. Todo se expresa como **tripletas** `sujeto — predicado — objeto`, y el conjunto de tripletas forma un **grafo dirigido y etiquetado**. Es la capa sobre la que se apoyan [[RDFS]], [[OWL]], [[SKOS]], [[SPARQL]] y [[SHACL]].

No es un formato de archivo ni una base de datos: es un **modelo abstracto**. Se serializa de varias formas (Turtle, JSON-LD, RDF/XML) y se almacena en triple stores, pero el modelo es siempre el mismo grafo de tripletas.

## La tripleta

La unidad atómica. Una afirmación mínima sobre algo:

```turtle
:juan  :trabajaEn  :acme .
#  ↑        ↑         ↑
# sujeto  predicado  objeto
```

Se lee como una oración: *"Juan trabaja en Acme"*. Cada tripleta es una arista del grafo: el sujeto y el objeto son nodos, el predicado es la arista etiquetada que los une.

Un grafo se arma acumulando tripletas — y lo importante es que **el orden no importa** y **no hay estructura previa que respetar**:

```turtle
@prefix :     <http://ejemplo.org/> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .

:juan  a           foaf:Person ;      # 'a' es atajo de rdf:type
       foaf:name   "Juan Pérez" ;
       :trabajaEn  :acme ;
       :edad       34 .

:acme  a           :Empresa ;
       foaf:name   "Acme S.A." ;
       :fundadaEn  1997 .
```

> [!tip] El punto y coma (`;`) repite el sujeto anterior, la coma (`,`) repite sujeto y predicado, el punto (`.`) cierra. Sin esas abreviaturas habría que escribir `:juan` cuatro veces.

## Los tres tipos de nodo

| Tipo | Qué es | Ejemplo |
|---|---|---|
| **IRI** | Identificador global de un recurso | `<http://ejemplo.org/juan>` |
| **Literal** | Un valor: texto, número, fecha | `"Juan Pérez"`, `34`, `"2026-08-03"^^xsd:date` |
| **Blank node** | Nodo anónimo, sin identidad global | `_:b1` o `[ :calle "Corrientes" ]` |

Reglas de posición: el **sujeto** puede ser IRI o blank node; el **predicado** es siempre un IRI; el **objeto** puede ser cualquiera de los tres.

> [!note] **Que un literal no pueda ser sujeto** tiene una consecuencia práctica: no podés decir cosas *sobre* un valor. Si necesitás afirmar algo sobre "34" (su unidad, su fecha de medición, su fuente), tenés que convertirlo en un recurso con IRI propio.

### Literales tipados y con idioma

```turtle
:producto :precio      "1250.50"^^xsd:decimal ;
          :fechaAlta   "2026-08-03"^^xsd:date ;
          :nombre      "Silla ergonómica"@es ;
          :nombre      "Ergonomic chair"@en .
```

El sufijo `^^` da el **tipo** (vocabulario XSD); el `@` da el **idioma**. Ambos importan: `"34"` y `"34"^^xsd:integer` son literales **distintos** y no matchean en una consulta [[SPARQL]].

> [!warning] Es una fuente clásica de consultas que devuelven vacío sin razón aparente: buscás `?x :edad 34` y el dato está cargado como `"34"` (string). Son valores diferentes para el motor.

### Blank nodes

Nodos sin identidad global. Sirven para agrupar sin inventar un IRI:

```turtle
:juan :direccion [
    :calle   "Av. Corrientes" ;
    :numero  1234 ;
    :ciudad  :BuenosAires
] .
```

> [!warning] **Los blank nodes no se pueden referenciar desde afuera del grafo** — su identificador es local y puede cambiar al re-serializar. Si algo va a ser referenciado, actualizado o citado desde otro dataset, **dale un IRI**. Los blank nodes son cómodos al escribir y dolorosos al mantener.

## Por qué IRIs y no strings

Es la decisión de diseño que hace que RDF sirva para integrar datos, y no se aprecia hasta que se ve el problema que resuelve.

Si dos sistemas usan la cadena `"cliente"`, no hay forma de saber si hablan de lo mismo. Si ambos usan `http://schema.org/Customer`, **es literalmente el mismo nodo del grafo** — la integración ocurre sin negociar nada.

> [!note] **La fusión de grafos es trivial en RDF**: unís los conjuntos de tripletas y listo. Los nodos con el mismo IRI se identifican automáticamente. No hay migración de esquemas, no hay mapeo de columnas, no hay claves foráneas que reconciliar. Esta es la propiedad que justifica todo el modelo.

Los **prefijos** son solo azúcar sintáctico para no escribir IRIs completos: `foaf:name` expande a `http://xmlns.com/foaf/0.1/name`.

## Grafo vs tabla: el cambio de mentalidad

| | Relacional | RDF |
|---|---|---|
| **Unidad** | Fila con columnas fijas | Tripleta suelta |
| **Esquema** | Definido antes de cargar | Ninguno; opcional después ([[SHACL]]) |
| **Campo ausente** | `NULL` en la columna | La tripleta simplemente no existe |
| **Agregar un atributo** | `ALTER TABLE` | Agregar una tripleta |
| **Unir dos fuentes** | Migrar esquemas, mapear | Unir los conjuntos de tripletas |
| **Identidad** | Clave primaria local | IRI global |

> [!warning] **La irregularidad es la norma, no la excepción.** En un grafo cada entidad tiene las propiedades que tiene: un empleado con teléfono y otro sin él conviven sin `NULL` ni columna vacía. Por eso en [[SPARQL]] `OPTIONAL` es imprescindible — asumir uniformidad es el reflejo relacional que más consultas rompe.

> [!tip] El costo del modelo sin esquema es que **nada te protege de un typo**: `:nombreCompelto` entra al grafo tan válido como `:nombreCompleto`. Ahí es donde entra [[SHACL]], que devuelve las garantías de esquema donde las necesitás.

## Serializaciones

RDF es un modelo; estos son los formatos concretos:

| Formato | Para qué | Nota |
|---|---|---|
| **Turtle** (`.ttl`) | Trabajo humano, control de versiones | El más legible. El default razonable |
| **N-Triples** (`.nt`) | Streaming, cargas masivas | Una tripleta por línea, sin abreviaturas |
| **JSON-LD** (`.jsonld`) | APIs web, SEO | JSON válido; lo consume cualquier stack web |
| **RDF/XML** (`.rdf`) | Legado | Verboso, diffs ilegibles. Evitalo salvo obligación |
| **TriG** / **N-Quads** | Datasets con **named graphs** | Turtle/N-Triples + un cuarto elemento |

> [!tip] Turtle para el repositorio (los diffs se revisan de verdad), JSON-LD para exponer datos a una aplicación web. RDF/XML solo si una herramienta vieja lo exige.

## Named graphs y cuádruplas

En la práctica casi nadie usa un grafo único: se agrupan tripletas en **named graphs**, agregando un cuarto elemento que identifica a qué grafo pertenece cada una.

```trig
:grafoRRHH {
    :juan :trabajaEn :acme .
}
:grafoVentas {
    :juan :cerroTrato :contrato-88 .
}
```

Para qué sirve: separar por **fuente** (saber de dónde vino cada dato), por **confianza**, por **versión temporal**, o para poder borrar un lote completo sin tocar el resto.

> [!note] Es lo que hace manejable un [[Knowledge graph]] real: sin named graphs, un grafo de millones de tripletas de procedencias distintas es imposible de auditar o de actualizar por partes.

## Reificación: decir cosas sobre una tripleta

Problema recurrente: querés afirmar algo **sobre una afirmación** — quién la dijo, cuándo, con qué confianza. Una tripleta suelta no tiene dónde colgar esos metadatos.

Las salidas:

- **Named graphs** — la más usada y la más simple: metés la tripleta en un grafo y anotás el grafo.
- **Reificación estándar RDF** (`rdf:Statement`, `rdf:subject`, `rdf:predicate`, `rdf:object`) — verbosa: cuatro tripletas extra por cada una que querés anotar. Rara vez vale la pena.
- **Reificar el hecho como entidad** — convertir la relación en un nodo propio. Es el patrón n-ario de [[05 - Ontology Design Patterns and Reuse]] y suele ser la mejor opción cuando el hecho tiene atributos propios.

## RDF solo no dice qué significa nada

Punto clave para entender el stack completo:

> [!warning] **RDF es pura estructura.** Sabe que `:juan :trabajaEn :acme` es una tripleta bien formada, y nada más: no sabe qué es *trabajar*, ni que Juan es una persona, ni que las personas no son empresas. El **significado** lo aportan las capas de arriba — [[RDFS]] con clases y jerarquía, [[OWL]] con axiomas lógicos y razonamiento.

Esa separación es deliberada: el modelo de datos es simple y estable, y encima se apila tanta semántica como el caso de uso necesite — el [[espectro semántico]] en forma de tecnologías.

## Errores comunes

- **Literales sin tipo consistente** — `"34"` vs `"34"^^xsd:integer` no matchean; consultas vacías sin causa aparente.
- **Blank nodes para cosas referenciables** — imposibles de citar o actualizar desde afuera. Dales IRI.
- **Inventar IRIs cuando existe un vocabulario estándar** — perdés interoperabilidad gratis. Buscá primero en schema.org, FOAF, Dublin Core, SKOS.
- **IRIs en un dominio que no controlás** o que va a desaparecer — los IRIs deben resolver en el tiempo.
- **Esperar uniformidad relacional** — el grafo es irregular por diseño; usá `OPTIONAL`.
- **Cargar todo en el grafo por defecto** — el grafo modela relaciones y semántica; volúmenes masivos de datos operativos suelen ir mejor en otro lado.

## Conexión en el vault

- Es la base del stack: [[RDFS]] le agrega clases y jerarquía, [[OWL]] axiomas y razonamiento, [[SKOS]] vocabularios ligeros, [[SPARQL]] consulta y [[SHACL]] validación.
- Un [[Knowledge graph]] es, en la práctica industrial, un grafo RDF (o similar) poblado a escala con datos reales.
- La irregularidad estructural del modelo es la razón por la que `OPTIONAL` es imprescindible en [[SPARQL]].
- La ausencia de esquema es exactamente el hueco que [[SHACL]] existe para cubrir.

## References

- [RDF 1.1 Concepts and Abstract Syntax](https://www.w3.org/TR/rdf11-concepts/) — W3C Recommendation
- [RDF 1.1 Turtle](https://www.w3.org/TR/turtle/) — la serialización recomendada
- [JSON-LD 1.1](https://www.w3.org/TR/json-ld11/) — RDF para stacks web
- [RDF 1.1 Primer](https://www.w3.org/TR/rdf11-primer/) — puerta de entrada

## Related

- [[RDFS]]
- [[OWL]]
- [[SPARQL]]
- [[SHACL]]
- [[SKOS]]
- [[Knowledge graph]]
- [[_Ontology Engineering|Ontology Engineering]]
