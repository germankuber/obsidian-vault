---
title: 04 - Modeling Decisions
libro: Ontology Engineering
autor: Elisa F. Kendall / Deborah L. McGuinness
capitulo: 4
created: 2026-08-03
tags:
  - libros/ontology-engineering
  - type/case-study
  - status/stub
aliases:
  - Modeling Decisions
  - Cap 4 - Decisiones de modelado
updated: 2026-08-03
---

# 04 - Modeling Decisions

> [!info] Capítulo 4 · *Ontology Engineering* — Elisa F. Kendall & Deborah L. McGuinness · Morgan & Claypool, *Synthesis Lectures on Data, Semantics, and Knowledge* (2019)
> El corazón técnico: **convertir el vocabulario analizado en axiomas formales**, y las decisiones recurrentes que eso implica. Cubre clase vs instancia, la construcción de jerarquías sanas, propiedades y sus características lógicas, restricciones y cardinalidades, clases **primitivas vs definidas**, disjunción, y las consecuencias del **mundo abierto** de [[OWL]] sobre todo lo anterior. Navegá: [[_Ontology Engineering|Ontology Engineering]] · anterior [[03 - Terminology and Domain Analysis]] · siguiente [[05 - Ontology Design Patterns]].

## Resumen

Este es el capítulo donde el vocabulario se vuelve lógica. Las autoras lo tratan como un catálogo de **decisiones recurrentes** —no como un tutorial de sintaxis [[OWL]]— porque el problema real del modelador no es escribir el axioma sino elegir cuál escribir. Cada decisión tiene alternativas legítimas, y la que corresponde depende del caso de uso y de las [[competency questions]], nunca de una regla universal.

Las decisiones se agrupan en cinco frentes. Primero, **clase vs instancia**: dónde termina el nivel de tipos y empieza el de individuos, con el caso incómodo de las entidades que son ambas cosas según el punto de vista (metamodelado). Segundo, la **jerarquía**: cómo construir taxonomías que no mientan, evitando la confusión entre subsunción y meronimia y las jerarquías profundas por prolijidad estética. Tercero, las **propiedades** y sus características lógicas —dominio, rango, funcionalidad, transitividad, simetría, inversas— que son axiomas con consecuencias inferenciales reales, no anotaciones decorativas. Cuarto, las **restricciones y cardinalidades**, incluyendo la distinción crítica entre restricción existencial y universal. Quinto, y con estatus propio, las **clases primitivas vs definidas**: la diferencia entre condiciones necesarias y condiciones necesarias y suficientes, que es lo que separa una taxonomía manual de una ontología que el razonador puede clasificar sola.

Atravesando todo el capítulo hay una advertencia constante: el **mundo abierto** de OWL cambia el significado de casi todas estas decisiones respecto de lo que un modelador de bases de datos espera. Una restricción de cardinalidad no valida que falte un dato — habilita una inferencia. Ese desajuste de expectativas es la fuente principal de frustración de quien llega a OWL desde el mundo relacional, y el capítulo vuelve sobre él una y otra vez.

## Clase o instancia — dónde cortar

La primera decisión, y la que más discusiones genera. Una **clase** representa un tipo, un conjunto de cosas; una **instancia** (individuo) representa una cosa concreta.

El problema es que muchas entidades del dominio admiten las dos lecturas según qué pregunta querés responder:

- *Golden Retriever* — ¿clase de perros, o instancia de la clase *Raza*?
- *Aspirina* — ¿clase de comprimidos concretos, o instancia de la clase *Medicamento*?
- *Boeing 747* — ¿clase de aviones, o instancia de *Modelo de aeronave*?

> [!note] **La respuesta la dan las competency questions.** Si necesitás preguntar *"¿cuántas razas registró el club canino en 2024?"*, entonces *Raza* es una clase y *Golden Retriever* su instancia. Si necesitás *"¿qué perros son Golden Retriever?"*, entonces *Golden Retriever* es una clase con instancias perro. Si necesitás ambas, entrás en territorio de metamodelado.

