---
title: Razonadores OWL
source: (OWL 2 Profiles, W3C; documentación de ELK, HermiT, Openllet; práctica de OBO Foundry)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/reference
  - status/done
aliases:
  - Razonadores OWL
  - razonador
  - razonadores
  - OWL reasoner
  - reasoner
updated: 2026-08-04
---

# Razonadores OWL

> [!note] Qué es
> Un **razonador** computa las consecuencias lógicas de los axiomas declarados. Es la herramienta distintiva del campo: no tiene equivalente exacto en el desarrollo de software convencional, porque no verifica sintaxis ni tipos — **deduce hechos que nadie escribió**.

## Qué hace exactamente

| Servicio | Qué computa | Para qué sirve |
|---|---|---|
| **Chequeo de consistencia** | ¿Existe algún modelo que satisfaga todos los axiomas? | Detectar contradicciones fatales |
| **Clasificación (TBox)** | La jerarquía **inferida** de subsunción | Comparar con la declarada: las diferencias son errores de modelado |
| **Satisfacibilidad** | ¿Puede esta clase tener instancias? | Detectar [[clases insatisfacibles]] |
| **Realización (ABox)** | A qué clases pertenece cada individuo | Lo que hace útiles a las clases definidas |
| **Explicación** | El conjunto mínimo de axiomas que causa una inferencia | La funcionalidad de debugging que más rinde |

> [!tip] **La jerarquía inferida es el mejor instrumento de revisión que tenés.** Compará lo que declaraste con lo que el razonador dedujo: si aparecen relaciones de subclase que no esperabas, o dos clases que resultan equivalentes sin que lo hayas querido, encontraste un error que ninguna inspección visual del archivo habría mostrado.

## Cuál usar

| Razonador | Perfil soportado | Velocidad | Estado | Cuándo |
|---|---|---|---|---|
| **ELK** | OWL 2 **EL** | Órdenes de magnitud más rápido | Activo | Ontologías enormes con jerarquías profundas. Es lo que hace viable clasificar SNOMED CT (~350k conceptos) en minutos |
| **HermiT** | OWL 2 **DL** completo | Lento en modelos complejos | Maduro; incluido en Protégé | Expresividad completa; el default cuando no podés restringirte a EL |
| **Openllet** | OWL 2 DL + reglas SWRL | Intermedio | Fork mantenido de Pellet | Cuando necesitás SWRL; **Pellet original está discontinuado** |
| **JFact** | OWL 2 DL | Lento | Mantenimiento | Alternativa de contraste cuando sospechás de un bug del razonador |
| **Embebidos en triple stores** | Subconjuntos (RDFS, OWL 2 RL) | Optimizados para volumen | Varía | Inferencia materializada sobre millones de tripletas en runtime |

> [!warning] **Pellet** aparece en toda la bibliografía (incluido Kendall & McGuinness 2019) pero el proyecto original está discontinuado. El sucesor mantenido es **Openllet**. Si una guía te dice "usá Pellet", traducí a Openllet.

## Perfil y razonador son la misma decisión

> [!note] **La elección de razonador y la de perfil [[OWL]] son la misma decisión tomada desde dos lados.** Si tu ontología usa una construcción que solo HermiT soporta, ELK deja de ser opción. Si podés restringirte a EL, ELK cambia por completo lo que es computacionalmente viable.

Los perfiles y qué sacrifican — ver [[Description Logic]] para el detalle formal:

| Perfil | Sacrifica | Pensado para |
|---|---|---|
| **EL** | Universales (`allValuesFrom`), inversos, disyunción, negación | Ontologías gigantes: biomedicina |
| **QL** | Cardinalidades, clases definidas complejas | Query answering: la consulta se reescribe a SQL |
| **RL** | Existenciales en el consecuente | Reglas sobre triple stores, volúmenes grandes |
| **DL** | Nada (máximo decidible) | Máxima expresividad, al costo de performance |

> [!tip] **Decidí el perfil temprano y modelá dentro de él.** Descubrir a los seis meses que necesitás salir de EL significa cambiar de razonador y multiplicar el tiempo de clasificación. `robot report` y las herramientas de validación de perfil te avisan cuando te saliste.

