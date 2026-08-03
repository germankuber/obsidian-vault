---
title: SPARQL
source: (conocimiento general + especificación W3C SPARQL 1.1 — verificar detalles de sintaxis contra la spec antes de citar)
author: W3C
created: 2026-08-03
tags:
  - semantic-web
  - type/technology
  - status/stub
aliases:
  - SPARQL
  - SPARQL 1.1
  - SPARQL Protocol and RDF Query Language
---

# SPARQL

> [!note] Definición
> **SPARQL** (*SPARQL Protocol and RDF Query Language*) es el lenguaje estándar del W3C para consultar grafos [[RDF]]. Es a un triple store lo que SQL es a una base relacional — con una diferencia de fondo: en vez de hacer *joins* entre tablas, describís un **patrón de grafo** y el motor busca todos los subgrafos que encajan.

## El modelo mental: pattern matching sobre un grafo

Los datos RDF son **tripletas** `sujeto — predicado — objeto`, que forman un grafo dirigido. Una consulta SPARQL es un **grafo con huecos**: escribís tripletas donde algunas posiciones son variables (prefijadas con `?`), y el motor devuelve todas las combinaciones de valores que hacen que ese patrón exista en los datos.

```sparql
PREFIX : <http://ejemplo.org/>

SELECT ?persona ?nombre
WHERE {
  ?persona a          :Empleado ;      # ?persona es un Empleado
           :nombre    ?nombre ;         # y tiene un nombre
           :trabajaEn :Acme .           # y trabaja en Acme
}
```

Se lee como una descripción, no como un procedimiento: *"traeme todo lo que sea un Empleado, tenga nombre, y trabaje en Acme"*. No hay `JOIN` porque las variables repetidas **ya son** el join: `?persona` aparece en tres tripletas, así que las tres tienen que hablar del mismo nodo.

> [!tip] El punto y coma (`;`) repite el sujeto anterior; la coma (`,`) repite sujeto y predicado; el punto (`.`) cierra la tripleta. Es la misma abreviatura de Turtle, y conocerla hace que las consultas se lean mucho mejor.

## Las cuatro formas de consulta

| Forma | Qué devuelve | Cuándo se usa |
|---|---|---|
| **`SELECT`** | Tabla de valores | El caso habitual: querés datos como filas y columnas |
| **`ASK`** | `true` / `false` | Verificar si un patrón existe — ideal para tests |
| **`CONSTRUCT`** | Un grafo RDF nuevo | Transformar, mapear entre vocabularios, exportar |
| **`DESCRIBE`** | Un grafo sobre un recurso | Explorar qué se sabe de una entidad (resultado no estandarizado) |

> [!note] **`CONSTRUCT` es la más subestimada.** Convierte SPARQL en una herramienta de **transformación**, no solo de consulta: entra un grafo con un vocabulario, sale otro grafo con otro vocabulario. Es la forma natural de implementar los mapeos de alineamiento entre ontologías.

```sparql
# CONSTRUCT: mapear vocabulario propio → vocabulario estándar
PREFIX :    <http://ejemplo.org/>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>

CONSTRUCT { ?p a foaf:Person ; foaf:name ?n . }
WHERE     { ?p a :Empleado   ; :nombre   ?n . }
```

## Construcciones que se usan todo el tiempo

### `OPTIONAL` — el left join de SPARQL

Sin él, si falta una sola tripleta del patrón, la fila entera desaparece. `OPTIONAL` la trae igual, con la variable sin ligar.

```sparql
SELECT ?empleado ?nombre ?telefono
WHERE {
  ?empleado a :Empleado ; :nombre ?nombre .
  OPTIONAL { ?empleado :telefono ?telefono . }   # si no tiene, la fila igual aparece
}
```

> [!warning] La causa número uno de *"mi consulta devuelve menos filas de las que debería"* es un patrón obligatorio que debería ser `OPTIONAL`. En un grafo, los datos son **irregulares por naturaleza**: no todas las entidades tienen las mismas propiedades. Asumir uniformidad es un reflejo del mundo relacional que acá no aplica.