- **Regla práctica** — si vas a contar, listar o adjuntar propiedades a la entidad *como tal* (la raza tiene un origen, un estándar, una fecha de reconocimiento), es instancia. Si vas a clasificar cosas bajo ella, es clase.
- **Metamodelado** — [[OWL]] 2 permite *punning*: usar el mismo IRI como clase y como individuo, y el razonador los trata como entidades separadas que comparten nombre. Resuelve el caso, pero **agrega complejidad conceptual**; no lo uses como salida fácil ante una duda que las competency questions podrían resolver.

> [!warning] Meter **instancias masivas de datos** dentro de la ontología es un error de diseño frecuente. La ontología modela el esquema conceptual; los millones de registros viven en un almacén que la referencia. Cargar el razonador con datos operativos degrada la performance sin dar nada a cambio.

## Construir jerarquías que no mientan

La jerarquía de subsunción es la columna vertebral del modelo, y también donde se cometen los errores más consecuentes — porque un error de jerarquía se **propaga por herencia** a todo lo que cuelga debajo.

Los errores recurrentes:

- **Confundir subsunción con meronimia** — el error que ya advertía [[03 - Terminology and Domain Analysis]]. *Rueda* no es subclase de *Auto*; es parte de un auto. Modelarlo como subclase hace que el razonador infiera que toda rueda es un auto.
- **Confundir subsunción con instanciación** — *Firulais* no es una subclase de *Perro*; es una instancia. La subclase se usa para tipos, no para individuos.
- **Jerarquías profundas por prolijidad** — cada nivel intermedio debe justificarse por una pregunta que responde o una propiedad que introduce. Los niveles decorativos agregan mantenimiento sin capacidad.
- **Herencia múltiple sin criterio** — colgar una clase de varios padres es legítimo, pero cuando los padres pertenecen a criterios de clasificación distintos (por función, por material, por origen) la jerarquía se vuelve un enredo. Mejor: separar los criterios en propiedades y dejar que el razonador clasifique.

> [!tip] **El test de subsunción**: *"¿todo X es también un Y, sin excepciones?"* Si tenés que decir "casi siempre" o "depende", no es subsunción. Aplicalo a cada arista de la jerarquía antes de aceptarla.

> [!warning] La subsunción **hereda todo**: propiedades, restricciones y consecuencias. Un axioma equivocado en un nodo alto contamina silenciosamente todas sus subclases. Por eso los errores de jerarquía son los más caros de descubrir tarde.

## Propiedades y sus características lógicas

En [[OWL]] las propiedades no son campos: son **relaciones con semántica formal**, y sus características habilitan inferencias reales.

- **Object property vs datatype property** — la primera relaciona dos individuos del dominio (*trabajaEn*, *esParteDe*); la segunda relaciona un individuo con un valor literal (*fechaNacimiento*, *peso*). La distinción no es cosmética: solo las object properties admiten características lógicas.
- **Dominio (`domain`) y rango (`range`)** — acá está la trampa mayor. En una base de datos, declarar un tipo **restringe** lo que se puede insertar. En OWL, dominio y rango son **axiomas de inferencia**: si declarás que el dominio de *trabajaEn* es *Persona* y luego afirmás que una silla *trabajaEn* una oficina, el razonador **no reporta un error** — infiere que esa silla es una Persona.
- **Funcional** — el individuo tiene a lo sumo un valor para esa propiedad (*tienePadreBiológico*). Habilita inferir que dos valores distintos son en realidad el mismo individuo.
- **Transitiva** — si A se relaciona con B y B con C, entonces A con C. Típica de *esParteDe* y *esAntepasadoDe*.
- **Simétrica / asimétrica** — *esHermanoDe* es simétrica; *esPadreDe* es asimétrica.
- **Inversa** — *tieneAutor* / *esAutorDe*. Declarar la inversa permite navegar y razonar en ambos sentidos sin duplicar afirmaciones.

> [!warning] **`domain` y `range` no validan: infieren.** Es probablemente el malentendido más costoso para quien viene del mundo relacional. Si querés que un dato mal tipado sea reportado como error, necesitás [[SHACL]], no OWL. Usar `domain`/`range` esperando validación produce ontologías que "no detectan errores" que nunca prometieron detectar.

