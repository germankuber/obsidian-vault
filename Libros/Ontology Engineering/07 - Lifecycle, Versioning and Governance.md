---
title: 07 - Lifecycle, Versioning and Governance
libro: Ontology Engineering
autor: Elisa F. Kendall / Deborah L. McGuinness
capitulo: 7
created: 2026-08-03
tags:
  - libros/ontology-engineering
  - type/case-study
  - status/stub
aliases:
  - Lifecycle, Versioning and Governance
  - Cap 7 - Ciclo de vida, versionado y governance
updated: 2026-08-03
---

# 07 - Lifecycle, Versioning and Governance

> [!info] Capítulo 7 · *Ontology Engineering* — Elisa F. Kendall & Deborah L. McGuinness · Morgan & Claypool, *Synthesis Lectures on Data, Semantics, and Knowledge* (2019)
> Qué pasa después del primer release: **versionado**, política de **IRIs**, gestión del cambio y sus consecuencias inferenciales, **deprecación** en vez de borrado, y la estructura de **governance** que decide quién puede cambiar qué. Es el capítulo que cierra la tesis del libro — una ontología sin governance se degrada exactamente igual que un código base sin ella. Navegá: [[_Ontology Engineering|Ontology Engineering]] · anterior [[06 - Evaluation and Testing]] · siguiente [[08 - Tools and Practical Considerations]].

## Resumen

La mayoría de los proyectos de ontología se piensan hasta el primer release y no más allá. Este capítulo argumenta que ahí empieza la parte larga: una ontología desplegada tiene **consumidores** —sistemas, datasets, mapeos, otras ontologías— y cada cambio que hagas se propaga a todos ellos. La diferencia con el software convencional es que en una ontología los cambios no solo rompen interfaces: **cambian lo que el sistema deduce**. Agregar un axioma puede invalidar datos que ayer eran consistentes.

El capítulo organiza el problema en tres frentes. El **versionado y la política de IRIs**: cómo identificar versiones, qué IRI usa un consumidor que quiere estabilidad frente a uno que quiere lo último, y por qué los IRIs son un compromiso de largo plazo que conviene diseñar antes del primer release y no después. La **gestión del cambio**: clasificar los cambios por su impacto inferencial —aditivos, restrictivos, correctivos— y entender que la categoría peligrosa no es la que uno esperaría. Y la **governance**: quién decide, cómo se aprueban los cambios, cómo se resuelven desacuerdos entre áreas que tienen visiones distintas del mismo concepto.

El hilo es el mismo de todo el libro llevado a su conclusión: si la ontología es un artefacto de software, necesita las mismas disciplinas —versionado, política de compatibilidad, deprecación, revisión de cambios, propiedad clara— y por las mismas razones. La diferencia es que la ontología suele tener **más consumidores y menos visibilidad** sobre quiénes son, lo que hace que el costo de romper compatibilidad sea más alto y más difícil de estimar.

## Versionado y política de IRIs

Los **IRIs** son la identidad de todo lo que la ontología define. Son también el compromiso más difícil de revertir: una vez que alguien publicó datos usando tu IRI, cambiarlo rompe esos datos.

Las decisiones que hay que tomar antes del primer release:

- **IRIs opacos vs descriptivos** — `:C_0042` versus `:Medicamento`. Los descriptivos son legibles y facilitan el trabajo diario; los opacos sobreviven a los cambios de nombre del concepto, que en dominios volátiles no son raros. Es un trade-off real sin respuesta universal.
- **IRI de la ontología vs IRI de versión** — el primero identifica la ontología como tal y no cambia nunca; el segundo identifica un release concreto. `owl:versionIRI` es el mecanismo estándar.
- **Estrategia de resolución** — un consumidor que importa el IRI genérico obtiene la última versión y se expone a cambios; uno que importa el versionIRI obtiene estabilidad y se queda atrás. Ambas necesidades son legítimas y hay que soportar las dos.
- **Persistencia** — los IRIs deben resolver en el tiempo. Usar un dominio que la organización controle y no vaya a abandonar es una decisión de infraestructura, no de modelado.

> [!warning] **Cambiar un IRI publicado rompe a todos los consumidores silenciosamente.** No hay error de compilación: los datos que referencian el IRI viejo simplemente dejan de conectar con el modelo, y las consultas empiezan a devolver menos resultados sin que nada falle visiblemente. Es el modo de falla más difícil de diagnosticar.

