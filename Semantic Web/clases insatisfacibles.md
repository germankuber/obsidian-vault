---
title: clases insatisfacibles
source: (OWL 2 Direct Semantics, W3C; Kendall & McGuinness 2019; práctica estándar de debugging DL)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/concept
  - status/done
aliases:
  - clases insatisfacibles
  - clase insatisfacible
  - unsatisfiable class
  - unsatisfiable classes
  - inconsistencia OWL
updated: 2026-08-04
---

# clases insatisfacibles

> [!note] Definición
> Una **clase insatisfacible** es una clase que, por sus propios axiomas, **no puede tener ninguna instancia** — el razonador la clasifica como equivalente a `owl:Nothing`. Es el síntoma diagnóstico central del modelado en [[OWL]].

> [!warning] **Una clase insatisfacible es SIEMPRE un error de modelado, nunca una decisión de diseño, y siempre bloqueante.** Es el razonador diciéndote que tu modelo afirma que algo del dominio **es imposible**. Si ese algo existe en la realidad, el modelo está roto.

## Insatisfacible vs inconsistente

Se confunden constantemente y no son lo mismo:

| | Clase insatisfacible | Ontología inconsistente |
|---|---|---|
| **Qué pasa** | Una clase no puede tener instancias | No existe ningún modelo que satisfaga todos los axiomas |
| **Alcance** | Localizado en una clase (o varias) | Global: toda la ontología |
| **Gravedad** | Grave, pero el resto sigue funcionando | Fatal: de una contradicción se deduce **cualquier cosa** |
| **Causa típica** | Axiomas de TBox que se contradicen | Un individuo **afirmado** como instancia de una clase insatisfacible |

> [!note] La relación entre ambas: mientras la clase insatisfacible se quede vacía, la ontología sigue siendo consistente. En cuanto alguien afirme `:x a :ClaseInsatisfacible`, la ontología **entera** se vuelve inconsistente y el razonador empieza a "inferir" literalmente todo. Por eso una insatisfacible es una bomba de tiempo, no un detalle cosmético.

## Por qué es el síntoma central

En [[Description Logic]], la satisfacibilidad es el servicio de razonamiento al que se reducen todos los demás: `A` es subsumida por `B` si y solo si `A ⊓ ¬B` es insatisfacible. Un razonador que sabe chequear satisfacibilidad sabe hacer todo lo demás — y por eso este es el síntoma que más información diagnóstica lleva.

## El daño se propaga en silencio

> [!warning] Una consulta que pide instancias de una clase insatisfacible **no falla ni da error: devuelve cero resultados**. Es lo peor posible, porque un resultado vacío parece un dato legítimo. El error puede vivir meses en el modelo y descubrirse cuando alguien nota que un reporte trae menos filas de las que debería.

Además se propaga **hacia abajo por herencia**: si una clase alta en la jerarquía se vuelve insatisfacible, todas sus subclases lo son también. Una insatisfacible temprana puede convertirse en decenas.

## Tabla de diagnóstico: síntoma → causa

Lo que se busca a las dos de la mañana.

| Síntoma observado | Causas típicas | Cómo confirmarlo |
|---|---|---|
| **Una clase queda bajo `owl:Nothing`** | Disjunción + subclase múltiple: la clase hereda de dos clases declaradas disjuntas | Buscá `owl:disjointWith` entre sus ancestros. Es la causa #1 |
| **Varias clases hermanas insatisfacibles a la vez** | Una disjunción declarada en el padre común, o un `rdfs:range` que choca con una restricción heredada | Revisá el ancestro común, no cada clase |
| **Toda la jerarquía bajo un nodo colapsa** | Una insatisfacible alta que se propagó por herencia | Buscá la clase insatisfacible **más alta** y arreglá esa: las demás suelen resolverse solas |
| **Cardinalidades que se pisan** | `minCardinality 2` con `maxCardinality 1`, o una propiedad funcional con `min 2` | Revisá todas las restricciones sobre la misma propiedad, incluidas las heredadas |
| **Restricción de rango contradictoria** | `allValuesFrom :A` en una clase y `allValuesFrom :B` en otra, con `:A` y `:B` disjuntas | Los universales se **acumulan** por herencia: se intersectan |
| **Datatype imposible** | `xsd:minInclusive 100` con `xsd:maxInclusive 50` | Revisá las facetas de datatype |
| **Todo se infiere como equivalente a todo** | `domain`/`range` demasiado laxos, o un `owl:equivalentClass` de más | Compará jerarquía declarada vs inferida |
| **La ontología entera inconsistente** | Un individuo afirmado en dos clases disjuntas, o instancia de una insatisfacible | Buscá el individuo, no la clase |
| **El razonador tarda de golpe muchísimo** | Cardinalidades cualificadas, muchas disyunciones, o salida del perfil OWL previsto | No es insatisfacibilidad: revisá el perfil. Ver [[Razonadores OWL]] |