### `FILTER` — restringir valores

```sparql
SELECT ?producto ?precio
WHERE {
  ?producto :precio ?precio .
  FILTER (?precio > 1000 && ?precio < 5000)
}
```

Funciona con `regex()`, `langMatches()`, `bound()`, comparaciones de fechas y aritmética.

> [!warning] `FILTER` **no** es lo mismo que `FILTER NOT EXISTS`. El primero descarta filas por el valor de una variable ya ligada; el segundo descarta filas porque **existe otro patrón** en el grafo. Confundirlos da resultados silenciosamente incorrectos.

### Property paths — recorrer la jerarquía

Es la construcción que más diferencia a SPARQL de SQL, y la que hace útil una taxonomía:

```sparql
# Todo lo que sea subclase de Medicamento a CUALQUIER profundidad
SELECT ?tipo WHERE { ?tipo rdfs:subClassOf* :Medicamento . }
```

| Operador | Significado |
|---|---|
| `p*` | Cero o más saltos por `p` |
| `p+` | Uno o más saltos |
| `p?` | Cero o uno |
| `p1/p2` | Encadenar: `p1` y después `p2` |
| `^p` | Recorrer `p` **al revés** |
| `p1\|p2` | `p1` o `p2` |

> [!tip] `rdfs:subClassOf*` es el idiom más útil del lenguaje. Sin él tendrías que saber de antemano cuántos niveles tiene la jerarquía. Con él, una consulta por *Medicamento* trae automáticamente antibióticos, penicilinas y amoxicilina sin nombrarlos.

### Agregación, orden y paginado

```sparql
SELECT ?departamento (COUNT(?empleado) AS ?total)
WHERE  { ?empleado :trabajaEn ?departamento . }
GROUP  BY ?departamento
HAVING (COUNT(?empleado) > 5)
ORDER  BY DESC(?total)
LIMIT  10
```

`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP_CONCAT`, `SAMPLE` — y `DISTINCT`, `OFFSET`, `VALUES` para inyectar un conjunto de valores fijos.

## SPARQL y el razonamiento

Acá está el punto que más confunde y el que más importa entender.

> [!warning] **SPARQL por sí solo NO razona.** Consulta las tripletas que **hay** en el grafo. Si tu ontología [[OWL]] dice que `:Gato rdfs:subClassOf :Mamífero` y preguntás por mamíferos, un motor sin inferencia **no te devuelve los gatos** — esa tripleta no está escrita, se deduce.

Las tres formas de resolverlo:

- **Materializar la inferencia** — correr el razonador antes y escribir las tripletas deducidas en el grafo. Consulta rápida; actualización cara.
- **Razonar en tiempo de consulta** — el triple store aplica reglas al consultar. Actualización barata; consulta más lenta.
- **Compensar en la consulta** — usar property paths (`rdfs:subClassOf*`) para recorrer la jerarquía a mano. No necesita razonador y cubre el caso más común, pero no reemplaza inferencia real (no deduce clases definidas ni detecta inconsistencias).

> [!note] En la práctica los triple stores ofrecen **niveles de inferencia** configurables (RDFS, OWL 2 RL, subconjuntos propios). Saber cuál está activo es imprescindible: la misma consulta devuelve resultados distintos según el nivel, y la diferencia no es visible en la consulta.

## Uso en ontology engineering: tests ejecutables

El uso que le da [[06 - Evaluation and Testing]]: cada [[competency questions|competency question]] se traduce a una consulta SPARQL y se convierte en un **test de aceptación** de la ontología.

```sparql
# CQ-07: ¿Qué medicamentos están contraindicados para insuficiencia renal?
SELECT ?medicamento WHERE {
  ?medicamento a                 :Medicamento ;
               :contraindicadoEn ?condicion .
  ?condicion   rdfs:subClassOf*  :InsuficienciaRenal .
}
```

