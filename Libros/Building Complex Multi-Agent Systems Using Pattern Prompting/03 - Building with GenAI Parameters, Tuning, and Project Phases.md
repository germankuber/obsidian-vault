---
title: "Building with GenAI: Parameters, Tuning, and Project Phases"
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 3
created: 2026-06-20
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/case-study
  - status/permanent
aliases:
  - Building with GenAI Parameters, Tuning, and Project Phases
  - Cap 3 - Building with GenAI
updated: 2026-07-05
---
# Building with GenAI: Parameters, Tuning, and Project Phases

> [!info] Capítulo 3 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> Construir un proyecto [[GenAI]] es distinto a cualquier proyecto no-AI: el trabajo central no es escribir código sino **resolver dependencias circulares mientras se configuran parámetros**. El capítulo enseña a pensar el tuning de los **15 parámetros principales** de una app GenAI (chunking, vector DB, [[LLM]]), a entender sus dependencias con un **dependency graph** (Fig 3.1) y un **pipeline breakdown** (ingestion / retrieval / generation, Fig 3.2), y lo aplica a un ejemplo concreto: una app de **Q&A sobre Harry Potter** con microarquitectura **[[RAG]]**. Profundiza el [[Chunking|chunking]] (5 estrategias + metáfora del small net vs trawler), la **[[LLM Temperature|temperature]]** del LLM y los **6 roles agénticos** ([[Judge]], [[Router]], [[Actor]], [[Guard]], [[Critic]], [[Editor]]) con su temperatura preferida, y cierra con las **3 fases** de un proyecto GenAI (project initiation, intermediate goals, crossing the finish line) y el **[[ETL]] pipeline**. Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[02 - Embeddings The Language of AI]] · siguiente [[04 - Building Your First RAG App]].

## Resumen

Si empezás un proyecto GenAI por primera vez, va a sentirse distinto a otros proyectos no-AI: la razón principal es que los proyectos GenAI requieren **resolver dependencias circulares mientras configurás parámetros**, y ponen un foco mucho más pesado en **optimizar la configuración** que en escribir código. Construir una app GenAI exitosa es encontrar los **valores óptimos de varios parámetros**, y el alto nivel de dependencia entre ellos significa que cambiar **un solo parámetro** puede romper el sistema y forzar cambios en otros, que a su vez crean nuevas incompatibilidades, y así sucesivamente. Es el mismo dolor que los desarrolladores conocen de las **dependencias circulares de librerías**; por suerte, igual que los lenguajes tienen package managers que las resuelven, este capítulo da una estrategia para resolverlas en GenAI. La advertencia cuantitativa es contundente: con **15 variables interdependientes en juego, el número posible de combinaciones llega a los miles de millones (billions)** — sin una buena estrategia, terminar la app puede llevar muchísimo tiempo.

El capítulo arranca con **principios y best practices** de tuning (4 reglas de sentido común: control del entorno, log versionado de tests, paralelizar, documentar los goals), enumera los **15 parámetros principales** que hay que tunear y los organiza con un **dependency diagram dirigido** (Fig 3.1): una flecha A → B significa que cambiar A obliga a re-tunear B. Esas relaciones se agrupan en 5 familias (Document → Chunking, Chunking → Retrieval, Retrieval ↔ Prompting, Model → Context/Tokens → Prompts, y Temperature → prompt behavior). El **pipeline breakdown** (Fig 3.2) muestra tres niveles de especialización —**ingestion, retrieval, generation**— cada uno con skills distintas, útil para dividir tareas en el equipo sin trabajar con propósitos cruzados.