> [!tip] Diseñá la política de IRIs **antes del primer release**. Es de las pocas decisiones de un proyecto de ontología que después es genuinamente cara de revertir — al nivel de cambiar el esquema de una base de datos en producción.

## Tipos de cambio y su impacto inferencial

La clasificación que organiza la gestión del cambio, y donde está el aporte menos obvio del capítulo:

- **Aditivos** — agregar clases, propiedades o individuos nuevos sin tocar lo existente. Generalmente seguros: lo que antes se inferría se sigue infiriendo.
- **Restrictivos** — agregar axiomas que **acotan** el modelo: disjunciones, cardinalidades, restricciones de rango. Son los peligrosos.
- **Correctivos** — arreglar errores de modelado. Cambian inferencias por definición, y ese es su propósito.
- **Refactorizaciones** — reorganizar sin cambiar la semántica. Seguros en teoría, y en la práctica solo si hay tests que lo demuestren.
- **Eliminaciones** — quitar clases o propiedades. Siempre rompen consumidores; nunca deberían hacerse sin deprecación previa.

> [!warning] **Los cambios restrictivos son la categoría traicionera.** Agregar una disjunción o una cardinalidad máxima parece un cambio menor y aditivo —estás agregando un axioma, no quitando nada— pero puede volver **inconsistentes datos que ayer eran válidos**. Un dataset con instancias que caen en ambas clases recién declaradas disjuntas pasa de correcto a inconsistente sin que nadie haya tocado los datos.

> [!note] En una ontología, la compatibilidad hacia atrás no se mide en la interfaz sino en las **inferencias**. Un cambio es compatible si todo lo que antes se deducía se sigue deduciendo y todo lo que antes era consistente lo sigue siendo. Por eso el diff que importa es el diff semántico que planteaba [[06 - Evaluation and Testing]], no el diff textual.

### Tabla 7.1 — Tipos de cambio y riesgo

| Tipo | Qué hace | Riesgo para consumidores | Requiere |
|---|---|---|---|
| **Aditivo** | Agrega entidades nuevas | Bajo | Release menor |
| **Restrictivo** | Agrega axiomas que acotan | **Alto — puede invalidar datos existentes** | Análisis de impacto + aviso |
| **Correctivo** | Arregla un error de modelado | Medio-alto; cambia inferencias a propósito | Documentar qué inferencias cambian |
| **Refactorización** | Reorganiza sin cambiar semántica | Bajo *si* hay tests que lo prueben | Suite de competency questions |
| **Eliminación** | Quita entidades | **Alto — rompe seguro** | Deprecación previa, nunca borrado directo |

> [!tip] La tabla es utilizable como checklist de revisión: clasificá cada cambio antes de aprobarlo, y exigí el análisis de impacto para las dos filas de riesgo alto.

## Deprecación en vez de borrado

La regla que se sigue de lo anterior: **no borres, deprecá**. Una entidad borrada rompe a todos los consumidores de golpe y sin aviso; una entidad deprecada sigue resolviendo mientras señaliza que va a desaparecer.

El mecanismo estándar:

- **Marcar con `owl:deprecated`** — la entidad sigue existiendo y resolviendo.
- **Documentar el reemplazo** — anotar con qué entidad se sustituye (`dcterms:isReplacedBy` o similar), para que el consumidor sepa hacia dónde migrar.
- **Definir un período de gracia** explícito, comunicado con anticipación.
- **Recién entonces**, y solo en un release mayor, eliminar.

> [!warning] Una ontología pública tiene consumidores que **no sabés que existen**. Cualquiera puede haber importado tu vocabulario sin avisarte. Esa asimetría —a diferencia de una API interna, donde podés enumerar los clientes— es lo que hace que la deprecación no sea una cortesía sino la única política responsable.

> [!tip] Registrá en cada release las entidades deprecadas, su reemplazo y su fecha estimada de eliminación. Es un artefacto barato que ahorra muchísimo soporte.

## Governance — quién decide

El frente organizativo, y donde el capítulo conecta con el problema humano que [[01 - Introduction]] planteaba. Las preguntas que hay que responder explícitamente:

