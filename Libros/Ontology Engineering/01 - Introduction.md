---
title: 01 - Introduction
libro: Ontology Engineering
autor: Elisa F. Kendall / Deborah L. McGuinness
capitulo: 1
created: 2026-08-03
tags:
  - libros/ontology-engineering
  - type/case-study
  - status/stub
aliases:
  - Introduction (Ontology Engineering)
  - Cap 1 - Ontology Engineering
updated: 2026-08-03
---

# 01 - Introduction

> [!info] Capítulo 1 · *Ontology Engineering* — Elisa F. Kendall & Deborah L. McGuinness · Morgan & Claypool, *Synthesis Lectures on Data, Semantics, and Knowledge* (2019)
> El capítulo fundante: define qué es (y qué no es) una **[[ontología]]** en sentido ingenieril, la ubica en el **[[espectro semántico]]** que va del glosario a la lógica formal, y planta la tesis del libro — una ontología es un **artefacto de software con ciclo de vida**, no un diagrama que se dibuja una vez ni un ejercicio filosófico. Presenta el porqué (integración de datos, búsqueda semántica, interoperabilidad, inferencia), los roles humanos involucrados y el panorama de estándares [[RDF]] / [[RDFS]] / [[OWL]] / [[SKOS]]. Navegá: [[_Ontology Engineering|Ontology Engineering]] · siguiente [[02 - Ontology Development Methodology]].

## Resumen

El capítulo abre desmontando un equívoco de entrada: el término **ontología** viene de la filosofía, donde nombra el estudio del ser y de qué existe, pero en ingeniería del conocimiento significa algo mucho más acotado y operativo. Kendall y McGuinness trabajan con la definición pragmática heredada de Gruber y refinada por la comunidad de Semantic Web: una ontología es una **especificación explícita y formal de una conceptualización compartida** de un dominio. Cada palabra de esa definición carga peso — *explícita* (los supuestos quedan escritos, no en la cabeza de alguien), *formal* (procesable por máquina, no prosa), *compartida* (el consenso de una comunidad, no la vista de un individuo), *conceptualización* (un modelo abstracto del dominio, no de una base de datos concreta).

Sobre esa base el capítulo introduce lo que es probablemente su aporte más citable: el **espectro semántico**. Los artefactos de vocabulario no son binarios (ontología / no-ontología) sino un continuo de expresividad creciente — listas de términos controlados, glosarios, taxonomías, thesauri, modelos conceptuales, y por último ontologías lógicas completas con capacidad de razonamiento automático. Cada escalón agrega poder expresivo y, simultáneamente, **costo de construcción y mantenimiento**. El mensaje de las autoras es deliberadamente anti-maximalista: el escalón correcto es el que resuelve tu caso de uso, y subir más alto de lo necesario es desperdicio, no virtud.

De ahí sale la pregunta de negocio — **por qué construir una ontología** — que el capítulo responde con casos de uso concretos: integrar datos heterogéneos que usan vocabularios distintos para lo mismo, habilitar búsqueda semántica que entienda sinónimos y jerarquías, lograr interoperabilidad entre sistemas y organizaciones, inferir conocimiento nuevo a partir del declarado, y —un beneficio subestimado— **documentar el significado compartido** dentro de una organización, donde a menudo el valor aparece antes de que ningún razonador se ejecute.

El resto del capítulo instala la tesis que vertebra el libro entero: la ontología es un **artefacto de ingeniería de software**. Tiene requisitos —capturados como **[[competency questions]]**, las preguntas concretas del dominio que la ontología debe poder responder—, diseño, implementación, testing, versionado, governance y mantenimiento evolutivo. Esto la distingue tajantemente del modo académico de producir ontologías —un ejercicio conceptual que se publica y se abandona— y explica por qué el libro se llama *Ontology Engineering* y no *Ontology Design*. Se presentan también los **roles humanos** en juego (expertos de dominio, ingenieros de conocimiento, desarrolladores, stakeholders) y el problema de traducción entre ellos, que las autoras tratan como riesgo de proyecto de primer orden y no como detalle organizativo. Cierra con un panorama de los estándares del W3C que el libro usará —[[RDF]], [[RDFS]], [[OWL]], [[SKOS]]— y un mapa de los capítulos siguientes.

## Qué es una ontología (y qué no es)

El capítulo arranca por la definición porque el término está sobrecargado. En filosofía, *ontología* (con artículo definido, "la ontología") es la rama de la metafísica que estudia la naturaleza del ser. En ingeniería del conocimiento se usa en plural y como sustantivo contable —"una ontología", "estas tres ontologías"— y designa un artefacto concreto que alguien construye, versiona y despliega.