Lo teórico baja a tierra con un ejemplo: una app que responde cualquier pregunta sobre los **7 libros de Harry Potter** en PDF. Se define la **Tabla 3.1** (4 queries con su ground truth), los goals/constraints (respuestas **< 5 oraciones**; misma respuesta al menos el **97%** de las veces) y la microarquitectura **[[RAG]]** simple (Fig 3.3). Sobre ese ejemplo se recorre el **chunking**: 5 estrategias evaluadas una por una con sus trade-offs, eligiendo la **estrategia 5** (chunks < 100 tokens con intelligent line breaking, chunk size subido a **600 tokens**), límite inicial de **5 chunks** ordenados por relevancia, y la metáfora del **small hand net vs trawler** (Fig 3.4). Sigue la **[[LLM Temperature|temperature]]** (predictibilidad de la respuesta) con el ejemplo del wand y los **6 roles agénticos** (Fig 3.6) y su temperatura preferida. Cierra con las **3 fases** del proyecto GenAI (Fig 3.7 el ETL pipeline, Fig 3.8 crossing the finish line). El cap. 4 construirá una app real con microarquitectura RAG poniendo en práctica todos estos parámetros.

## Tuning GenAI systems: principles and practices

Este capítulo da el puntapié al **tuning de parámetros**, una skill central para construir apps GenAI. Los parámetros a tunear viven en **múltiples componentes** de la app: [[LLM|LLMs]], document chunkers y [[Vector Database|vector databases]]. La lista de parámetros y estrategias del capítulo **no es exhaustiva**, y la enseñanza clave es metodológica.

> [!note] Lo importante no es memorizar nombres de parámetros ni estrategias específicas, sino **aprender a pensar el tuning** y qué procesos seguir. Traducir objetivos de alto nivel en outputs confiables requiere un enfoque granular para moldear el entorno del modelo, empezando por los parámetros core.

Antes de entrar a las estrategias, el libro repasa 4 best practices simples pero de sentido común:

> [!tip] Las 4 best practices iniciales de tuning:
> - **Maintain tight control over your environment** — cuando optimizás un parámetro, asegurate de que nadie más esté haciendo modificaciones simultáneas al sistema.
> - **Keep a log of all your tests, preferably under version control** — vas a querer volver atrás a ver qué se probó y qué resultados dio.
> - **Parallelize as much as possible** — no hay requisito de testear parámetros secuencialmente; si podés levantar múltiples entornos, mejorás muchísimo la velocidad a la que testeás combinaciones.
> - **Document the goals of your system** — para decidir después entre las opciones que ofrecen distintos settings; identificar goals puede incluir construir **customer profiles**, correr **focus groups**, etc.

## Tuning the parameters of your GenAI project

Los parámetros tuneables están en vector databases, document chunkers y LLMs. Entre otras cosas permiten controlar el **tamaño de los documentos** devueltos del store local, ajustar el **grado de creatividad** permitido en las respuestas del LLM, y determinar **cómo se agrupan y buscan** los documentos. El objetivo último es que el LLM genere la **mejor respuesta posible** al prompt. Saber tunear los parámetros de abajo alcanza para la mayoría de los proyectos.

### Tabla 3.A — Los 15 parámetros principales a tunear

| # | Parámetro | Dónde vive / qué controla |
|---|---|---|
| 1 | **Chunking strategy** | Cómo se divide un documento grande en chunks |
| 2 | **Document structures** | Tablas, headings, pages, sections del documento |
| 3 | **Document types** | Formatos (contracts, invoices, long-form PDFs, etc.) |
| 4 | **Chunk overlap** | Solape entre chunks adyacentes |
| 5 | **Chunk size** | Tamaño de cada chunk |
| 6 | **Similarity function** | cosine / dot-product / hybrid scoring |
| 7 | **Max. chunks returned** | Cuántos chunks devuelve la retrieval (top-k) |
| 8 | **Vector database repository structure** | single index vs multiple indexes, per-document-type collections, metadata filters, hybrid indexes |
| 9 | **LLM temperature** | Cuán predecible/creativa es la respuesta |
| 10 | **LLM** | El modelo elegido |
| 11 | **LLM version** | La versión del modelo |
| 12 | **Prompts** | Las plantillas de prompt |
| 13 | **Max tokens** | Límite de tokens de salida |
| 14 | **Context size of LLM** | Ventana de contexto del modelo |
| 15 | **Chunk limit** | Tope de chunks por documento / etapa del pipeline |

