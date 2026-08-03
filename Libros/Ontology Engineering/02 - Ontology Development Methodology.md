---
title: 02 - Ontology Development Methodology
libro: Ontology Engineering
autor: Elisa F. Kendall / Deborah L. McGuinness
capitulo: 2
created: 2026-08-03
tags:
  - libros/ontology-engineering
  - type/case-study
  - status/stub
aliases:
  - Ontology Development Methodology
  - Cap 2 - Metodología de desarrollo
updated: 2026-08-03
---

# 02 - Ontology Development Methodology

> [!info] Capítulo 2 · *Ontology Engineering* — Elisa F. Kendall & Deborah L. McGuinness · Morgan & Claypool, *Synthesis Lectures on Data, Semantics, and Knowledge* (2019)
> Del *qué es* al *cómo se construye*: el capítulo operacionaliza la tesis del capítulo anterior — si la [[ontología]] es un artefacto de software, necesita un **proceso de desarrollo**. Cubre el ciclo de vida iterativo, la captura de requisitos vía **[[competency questions]]**, la definición de alcance y sus fronteras, la elicitación de conocimiento con expertos de dominio, la evaluación de **reuso** antes de construir, y los criterios para decidir cuándo la ontología está lista. Navegá: [[_Ontology Engineering|Ontology Engineering]] · anterior [[01 - Introduction]] · siguiente [[03 - Terminology and Domain Analysis]].

## Resumen

Si el capítulo 1 argumentó que una ontología es un artefacto de software, este capítulo cobra la factura: un artefacto de software necesita un **proceso**. Y el proceso que las autoras proponen es explícitamente **iterativo e incremental**, no cascada. La razón es de fondo, no de moda metodológica: nadie —ni el experto de dominio, ni el ontologista— sabe al empezar qué conceptos hacen falta ni con qué granularidad. El conocimiento del dominio se descubre modelando, y los errores de modelado solo se vuelven visibles cuando la ontología se prueba contra preguntas reales. Un proceso que exige congelar los requisitos antes de modelar contradice cómo funciona la actividad.

El corazón operativo del capítulo son las **competency questions**: las preguntas concretas que la ontología debe poder responder. Ya aparecieron en el capítulo 1 como criterio de terminación; acá se convierten en el instrumento central del proceso. Cumplen tres funciones simultáneas que normalmente estarían en documentos separados — son el **documento de requisitos** (qué tiene que cubrir), la **definición de alcance** (lo que ninguna pregunta toca, queda afuera) y la **suite de tests** (la ontología está lista cuando las responde todas). Esa triple función es lo que las hace tan valiosas: alinean stakeholders sin necesidad de que nadie entienda lógica descriptiva, porque una pregunta de negocio se discute en lenguaje de negocio.

Alrededor de ese núcleo el capítulo despliega las demás actividades del ciclo: **definir alcance y fronteras** (con el foco puesto en lo que queda *afuera*, que es donde los proyectos se desbordan), **elicitar conocimiento** del experto de dominio con técnicas que no le exijan hablar en axiomas, **evaluar reuso** de ontologías y vocabularios existentes antes de escribir nada propio, **modelar** en ciclos cortos, y **evaluar** contra las competency questions y contra la consistencia lógica. El capítulo insiste en que estas actividades no son fases secuenciales sino hilos que conviven en cada iteración.

El hilo conductor: **el proceso existe para contener dos riesgos específicos** — el modelado sin fin (la ontología que nunca está terminada porque nadie definió qué es terminar) y la divergencia silenciosa entre el modelo formal y el dominio real (el experto valida lo que dijo, pero nunca las consecuencias lógicas de lo que dijo). Casi todo lo demás del capítulo es instrumental a esos dos problemas.

## El ciclo de vida es iterativo, no cascada

La primera decisión de proceso que el capítulo establece es que el desarrollo de ontologías es **iterativo e incremental por naturaleza**, y que intentar imponerle un modelo cascada es una fuente activa de fracaso.

El argumento no es estilístico. Es que la información necesaria para diseñar bien **no existe al comienzo del proyecto**:

