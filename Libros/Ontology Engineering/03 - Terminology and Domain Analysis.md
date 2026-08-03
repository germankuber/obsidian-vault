---
title: 03 - Terminology and Domain Analysis
libro: Ontology Engineering
autor: Elisa F. Kendall / Deborah L. McGuinness
capitulo: 3
created: 2026-08-03
tags:
  - libros/ontology-engineering
  - type/case-study
  - status/stub
aliases:
  - Terminology and Domain Analysis
  - Cap 3 - Terminología y análisis del dominio
updated: 2026-08-03
---

# 03 - Terminology and Domain Analysis

> [!info] Capítulo 3 · *Ontology Engineering* — Elisa F. Kendall & Deborah L. McGuinness · Morgan & Claypool, *Synthesis Lectures on Data, Semantics, and Knowledge* (2019)
> El trabajo previo al modelado formal: **recolectar, analizar y normalizar el vocabulario real del dominio** antes de convertirlo en clases y propiedades. Cubre las fuentes de terminología, la distinción entre **término** y **concepto**, el tratamiento de sinónimos, homónimos y ambigüedad, la construcción de un glosario acordado, y cómo el análisis del dominio decide qué se vuelve clase, qué propiedad y qué instancia. Navegá: [[_Ontology Engineering|Ontology Engineering]] · anterior [[02 - Ontology Development Methodology]] · siguiente [[04 - Modeling Decisions]].

## Resumen

Entre las [[competency questions]] del capítulo anterior y los axiomas formales del siguiente hay un paso que los proyectos suelen saltear y luego pagan caro: **entender y normalizar el vocabulario que el dominio ya usa**. Las autoras insisten en que el vocabulario no se inventa — ya existe, disperso en glosarios corporativos, manuales, formularios, esquemas de bases de datos y en la cabeza de los expertos. La tarea del ontologista es recolectarlo, analizarlo y depurarlo antes de formalizar nada.

El aporte conceptual central del capítulo es la distinción entre **término** y **concepto**. Un término es una etiqueta lingüística — una cadena de texto que la gente usa. Un concepto es la unidad de significado a la que ese término refiere. La relación entre ambos no es uno a uno, y ahí está todo el problema: varios términos pueden nombrar el mismo concepto (**sinonimia**), y un mismo término puede nombrar conceptos distintos según el contexto (**homonimia** / polisemia). Una ontología modela **conceptos**; los términos son etiquetas que se le cuelgan. Confundir los dos niveles produce ontologías que duplican conceptos porque el dominio los llamaba distinto, o que colapsan conceptos distintos porque compartían nombre.

De esa distinción salen las actividades del capítulo: **recolectar términos** de todas las fuentes disponibles, **agruparlos por concepto** resolviendo sinonimia, **desambiguar** los homónimos, **acordar un término preferido** por concepto (con los demás como alternativos), y producir un **glosario** validado por los expertos. El glosario resultante no es documentación de cortesía: es el insumo directo del modelado, y es donde los desacuerdos entre stakeholders aparecen mientras todavía son baratos de resolver.

El capítulo cierra con el **análisis del dominio** propiamente dicho: mirar el vocabulario normalizado y decidir qué papel juega cada cosa en el modelo — qué se vuelve clase, qué propiedad, qué instancia, y qué relación estructura la jerarquía. Es la bisagra hacia las decisiones de modelado del capítulo siguiente.

## El vocabulario ya existe — hay que encontrarlo

La primera afirmación del capítulo es liberadora y a la vez exigente: **nadie inventa el vocabulario de un dominio**. Toda organización que opera en un dominio ya desarrolló términos para hablar de él, y ese vocabulario está escrito en alguna parte.

Las fuentes habituales, ordenadas más o menos por rendimiento:

- **Glosarios corporativos y diccionarios de datos** — cuando existen, son la fuente más directa. Suelen estar desactualizados, pero dan el punto de partida.
- **Esquemas de bases de datos** — nombres de tablas y columnas revelan el vocabulario operativo real, el que efectivamente se usa. Cuidado: mezclan vocabulario del dominio con artefactos de implementación.
- **Formularios y pantallas** — las etiquetas que ve un usuario final son vocabulario acordado y validado por el uso.
- **Manuales de procedimiento y documentación normativa** — especialmente ricos en dominios regulados, donde el término tiene consecuencias legales.
- **Estándares y vocabularios del sector** — el candidato natural para el [[reuso]] que planteaba el capítulo anterior.
- **Los expertos de dominio** — la fuente que llena los huecos que ninguna de las anteriores cubre, y la única que puede explicar por qué dos términos que parecen sinónimos no lo son.