![[B34134_3_1.png|612]]
*Figure 3.1 – Dependency chart: an arrow A → B means changing A will likely require changing B*

> [!note] Es un grafo dirigido: una flecha de un parámetro a otro (A → B) significa que si cambiás A, probablemente necesites cambiar B para mantener la retrieval y la generación consistentes en calidad, costo y estabilidad.

Las relaciones del grafo se entienden examinando cómo el cambio en uno influye en los otros a través del sistema. El libro las agrupa en **5 familias**:

**1) Document → Chunking**
- **Document types and structures → chunking strategy**: distintos formatos (contracts, invoices, long-form PDFs) y estructuras (tables, headings, pages, sections) fuerzan distintos chunk boundaries.
- **Chunking strategy → chunk size, overlap, and chunk limit**: una vez que elegís las reglas (by headings, by pages, by semantic splits) hay que tunear size y overlap, y fijar safety limits.

**2) Chunking → Retrieval**
- **Chunk size and overlap → chunk limit**: chunks más grandes o más overlap aumentan el conteo total de chunks para el mismo documento.
- **Chunk limit → max. chunks returned**: si capás cuántos chunks existen por documento (o por etapa del pipeline), eso restringe qué puede devolver la retrieval.
- **Vector database structure → similarity function**: la estructura del repositorio (single index vs multiple indexes, per-document-type collections, metadata filters, hybrid indexes) afecta qué método de similitud es apropiado y cómo se computa o combina.
- **Similarity function → max. chunks returned**: cambiar la similitud (cosine vs dot-product vs hybrid scoring) cambia las distribuciones de score, así que el top-k óptimo (chunk limit) suele cambiar también.

**3) Retrieval ↔ Prompting** (bidireccional)
- **Max. chunks returned → prompts**: los prompts deben estructurarse según cuántos chunks alimentás (format, citations, compression, ordering).
- **Prompts → max. chunks returned**: si los prompts requieren más evidencia o citas, solés subir el top-k; si requieren grounding más estricto, bajás el top-k y subís el similarity threshold.

**4) Model → Context/Tokens → Prompts**
- **LLM → version → context size**: distintas versiones implican típicamente distintas restricciones de context window.
- **Context size → max tokens and prompts**: las plantillas de prompt y los límites de salida deben caber dentro de la context window del modelo.
- **Max tokens → prompts**: si bajás el máximo de output tokens, los prompts suelen necesitar constraints más fuertes, más instrucciones de summarization o un answer format distinto.

**5) Temperature ties to prompt behavior**
- **LLM temperature → prompts**: la rigurosidad del prompt suele necesitar aumentar cuando sube la temperature (para preservar determinismo y fidelidad); los prompts pueden relajarse cuando la temperature es baja.
- **LLM version → temperature**: distintas versiones pueden comportarse distinto a la misma temperature, así que a menudo hay que re-tunear.

> [!tip] Un conocimiento profundo de los grafos de Fig 3.1 y Fig 3.2 puede **reducir varios órdenes de magnitud** el tiempo que pasás construyendo y tuneando apps GenAI, y ayuda a **dividir tareas en el equipo** para que no trabajen con propósitos cruzados.

![[B34134_3_2.png|1149]]
*Figure 3.2 – Pipeline breakdown: ingestion, retrieval, and generation layers with their cross-layer coupling points*

> [!note] El pipeline tiene **tres niveles de especialización**, cada uno con skills y expertise distintas: **ingestion** (cargar y chunkear documentos), **retrieval** (búsqueda semántica y selección de chunks) y **generation** (el LLM produciendo la respuesta). El diagrama también marca los **cross-layer coupling points** que conectan las capas.