- El experto de dominio no sabe qué conceptos son relevantes hasta que ve un modelo que no los tiene y falla.
- El ontologista no sabe qué granularidad hace falta hasta que intenta responder una competency question y descubre que el modelo no distingue lo suficiente.
- Los consumidores de la ontología descubren requisitos nuevos cuando ven la primera versión funcionando.
- Las consecuencias lógicas de un conjunto de axiomas rara vez son obvias hasta que un razonador las computa.

> [!note] **El ciclo básico**: definir un subconjunto acotado del alcance → escribir competency questions para ese subconjunto → modelar lo mínimo que las responda → evaluar (¿las responde? ¿es consistente?) → validar con el experto → ampliar el alcance y repetir. Cada vuelta produce una ontología **usable**, no un fragmento inservible que solo cobra sentido al final.

> [!tip] Arrancá por el subconjunto del dominio que tenga **el caso de uso más claro y el consumidor más cercano**. Una ontología pequeña que un sistema real ya está usando genera feedback verdadero y justifica la siguiente iteración. Una ontología grande que nadie consume acumula errores de modelado que nadie detecta.

> [!warning] El modo de falla característico es el **modelado sin fin**: sin criterio explícito de terminación, siempre hay un concepto más que agregar, una distinción más fina que hacer, un caso borde más que cubrir. El proyecto deriva hacia el modelado por el modelado y muere por agotamiento presupuestario, no por haber fallado técnicamente.

## Competency questions — requisitos, alcance y tests a la vez

Es el instrumento central del capítulo y probablemente la técnica más práctica del libro entero. Una **competency question** es una pregunta concreta, en lenguaje natural, que la ontología tiene que poder responder una vez construida.

Ejemplos de la forma que toman:

- *¿Qué medicamentos están contraindicados para un paciente con insuficiencia renal?*
- *¿Qué componentes de este producto provienen de proveedores de una región determinada?*
- *¿Qué empleados tienen la certificación requerida para operar este equipo?*
- *¿Qué pólizas cubren este tipo de siniestro en esta jurisdicción?*

Lo que las hace poderosas es que cumplen **tres roles a la vez**:

- **Requisitos** — definen qué tiene que estar modelado. Si una pregunta menciona *contraindicación*, *insuficiencia renal* y *medicamento*, esos tres conceptos y la relación entre ellos tienen que existir en la ontología.
- **Alcance** — y su contracara, la frontera. Lo que **ninguna** competency question toca, queda deliberadamente afuera. Este es el uso más subestimado y el más valioso políticamente: da un criterio objetivo para rechazar pedidos de modelado que nadie va a usar.
- **Tests** — son la suite de aceptación. La ontología está lista cuando las responde todas. Y son ejecutables: se traducen a consultas [[SPARQL]] que corren contra la ontología poblada y verifican que devuelvan lo esperado.

> [!note] **Las competency questions se escriben en lenguaje de negocio, no en lenguaje formal.** Esa es su virtud política: un stakeholder que no sabe qué es una lógica descriptiva puede discutir, aprobar y priorizar una lista de preguntas. Es el único artefacto del proyecto que todos los roles pueden leer sin traducción.

> [!tip] Escribilas **antes** de modelar nada, y en cantidad suficiente para cubrir el alcance de la iteración —típicamente entre cinco y quince por iteración—. Priorizalas: no todas valen lo mismo, y las que corresponden al caso de uso principal son las que gobiernan las decisiones de diseño cuando hay tensión.

> [!warning] Una competency question demasiado vaga (*"¿qué sabemos sobre nuestros clientes?"*) no sirve para nada: no acota, no testea, no se traduce a consulta. Si no podés imaginar la respuesta como una lista concreta de resultados, la pregunta todavía no está lista.

### Tabla 2.1 — Las tres funciones de una competency question

| Función | Qué responde | Cuándo se usa | Qué pasa si falta |
|---|---|---|---|
| **Requisito** | ¿Qué tiene que estar modelado? | Al inicio de la iteración, antes de modelar | Se modela por intuición; aparecen conceptos que nadie pidió |
| **Alcance / frontera** | ¿Qué queda afuera? | Al negociar el alcance con stakeholders | El alcance se desborda; el proyecto no termina nunca |
| **Test de aceptación** | ¿Está lista la ontología? | Al cerrar la iteración, como consulta ejecutable | No hay criterio objetivo de terminación |

