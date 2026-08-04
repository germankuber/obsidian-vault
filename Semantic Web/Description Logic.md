---
title: Description Logic
source: (Baader et al., The Description Logic Handbook; OWL 2 Direct Semantics, W3C)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/concept
  - status/done
aliases:
  - Description Logic
  - Description Logics
  - lógica descriptiva
  - lógicas descriptivas
  - DL
updated: 2026-08-04
---

# Description Logic

> [!note] Definición
> Las **lógicas descriptivas** (DL) son una familia de lenguajes formales para representar conocimiento estructurado, diseñados como un **compromiso deliberado entre expresividad y decidibilidad**. Son fragmentos de la lógica de primer orden elegidos de modo que el razonamiento automático **termine siempre** y en tiempo acotado.

Es el fundamento formal de [[OWL]]. Entender DL es lo que convierte el modelado en OWL de memorización de construcciones a derivación desde principios.

## Por qué existen: el compromiso central

La lógica de primer orden es muy expresiva y **indecidible**: no existe algoritmo que garantice responder si una fórmula arbitraria se sigue de un conjunto de premisas. Para un razonador automático eso es fatal — podría no terminar nunca.

Las DL resuelven el problema por **restricción**: se elige un subconjunto de la lógica de primer orden lo bastante expresivo para modelar dominios reales y lo bastante limitado para que el razonamiento sea decidible.

> [!note] **Este compromiso es la razón de ser de todo el diseño de OWL**, incluidos sus perfiles. Cada construcción que un lenguaje DL admite se paga en complejidad computacional, y esa contabilidad es explícita en la literatura del campo.

## TBox y ABox — la división que ordena todo

La distinción más útil de DL, y la que casi nunca se explica cuando se enseña OWL directamente:

| | **TBox** (*terminological*) | **ABox** (*assertional*) |
|---|---|---|
| **Qué contiene** | El esquema: clases, propiedades, axiomas | Los datos: individuos y sus afirmaciones |
| **Ejemplo** | *Todo Padre es una Persona con al menos un hijo* | *Juan es un Padre; Juan tieneHijo María* |
| **En OWL** | `owl:Class`, `rdfs:subClassOf`, restricciones | `a :Padre`, `:juan :tieneHijo :maria` |
| **Análogo** | El esquema de la base de datos | Las filas de las tablas |
| **Tamaño típico** | Cientos a miles de axiomas | Millones a miles de millones de tripletas |

> [!tip] **Esta división explica una de las advertencias más repetidas del campo**: "no metas instancias masivas en la ontología". La ontología es la TBox; los datos operativos son la ABox y viven en una triple store. El razonamiento sobre TBox (clasificación) es caro pero acotado; sobre ABox gigante es otra clase de problema.

Los dos tipos de razonamiento correspondientes:

- **Clasificación (TBox)** — computar la jerarquía inferida: qué clase es subclase de cuál según los axiomas. Es lo que corre Protégé cuando apretás *"start reasoner"*.
- **Realización / instance checking (ABox)** — determinar a qué clases pertenece cada individuo, incluidas las que nadie declaró. Es lo que hace útiles a las clases definidas.

Hay una tercera pieza que OWL 2 agrega: la **RBox**, con los axiomas sobre propiedades (jerarquía de roles, cadenas de propiedades, transitividad).

## Los constructores y la notación

Las DL nombran sus lenguajes por las construcciones que admiten. La base es **𝒜ℒ𝒞** (*Attributive Language with Complements*):

| Constructor | Notación DL | En OWL | Significado |
|---|---|---|---|
| **Concepto atómico** | `Persona` | `owl:Class` | Una clase nombrada |
| **Top / Bottom** | `⊤` / `⊥` | `owl:Thing` / `owl:Nothing` | Todo / nada |
| **Conjunción** | `Persona ⊓ Adulto` | `owl:intersectionOf` | Y lógico |
| **Disyunción** | `Hombre ⊔ Mujer` | `owl:unionOf` | O lógico |
| **Negación** | `¬Vegetariano` | `owl:complementOf` | No |
| **Restricción existencial** | `∃tieneHijo.Persona` | `owl:someValuesFrom` | Tiene **al menos un** hijo que es Persona |
| **Restricción universal** | `∀tieneHijo.Persona` | `owl:allValuesFrom` | **Todos** sus hijos (si los tiene) son Personas |