> [!tip] Examinando la Fig 3.2 se ven los **tres roles de equipo naturales**: un especialista de ingestion, uno de retrieval y uno de generation. Conocer el pipeline así evita que los miembros del equipo trabajen at cross-purposes.

## Configuring a GenAI application to answer any question about the Harry Potter books

Tunear GenAI es **parte ciencia, parte arte**. Tus primeros valores para los parámetros van a estar lejos de los óptimos, así que el tuning es un proceso **iterativo de modificar y testear**. Asumamos un proyecto cuyo objetivo es una app GenAI que responda preguntas de los libros de Harry Potter, y que arrancamos con **los 7 libros en PDF**.

Para los no familiarizados con la serie: son novelas de fantasía de **J.K. Rowling**, que siguen el viaje de un joven mago en **Hogwarts** mientras combate al dark lord **Voldemort**. Abarcan 7 libros, exploran temas de amistad, valentía y la lucha entre el bien y el mal, y han inspirado **8 películas**.

### Tabla 3.1 — Sample queries and ground truth answers for the Harry Potter Q&A application

| Question | Ground Truth Answer |
|---|---|
| What magic is used in Harry Potter? | Witchcraft and wizardry, performed through spells, charms, curses, potions, and magical objects. |
| Why does Harry Potter have a lightning-shaped scar? | He got it as a baby when Lord Voldemort tried to kill him with the Killing Curse, which rebounded. |
| What is a wand used for in Harry Potter? | A wand channels a wizard's or witch's magic to cast spells. |
| What kind of creature is Hagrid's pet, and what is its name? | A giant three-headed dog named Fluffy. |

Para cada pregunta se establece una **"ground truth"** contra la que comparar el output del LLM. En un proyecto con más datos se crearían **al menos 120 queries**, incluyendo **multi-hop questions** (que requieren más de una oración) y preguntas amplias como "What magic is used?" que tienen muchas respuestas válidas. Cuatro preguntas alcanzan para este ejemplo.

> [!note] **Goals y constraints del ejemplo:**
> - Las respuestas a las queries deben ser **cortas y al punto — menos de 5 oraciones**.
> - Una pregunta dada debe hacer que el LLM devuelva el **mismo resultado al menos el 97% de las veces**.

La microarquitectura es la **[[RAG]]** (retrieval-augmented generation) simple de la Fig 3.3. Entender el framework en detalle no es importante acá (el cap. 4 lo trata en profundidad); por ahora solo interesa **qué chunks de texto** se devuelven de la vector database.

![[B34134_3_3.png|716]]
*Figure 3.3 – RAG architecture model: the user's question triggers a semantic search, following which relevant chunks are retrieved and added to the prompt, and the LLM generates a response*

> [!note] La pregunta del usuario dispara una **semantic search**, tras la cual se recuperan los **chunks relevantes** y se agregan al prompt, y el LLM genera una respuesta.

Normalmente el paso siguiente sería identificar el LLM y su versión, pero se **saltea** porque la sección es teórica y no se usará un LLM real en este capítulo; las discusiones no dependen de modelos específicos. Con la arquitectura, los goals/constraints, el conocimiento del dataset (los libros) y un set diverso de sample questions ya definidos, se arranca el tuning. Nota importante: hay elecciones malas (incluso que rompen el sistema entero), pero **rara vez hay una única elección correcta**.

### Chunking strategies

La **[[Chunking|chunking strategy]]** es la primera elección de la rama derecha del dependency tree (Fig 3.1) y **no está fija antes del development time**. Ya conocemos los nodos por encima de chunking strategy: **document type (PDF)** y **document structure (chapters)**. La chunking strategy determina cómo tomar un documento grande y dividirlo en chunks más chicos; acá se chunkean PDFs (hay otros tipos —DOCX, XLSX, JSON, raw text, imágenes— pero el proceso de pensamiento se adapta fácil).

Las **5 estrategias** consideradas para los libros de HP:

