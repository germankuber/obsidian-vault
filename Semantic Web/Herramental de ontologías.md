---
title: Herramental de ontologías
source: (ROBOT — BMC Bioinformatics 2019; Ontology Development Kit — Database 2022; docs de Protégé, pySHACL, Ontop)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/reference
  - status/done
aliases:
  - Herramental de ontologías
  - herramientas de ontologías
  - ontology tooling
  - ROBOT
  - ODK
updated: 2026-08-04
---

# Herramental de ontologías

> [!info] Estado a agosto 2026
> El herramental que aparece en la bibliografía clásica —incluido [[_Ontology Engineering|Kendall & McGuinness (2019)]]— quedó parcialmente desactualizado. Esta nota es el mapa vigente: qué sigue siendo estándar, qué se abandonó, y qué apareció después y hoy es prácticamente obligatorio.

## Lo que cambió desde 2019

| Herramienta | Estado 2026 | Qué hacer |
|---|---|---|
| **Protégé** | Vigente. 5.6.8 (sep 2025) | Sigue siendo el estándar de facto |
| **HermiT, ELK** | Vigentes | Sin cambios en su rol |
| **Pellet** | **Discontinuado** | Usar **Openllet** (fork mantenido) |
| **Blazegraph** | **Abandonado desde 2020** | No adoptar. Wikidata mismo está migrando |
| **Virtuoso, GraphDB, Stardog, AllegroGraph** | Vigentes | Sin cambios sustanciales de rol |
| **ROBOT** | **Estándar de facto para automatización** — no está en el libro | Adoptar si versionás una ontología |
| **ODK** | Toolkit de repositorio completo — no está en el libro | Adoptar para proyectos serios |
| **SHACL** | Recomendación W3C, ecosistema maduro | pySHACL, TopBraid, soporte nativo en varias stores |
| **Oxigraph, Qlever** | Nuevos, livianos, rápidos | Alternativas modernas para casos acotados |
| **RDF-star / SPARQL 1.2** | En track de Recommendation | Verificar soporte del motor antes de comprometerse |

---

## Edición

**Protégé** (Stanford, open source, gratuito) sigue siendo el estándar. Versión estable **5.6.8** (sep 2025). Existe también **WebProtégé** para trabajo colaborativo en navegador.

Lo que aporta más allá de escribir axiomas:

- **Jerarquía declarada vs inferida**, lado a lado.
- **Integración con razonadores** — clasificar sin salir del entorno.
- **Explicaciones de inferencia** — la funcionalidad más valiosa. Ver [[Razonadores OWL]].
- **Detección de errores comunes** de modelado.

> [!warning] **Editar en GUI y versionar en Git conviven mal si no cuidás la serialización.** Protégé puede reordenar el archivo al guardar y producir diffs enormes que no reflejan el cambio real. Fijá el formato de salida (Turtle) y usá `robot convert` para normalizar antes de commitear, o el control de versiones deja de servir para revisión.

---

## Automatización: ROBOT

**La pieza que falta en toda la bibliografía anterior a 2019 y que hoy es el estándar de facto.** ROBOT es una herramienta de línea de comandos construida sobre la OWL API — la misma base que usa Protégé — pensada para automatizar workflows de ontologías.

Los comandos que más se usan:

| Comando | Qué hace |
|---|---|
| `robot reason` | Corre el razonador; **falla si hay [[clases insatisfacibles]]** |
| `robot report` | Chequeos de calidad: labels faltantes, IRIs mal formados, definiciones ausentes |
| `robot verify` | Ejecuta consultas SPARQL como tests: si devuelven filas, falla |
| `robot diff` | **Diff semántico** entre dos versiones — qué axiomas cambiaron, no qué líneas |
| `robot extract` | Extrae un módulo mínimo de una ontología grande, preservando inferencias relevantes |
| `robot merge` | Fusiona ontologías e imports |
| `robot convert` | Normaliza serialización (clave para diffs revisables) |
| `robot annotate` | Agrega metadatos de versión y release |
| `robot template` | Genera OWL desde planillas — curación en spreadsheet |

> [!tip] **`robot verify` es lo que convierte las [[competency questions]] en tests reales.** Escribís la CQ como SPARQL, definís que no debe devolver resultados inesperados, y el build falla cuando el modelo deja de responderla. Es la pieza que la teoría siempre pidió y que nadie sabía cómo implementar.

> [!note] `robot diff` merece atención especial: es la implementación del **diff semántico**. Dos cambios de una línea pueden tener consecuencias inferenciales radicalmente distintas —agregar una disjunción puede invalidar cientos de individuos—. Revisar el diff textual no alcanza; ROBOT muestra qué axiomas se agregaron, quitaron o cambiaron.