> [!tip] La tabla deja claro por qué las competency questions valen más que un documento de requisitos tradicional: **un solo artefacto cubre tres necesidades** que de otro modo requerirían tres documentos desincronizados entre sí.

## Alcance y fronteras — el arte de decir qué queda afuera

Definir el alcance de una ontología es, en la práctica, definir sus **fronteras**. Y el trabajo difícil no está en enumerar lo que se incluye —eso surge naturalmente de las competency questions— sino en declarar explícitamente lo que **no** se modela.

Las decisiones de frontera recurrentes:

- **Profundidad de la jerarquía** — ¿modelamos *medicamento* o bajamos hasta *principio activo*, *presentación* y *lote*? La respuesta la da el caso de uso, no la completitud del dominio.
- **Dominios adyacentes** — una ontología de productos toca proveedores, logística, regulación y contabilidad. Cada frontera es una decisión: se modela, se referencia una ontología externa, o se ignora.
- **Instancias vs clases** — ¿los datos concretos viven en la ontología o en un almacén separado que la referencia? Meter instancias masivas en la ontología suele ser un error de diseño con consecuencias serias de performance.
- **Granularidad temporal y de versionado** — ¿la ontología modela el estado actual del dominio o su evolución histórica? Modelar tiempo multiplica la complejidad y rara vez es gratis.

> [!warning] **Las fronteras no declaradas se desbordan solas.** Si nadie escribió "los procesos contables quedan fuera de alcance", en algún momento alguien va a pedir modelar una factura, después un asiento, y el proyecto habrá adquirido un segundo dominio sin que nadie lo decidiera. Escribí las exclusiones con la misma formalidad que las inclusiones.

> [!tip] Un artefacto barato y muy efectivo: junto a la lista de competency questions, mantené una lista de **non-goals** — preguntas que explícitamente NO se van a responder, con una línea de justificación. Es el documento al que se apunta cuando aparece el pedido fuera de alcance, y evita rediscutir la misma frontera cada trimestre.

## Elicitación de conocimiento del experto de dominio

El capítulo 1 planteó que el problema central es de **traducción** entre el experto de dominio y el ontologista. Este capítulo aporta las técnicas concretas para atacarlo.

El punto de partida es una asimetría irreductible: el experto sabe el dominio pero no sabe formalizar; el ontologista sabe formalizar pero no sabe el dominio. Ninguno de los dos puede hacer el trabajo solo, y la comunicación entre ambos es donde el proyecto se gana o se pierde.

Las técnicas que funcionan:

- **Partir de las competency questions** — pedirle al experto que formule las preguntas que su trabajo requiere responder es mucho más productivo que pedirle que enumere conceptos. La gente sabe qué necesita saber; no sabe qué categorías usa implícitamente.
- **Trabajar sobre documentos existentes** — glosarios corporativos, manuales de procedimiento, formularios, esquemas de bases de datos, taxonomías de negocio. El vocabulario del dominio ya está escrito en alguna parte; casi nunca hay que inventarlo desde cero.
- **Analizar ejemplos concretos** — pedirle al experto que recorra un caso real y describa qué entidades intervienen y cómo se relacionan. Los casos concretos revelan distinciones que la descripción abstracta oculta.
- **Confrontar casos borde** — *"¿esto también cuenta como X?"* es la pregunta que descubre dónde están las fronteras reales de un concepto y dónde el experto mismo no tenía una respuesta consolidada.
- **Devolver el modelo en lenguaje natural** — mostrarle al experto lo que el modelo *dice*, parafraseado, en lugar de mostrarle axiomas que no puede leer.

> [!warning] **Validar afirmaciones no alcanza: hay que validar consecuencias.** Este es el punto más importante de toda la sección. El experto revisa lo que el ontologista escribió y dice "sí, es correcto" — pero un conjunto de axiomas correctos individualmente puede tener consecuencias lógicas que el experto jamás habría aceptado. Declarar dos clases como disjuntas es una afirmación inocente que **prohíbe** una categoría de instancias; si en el dominio real esas instancias existen, el modelo acaba de divergir del mundo sin que nadie lo note. Mostrale al experto lo que el **razonador infiere**, no solo lo que vos escribiste.

> [!tip] Un ciclo de validación efectivo: poblá la ontología con instancias reales del dominio, corré el razonador, y llevale al experto las clasificaciones inferidas. *"El sistema dedujo que este caso es de tipo X — ¿es correcto?"* Esa pregunta detecta errores de modelado que ninguna revisión de axiomas detecta.