- **¿Quién es dueño de la ontología?** Tiene que haber un responsable identificable. Sin propiedad clara, los cambios se estancan o se hacen sin criterio.
- **¿Quién puede proponer cambios?** Idealmente cualquiera; ese es el punto de tener un proceso.
- **¿Quién los aprueba?** Un comité, un ontologista jefe, un grupo de expertos por dominio. La escala del proyecto define la formalidad razonable.
- **¿Cómo se resuelven desacuerdos entre áreas?** El caso duro: dos áreas con definiciones incompatibles del mismo concepto. Necesita un mecanismo de escalamiento definido de antemano, cuando no hay un conflicto concreto sobre la mesa.
- **¿Cómo se comunican los cambios?** Release notes con foco en qué inferencias cambiaron, no solo qué archivos.

> [!note] **La governance de ontologías es más pesada que la de código** por una razón estructural: la ontología codifica **acuerdos entre áreas**, no decisiones técnicas. Cambiar la definición de *cliente* no es refactorizar — es renegociar un acuerdo organizativo. Por eso el proceso de aprobación necesita representación de las áreas afectadas, no solo criterio técnico.

> [!warning] El modo de falla organizativo más común: la ontología se construye en un proyecto con financiamiento, se despliega, y al terminar el proyecto **nadie queda a cargo**. Los consumidores siguen usándola, el dominio sigue cambiando, y el modelo se degrada hasta que alguien decide que "no sirve" y empieza uno nuevo. El ciclo se repite.

> [!tip] Definí governance **desde el día uno**, aunque sea mínima: un responsable, un canal de propuestas, una política de release. Retrofittear governance sobre una ontología ya desplegada y con consumidores es carísimo y políticamente difícil.

## El ciclo de vida completo

El capítulo cierra articulando el ciclo completo, que es la tesis del libro en forma operativa:

1. **Requisitos** — casos de uso y [[competency questions]] ([[02 - Ontology Development Methodology]]).
2. **Análisis** — terminología y vocabulario del dominio ([[03 - Terminology and Domain Analysis]]).
3. **Diseño** — decisiones de modelado y patrones ([[04 - Modeling Decisions]], [[05 - Ontology Design Patterns and Reuse]]).
4. **Evaluación** — verificación y validación ([[06 - Evaluation and Testing]]).
5. **Release** — versionado, IRIs, documentación.
6. **Mantenimiento** — gestión del cambio, deprecación, governance.
7. **Evolución** — el dominio cambia y el ciclo vuelve a empezar.

> [!note] **El ciclo no termina.** Una ontología viva evoluciona con su dominio; el objetivo nunca fue terminarla sino mantenerla en un estado siempre desplegable, exactamente como el software. La ontología "terminada" es, casi siempre, una ontología abandonada.

## Para aplicar

- **Diseñá la política de IRIs antes del primer release** — opacos vs descriptivos, versionIRI, dominio persistente.
- **Clasificá cada cambio por tipo** antes de aprobarlo, y exigí análisis de impacto para restrictivos y eliminaciones.
- **Nunca borres: deprecá** con `owl:deprecated`, documentá el reemplazo y definí período de gracia.
- **Publicá release notes centradas en qué inferencias cambiaron**, no en qué archivos se tocaron.
- **Nombrá un responsable desde el día uno**, aunque el proyecto sea chico.
- **Definí el mecanismo de escalamiento para desacuerdos entre áreas** antes de que aparezca el primer conflicto.
- **Corré la suite de competency questions antes de cada release** — es tu red de seguridad contra regresiones inferenciales.

## Conexiones

- [[_Ontology Engineering|Ontology Engineering]] — el MOC del libro.
- [[06 - Evaluation and Testing]] — capítulo anterior: los tests que hacen posible cambiar con seguridad · [[08 - Tools and Practical Considerations]] — capítulo siguiente: el herramental que soporta todo esto.
- [[01 - Introduction]] — la tesis de "ontología = artefacto de software" que este capítulo lleva a su conclusión operativa.
- [[OWL]] — `owl:versionIRI`, `owl:deprecated`, `owl:imports`: los mecanismos estándar de versionado.
- [[Semantic Versioning]] — el paralelo del software; adaptarlo requiere pensar la compatibilidad en términos inferenciales.
- [[Architecture Decision Record (ADR)]] — el registro de decisiones de modelado como parte del proceso de governance.
- [[Data Governance]] — el marco organizativo más amplio donde esta governance se inserta; **candidato a nota propia**.
