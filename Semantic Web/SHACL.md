---
title: SHACL
source: (conocimiento general + especificación W3C SHACL — verificar sintaxis contra la spec antes de citar)
author: W3C
created: 2026-08-03
tags:
  - semantic-web
  - type/technology
  - status/stub
aliases:
  - SHACL
  - Shapes Constraint Language
  - SHACL Shapes
---

# SHACL

> [!note] Definición
> **SHACL** (*Shapes Constraint Language*) es el estándar del W3C para **validar** grafos [[RDF]] contra un conjunto de restricciones estructurales. Definís *shapes* —formas que los datos deben cumplir— y el validador te devuelve un informe con cada violación: qué nodo falló, qué restricción, y por qué.

Es la respuesta a una necesidad que [[OWL]] deliberadamente no cubre: **verificar que los datos estén completos y bien formados**.

## El problema que resuelve: OWL infiere, SHACL valida

Esta es la distinción que hay que entender antes que cualquier sintaxis, y la que más confunde a quien llega desde bases de datos.

[[OWL]] opera bajo **mundo abierto** (*open world assumption*): lo que no está declarado no es falso, es **desconocido**. Si declarás que todo `Empleado` tiene un `:legajo` y cargás un empleado sin legajo, OWL **no reporta un error** — asume que el legajo existe y todavía no lo sabemos. Peor: si declarás `:legajo` como funcional y un empleado tiene dos valores distintos, OWL **infiere que esos dos valores son el mismo individuo**.

SHACL opera bajo **mundo cerrado**: si el dato no está, falta. Punto.

| | [[OWL]] | SHACL |
|---|---|---|
| **Qué hace** | Infiere hechos nuevos | Valida los hechos que hay |
| **Semántica** | Mundo abierto | Mundo cerrado |
| **Dato ausente** | Desconocido, no es error | **Violación** |
| **Salida** | Tripletas deducidas, inconsistencias | Informe de validación con cada violación |
| **Pregunta típica** | *¿Qué se deduce de esto?* | *¿Estos datos sirven para lo que necesito?* |
| **Rol mental** | Motor de razonamiento | Suite de tests / *schema* |

> [!warning] **`rdfs:domain` y `rdfs:range` no validan nada.** Es el malentendido más costoso del stack semántico. Si declarás que el dominio de `:trabajaEn` es `:Persona` y después afirmás que una silla `:trabajaEn` una oficina, OWL **no falla** — infiere que la silla es una Persona. Si querés que eso sea un error, necesitás SHACL.

> [!note] No compiten: **se complementan**. El flujo habitual en producción es razonar primero con OWL (para que las clasificaciones inferidas existan en el grafo) y validar después con SHACL sobre el grafo ya enriquecido.

## Cómo se ve una shape

Una *shape* es un nodo RDF que describe restricciones. Se escribe en el mismo Turtle que los datos — SHACL es RDF describiendo RDF:

```turtle
@prefix sh:   <http://www.w3.org/ns/shacl#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix :     <http://ejemplo.org/> .

:EmpleadoShape
    a               sh:NodeShape ;
    sh:targetClass  :Empleado ;          # aplica a toda instancia de :Empleado

    sh:property [
        sh:path     :legajo ;
        sh:datatype xsd:string ;
        sh:minCount 1 ;                   # obligatorio
        sh:maxCount 1 ;                   # uno solo
        sh:pattern  "^EMP-[0-9]{5}$" ;
        sh:message  "El legajo debe existir y tener formato EMP-00000." ;
    ] ;

    sh:property [
        sh:path     :trabajaEn ;
        sh:class    :Departamento ;       # el valor debe ser un Departamento
        sh:minCount 1 ;
    ] .
```

Leído en voz alta: *"todo Empleado tiene exactamente un legajo, string, con este formato; y trabaja en al menos un Departamento"*.

## Targets — a qué nodos aplica la shape