> [!tip] Declarar una propiedad como transitiva o funcional tiene **costo de razonamiento**. En ontologías grandes, características usadas sin necesidad real degradan seriamente el tiempo de clasificación. Declaralas cuando habiliten una inferencia que alguna competency question necesita.

### Tabla 4.1 — Características de propiedades y qué habilitan

| Característica | Qué afirma | Ejemplo | Qué infiere el razonador |
|---|---|---|---|
| **Funcional** | A lo sumo un valor por individuo | *tieneMadreBiológica* | Si hay dos valores, son el mismo individuo |
| **Inversa funcional** | El valor identifica al sujeto | *númeroDeDocumento* | Dos sujetos con el mismo valor son el mismo individuo |
| **Transitiva** | Se encadena | *esParteDe* | Parte de una parte es parte del todo |
| **Simétrica** | Vale en ambos sentidos | *esHermanoDe* | Si A→B, entonces B→A |
| **Asimétrica** | Nunca vale al revés | *esPadreDe* | Si A→B, B→A es inconsistente |
| **Inversa** | Relación opuesta | *tieneAutor* / *esAutorDe* | Navegación y razonamiento bidireccional |

> [!note] Cada fila es un axioma con consecuencias computables. La tabla es la respuesta a *"¿por qué declararía esto?"*: se declara cuando la inferencia de la última columna es algo que el caso de uso necesita.

## Restricciones y cardinalidades

Las restricciones acotan qué valores puede tomar una propiedad para los miembros de una clase. La distinción fundamental:

- **Restricción existencial (`someValuesFrom`)** — *todo Padre tiene al menos un Hijo*. Afirma existencia.
- **Restricción universal (`allValuesFrom`)** — *todos los hijos de un Padre son Personas*. Afirma que **si** hay valores, son de ese tipo — pero **no exige que haya alguno**.
- **Cardinalidad** — mínima, máxima o exacta: *un auto tiene exactamente cuatro ruedas*.

> [!warning] **La restricción universal no implica existencia.** Declarar que *todos los ingredientes de una PizzaVegetariana son vegetales* es satisfecho trivialmente por una pizza **sin ningún ingrediente**. Es el error clásico del tutorial de la pizza, y sigue siendo el más cometido: si querés exigir que haya al menos un vegetal, necesitás también una restricción existencial.

> [!note] Bajo **mundo abierto**, una cardinalidad mínima de 1 no significa "este dato es obligatorio y falta". Significa que el razonador **puede inferir** que ese valor existe aunque no esté declarado. OWL nunca se queja de un dato ausente: asume que todavía no lo sabemos.

## Clases primitivas vs definidas

Es la distinción de mayor rendimiento del capítulo y la que separa una ontología que el razonador puede aprovechar de una taxonomía dibujada a mano.

- **Clase primitiva** — se describe con **condiciones necesarias**. Todo *Vino* tiene un productor y una añada, pero cumplir esas condiciones no te convierte en Vino. La pertenencia se **afirma** manualmente.
- **Clase definida** — se describe con **condiciones necesarias y suficientes**. *VinoTinto* = todo vino cuyo color es tinto. Cualquier individuo que cumpla la condición **es** un VinoTinto, lo haya dicho alguien o no.

> [!note] **Las clases definidas son las que hacen que el razonador trabaje para vos.** Declarás las condiciones una vez y el razonador clasifica automáticamente cada individuo y cada subclase que las cumpla. Sin clases definidas, toda la jerarquía es manual y el razonador solo verifica consistencia — desaprovechás la razón principal para haber subido al escalón [[OWL]] del [[espectro semántico]].

> [!tip] Patrón habitual: modelá la jerarquía **primitiva** con las distinciones esenciales del dominio (las que no dependen de propiedades variables), y agregá clases **definidas** para las categorías que se derivan de propiedades — *ClienteMoroso*, *PacienteDeAltoRiesgo*, *PedidoUrgente*. Esas categorías cambian solas cuando cambian los datos, sin reclasificar nada a mano.