---

## Repositorio completo: ODK

El **Ontology Development Kit** inicializa un repositorio de ontología con estructura de directorios, un `Makefile` con workflows de release automatizados, CI configurado y documentación. Viene de la comunidad OBO Foundry (biomedicina) pero es agnóstico de dominio.

Lo que trae de fábrica:

- Estructura de directorios estándar (`src/ontology`, imports, patrones).
- Pipeline de release reproducible.
- **CI configurado**: al abrir un pull request corre tests estándar — clases insatisfacibles, cross-references mal formadas, labels faltantes.
- Soporte de templating: **DOSDP** y ROBOT templates, para curar contenido en planillas y compilarlo a OWL.

> [!tip] Si vas a mantener una ontología con más de una persona y más de un release, ODK ahorra semanas de configuración. Si es un modelo chico de un solo autor, ROBOT solo alcanza.

---

## Pipeline de CI

El equivalente concreto del pipeline que la teoría plantea:

```yaml
# Conceptual — adaptar al runner que uses
steps:
  - robot convert --input onto.owl --format ttl --output onto.ttl   # normalizar
  - robot reason  --input onto.ttl --reasoner ELK                   # falla si hay insatisfacibles
  - robot report  --input onto.ttl --fail-on ERROR                  # convenciones
  - robot verify  --input onto.ttl --queries tests/*.rq             # competency questions
  - robot diff    --left main.ttl --right onto.ttl                  # diff semántico al PR
```

> [!warning] Sin este pipeline, los errores se descubren cuando alguien nota que una consulta devuelve basura — típicamente meses después del commit que lo causó, y con el contexto de esa decisión ya perdido.

---

## Validación: SHACL en la práctica

[[SHACL]] es Recomendación W3C y hoy tiene ecosistema maduro. Es la respuesta a lo que [[OWL]] deliberadamente no hace: **validar** bajo mundo cerrado.

| Herramienta | Contexto |
|---|---|
| **pySHACL** | Python, la más usada para scripting y CI |
| **TopBraid SHACL API** | Java, implementación de referencia |
| **Soporte nativo** | GraphDB, Stardog, Jena y otros validan SHACL directamente |

```turtle
@prefix sh: <http://www.w3.org/ns/shacl#> .

:MedicamentoShape  a  sh:NodeShape ;
    sh:targetClass  :Medicamento ;
    sh:property [
        sh:path      :principioActivo ;
        sh:minCount  1 ;                       # OBLIGATORIO — OWL no puede exigir esto
        sh:message   "Todo medicamento debe declarar al menos un principio activo"@es ;
    ] ;
    sh:property [
        sh:path      :dosisMaxima ;
        sh:datatype  xsd:decimal ;
        sh:minExclusive 0 ;
    ] .
```

> [!note] **La diferencia con OWL, en una línea**: `sh:minCount 1` **falla** si el dato no está; `owl:minCardinality 1` **infiere** que existe aunque no esté declarado. Ver [[Description Logic]] para la raíz del asunto (open world assumption).

> [!tip] El patrón que funciona en producción: **OWL para el modelo conceptual y la inferencia; SHACL para validar los datos que entran**. No compiten — cubren necesidades distintas y la mayoría de los sistemas serios usan ambos.

---

## Trabajo programático

Nada de esto está en la bibliografía clásica, y es como se trabaja realmente:

| Herramienta | Lenguaje | Para qué |
|---|---|---|
| **rdflib** | Python | Manipular grafos RDF, parsear, serializar, consultar |
| **Owlready2** | Python | Trabajar con ontologías OWL como objetos Python; razonamiento integrado |
| **pySHACL** | Python | Validación SHACL |
| **RDF4J** | Java | Framework completo: parsing, storage, SPARQL, inferencia |
| **OWL API** | Java | La base de Protégé y ROBOT; manipulación de axiomas OWL |
| **Oxigraph** | Rust (bindings Python/JS) | Triple store embebible, rápida y liviana |
| **sophia** | Rust | Toolkit RDF de alto rendimiento |

> [!tip] Para pipelines de datos en Python, la combinación habitual es **rdflib** para construir el grafo + **pySHACL** para validar + una triple store para persistir. Owlready2 conviene cuando el trabajo es sobre la ontología misma más que sobre los datos.

---

## Triple stores