1. **Treat the entire PDF book as one chunk** — la query "What kind of creature is Hagrid's pet…?" encuentra texto relevante en **dos lugares** de *Harry Potter and the Sorcerer's Stone*: **Cap. 9 (The Midnight Duel) → Fluffy, el perro de tres cabezas**, y **Cap. 14 (Norbert the Norwegian Ridgeback) → el dragón bebé de Hagrid**. Pero el libro tiene **más de 250 páginas**; cargarlo entero puede fallar por **memoria** o por **exceder el token limit** del LLM o embedding model. Aun sin error de memoria, hay **latencia** por procesar todo el libro. Si además trackeamos el **historial de conversación**, cada nuevo request se vuelve enorme. Y como el costo de la API se mide en tokens, la estrategia sería **muy cara**. Sirve para documentos chicos como **JSON responses**, no para los libros. Descartada.
2. **Chunk by chapter** — la query "What magic is used by Harry Potter?" hace match semántico con **muchos capítulos a lo largo de los 7 libros**, requiriendo mucha memoria y arrastrando todos los problemas de la estrategia 1. Drawback extra: un capítulo cubre **muchos temas**, algunos no relevantes, y quizá **solo una oración** trata sobre magia; como cada capítulo es grande hay que **limitar los chunks devueltos**, con lo que puede haber **data relevante en chunks que no devolvemos** por falta de lugar. Sirve para colecciones de documentos chicos no relacionados (un **product inventory**), no para los libros. Descartada.
3. **Chunk by sub-chapter** — parece prometedora (bajaría chunk size y la dependencia entre chunks), pero los libros de HP **no tienen sub-capítulos**, así que esta opción **no está disponible**.
4. **Chunks de menos de 100 tokens** — 100 es un número arbitrario para el ejemplo. Da **mayor precisión** en los datos que procesa el LLM y reduce el tamaño de los chunks devueltos, resolviendo lo de las estrategias 1 y 2. Pero si la query es muy específica (la mascota de Hagrid), los matches semánticos pueden limitarse a **un solo lugar**; ahí querríamos **el máximo contexto posible** (unas oraciones antes y después), y limitar a 100 tokens **saltea chunks vecinos útiles**. Otro problema: un chunk puede **cortar a mitad de oración**, p.ej. terminar en *"Hagrid's pet is a kind of …"* y continuar en el chunk siguiente.
5. **Chunks de menos de 100 tokens con intelligent line breaking, con chunk size aumentado a 600 tokens** — el approach inteligente **no corta a mitad de oración** y, al subir el chunk size a **600 tokens**, recupera **más oraciones antes y después** del texto que matchea. **Es la estrategia elegida.**

Además se **limita el número de chunks devueltos** para no quedar sepultados ante una query que matchea en muchos lugares: se elige un límite inicial de **5 chunks**, sabiendo que se puede revisar luego si falta data en chunks más allá del quinto. **Los chunks se ordenan por relevancia.**

> [!warning] El chunking es un equilibrio: chunks demasiado chicos pierden contexto y cortan oraciones; chunks demasiado grandes traen ruido, encarecen y arriesgan exceder memoria/tokens. La estrategia 5 (intelligent line breaking + 600 tokens + top-5 por relevancia) es el punto de balance para esta colección.

![[B34134_3_4.png|469]]
*Figure 3.4 – Chunking by size: a small net (small chunks) offers precision and flexibility; a trawler (large chunks) offers breadth at the cost of sorting effort*

> [!note] Metáfora de pesca: con una **red chica de mano** es más probable pescar exactamente el pez que querés, con menos esfuerzo descartando los que no, y podés tirar la red en **muchos lugares**. Con una **red grande (trawler)** capturás muchos peces de golpe, incluyendo los que no querés, requiriendo mucho más procesamiento para clasificar; además el trawler **no puede tirar en muchos lugares**, así que su captura refleja solo el **único lugar** donde decidió pescar.

Con chunk size, top-k (chunk limit), chunking strategy y **chunk overlap** (visto en [[02 - Embeddings The Language of AI]]) fijados, se pasa al próximo parámetro: temperature.