La definición operativa que el libro adopta desciende de la formulación clásica de **Tom Gruber**: una ontología es una *especificación explícita de una conceptualización*, a la que la comunidad agregó los calificativos *formal* y *compartida*. Conviene desarmarla:

- **Explícita** — los supuestos sobre el dominio están escritos y son inspeccionables. Lo contrario es el conocimiento tácito que vive en la cabeza de los expertos o enterrado en el código de una aplicación.
- **Formal** — expresada en un lenguaje con semántica definida, de modo que una máquina pueda procesarla sin ambigüedad. Un documento Word que describe conceptos en prosa no es una ontología, por más riguroso que sea.
- **Compartida** — refleja el consenso de una comunidad de práctica. Una ontología que solo su autor entiende falló en su propósito, porque el valor está justamente en el acuerdo.
- **Conceptualización** — modela el dominio, no una implementación. Es la diferencia entre modelar *qué es un paciente* y modelar *la tabla `patients` de este sistema*.

> [!note] **Una ontología modela el dominio, no la aplicación.** Este es el criterio que separa una ontología de un esquema de base de datos. El esquema responde a las necesidades de un sistema concreto (performance, normalización, la query que hay que servir); la ontología describe cómo es el mundo del dominio, con independencia de qué sistema lo consuma. Por eso una ontología puede sobrevivir a la aplicación que la originó y ser reusada por otras.

> [!warning] La confusión más frecuente en la práctica es tratar la ontología como un diagrama entidad-relación con otro nombre. No lo es: el modelo E-R describe estructuras de almacenamiento, la ontología describe **significado** — y ese significado tiene consecuencias lógicas que un razonador puede computar.

## El espectro semántico

Es la idea vertebral del capítulo y la que más se cita del libro. Los artefactos que capturan vocabulario de un dominio no se dividen en "ontologías" y "no-ontologías": forman un **continuo de expresividad creciente**. En cada escalón se puede decir más sobre el dominio, y una máquina puede inferir más — pero también sube el costo de construir, acordar y mantener.

De menor a mayor expresividad:

- **Vocabulario controlado / lista de términos** — un conjunto acordado de términos permitidos, sin relaciones entre ellos. Resuelve el problema básico de que cada uno escriba lo mismo de forma distinta.
- **Glosario** — términos con definiciones en lenguaje natural. Útil para humanos; una máquina no puede hacer nada con la definición.
- **Taxonomía** — términos organizados jerárquicamente por relaciones de subsunción (*es un tipo de*). Aparece la primera estructura explotable por máquina: la herencia.
- **Thesaurus** — agrega relaciones además de la jerárquica: términos más amplios y más estrechos, términos relacionados, sinónimos y términos preferidos. Es el territorio de [[SKOS]].
- **Modelo conceptual** — clases con propiedades, relaciones tipadas, restricciones de cardinalidad. Se acerca a lo que un modelador de datos reconocería, pero orientado al dominio.
- **Ontología lógica formal** — axiomas expresados en lógica descriptiva: disjunción, restricciones de propiedad, clases definidas por condiciones necesarias y suficientes, propiedades transitivas e inversas. Habilita **razonamiento automático**: clasificación, detección de inconsistencias, inferencia de hechos no declarados. Es el territorio de [[OWL]].

> [!tip] **Elegí el escalón por el caso de uso, no por ambición.** Si tu problema es que tres equipos llaman distinto a la misma cosa, un vocabulario controlado lo resuelve. Si necesitás que el sistema deduzca que un paciente cumple los criterios de un ensayo clínico sin que nadie lo haya declarado, ahí sí necesitás una ontología lógica. Construir OWL con razonamiento donde alcanzaba SKOS es sobre-ingeniería: pagás complejidad, tiempo de modelado y costo de mantenimiento por capacidad que nadie va a usar.

> [!warning] El error simétrico también existe y es más caro a largo plazo: quedarse en taxonomía cuando el caso de uso pide inferencia lleva a que la lógica faltante termine hardcodeada en la aplicación, dispersa y duplicada en cada sistema consumidor. Ahí se pierde justamente el beneficio de tener el significado en un lugar declarativo y único.

### Tabla 1.1 — El espectro semántico