| Target | Qué selecciona |
|---|---|
| `sh:targetClass` | Todas las instancias de una clase |
| `sh:targetNode` | Nodos concretos, enumerados |
| `sh:targetSubjectsOf` | Todo lo que sea **sujeto** de una propiedad |
| `sh:targetObjectsOf` | Todo lo que sea **objeto** de una propiedad |

> [!tip] `sh:targetObjectsOf` es útil para validar el extremo receptor de una relación sin que importe la clase del nodo — por ejemplo, "todo lo que alguien referencie con `:responsable` tiene que tener email", sin exigir que sea de una clase concreta.

## Las restricciones que se usan siempre

**Cardinalidad y tipo:**

- `sh:minCount` / `sh:maxCount` — cuántos valores. `minCount 1` es "obligatorio", la restricción que OWL no puede expresar.
- `sh:datatype` — tipo literal (`xsd:string`, `xsd:date`, `xsd:integer`).
- `sh:class` — el valor debe ser instancia de una clase.
- `sh:nodeKind` — IRI, blank node o literal.

**Valores:**

- `sh:minInclusive` / `sh:maxInclusive` — rangos numéricos o de fechas.
- `sh:minLength` / `sh:maxLength` — longitud de texto.
- `sh:pattern` — expresión regular.
- `sh:in` — lista cerrada de valores permitidos.
- `sh:hasValue` — debe incluir un valor concreto.
- `sh:languageIn` — idiomas permitidos en literales.

**Lógicas y comparaciones:**

- `sh:not`, `sh:and`, `sh:or`, `sh:xone` — combinaciones booleanas de shapes.
- `sh:equals`, `sh:disjoint`, `sh:lessThan` — comparar dos propiedades del mismo nodo.
- `sh:node` — el valor debe cumplir **otra shape** completa. Así se componen shapes anidadas.
- `sh:closed true` — prohíbe propiedades no declaradas en la shape. Cierra el modelo.

> [!tip] `sh:closed true` es la restricción más útil para higiene de datos: detecta typos en nombres de propiedad (`:nombreCompelto`) que de otro modo entran al grafo en silencio y nadie nota nunca.

## El informe de validación

La salida de SHACL **también es un grafo RDF**, lo cual permite consultarla con [[SPARQL]] o procesarla programáticamente:

```turtle
[] a sh:ValidationReport ;
   sh:conforms false ;
   sh:result [
       a                             sh:ValidationResult ;
       sh:resultSeverity             sh:Violation ;
       sh:focusNode                  :empleado-4711 ;
       sh:resultPath                 :legajo ;
       sh:sourceConstraintComponent  sh:MinCountConstraintComponent ;
       sh:resultMessage              "El legajo debe existir y tener formato EMP-00000." ;
   ] .
```

Cada resultado dice **qué nodo** falló (`sh:focusNode`), **qué propiedad** (`sh:resultPath`) y **qué restricción** (`sh:sourceConstraintComponent`).

**Tres niveles de severidad**, y usarlos bien cambia la utilidad del sistema:

| Severidad | Cuándo | Uso típico |
|---|---|---|
| `sh:Violation` | El dato es inutilizable | Bloquea la carga o el deploy |
| `sh:Warning` | Es un problema, no bloqueante | Deuda de calidad a revisar |
| `sh:Info` | Nota informativa | Métricas, oportunidades de mejora |

> [!tip] Escribí siempre `sh:message` en lenguaje de negocio. El mensaje por defecto del validador es correcto y críptico; un mensaje propio convierte el informe en algo que alguien fuera del equipo técnico puede leer y accionar.

## SHACL-SPARQL: restricciones arbitrarias

Cuando las restricciones predefinidas no alcanzan, podés escribir la condición en [[SPARQL]]:

```turtle
:FechasCoherentesShape
    a              sh:NodeShape ;
    sh:targetClass :Contrato ;
    sh:sparql [
        sh:message "La fecha de fin no puede ser anterior a la de inicio." ;
        sh:select """
            SELECT $this WHERE {
              $this :fechaInicio ?ini ; :fechaFin ?fin .
              FILTER (?fin < ?ini)
            }
        """ ;
    ] .
```