## La causa #1, en detalle

El patrón que produce la mayoría de los casos reales — tres axiomas, cada uno validado por separado y correcto:

```turtle
# Reunión 1 — tipos de póliza
:PolizaVida         rdfs:subClassOf   :Poliza .
:PolizaPatrimonial  rdfs:subClassOf   :Poliza .
:PolizaVida         owl:disjointWith  :PolizaPatrimonial .   # ← el axioma clave

# Reunión 2 — productos
:SeguroDeVidaConAhorro  rdfs:subClassOf  :PolizaVida .

# Reunión 3 — coberturas
:SeguroDeVidaConAhorro  rdfs:subClassOf  :PolizaPatrimonial .
```

La cadena: todo `SeguroDeVidaConAhorro` es `PolizaVida` **y** `PolizaPatrimonial`; nada puede ser ambas; por lo tanto **nada puede ser un `SeguroDeVidaConAhorro`**.

> [!warning] **El producto existe, se vende, tiene miles de clientes — y la ontología acaba de declarar que es imposible.** Nadie afirmó eso: se dedujo. La divergencia entre modelo y mundo no está en ninguna afirmación, está en su **combinación**.

La discusión real que abre no es técnica: ¿el ramo vida y el patrimonial son verdaderamente excluyentes, o son dos **componentes** que un producto puede combinar? La disjunción era demasiado fuerte — pero eso solo se vuelve visible cuando alguien ve su consecuencia.

## El procedimiento de debugging

1. **Empezá por la insatisfacible más alta** en la jerarquía. Las de abajo suelen ser propagación.
2. **Pedí la explicación al razonador.** Protégé da el **conjunto mínimo de axiomas responsables** (*justification*). Es la funcionalidad que más rinde del editor.
3. **Sospechá primero de las disjunciones.** Son la causa más frecuente y la más fácil de declarar a la ligera.
4. **Revisá las restricciones heredadas**, no solo las locales. Los universales se intersectan hacia abajo.
5. **Llevá la consecuencia al experto, no el axioma.** *"El modelo dice que no puede existir ningún seguro de vida con ahorro, ¿es correcto?"* — el experto responde en dos segundos. El axioma OWL no se lo podés mostrar.

> [!tip] **La pregunta correcta invierte la dirección de la validación.** En vez de mostrar lo que escribiste, mostrás lo que el sistema dedujo. Ninguna revisión de axiomas uno por uno encuentra lo que este ciclo encuentra — porque cada axioma, por separado, es correcto.

## Prevención

- **Declará disjunción deliberadamente**, no por reflejo. Preguntá siempre: *"¿puede algo del dominio ser ambas cosas a la vez?"*. Si dudás, no la declares.
- **Corré el razonador después de cada bloque de modelado**, no al final. Detectás la causa mientras todavía recordás qué cambiaste.
- **Automatizalo en CI**: `robot reason` falla el build si aparece alguna. Ver [[Herramental de ontologías]].
- **Ante la duda, primitiva antes que definida.** Equivocarse en una clase primitiva deja un hueco visible; equivocarse en una definida produce clasificaciones erróneas silenciosas.
- **Modelá los componentes en vez de forzar disjunción.** Cuando dos categorías "excluyentes" se combinan en un producto real, casi siempre eran componentes, no tipos.

## Conexión en el vault

- [[Description Logic]] — la satisfacibilidad como servicio de razonamiento al que se reducen los demás.
- [[04 - Modeling Decisions]] — la disjunción como el axioma más subestimado y peligroso.
- [[02 - Ontology Development Methodology]] — el caso de seguros completo, con la validación de consecuencias.
- [[06 - Evaluation and Testing]] — el razonador como herramienta de verificación y el pipeline de CI.
- [[Razonadores OWL]] — cuál usar y cómo leer sus explicaciones.
- [[Ontology Design Patterns (ODP)]] — los antipatrones que producen insatisfacibles.

## References

- [OWL 2 Direct Semantics](https://www.w3.org/TR/owl2-direct-semantics/) — W3C.
- Horridge, M., Parsia, B. & Sattler, U. (2008) — *Laconic and Precise Justifications in OWL*. El trabajo detrás de las explicaciones de Protégé.
- Kendall, E. & McGuinness, D. (2019) — *Ontology Engineering*, caps. 2 y 6.

## Related

- [[OWL]]
- [[Description Logic]]
- [[Razonadores OWL]]
- [[Herramental de ontologías]]
- [[Ontology Design Patterns (ODP)]]
- [[06 - Evaluation and Testing]]