Sobre esa base, cada letra agregada es una capacidad más — y complejidad computacional más alta:

| Letra | Qué agrega | En OWL |
|---|---|---|
| **𝒮** | 𝒜ℒ𝒞 + roles transitivos | `owl:TransitiveProperty` |
| **ℋ** | Jerarquía de roles | `rdfs:subPropertyOf` |
| **ℛ** | Cadenas de propiedades, roles reflexivos/irreflexivos | `owl:propertyChainAxiom` |
| **𝒪** | Nominales (clases definidas por enumeración) | `owl:oneOf` |
| **ℐ** | Roles inversos | `owl:inverseOf` |
| **𝒬** | Restricciones de cardinalidad cualificadas | `owl:qualifiedCardinality` |
| **𝒟** | Tipos de datos | `xsd:integer`, etc. |

> [!note] **[[OWL]] 2 DL corresponde a `SROIQ(D)`.** Ahora ese nombre críptico se lee: 𝒜ℒ𝒞 + transitividad + jerarquía de roles + cadenas + nominales + inversos + cardinalidad cualificada + datatypes. Es el fragmento más expresivo que sigue siendo decidible.

## Los tres servicios de razonamiento

Todo lo que un razonador hace se reduce a estas operaciones, y todas se reducen entre sí:

1. **Satisfacibilidad de conceptos** — ¿puede esta clase tener alguna instancia? Si no, es **insatisfacible** y hay un error de modelado. Ver [[clases insatisfacibles]].
2. **Subsunción** — ¿toda instancia de A es necesariamente instancia de B? Es lo que computa la jerarquía inferida.
3. **Consistencia de la ontología** — ¿existe algún modelo que satisfaga todos los axiomas a la vez? Una ontología inconsistente es inservible: de una contradicción se sigue cualquier cosa.

> [!tip] **La reducción clave**: `A` es subsumida por `B` si y solo si `A ⊓ ¬B` es insatisfacible. Por eso un razonador que sabe chequear satisfacibilidad sabe hacer todo lo demás — y por eso la clase insatisfacible es el síntoma diagnóstico central.

## Las dos suposiciones que rompen la intuición SQL

Es el punto donde más gente se estrella al llegar desde bases de datos.

**Open World Assumption (OWA)** — lo que no está declarado es **desconocido**, no falso.

```turtle
:juan a :Persona .
# Pregunta: ¿Juan tiene hijos?
# SQL diría: no (no hay filas).
# DL dice: no lo sé. Podría tenerlos y no habérmelo dicho todavía.
```

**No Unique Name Assumption (no-UNA)** — dos IRIs distintos **pueden** referirse al mismo individuo, salvo que se declare lo contrario.

```turtle
:juan   :tieneMadre  :maria .
:juan   :tieneMadre  :mariaGomez .
# Con tieneMadre declarada funcional, DL NO reporta error:
# infiere que :maria y :mariaGomez son el MISMO individuo.
```

> [!warning] Estas dos suposiciones son **decisiones de diseño, no limitaciones**. DL fue pensada para la web: información incompleta, distribuida y aportada por múltiples fuentes que no se conocen entre sí. Bajo ese supuesto, asumir que lo ausente es falso sería incorrecto. Si tu necesidad es *"avisame que este campo está vacío"*, la herramienta es [[SHACL]], no OWL.

Para forzar unicidad cuando la necesitás: `owl:differentFrom`, `owl:AllDifferent`, o `owl:hasKey`.

## Los perfiles de OWL 2 como puntos del compromiso

Los perfiles son sub-lenguajes DL elegidos para garantizar complejidad tratable. Cada uno sacrifica construcciones distintas según el caso de uso:

| Perfil | Fragmento DL | Complejidad | Qué sacrifica | Para qué está pensado |
|---|---|---|---|---|
| **EL** | `EL++` | PTime (polinomial) | Universales, inversos, disyunción, negación | Ontologías **enormes** con jerarquías profundas — biomedicina (SNOMED CT, Gene Ontology) |
| **QL** | `DL-Lite` | AC⁰ en datos | Cardinalidades, clases definidas complejas | **Query answering** sobre bases relacionales: la consulta se reescribe a SQL |
| **RL** | `pD*`-like | PTime | Existenciales en el consecuente | Razonamiento por **reglas** sobre triple stores y volúmenes grandes |
| **DL** | `SROIQ(D)` | N2ExpTime (peor caso) | Nada (es el máximo decidible) | Máxima expresividad |