## Reuso antes que construcción

El capítulo insiste —y es una constante de todo el libro— en que **la norma del campo es reusar, no construir de cero**. Antes de escribir una clase propia, la pregunta obligatoria es si alguien ya modeló ese fragmento del dominio.

Los motivos van más allá del ahorro de esfuerzo:

- **Interoperabilidad gratis** — si dos organizaciones usan el mismo vocabulario establecido, el intercambio de datos entre ellas ya no requiere negociar el significado. Este beneficio es a menudo mayor que el ahorro de tiempo.
- **Calidad probada** — una ontología usada por muchos ya recorrió los casos borde que a vos te faltan encontrar, y sus errores más groseros ya fueron corregidos.
- **Mantenimiento distribuido** — el vocabulario externo lo mantiene alguien más, con recursos que tu proyecto probablemente no tiene.
- **Legitimidad** — en dominios regulados, usar el vocabulario estándar del sector no es una preferencia técnica sino un requisito de facto.

Pero el reuso tiene su propio conjunto de decisiones y de costos:

- **Adoptar completa vs importar parcialmente** — traer una ontología grande entera arrastra su complejidad y sus compromisos de modelado, incluso los que no necesitás.
- **Alineamiento vs importación** — a veces conviene mantener el vocabulario propio y declarar **mapeos** hacia el externo (equivalencias, subsunción) en lugar de importar. Desacopla, al costo de mantener el mapeo.
- **Dependencia de versiones** — si la ontología externa cambia, tu modelo hereda el cambio. Hay que versionar la dependencia igual que se versiona una librería.
- **Desajuste de compromiso ontológico** — dos ontologías pueden usar el mismo término con distinciones incompatibles. Forzar el alineamiento produce modelos que mienten sutilmente.

> [!warning] **Reusar mal es peor que no reusar.** Importar una ontología grande "por las dudas" arrastra cientos de clases irrelevantes, sus axiomas, sus compromisos de modelado y su carga de razonamiento. Si necesitás cinco clases de un vocabulario de mil, importar el módulo mínimo o declarar mapeos es casi siempre mejor que la importación completa.

> [!tip] Evaluá una ontología candidata contra tus propias **competency questions**: si no ayuda a responder ninguna, no la reuses por prestigio. El criterio es el mismo que gobierna todas las demás decisiones del proceso.

## El ciclo de modelado y evaluación

Con requisitos, alcance y decisiones de reuso resueltas, la iteración entra en su núcleo: modelar y evaluar. El capítulo mantiene los ciclos **cortos** deliberadamente, porque la retroalimentación temprana es lo que evita construir sobre un error de modelado.

Las actividades de la iteración:

1. **Modelar el fragmento** — las clases, propiedades y axiomas mínimos que responden las competency questions de esta vuelta. Mínimo es la palabra operativa: agregar "porque en algún momento va a hacer falta" es cómo se acumula el modelado sin fin.
2. **Poblar con instancias de prueba** — datos reales del dominio, aunque sean pocos. Un modelo sin instancias no revela sus problemas.
3. **Correr el razonador** — verificar consistencia lógica y detectar **clases insatisfacibles** (clases que por sus axiomas no pueden tener ninguna instancia: siempre son un error de modelado, nunca una decisión de diseño).
4. **Ejecutar las competency questions** — traducidas a consultas [[SPARQL]], contra la ontología poblada. ¿Devuelven lo que el experto espera?
5. **Validar consecuencias con el experto** — llevarle las inferencias, no los axiomas.
6. **Registrar decisiones** — por qué se modeló así y qué alternativas se descartaron. Sin esto, la siguiente persona rehace la discusión desde cero.

> [!note] **La evaluación tiene dos ejes independientes y ambos son necesarios.** *Verificación*: ¿la ontología está bien construida? (consistente, sin clases insatisfacibles, con nomenclatura coherente). *Validación*: ¿la ontología modela el dominio correcto? (responde las competency questions, el experto acepta las inferencias). Una ontología puede ser impecablemente consistente y describir un dominio que no existe.