> [!tip] Para tests, **`ASK` suele ser mejor que `SELECT`**: devuelve un booleano, que es exactamente lo que un test necesita afirmar. `ASK { ... }` responde *"¿el grafo satisface esta condición?"* sin traer datos.

> [!warning] Una competency question que **no se puede escribir como consulta** no está lista: es demasiado vaga. Si no podés expresarla en SPARQL, tampoco vas a poder decidir si la ontología la responde — y perdiste el criterio de terminación que era su razón de ser.

## Actualización: SPARQL Update

El lenguaje también escribe, con `INSERT DATA`, `DELETE DATA`, `DELETE/INSERT ... WHERE`, `LOAD` y `CLEAR`:

```sparql
DELETE { ?p :estado :Activo . }
INSERT { ?p :estado :Inactivo . }
WHERE  { ?p a :Empleado ; :fechaBaja ?f . }
```

> [!warning] `DELETE WHERE` sin restricciones suficientes borra mucho más de lo que esperás y **no hay rollback** en la mayoría de los triple stores. Probá siempre el patrón con un `SELECT` primero, mirá qué trae, y recién entonces convertilo en `DELETE`.

## El protocolo

SPARQL no es solo un lenguaje: define también un **protocolo HTTP**. Un *endpoint* SPARQL es una URL que recibe consultas por `GET` o `POST` y devuelve resultados en JSON, XML, CSV o Turtle. Eso hace que cualquier triple store sea consultable desde cualquier lenguaje sin driver propietario — y habilita **consultas federadas** con `SERVICE`, que ejecutan parte del patrón en un endpoint remoto y combinan los resultados.

```sparql
SELECT ?ciudad ?poblacion WHERE {
  ?ciudad :estaEn :Argentina .
  SERVICE <https://query.wikidata.org/sparql> {   # se resuelve en Wikidata
    ?ciudad wdt:P1082 ?poblacion .
  }
}
```

## Errores comunes

- **Olvidar `OPTIONAL`** — la causa número uno de resultados incompletos.
- **Esperar inferencia sin razonador** — la consulta está bien y el resultado igual viene vacío.
- **`FILTER` mal ubicado** — dentro de un `OPTIONAL` filtra solo esa parte del patrón; fuera, filtra el resultado combinado. No es lo mismo.
- **Confundir `FILTER` con `FILTER NOT EXISTS`** — semánticas distintas, errores silenciosos.
- **`DISTINCT` para tapar duplicados** — casi siempre el duplicado indica un patrón mal escrito, no un problema de presentación.
- **Consultas sin `LIMIT` contra endpoints públicos** — timeout garantizado.

## Conexión en el vault

- Es el mecanismo que convierte las [[competency questions]] en tests ejecutables ([[06 - Evaluation and Testing]]).
- Consulta grafos [[RDF]]; el razonamiento lo aporta [[OWL]] y la validación estructural [[SHACL]] — tres piezas distintas que se confunden seguido.
- Los property paths son lo que hace **útil** a una taxonomía: sin `subClassOf*`, una jerarquía es decorativa a efectos de consulta.
- Es también la vía de acceso a un [[Knowledge graph]] en producción — donde el término "ontología" a menudo desaparece aunque el trabajo sea el mismo.

## References

- [SPARQL 1.1 Query Language](https://www.w3.org/TR/sparql11-query/) — W3C Recommendation
- [SPARQL 1.1 Update](https://www.w3.org/TR/sparql11-update/) — W3C Recommendation
- [SPARQL 1.1 Federated Query](https://www.w3.org/TR/sparql11-federated-query/) — W3C Recommendation

## Related

- [[RDF]]
- [[OWL]]
- [[SHACL]]
- [[competency questions]]
- [[Knowledge graph]]
- [[_Ontology Engineering|Ontology Engineering]]