> [!tip] Empezá por las fuentes escritas y usá al experto para **resolver conflictos**, no para enumerar desde cero. Es mucho más productivo llevarle una lista de treinta términos con dudas concretas que pedirle "contame qué conceptos maneja tu área".

> [!warning] Los esquemas de bases de datos son una fuente valiosa pero contaminada: contienen vocabulario del dominio mezclado con decisiones de implementación (`cliente_tmp`, `flag_activo`, `tabla_maestra_v2`). Extraé el vocabulario del dominio y descartá el resto — importar el esquema tal cual es exactamente el error de modelar la aplicación en vez del dominio que advertía [[01 - Introduction]].

## Término vs concepto — la distinción que ordena todo

Es el núcleo conceptual del capítulo y la fuente de la mayoría de los errores que aparecen después.

- **Término** — la etiqueta lingüística. Una cadena de caracteres que alguien usa para referirse a algo. Vive en un idioma, en un registro, en una comunidad.
- **Concepto** — la unidad de significado. Lo que existe en el dominio con independencia de cómo se lo llame.

La ontología modela **conceptos**. Los términos se adjuntan como etiquetas (`rdfs:label`, `skos:prefLabel`, `skos:altLabel`). Esta separación es lo que permite que la misma ontología sirva a comunidades que usan vocabularios distintos.

> [!note] **La relación término ↔ concepto es de muchos a muchos.** Varios términos pueden apuntar al mismo concepto (sinonimia), y un término puede apuntar a varios conceptos (homonimia). Una ontología bien construida resuelve ambas ambigüedades explícitamente en vez de heredarlas.

Los dos fenómenos que hay que resolver:

- **Sinonimia** — *cliente*, *customer*, *titular de cuenta* y *contraparte* pueden ser el mismo concepto según el área que hable. Si cada área impone su término, la ontología termina con cuatro clases para una sola cosa, y la integración de datos —el motivador original— falla exactamente donde debía funcionar.
- **Homonimia / polisemia** — *póliza* significa cosas distintas para el área comercial (el producto vendido) y para el área legal (el documento contractual). *Cuenta* en contabilidad no es *cuenta* en sistemas. El mismo término, conceptos distintos.

> [!warning] **La homonimia es más peligrosa que la sinonimia** porque no se ve. Dos áreas usan la misma palabra, asumen que hablan de lo mismo, y el desacuerdo solo aparece cuando el sistema produce un resultado que a una de las dos le parece absurdo. La sinonimia genera duplicación —visible y corregible—; la homonimia genera **acuerdos falsos**.

> [!tip] La pregunta que detecta homónimos: *"¿me das un ejemplo concreto de esto?"* Si dos personas dan ejemplos que no se solapan, están usando el mismo término para conceptos distintos, y acabás de encontrar un problema que habría explotado tres meses después.

### Tabla 3.1 — Sinonimia vs homonimia

| Fenómeno | Qué pasa | Cómo se detecta | Cómo se resuelve | Riesgo si no se trata |
|---|---|---|---|---|
| **Sinonimia** | Varios términos, un concepto | Dos términos con la misma definición y los mismos ejemplos | Un término preferido + los demás como alternativos (`skos:altLabel`) | Conceptos duplicados; falla la integración |
| **Homonimia** | Un término, varios conceptos | Ejemplos concretos que no se solapan entre áreas | Conceptos separados, términos calificados por contexto | **Acuerdo falso**: nadie nota el desacuerdo |

> [!note] La tabla explica por qué el glosario no es documentación de cortesía: es el instrumento que **fuerza a explicitar** ambas ambigüedades mientras todavía son baratas de resolver.

## Construir el glosario acordado

El producto tangible de esta etapa es un **glosario**: la lista de conceptos del dominio, cada uno con su término preferido, sus términos alternativos y una definición en lenguaje natural validada por los expertos.

Lo que hace bueno a un glosario:

- **Una definición por concepto, no por término** — la definición describe el concepto; los términos son etiquetas que apuntan a él.
- **Definiciones que discriminan** — una buena definición permite decidir si un caso concreto cae adentro o afuera. *"Un cliente es alguien que nos compra"* no discrimina (¿alguien que compró una vez hace diez años?, ¿alguien que tiene contrato pero nunca compró?).
- **Ejemplos y contraejemplos** — especialmente contraejemplos. Los casos que **no** son X hacen más por fijar un concepto que tres párrafos de definición abstracta.
- **Término preferido explícito** — la decisión de qué etiqueta es la canónica, tomada y registrada, no dejada al azar del primero que modele.
- **Trazabilidad a la fuente** — de dónde salió el término y quién validó la definición.