### Temperature

La **[[LLM Temperature|temperature]]** mide cuán predecible es la respuesta del LLM respecto de lo que el usuario espera. Pero no siempre queremos respuestas casi idénticas: para generar varias opciones de línea de apertura de un email importante, valoramos la **variedad**. Para cubrir ambos casos, los LLMs permiten setear la temperature, un número que determina cuán únicas son las respuestas al llamar al LLM con el **mismo prompt** varias veces. **Temperature alta → respuestas más variadas; temperature baja → respuestas más consistentes.**

Ejemplo con el prompt "What is a wand used for in Harry Potter?", donde el LLM podría hallar **dos respuestas viables** según los datos de la vector DB:
- **Casting spells**
- **Focusing magical energy**

> [!note] Con **temperature muy baja, cerca de cero**: la misma respuesta casi siempre — de 100 llamadas, quizá **99 devuelven "Casting spells"**. Con **temperature cerca o por encima de uno**: respuestas más variadas con frecuencia similar — "Focusing magical energy" aparecería **casi tan seguido** como "Casting spells".

Cuándo conviene cada una: la pregunta "What kind of creature is Hagrid's pet, and what is its name?" **requiere la misma respuesta cada vez** (temperature baja). En cambio "What magic is used by Harry Potter?" admite **muchas respuestas buenas** y no necesitás respuestas idénticas (la temperature no necesita ser baja). La elección es sutil: hay que pensar el use case y revisar qué otros parámetros soporta el LLM.

![[B34134_3_5.png]]
*Figure 3.5 – Temperature: low temperature (clownfish) produces predictable, similar responses; high temperature (diverse sea life) produces varied, creative responses*

> [!note] Metáfora marina: una **temperature baja** es como un cardumen de clownfish (todos parecidos → predecible); una **temperature alta** es como la **diversidad de vida marina** (variada, creativa).

#### Los 6 roles agénticos y su temperature preferida

Las microarquitecturas GenAI tienen uno o más componentes **altamente especializados**. La flexibilidad que se le da al LLM para devolver algo distinto a la respuesta más probable suele depender del **rol** del componente:

![[B34134_3_6.png]]
*Figure 3.6 – Agentic roles: each role has a characteristic temperature preference*

> [!note] El contenido de la figura se reproduce íntegro en la tabla de roles de arriba.

### Tabla 3.B — Roles agénticos y su temperature preference

| Rol | Qué hace | Temperature |
|---|---|---|
| **[[Judge]]** | Evalúa respuestas de **múltiples data sources** y determina cuál es mejor o qué afirmación es verdadera. La creatividad rara vez sirve para juzgar. | **Baja** |
| **[[Router]]** | Dirige un request a un **componente LLM especializado**, usando el texto recuperado de la vector DB para decidir el ruteo. Una temperature alta (y la chunking strategy) puede producir respuestas distintas al mismo prompt y rutear a componentes distintos con resultados muy diferentes. | **Baja** |
| **[[Actor]]** | Ejecuta una operación e **interactúa con el mundo externo** (p.ej. llamar una API o consultar la vector DB). A menudo convierte **raw data a formato legible** por humanos. Generalmente determinístico. | **Baja** |
| **[[Guard]]** | Garantiza la seguridad de los usuarios **detectando y rechazando alucinaciones, lenguaje ofensivo y contenido inapropiado**. No se supone que sea creativo. | **Baja** |
| **[[Critic]]** | Evalúa la **calidad** de una respuesta contra criterios definidos como **accuracy, relevance o completeness**. Requiere juicio cuidadoso y consistente. | **Baja** |
| **[[Editor]]** | **Acorta, alarga o agrega estilo** a una respuesta de texto. Mayor creatividad puede servir, según los requisitos. | **Alta** (más diverso/creativo) |