| Escalón | Qué agrega | Puede inferir la máquina | Estándar típico | Caso de uso representativo |
|---|---|---|---|---|
| **Vocabulario controlado** | Términos acordados | Nada (solo validación de pertenencia) | Listas, code sets | Normalizar valores de un campo |
| **Glosario** | Definiciones en lenguaje natural | Nada | Documento, [[SKOS]] parcial | Alinear el entendimiento humano |
| **Taxonomía** | Jerarquía *es-un-tipo-de* | Herencia por subsunción | [[SKOS]], [[RDFS]] | Navegación y clasificación por facetas |
| **Thesaurus** | Sinónimos, términos relacionados, broader/narrower | Expansión de consultas | [[SKOS]] | Búsqueda que tolera sinónimos |
| **Modelo conceptual** | Clases, propiedades tipadas, cardinalidades | Validación estructural | [[RDFS]], [[OWL]] básico, [[SHACL]] | Integración de esquemas |
| **Ontología lógica** | Axiomas formales, clases definidas, disjunción | Clasificación, consistencia, hechos nuevos | [[OWL]] 2 | Inferencia y razonamiento automático |

> [!note] La tabla deja clara la relación que el capítulo quiere instalar: **expresividad e inferencia crecen juntas, y el costo con ellas**. No hay un escalón "mejor" en abstracto — hay un escalón adecuado para cada pregunta que el sistema tenga que responder.

## Por qué construir una ontología

El capítulo insiste en anclar la decisión en casos de uso concretos, porque una ontología sin caso de uso es un modelo que nadie sabe cuándo está terminado. Los motivadores recurrentes:

- **Integración de datos heterogéneos** — el caso clásico. Distintas fuentes usan vocabularios distintos para las mismas entidades (`cliente`, `customer`, `cuenta_titular`). La ontología provee el modelo común contra el cual se mapean las fuentes, en lugar de construir mapeos punto-a-punto que crecen cuadráticamente con la cantidad de sistemas.
- **Búsqueda semántica y recuperación de información** — permitir que una consulta por *"cardiopatía"* recupere documentos que dicen *"insuficiencia cardíaca"*, porque la ontología sabe que uno es un tipo del otro. Sin ese conocimiento la búsqueda es puramente léxica.
- **Interoperabilidad entre sistemas y organizaciones** — un vocabulario compartido con semántica explícita permite que dos organizaciones intercambien datos sin negociar el significado en cada integración. Es el motivador dominante en dominios regulados (salud, finanzas, defensa).
- **Inferencia de conocimiento nuevo** — derivar hechos no declarados explícitamente a partir de los axiomas. Es la capacidad que justifica subir hasta el escalón de [[OWL]], y la que no se puede obtener de ningún escalón inferior.
- **Documentación del significado compartido** — el beneficio más subestimado. El proceso mismo de construir la ontología obliga a la organización a explicitar y acordar qué significan sus términos, y ese acuerdo tiene valor **antes de que ningún razonador se ejecute**. Muchos proyectos cosechan aquí su retorno principal.

### Competency questions — el instrumento que ancla todo lo demás

Todos esos motivadores comparten un problema: son demasiado vagos para guiar el modelado. *"Integrar datos"* no dice qué clases hacen falta. El capítulo introduce acá el instrumento que resuelve esa vaguedad y que va a reaparecer en cada decisión del libro.

> [!note] **Definición.** Una **[[competency questions|competency question]]** es una pregunta concreta, formulada en lenguaje natural y en el vocabulario del negocio, que la ontología debe poder responder una vez construida. No es una pregunta sobre la ontología ("¿cuántas clases tiene?") sino una pregunta **del dominio** que el sistema tendrá que contestar: *"¿qué medicamentos están contraindicados para un paciente con insuficiencia renal?"*, *"¿qué empleados tienen la certificación para operar este equipo?"*, *"¿qué pólizas cubren este siniestro en esta jurisdicción?"*.

Lo que las hace el instrumento central del libro es que cumplen **tres funciones a la vez**, que normalmente vivirían en documentos separados y desincronizados:

- **Requisitos** — definen qué tiene que estar modelado. Si una pregunta menciona *contraindicación*, *insuficiencia renal* y *medicamento*, esos conceptos y sus relaciones tienen que existir en la ontología.
- **Alcance** — y su contracara, la frontera: lo que **ninguna** competency question toca queda deliberadamente afuera. Da un criterio objetivo para rechazar pedidos de modelado que nadie va a usar.
- **Criterio de terminación** — la ontología está lista cuando las responde todas. Y son verificables: se traducen a consultas [[SPARQL]] que se ejecutan contra la ontología poblada.

> [!tip] Escribilas **antes de modelar nada**. Sin ellas el modelado no tiene fin natural —siempre hay un concepto más que agregar— y el proyecto deriva hacia el modelado por el modelado. Con ellas, tenés a la vez el punto de partida y la línea de llegada.