> [!warning] **N2ExpTime en el peor caso** suena catastrófico y en la práctica rara vez se manifiesta: el rendimiento real depende de **qué construcciones usás**, no de cuántas clases tenés. Una ontología de 1.000 clases con cardinalidades cualificadas y disyunciones puede ser mucho más lenta que una de 300.000 clases en perfil EL. SNOMED CT tiene ~350.000 conceptos y clasifica en minutos con ELK, precisamente porque está en EL.

> [!tip] **Elegí el perfil temprano y modelá dentro de él.** La elección de perfil y la de razonador son la misma decisión tomada desde dos lados: si usás construcciones fuera de EL, ELK deja de ser una opción y quedás con HermiT. Ver [[Razonadores OWL]].

## Qué NO puede expresar

Límites que conviene conocer antes de chocarlos:

- **Relaciones n-arias** — DL solo tiene predicados binarios. Un hecho con más participantes exige **reificación** (ver el patrón n-ario en [[Ontology Design Patterns (ODP)]]).
- **Excepciones y razonamiento por defecto** — *"las aves vuelan, salvo los pingüinos"* no se expresa. DL es **monótona**: agregar información nunca invalida una conclusión previa.
- **Aritmética y agregación** — contar, sumar, comparar valores. Eso es trabajo de [[SPARQL]].
- **Cierre transitivo en consultas** — se declara la propiedad como transitiva, pero preguntar "todos los ancestros" es un property path de SPARQL (`rdfs:subClassOf*`).
- **Tiempo y cambio** — no hay noción nativa. Requiere patrones (fluents, relaciones indexadas por tiempo).
- **Probabilidad e incertidumbre** — DL es binaria. Existen extensiones probabilísticas, ninguna estándar.

> [!note] La **monotonicidad** es la limitación conceptualmente más importante y la menos intuitiva: en DL nunca podés retractar una conclusión agregando datos. Esto choca frontalmente con cómo funciona el conocimiento de sentido común, y es la razón de fondo por la que las ontologías no reemplazan a los sistemas de reglas ni a los modelos estadísticos.

## Conexión en el vault

- Es el fundamento formal de [[OWL]]: los perfiles EL/QL/RL/DL son puntos elegidos del compromiso expresividad-decidibilidad.
- La distinción **TBox/ABox** explica por qué el esquema y los datos masivos se separan — advertencia central de [[04 - Modeling Decisions]] y [[08 - Tools and Practical Considerations]].
- **Satisfacibilidad** es el servicio que convierte a las [[clases insatisfacibles]] en el síntoma diagnóstico principal.
- **OWA y no-UNA** son la raíz de por qué existe [[SHACL]] y de por qué `domain`/`range` infieren en vez de validar.
- [[Razonadores OWL]] — las implementaciones concretas de estos servicios.
- [[espectro semántico]] — DL es lo que habita el escalón superior del continuo.

## References

- Baader, F., Calvanese, D., McGuinness, D., Nardi, D. & Patel-Schneider, P. — *The Description Logic Handbook*, Cambridge University Press. La referencia canónica (McGuinness, coautora, es también autora del libro *Ontology Engineering*).
- [OWL 2 Web Ontology Language Direct Semantics](https://www.w3.org/TR/owl2-direct-semantics/) — W3C. La semántica DL formal de OWL 2.
- [OWL 2 Web Ontology Language Profiles](https://www.w3.org/TR/owl2-profiles/) — W3C. EL, QL y RL con sus garantías de complejidad.
- Horrocks, I., Kutz, O. & Sattler, U. (2006) — *The Even More Irresistible SROIQ*. El fragmento que fundamenta OWL 2 DL.

## Related

- [[OWL]]
- [[ontología]]
- [[clases insatisfacibles]]
- [[Razonadores OWL]]
- [[SHACL]]
- [[espectro semántico]]
- [[04 - Modeling Decisions]]