## Performance: qué la determina realmente

> [!warning] La complejidad de OWL 2 DL es **N2ExpTime en el peor caso**. Suena catastrófico y en la práctica casi nunca se manifiesta: **el rendimiento depende de qué construcciones usás, no de cuántas clases tenés**. Una ontología de 1.000 clases con cardinalidades cualificadas y disyunciones puede ser mucho más lenta que una de 300.000 clases en perfil EL.

Las construcciones que más cuestan, aproximadamente en orden:

1. **Cardinalidades cualificadas** (`owl:qualifiedCardinality`) — las más caras.
2. **Disyunción** (`owl:unionOf`) — obliga al razonador a explorar ramas.
3. **Negación** y clases complemento.
4. **Roles inversos** combinados con restricciones.
5. **Nominales** (`owl:oneOf`) mezclados con lo anterior.

> [!tip] **Medí el tiempo de clasificación desde temprano**, no cuando ya es tarde para simplificar. Si de golpe se dispara, la causa suele ser un axioma agregado en la última sesión — no un crecimiento gradual. Un `robot reason` en CI con timeout te lo detecta el mismo día.

## Materializar o razonar en consulta

Decisión de runtime, distinta de la de diseño:

| | Materializar (forward chaining) | En consulta (backward chaining) |
|---|---|---|
| **Cuándo infiere** | Al cargar o actualizar | Al consultar |
| **Consulta** | Rápida | Más lenta |
| **Escritura** | Cara: hay que reinferir | Barata |
| **Espacio** | Crece, a veces mucho | Sin overhead |

> [!tip] La mayoría de los sistemas productivos materializan RDFS u OWL 2 RL —barato y suficiente para consulta— y reservan el razonamiento DL completo para la fase de diseño y CI, no para runtime.

## Leer una explicación

Cuando el razonador infiere algo inesperado, pedile la **justificación**: el conjunto mínimo de axiomas del que se sigue esa conclusión.

En Protégé aparece como `?` junto a la inferencia. Cómo leerla:

1. **Es un conjunto mínimo** — si sacás cualquier axioma, la inferencia desaparece. Eso significa que **cada uno es necesario**, así que el error está en alguno de ellos.
2. **Buscá el axioma más fuerte** — disjunciones, universales y cardinalidades son los sospechosos habituales. Los `subClassOf` simples rara vez son el problema.
3. **Puede haber varias justificaciones** para la misma inferencia. Arreglar una no elimina la conclusión si otra sigue en pie.

> [!warning] Sin explicaciones, debuggear una inferencia inesperada en una ontología mediana es genuinamente difícil — estás buscando una combinación de axiomas, no un axioma. Es la razón principal para modelar en Protégé aunque edites el Turtle a mano.

## Conexión en el vault

- [[Description Logic]] — los servicios de razonamiento y por qué se reducen a satisfacibilidad.
- [[clases insatisfacibles]] — el síntoma principal y su tabla de diagnóstico.
- [[OWL]] — los perfiles como palanca de performance.
- [[Herramental de ontologías]] — ROBOT para automatizar el razonamiento en CI.
- [[06 - Evaluation and Testing]] — verificación vs validación, y el pipeline.

## References

- [OWL 2 Web Ontology Language Profiles](https://www.w3.org/TR/owl2-profiles/) — W3C.
- [ELK reasoner](https://github.com/liveontologies/elk-reasoner) — perfil EL, alto rendimiento.
- [HermiT](http://www.hermit-reasoner.com/) — OWL 2 DL completo.
- [Openllet](https://github.com/Galigator/openllet) — fork mantenido de Pellet, con soporte SWRL.
- Horridge, M., Parsia, B. & Sattler, U. (2008) — *Laconic and Precise Justifications in OWL*.

## Related

- [[OWL]]
- [[Description Logic]]
- [[clases insatisfacibles]]
- [[Herramental de ontologías]]
- [[06 - Evaluation and Testing]]