| Store | Licencia | Fuerte en | Nota |
|---|---|---|---|
| **GraphDB** (Graphwise, ex-Ontotext) | Comercial, free edition | Inferencia materializada, SHACL, madurez | El default empresarial habitual |
| **Stardog** | Comercial | Razonamiento en consulta, virtualización | Fuerte en OBDA |
| **Virtuoso** | Open source + comercial | Volumen muy grande, SPARQL maduro | Detrás de DBpedia |
| **Apache Jena (TDB2/Fuseki)** | Apache 2.0 | Ecosistema Java, gratuito | El open source de referencia |
| **Amazon Neptune** | Servicio gestionado | Operación sin ops; RDF **y** property graph | Descendiente de Blazegraph |
| **Oxigraph** | MIT | Liviana, embebible, rápida | Excelente para desarrollo y casos acotados |
| **AllegroGraph** | Comercial | Razonamiento, escala | — |
| **Qlever** | Open source | Consultas muy rápidas sobre datasets enormes | Wikidata completo con latencia baja |
| ~~**Blazegraph**~~ | — | — | **Abandonado desde 2020. No adoptar.** |

Criterios de selección: **volumen**, **capacidad de razonamiento** (materializado vs en consulta), **soporte SHACL**, y **soporte RDF-star** si vas a necesitar metadatos sobre tripletas.

> [!warning] La distancia entre *"la ontología clasifica bien en Protégé"* y *"el sistema responde consultas en producción con datos reales"* es grande y sorprende a muchos equipos. Probá con **volumen realista desde temprano**.

---

## Poblado de datos

El gap que ninguna metodología clásica cubre y que es el 70% del esfuerzo real.

| Enfoque | Herramienta | Cuándo |
|---|---|---|
| **R2RML** | Estándar W3C; implementado en varios motores | Mapeo declarativo de relacional a RDF |
| **RML** | Extensión de R2RML | Además de SQL: CSV, JSON, XML |
| **Virtual KG (OBDA)** | **Ontop** | No materializar: la consulta SPARQL se reescribe a SQL. El perfil OWL 2 QL existe para esto |
| **Extracción desde texto** | Pipelines con LLM | Ver [[Ontología y LLMs]] |
| **Programático** | rdflib, RDF4J | Ingesta desde APIs y streams |

> [!tip] **Ontop y el enfoque virtual son la opción más subestimada.** Si los datos ya viven en una base relacional que otros sistemas usan, materializarlos como RDF crea un problema de sincronización permanente. La virtualización lo evita: los datos se quedan donde están y la ontología es una capa de acceso semántico.

---

## Calidad

- **OOPS!** (OntOlogy Pitfall Scanner) — detecta *pitfalls* comunes de modelado vía web o API.
- **`robot report`** — chequeos de convención integrables en CI.
- **Validación de perfil** — verificar que no te saliste del perfil OWL previsto, que es la causa silenciosa de que el razonamiento se dispare.

---

## Conexión en el vault

- [[Razonadores OWL]] — cuál usar, perfiles y performance.
- [[clases insatisfacibles]] — lo que `robot reason` detecta y cómo diagnosticarlo.
- [[SHACL]] — la validación que OWL no cubre.
- [[Knowledge graph]] — dónde viven los datos y el debate RDF vs property graph.
- [[Ontología y LLMs]] — extracción asistida como fuente de poblado.
- [[08 - Tools and Practical Considerations]] — el capítulo del libro que esta nota actualiza.
- [[06 - Evaluation and Testing]] — el pipeline conceptual que ROBOT implementa.

## References

- Jackson, R. et al. (2019) — *ROBOT: A Tool for Automating Ontology Workflows*. [BMC Bioinformatics 20:407](https://doi.org/10.1186/s12859-019-3002-3).
- Matentzoglu, N. et al. (2022) — *Ontology Development Kit: a toolkit for building, maintaining and standardizing biomedical ontologies*. [Database, baac087](https://doi.org/10.1093/database/baac087).
- [Protégé](https://protege.stanford.edu/) — Stanford. Estable 5.6.8.
- [SHACL](https://www.w3.org/TR/shacl/) — W3C Recommendation.
- [R2RML](https://www.w3.org/TR/r2rml/) — W3C Recommendation.
- [Ontop](https://ontop-vkg.org/) — virtual knowledge graphs.
- [OBO Foundry — Ontology tools and resources](https://obofoundry.org/resources).

## Related

- [[Razonadores OWL]]
- [[clases insatisfacibles]]
- [[SHACL]]
- [[OWL]]
- [[Knowledge graph]]
- [[Ontología y LLMs]]
- [[08 - Tools and Practical Considerations]]