> [!tip] Patrón general: **casi todos los roles prefieren temperature baja** (Judge, Router, Actor, Guard, Critic), porque su trabajo es preciso, determinístico o de seguridad. El **Editor** es la excepción: su tarea estilística se beneficia de la **variedad** de una temperature alta.

## Best practices for building a GenAI project

El objetivo de un buen proceso es **minimizar dependencias dentro del equipo** y aumentar la productividad de sus miembros. Un proyecto GenAI se parte en **tres fases distintas**: lograr entender el producto y recolectar datos de testing; tunear exitosamente los parámetros; y dar los toques finales antes del release.

### Project initiation

En la fase de iniciación se junta la **información externa** que servirá de input para decidir los settings de los parámetros:
- A partir de un **entendimiento profundo de los usuarios** del producto, crear una **lista de test prompts**.
- **Identificar las microarquitecturas**.
- **Identificar los documentos** de la colección. En organizaciones grandes están en varias ubicaciones: **email servers, document management systems, local hard drives**.
- **Identificar los tipos de documentos** (spreadsheets, PDFs, etc.) que se ingestarán en la vector database.
- **Identificar una chunking strategy** examinando una **muestra random** de documentos. A veces los documentos son **miles** y requieren una strategy más compleja que agrupe documentos en collections por **type, source, size, content y topic**.
- **Identificar y documentar UX factors**: response times aceptables, requisitos de complejidad y **SLAs (service-level agreements)**.
- **Desarrollar un sistema de clasificación de documentos** por **type, source location, topic, age**, etc. Usar **modelos AI de clasificación automática** si hace falta.
- **Crear un repositorio de documentos de muestra** representativos, con **metadata** (descripción, source, creation date, author).
- **Identificar las temperature settings correctas** para los sample prompts.

### Intermediate goals

Mapeados los procesos, se **mueven los documentos** desde donde están (típicamente una database, document management system o file server) a una vector database:
- Construir un **[[ETL]] (Extract, Transform, Load) pipeline** que carga los documentos del source, los chunkea e ingesta los chunks en la vector database. Ingestar documentos en la vector DB es un paso crucial; se construye un **ETL avanzado en los Caps. 5 y 6**.
- Usar el pipeline para **ingestar los documentos** en la vector DB y **correr los sample prompts**.
- **Iterar**: ingestar documentos, invocar la vector DB con queries, evaluar los resultados, y **testear la chunking strategy después de cada iteración**.

![[B34134_3_7.png]]
*Figure 3.7 – Data pipeline: the ETL process extracts documents from multiple sources, transforms them through chunking and embedding, and loads them into a vector database*

> [!note] El proceso **ETL extrae** documentos de múltiples fuentes, los **transforma** mediante chunking y embedding, y los **carga (load)** en una vector database.

### Crossing the finish line

Antes de liberar la app para testing y deployment, analizar:
- Según el diseño de la microarquitectura, **verificar que los prompts devuelven las respuestas deseadas**. Usar un LLM para **generar test inputs** si hace falta.
- Si hay **múltiples componentes, testear cada uno independientemente**. Se puede usar un LLM para analizar respuestas si ayuda.
- Preguntarse: ¿la **clasificación de documentos** funciona como se espera? ¿Hay **suficiente independencia** en los grupos de documentos como para justificar que **cada grupo tenga su propia collection** en la vector database?

![[B34134_3_8.png]]
*Figure 3.8 – Crossing the finish line: the final phase of a GenAI project before release*

> [!note] La **fase final** de un proyecto GenAI antes del release.

## Citas

> "GenAI projects require solving circular dependencies while configuring parameters. GenAI projects also place a heavier focus on optimizing configuration parameters than on writing code."

> "With 15 interdependent variables at play, the possible number of combinations runs into the billions."

> "An arrow from one parameter to another, e.g., A → B means that if you change one parameter (A), you will likely need to change another (B) to keep retrieval and generation of quality, cost, and stability consistent."

> "Tuning GenAI is part science and part art. Your first guesses at the values for the parameters will likely be far from their optimal values."