> [!tip] Escribí las definiciones de modo que un recién llegado al dominio pueda **clasificar casos concretos** con ellas. Ese es el test de calidad: si dos personas leen la definición y clasifican distinto el mismo caso, la definición todavía no sirve.

> [!warning] El glosario es donde aparecen los **desacuerdos reales entre áreas**, y eso es una virtud, no un problema. Si dos áreas no logran acordar la definición de *cliente*, ese conflicto ya existía — solo que estaba oculto y produciendo inconsistencias en los datos. Descubrirlo acá es barato; descubrirlo después de modelar es caro; descubrirlo en producción es carísimo.

## Análisis del dominio — de vocabulario a estructura

Con el vocabulario normalizado, el capítulo pasa al **análisis**: mirar los conceptos y decidir qué papel juega cada uno en el modelo. Es el trabajo que prepara las decisiones de modelado formales del capítulo siguiente.

Las preguntas que guían el análisis:

- **¿Esto es una clase o una instancia?** *Perro* es una clase; *Firulais* es una instancia. Pero *Golden Retriever* puede ser cualquiera de las dos según el caso de uso —una clase de perros, o una instancia de la clase *Raza*—. La respuesta la dan las [[competency questions]], no la intuición.
- **¿Esto es una clase o un valor de propiedad?** *Color* raramente es una clase; *Rojo* suele ser un valor. Pero en un dominio donde los colores tienen propiedades propias (código, gama, proveedor), pasan a ser entidades.
- **¿Esto es una propiedad o una relación?** Distinguir atributos con valores literales (una fecha, un número) de relaciones entre entidades del dominio.
- **¿Qué relaciones estructuran la jerarquía?** La subsunción (*es un tipo de*) es la columna vertebral, pero conviene no confundirla con otras relaciones que parecen jerárquicas y no lo son.

> [!warning] **La confusión más frecuente y más costosa: *es-un-tipo-de* vs *es-parte-de*.** Una rueda no es un tipo de auto — es parte de un auto. Modelar la meronimia (parte-todo) como subsunción rompe la herencia: el razonador va a inferir que toda rueda es un auto, con todas las propiedades del auto. Es un error que parece inocente en el diagrama y produce inferencias absurdas apenas corre el razonador.

> [!tip] El test de subsunción: *"¿todo X es también un Y?"* Si la respuesta es sí sin excepciones, es subsunción. *Todo Golden Retriever es un perro* → sí, subsunción. *Toda rueda es un auto* → no, es meronimia, y necesita una relación propia.

> [!note] **Estas decisiones no son intrínsecas al dominio: dependen del caso de uso.** No existe la respuesta "correcta" sobre si *Golden Retriever* es clase o instancia en abstracto. Existe la respuesta correcta **para las preguntas que la ontología tiene que responder**. Este es el mismo criterio que gobierna todo el libro — el caso de uso manda, no la ambición de completitud.

## Para aplicar

- **Recolectá el vocabulario de fuentes escritas primero** — glosarios, esquemas, formularios, normativa — y reservá al experto para resolver conflictos, no para enumerar de cero.
- **Separá término de concepto desde el inicio**: una definición por concepto, y los términos como etiquetas (`skos:prefLabel` / `skos:altLabel`).
- **Cazá homónimos pidiendo ejemplos concretos** a cada área por separado. Si los ejemplos no se solapan, son conceptos distintos con el mismo nombre.
- **Escribí contraejemplos en cada definición** — los casos que NO son X fijan el concepto mejor que la definición positiva.
- **Aplicá el test de subsunción** (*"¿todo X es también un Y?"*) antes de colgar cualquier concepto de una jerarquía.
- **Registrá los desacuerdos de definición que aparezcan** — son hallazgos valiosos sobre la organización, no ruido a resolver rápido.

## Conexiones

- [[_Ontology Engineering|Ontology Engineering]] — el MOC del libro.
- [[02 - Ontology Development Methodology]] — capítulo anterior: el proceso dentro del cual esta etapa vive · [[04 - Modeling Decisions]] — capítulo siguiente: convertir este vocabulario analizado en axiomas formales.
- [[SKOS]] — el estándar hecho a medida para esta capa: conceptos con etiquetas preferidas y alternativas, sin exigir formalización lógica.
- [[competency questions]] — el criterio que decide clase vs instancia y toda otra duda de análisis.
- [[espectro semántico]] — un glosario bien hecho ya es un escalón del espectro y, para muchos casos de uso, el escalón suficiente.
- [[Ubiquitous Language]] — el paralelo directo en Domain-Driven Design: el mismo problema de vocabulario compartido, atacado desde el diseño de software.