> [!note] Su otra virtud es **política**: se escriben en lenguaje de negocio, así que un stakeholder que no sabe qué es una lógica descriptiva puede leerlas, discutirlas y priorizarlas. Es el único artefacto del proyecto que todos los roles comprenden sin traducción. [[02 - Ontology Development Methodology]] las desarrolla como método completo.

## La ontología como artefacto de software

Acá está la tesis que sostiene el libro completo y que justifica la palabra *Engineering* del título. Una ontología **no** es un diagrama conceptual que se dibuja una vez, se publica y se olvida. Es un artefacto de software, y en consecuencia tiene todo el ciclo de vida de uno:

- **Requisitos** — capturados como competency questions, casos de uso y alcance explícito del dominio (con sus fronteras: qué queda afuera).
- **Diseño** — decisiones de modelado, reuso de ontologías existentes, elección del nivel de expresividad, aplicación de patrones de diseño.
- **Implementación** — codificación en el lenguaje elegido, con las convenciones de nomenclatura y URIs que la organización adopte.
- **Testing** — verificación contra las competency questions, chequeo de consistencia con un razonador, detección de clases insatisfacibles.
- **Versionado y release** — la ontología cambia, y los sistemas que dependen de ella necesitan estabilidad. Versionado, políticas de deprecación y compatibilidad hacia atrás son problemas reales.
- **Governance** — quién puede cambiarla, cómo se aprueban los cambios, cómo se resuelven los desacuerdos entre stakeholders.
- **Mantenimiento evolutivo** — el dominio cambia, y la ontología con él.

> [!note] **La consecuencia práctica de tratarla como software**: aplican las disciplinas que ya conocés. Control de versiones, revisión de cambios, integración continua que corre el razonador, releases con notas, deprecación en vez de borrado. Una ontología sin governance ni versionado se degrada exactamente igual que un código base sin ellos.

> [!warning] El modo de falla característico del enfoque académico es la **ontología huérfana**: técnicamente elegante, lógicamente impecable, publicada en un paper — y sin ningún sistema que la consuma ni nadie que la mantenga. El libro está escrito en buena medida contra ese resultado.

## Roles y stakeholders

Construir una ontología es una actividad **socio-técnica**, y el capítulo trata el factor humano como riesgo de proyecto de primer orden, no como nota al pie:

- **Expertos de dominio** — poseen el conocimiento a modelar, pero normalmente no saben expresarlo formalmente ni conocen las consecuencias lógicas de lo que afirman.
- **Ingenieros de conocimiento / ontologistas** — saben modelar formalmente, pero no conocen el dominio. Su trabajo real es de elicitación: extraer del experto lo que sabe y traducirlo a axiomas.
- **Desarrolladores de aplicaciones** — consumen la ontología. Sus necesidades condicionan qué tiene que estar modelado y con qué nivel de detalle; si no participan, la ontología termina resolviendo problemas que nadie tiene.
- **Stakeholders de negocio** — financian el proyecto y definen los casos de uso que justifican su existencia.

> [!warning] **El problema central es de traducción, no de tecnología.** El experto de dominio dice algo en su jerga; el ontologista lo formaliza; la formalización tiene consecuencias lógicas que el experto no anticipó (por ejemplo, declarar dos clases como disjuntas prohíbe instancias que en la realidad del experto sí existen). Sin ciclos de validación explícitos donde el experto revise las consecuencias —y no solo las afirmaciones— el modelo diverge silenciosamente del dominio real. La mayoría de los fracasos de proyecto viven acá, no en la elección de herramientas.

## Panorama de estándares

El capítulo cierra su parte conceptual presentando los estándares del W3C que el libro va a usar. La lógica de la lista es acumulativa: cada uno se apoya en el anterior y agrega expresividad, replicando el espectro semántico en forma de tecnologías concretas.