La consulta devuelve los nodos **que violan** la restricción. `$this` es el nodo bajo validación.

> [!warning] SHACL-SPARQL es la escotilla de escape, no el default. Las restricciones declarativas son más legibles, más rápidas y las entiende cualquier herramienta; una shape llena de SPARQL embebido se vuelve tan opaca como el código que querías evitar.

## Dónde encaja en un sistema real

- **Validación en ingesta** — antes de escribir en el triple store. Rechazás lo que no cumple, en vez de contaminar el grafo.
- **Contrato de API** — la shape documenta y hace cumplir qué forma tiene el payload RDF que aceptás.
- **Tests de calidad en CI** — validar el grafo en cada cambio, igual que un test suite. Falla el build si aparece una violación.
- **Documentación ejecutable** — la shape *es* la especificación de la forma de tus datos, y no puede quedar desactualizada porque se ejecuta.
- **Generación de UI** — varias herramientas derivan formularios directamente de las shapes.

> [!note] Pensarlo como **el schema que RDF no tiene**. Un grafo RDF acepta cualquier tripleta: esa flexibilidad es su virtud y su riesgo. SHACL te deja recuperar las garantías de un esquema **solo donde las necesitás**, sin renunciar a la apertura del modelo en el resto.

## SHACL vs ShEx

El otro lenguaje de validación de RDF es **ShEx** (*Shape Expressions*). Diferencias prácticas:

- **SHACL** es Recomendación del W3C, se escribe en RDF, tiene más adopción industrial y soporte en triple stores.
- **ShEx** tiene una sintaxis más compacta y una semántica basada en gramáticas; fuerte en el mundo bioinformático y en Wikidata.

> [!tip] Para un proyecto nuevo en un stack estándar, SHACL es la apuesta segura: es Recomendación W3C y los triple stores lo traen integrado.

## Errores comunes

- **Esperar que [[OWL]] valide** — el error raíz. `domain`/`range`/cardinalidades de OWL infieren, no validan.
- **Validar antes de razonar** — si tu shape depende de clasificaciones inferidas, corré el razonador primero o vas a marcar como violación algo que el razonador habría resuelto.
- **Shapes gigantes** — una `NodeShape` con treinta `sh:property` es tan poco mantenible como una función de 500 líneas. Componé con `sh:node`.
- **Sin `sh:message`** — informes ilegibles para cualquiera que no haya escrito la shape.
- **Todo en `sh:Violation`** — si todo bloquea, el equipo empieza a ignorar el informe. Usá las tres severidades.
- **Olvidar `sh:closed`** — sin él, un typo en un nombre de propiedad entra al grafo sin que nada lo note.

## Conexión en el vault

- Es la contraparte de [[OWL]]: uno infiere bajo mundo abierto, el otro valida bajo mundo cerrado. La confusión entre ambos es la trampa recurrente que atraviesa [[_Ontology Engineering|Ontology Engineering]].
- Se apoya en [[SPARQL]] para restricciones arbitrarias, y su informe de validación es un grafo consultable con SPARQL.
- Complementa las [[competency questions]] en [[06 - Evaluation and Testing]]: las CQ validan que el **modelo** responda lo que debe; SHACL valida que los **datos** tengan la forma que el modelo espera. Ejes distintos, ambos necesarios.
- En un [[Knowledge graph]] en producción suele ser la pieza que más se usa a diario — más que el razonador.

## References

- [SHACL — Shapes Constraint Language](https://www.w3.org/TR/shacl/) — W3C Recommendation
- [SHACL Advanced Features](https://www.w3.org/TR/shacl-af/) — W3C Note (reglas, funciones)
- [Shape Expressions (ShEx)](http://shex.io/) — la alternativa

## Related

- [[OWL]]
- [[RDF]]
- [[SPARQL]]
- [[competency questions]]
- [[Knowledge graph]]
- [[_Ontology Engineering|Ontology Engineering]]