> [!warning] Marcar como definida una clase cuyas condiciones no son realmente suficientes produce **clasificaciones erróneas silenciosas**: el razonador va a meter individuos donde no corresponde y nadie lo va a notar hasta que una consulta devuelva basura.

### Tabla 4.2 — Primitiva vs definida

| Aspecto | Clase primitiva | Clase definida |
|---|---|---|
| **Condiciones** | Necesarias | Necesarias **y** suficientes |
| **Pertenencia** | Se afirma manualmente | La infiere el razonador |
| **Uso típico** | Distinciones esenciales del dominio | Categorías derivadas de propiedades |
| **Ejemplo** | *Vino*, *Persona*, *Producto* | *VinoTinto*, *ClienteMoroso*, *PedidoUrgente* |
| **Si te equivocás** | Falta una clasificación | Clasificación errónea silenciosa |

> [!tip] La última fila marca la asimetría de riesgo: equivocarse en primitiva es un hueco visible; equivocarse en definida es un error invisible. Ante la duda, primitiva.

## Disjunción y el mundo abierto

- **Disjunción** — declarar que dos clases no comparten instancias (*Hombre* y *Mujer* disjuntas, *Vivo* y *Muerto* disjuntas). Sin disjunción explícita, OWL **no asume** que las clases sean distintas: mientras nadie lo prohíba, un individuo podría pertenecer a ambas.
- **Mundo abierto (open world assumption)** — lo no declarado es **desconocido**, no falso. Es la diferencia semántica de fondo con las bases de datos.
- **Sin nombres únicos (no unique name assumption)** — dos IRIs distintos pueden referirse al mismo individuo, salvo que se declare lo contrario.

> [!warning] **La disjunción es el axioma más subestimado y el más peligroso.** Es indispensable para que el razonador detecte inconsistencias reales, pero declararla a la ligera produce **clases insatisfacibles**: si declarás *Vegetariano* y *Carnívoro* disjuntos y después modelás algo que es ambos, esa clase no puede tener instancias. El razonador lo reporta; nunca es cosmético.

> [!note] La combinación de mundo abierto y ausencia de nombres únicos es lo que hace que OWL se comporte de forma contraintuitiva para quien viene de SQL. **OWL infiere, [[SHACL]] valida.** Si tu necesidad es "reportar que este campo está vacío", OWL no es la herramienta — y no por limitación, sino por diseño.

## Para aplicar

- **Resolvé clase vs instancia con las competency questions**, no con la intuición; usá punning solo si el caso realmente lo exige.
- **Aplicá el test de subsunción a cada arista** de la jerarquía antes de aceptarla, y separá explícitamente meronimia de subsunción.
- **Declará características de propiedad solo cuando habiliten una inferencia que necesitás** — cada una cuesta tiempo de razonamiento.
- **No uses `domain`/`range` esperando validación**: si necesitás validar, sumá [[SHACL]].
- **Combiná existencial y universal** cuando quieras exigir que exista al menos un valor del tipo correcto; la universal sola no lo hace.
- **Usá clases definidas para las categorías derivadas** (moroso, urgente, de alto riesgo) y dejá que el razonador las mantenga.
- **Declará disjunción deliberadamente** y corré el razonador después de cada una: cualquier clase insatisfacible es bloqueante.

## Conexiones

- [[_Ontology Engineering|Ontology Engineering]] — el MOC del libro.
- [[03 - Terminology and Domain Analysis]] — capítulo anterior: el vocabulario analizado que acá se formaliza · [[05 - Ontology Design Patterns]] — capítulo siguiente: soluciones reusables a estas mismas decisiones.
- [[OWL]] — el lenguaje donde estas decisiones se expresan; **candidato a nota propia con los perfiles EL/QL/RL**.
- [[SHACL]] — la contraparte de validación; la respuesta a lo que OWL deliberadamente no hace.
- [[competency questions]] — el criterio que resuelve cada decisión de este capítulo.
- [[espectro semántico]] — las clases definidas son la capacidad que justifica haber subido al escalón OWL.
- [[Description Logic]] — el fundamento formal detrás de estas construcciones; **candidato a nota propia**.