- **[[RDF]]** (*Resource Description Framework*) — el modelo de datos base. Todo se expresa como **tripletas** sujeto-predicado-objeto, formando un grafo. Es la capa sintáctica y estructural sobre la que se construye todo lo demás.
- **[[RDFS]]** (*RDF Schema*) — agrega el vocabulario mínimo para describir clases y propiedades: `rdfs:subClassOf`, `rdfs:subPropertyOf`, `rdfs:domain`, `rdfs:range`. Alcanza para taxonomías y modelos conceptuales simples.
- **[[OWL]] 2** (*Web Ontology Language*) — el lenguaje de ontologías propiamente dicho, fundado en **lógicas descriptivas**. Agrega disjunción, cardinalidades, restricciones de propiedad, clases definidas por condiciones necesarias y suficientes, propiedades transitivas, simétricas e inversas. Define **perfiles** (EL, QL, RL) que negocian expresividad contra complejidad computacional del razonamiento. McGuinness es coautora de la especificación original.
- **[[SKOS]]** (*Simple Knowledge Organization System*) — pensado para taxonomías, thesauri y esquemas de clasificación. Modela conceptos con etiquetas preferidas y alternativas, relaciones broader/narrower y related. Es deliberadamente **menos expresivo que OWL** y por eso mucho más barato de construir y mantener: el escalón correcto para vocabularios que no necesitan razonamiento.
- **[[SHACL]]** (*Shapes Constraint Language*) — validación de grafos RDF contra restricciones estructurales. Cubre una necesidad que OWL no atiende bien por su semántica de **mundo abierto**: OWL infiere, SHACL valida.

> [!warning] **Mundo abierto vs mundo cerrado** — la trampa conceptual que más golpea a quien viene de bases de datos. [[OWL]] opera bajo *open world assumption*: lo que no está declarado no es falso, es **desconocido**. Una base de datos asume lo contrario (*closed world*): si no está en la tabla, no existe. Por eso OWL no sirve para validar que un dato falta —para él simplemente no fue declarado todavía— y por eso existe [[SHACL]]. Confundir ambas semánticas produce ontologías que "no detectan errores" que en realidad nunca prometieron detectar.

> [!note] La relación entre los estándares es de **capas acumulativas**, no de alternativas competidoras: RDF da el grafo, RDFS le pone tipos y jerarquía, OWL agrega axiomas lógicos y razonamiento, SKOS ofrece un atajo barato para vocabularios, SHACL valida la forma de los datos.

## Estructura del libro

El capítulo termina mapeando el recorrido. El libro está organizado como un **manual de proyecto**, siguiendo el orden en que las decisiones aparecen en la práctica: primero metodología y captura de requisitos (casos de uso, competency questions, alcance), luego terminología y análisis del vocabulario del dominio, después las decisiones de modelado y los patrones de diseño reusables, y por último las cuestiones de ciclo de vida — testing, evaluación, versionado, governance y mantenimiento.

> [!tip] La brevedad es deliberada: es un volumen de la serie *Synthesis Lectures*, pensado como guía práctica condensada y no como tratado exhaustivo. Leelo como el manual que te dice **qué decisiones hay que tomar y en qué orden**, apoyándote en las especificaciones del W3C para el detalle técnico de cada lenguaje.

## Para aplicar

- **Escribí las competency questions antes de modelar** — cinco a diez preguntas concretas que la ontología debe responder. Son tu documento de requisitos y tu criterio de terminación.
- **Ubicá tu caso en el espectro semántico y quedate en el escalón mínimo que lo resuelva** — si nadie va a correr un razonador, no necesitás [[OWL]]; [[SKOS]] es más barato de construir y de mantener.
- **Buscá ontologías existentes antes de escribir una nueva** — el reuso es la norma en este campo, no la excepción. Reusar un vocabulario establecido te da además interoperabilidad gratis.
- **Definí governance desde el día uno** — quién aprueba cambios, cómo se versiona, qué política de deprecación aplica. Retrofittear governance sobre una ontología ya desplegada es carísimo.
- **Involucrá al experto de dominio en la validación de consecuencias, no solo de afirmaciones** — mostrale lo que el razonador infiere, no solo lo que vos escribiste. Ahí aparecen los desacuerdos que importan.
- **Tratála como código** — repositorio, control de versiones, revisión de cambios, y un chequeo de consistencia con razonador corriendo en integración continua.

## Conexiones

- [[_Ontology Engineering|Ontology Engineering]] — el MOC del libro.
- [[02 - Ontology Development Methodology]] — capítulo siguiente: la metodología de proyecto que operacionaliza esta introducción.
- [[espectro semántico]] — la idea vertebral del capítulo; **candidato fuerte a nota propia**.
- [[competency questions]] — el mecanismo de requisitos que atraviesa todo el libro; **candidato a nota propia**.
- [[ontología]] · [[RDF]] · [[RDFS]] · [[OWL]] · [[SKOS]] · [[SHACL]] — los estándares del stack semántico.
- [[Knowledge graph]] — el pariente cercano e industrialmente dominante de la ontología; conviene una nota que contraste ambos términos.
- [[Graph RAG]] — dónde este conocimiento se cruza con el trabajo de recuperación aumentada del vault.
- [[Model Context Protocol (MCP)]] — otro caso de vocabulario compartido para interoperabilidad, en el dominio agéntico.