> "You can think of the chunking strategy as choosing whether to fish with a small hand net or a trawler."

> "Temperature is a measure of how predictable the LLM's response is relative to what the user expects."

> "A guard ensures the safety of users by detecting and rejecting hallucinations, offensive language, and inappropriate content."

## Para aplicar

- **Tratar el tuning como el trabajo central, no el código** — presupuestá tiempo para iterar configuración; con ~15 parámetros interdependientes las combinaciones son enormes, así que la estrategia y el grafo de dependencias importan más que escribir código.
- **Aplicar las 4 best practices desde el día 1** — entorno controlado (nadie toca mientras testeás), **log de tests bajo version control**, **paralelizar** entornos, y **documentar los goals** (customer profiles, focus groups).
- **Dibujar el dependency graph antes de tunear** — usá las 5 familias (Document→Chunking, Chunking→Retrieval, Retrieval↔Prompting, Model→Context/Tokens→Prompts, Temperature→prompts) para saber qué re-tunear al tocar un parámetro y evitar loops infinitos.
- **Dividir el equipo por las 3 capas del pipeline** — ingestion / retrieval / generation, cada una con su especialista, para no trabajar at cross-purposes.
- **Definir ground-truth queries y constraints medibles** — al estilo Tabla 3.1 (apuntá a **≥120 queries** en un proyecto real, con multi-hop y broad questions), con constraints como "**< 5 oraciones**" y "**misma respuesta ≥97% de las veces**".
- **Elegir chunking por iteración, no por dogma** — descartá estrategias contra tus queries (entero/by-chapter fallan por memoria/costo/ruido); preferí **intelligent line breaking + chunk size ~600 + top-5 por relevancia** para evitar cortes a mitad de oración y conservar contexto vecino.
- **Setear temperature por rol del componente** — baja para Judge/Router/Actor/Guard/Critic (precisión, determinismo, seguridad); alta solo para Editor (variedad estilística).
- **Construir un ETL pipeline para la ingesta** — Extract (del source) → Transform (chunking + embedding) → Load (vector DB); iterá ingesta→query→evaluación testeando la chunking strategy en cada vuelta (ETL avanzado en Caps. 5 y 6).
- **Hacer un checklist de "crossing the finish line"** — verificar prompts contra ground truth (generá test inputs con un LLM si hace falta), testear cada componente por separado, y validar si la clasificación de documentos justifica **collections separadas** en la vector DB.

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro.
- [[02 - Embeddings The Language of AI]] — capítulo anterior: introdujo embeddings, vector databases y **chunk overlap**; este capítulo retoma el chunking en profundidad con el ejemplo de Harry Potter y formaliza el tuning de parámetros.
- [[04 - Building Your First RAG App]] — capítulo siguiente (placeholder): construye una **app real** con microarquitectura **[[RAG]]**, poniendo en práctica los parámetros de este capítulo.
- [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape]] — los componentes especializados (roles agénticos) son la concreción de la **IA agéntica como arquitectura de componentes** del cap. 1.
- [[GenAI]] · [[LLM]] — el dominio y el modelo cuyo comportamiento se tunea.
- **[[LLM Temperature]]** — predictibilidad vs creatividad de la respuesta (candidata a nota propia).
- [[RAG]] — microarquitectura de retrieval-augmented generation usada en el ejemplo (se profundiza en el cap. 4).
- **[[Chunking]]** — chunk size / overlap / strategy / limit; las 5 estrategias y la metáfora net vs trawler.
- **[[ETL]]** — Extract, Transform, Load pipeline para mover documentos a la vector DB (candidata a nota propia).
- Roles agénticos (candidatos a nota propia): **[[Judge]]** · **[[Router]]** · **[[Actor]]** · **[[Guard]]** · **[[Critic]]** · **[[Editor]]**.
- [[Vector Database]] · [[Cosine Similarity]] — la retrieval sobre la que operan los parámetros de chunking y similarity function.