> [!warning] Las **clases insatisfacibles** son la señal de alarma más clara y la más ignorada. Si el razonador reporta que una clase no puede tener instancias, hay una contradicción en los axiomas — típicamente disjunciones declaradas a la ligera o restricciones de cardinalidad que se pisan entre sí. Nunca es cosmético, y nunca conviene postergarlo.

### Tabla 2.2 — Verificación vs validación

| Eje | Pregunta | Cómo se comprueba | Quién la resuelve |
|---|---|---|---|
| **Verificación** | ¿Está bien construida? | Razonador: consistencia, clases insatisfacibles, jerarquía inferida | Ontologista |
| **Validación** | ¿Modela el dominio correcto? | Competency questions ejecutadas + revisión de inferencias | Experto de dominio |

> [!note] Los dos ejes requieren **personas distintas** y **herramientas distintas**. Confundirlos es un error de proceso frecuente: el ontologista corre el razonador, ve verde, y declara la ontología terminada sin que ningún experto haya mirado una sola inferencia.

## Cuándo la ontología está lista

El capítulo cierra con el criterio de terminación, que es la contracara del riesgo de modelado sin fin planteado al principio.

Una ontología está lista para la iteración en curso cuando:

- **Responde todas las competency questions** de esa iteración, verificado con consultas ejecutadas contra datos reales.
- **Es lógicamente consistente** y no tiene clases insatisfacibles.
- **El experto de dominio validó las inferencias**, no solo los axiomas.
- **Tiene consumidor** — algún sistema o proceso real la está usando, o está listo para usarla.
- **Está documentada y versionada** — con las decisiones de modelado registradas y un release identificable.

> [!tip] "Lista" siempre significa **lista para esta iteración**, nunca lista definitivamente. Una ontología viva evoluciona con su dominio; el objetivo del proceso no es terminarla sino mantenerla en un estado siempre desplegable, exactamente como el software.

> [!warning] El criterio que **no** sirve es "cuando esté completa". Ningún dominio se modela completamente, y perseguir la completitud es la forma más común de que un proyecto de ontología nunca entregue nada. La completitud se define contra el caso de uso, no contra el dominio.

## Para aplicar

- **Escribí entre cinco y quince competency questions por iteración**, en lenguaje de negocio, priorizadas — y no modeles nada antes de tenerlas.
- **Mantené una lista explícita de non-goals** junto a las competency questions: preguntas que NO se van a responder, con su justificación en una línea.
- **Traducí las competency questions a consultas [[SPARQL]]** y corrélas en cada iteración: es tu suite de tests de aceptación, y debe ser ejecutable, no aspiracional.
- **Corré el razonador en cada ciclo** y tratá cualquier clase insatisfacible como bloqueante, nunca como deuda a postergar.
- **Validá con el experto mostrándole inferencias, no axiomas** — poblá con instancias reales y preguntá si las clasificaciones deducidas son correctas.
- **Buscá ontologías existentes antes de escribir la primera clase**, y evaluá cada candidata contra tus propias competency questions.
- **Registrá las decisiones de modelado y sus alternativas descartadas** — es el equivalente a los ADR del desarrollo de software.
- **Arrancá por el fragmento con el consumidor más cercano** — feedback real desde la primera iteración vale más que un modelo grande sin usuarios.

## Conexiones

- [[_Ontology Engineering|Ontology Engineering]] — el MOC del libro.
- [[01 - Introduction]] — capítulo anterior: define qué es una ontología y planta la tesis de artefacto de software que este capítulo operacionaliza · [[03 - Terminology and Domain Analysis]] — capítulo siguiente: el análisis del vocabulario del dominio que alimenta el modelado.
- [[competency questions]] — el instrumento central del capítulo; **candidato fuerte a nota propia**.
- [[espectro semántico]] — la decisión de qué escalón usar se toma dentro de este proceso, guiada por las competency questions.
- [[SPARQL]] — el lenguaje en que las competency questions se vuelven tests ejecutables; **candidato a nota propia**.
- [[OWL]] · [[SHACL]] — verificación por razonamiento vs validación estructural, los dos mecanismos de evaluación.
- [[Test-Driven Development]] — el paralelo es directo: competency questions son a la ontología lo que los tests al código, escritos antes y usados como criterio de terminación.
- [[Architecture Decision Record (ADR)]] — el registro de decisiones de modelado cumple la misma función en este proceso.
