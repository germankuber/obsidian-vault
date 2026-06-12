---
title: 03 - Introduction to Haystack by deepset
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 3
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Introduction to Haystack by deepset
  - Cap 3 - Haystack
updated: 2026-06-12
---

# 03 - Introduction to Haystack by deepset

> [!info] Capítulo 3 · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> Primer deep dive en la construcción de la **tool layer** con [[Haystack 2.0]]. Presenta los building blocks fundamentales — **components**, **pipelines**, **SuperComponents**, **tools** y **agents** — sobre una arquitectura de **directed graph (DG) explícito** con typed sockets. La tesis del capítulo: el verdadero valor de Haystack es ser un *pipeline-first engine* que produce tools deterministas, auditables y deployables (vía [[Hayhooks]]/[[Model Context Protocol (MCP)|MCP]]) que un orquestador ágentico ([[LangGraph]]) puede invocar. Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · anterior [[02 - Diving Deep into Large Language Models]] · siguiente [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]].

## Resumen

El capítulo abre la **Parte 2** del libro (construir la *tool layer*) y empieza a dominar la "física fundamental" de [[Haystack 2.0]] antes de construir [[RAG]] avanzado (Cap 4) o microservicios (Caps 7-8). Tras un overview de **deepset** —fundada en 2018 por Milos Rusic, Malte Pietsch y Timo Möller alrededor del auge de [[BERT]], con orígenes en el framework **FARM** y modelos como `roberta-base-squad2`, fusionados en Haystack en 2021— explica que la industria pasó del fine-tuning especializado a los LLMs general-purpose, y que deepset re-arquitecturó completamente el framework: **Haystack 2.0**, una plataforma para la era de la IA generativa que orquesta LLMs vía API para RAG complejo y agentes autónomos.

El núcleo conceptual es el **paradigm shift** de Haystack 1.x a 2.x (Tabla 3.1): de YAML/implicit data-passing a un **directed graph (DG)/[[DAG]] explícito** construido en Python puro, donde el developer instancia un `Pipeline`, agrega `Component` con nombre y conecta **typed sockets** con `.connect()`. Esto habilita workflows no-lineales (conditional routing, parallel branches), hace el sistema transparente al debugging nativo de Python y permite visualizarlo con `.draw()`. La filosofía descansa en tres pilares: **Python-native, explicit, modular**. La fortaleza para el libro no es tanto el soporte ágentico nativo de Haystack sino su **fiabilidad**: un DG con contratos estrictos provee la plataforma estable para crear los tools deterministas sobre los que los agentes razonan.

Luego desarrolla la **jerarquía de abstracciones** que escala de la unidad atómica al agente: **Component → Pipeline → [[SuperComponent]] → Tool → Agent**. Un *component* es una clase Python con el decorador `@component` y un método `run()`, con typed input/output sockets; un *pipeline* orquesta components en un DG; un *SuperComponent* empaqueta un subgrafo reutilizable como un único component; un *tool* es un wrapper semántico (name + description) sobre un component o pipeline; un *agent* razona en loop (thought → action → observation) eligiendo tools. El capítulo subraya el trade-off de esta jerarquía: da **fiabilidad** (el action step se respalda en un engine type-safe) pero introduce **context opacity** (el Agent encapsula el reasoning loop como caja negra, dificultando la curación de memoria que sí ofrece [[LangGraph]]).

El recorrido cubre cuatro grandes bloques: (1) las **features y ventajas** (transparencia/debuggability, flexibilidad vía DGs, escalabilidad con [[Hayhooks]] como REST/MCP, arquitectura agent-supporting, extensibilidad vía `@component`, y ecosistema comprehensivo sin vendor lock-in); (2) los **casos de uso** (RAG con separación indexing/query pipeline; los tres retrievers —semantic/dense, lexical/sparse [[BM25]], hybrid con [[Reciprocal Rank Fusion (RRF)|RRF]] + reranking [[cross-encoder]]; RAG-as-a-tool y multi-tool agents con su trace [[ReAct]]); (3) un **deep dive en la arquitectura** (data classes, document stores, las 6 categorías de components); y (4) cómo **incorporar Haystack al workflow** (assess → diseño visual-first con `.draw()` → roadmap incremental de 5 pasos → troubleshooting). Cierra anticipando el Cap 4: ensamblar estos components en pipelines funcionales con código hands-on.

## Un overview de deepset

deepset fue **fundada en 2018** por Milos Rusic, Malte Pietsch y Timo Möller, alrededor del auge de los modelos [[BERT]], para construir herramientas NLP escalables. Inicialmente la empresa se enfocó en fine-tuning open source vía su framework **FARM** y en modelos populares de Hugging Face como `roberta-base-squad2`. En **2021**, las features core de FARM se fusionaron en su framework insignia, **Haystack**.

Al desplazarse la industria del fine-tuning especializado hacia LLMs general-purpose, deepset adaptó el framework re-arquitecturándolo por completo. El resultado es **Haystack 2.0**: una plataforma diseñada específicamente para la era de la IA generativa. Más allá de la simple extracción, orquesta LLMs API-based para construir sistemas RAG complejos y agentes autónomos — una evolución que transforma a Haystack de herramienta NLP especializada a plataforma versátil para el desarrollo de IA moderna.

## Haystack 2.0: el framework de orquestación LLM evolucionado

Haystack 2.0 representa un **paradigm shift fundamental**: de framework de búsqueda especializado a toolkit general-purpose, altamente modular, para ingeniería de IA moderna. Su filosofía descansa en tres pilares — **Python-native, explicit y modular** — y al alejarse de estructuras predefinidas constreñidas, se posiciona como un framework robusto y developer-friendly capaz de orquestar los workflows production-grade que demanda la generación actual de aplicaciones LLM.

El avance más significativo es la adopción de una arquitectura de **directed graph (DG) explícito**. Los developers ya no dependen de implicit data passing; en su lugar **instancian explícitamente un objeto `Pipeline`, agregan instancias de `Component` y definen estrictamente el flujo de datos conectando typed sockets**. Esta transparencia habilita workflows sofisticados y no-lineales — como conditional routing (ej. *if the query is in German, route to Model A*) y parallel processing branches (ej. correr keyword y vector search simultáneamente) — y hace al sistema completamente transparente para las herramientas de debugging estándar de Python.

> [!note] Aunque el framework incluye soporte nativo para agents y loops, su mayor fortaleza para los fines del libro es la **fiabilidad**. Al reconstruir el fundamento alrededor de un DG explícito con contratos estrictos, Haystack provee una plataforma estable para crear los **tools deterministas** sobre los que los agentes avanzados se apoyan. Es una alternativa atractiva para equipos de ingeniería que priorizan debuggability y estabilidad.

### Tabla 3.1 — Haystack 1.x vs. Haystack 2.0: a paradigm shift

| Feature | Haystack 1.x (legacy) | Haystack 2.x (current) |
|---|---|---|
| **Pipeline definition** | Archivos YAML o la clase `Pipeline()` con implicit node naming | La clase `Pipeline()` con métodos explícitos `.add_component()` y `.connect()` en Python puro |
| **Component definition** | Class-based, requiere heredar de una clase `BaseComponent` | Decorator-based, usando el simple decorador `@component` sobre una clase Python estándar |
| **Data flow** | Implicit passing de un diccionario entre components, difícil de rastrear | Input/output sockets explícitos y tipados, imponiendo un data contract claro |
| **Debugging** | Requería conocimiento framework-specific, podía ser opaco | Debugging Python-native; los pipelines se visualizan con `.draw()` para ver la estructura exacta del grafo y el data flow |
| **Core abstraction** | Una secuencia lineal de nodos, lógica compleja difícil | Un **directed acyclic graph (DAG)**, soporta nativamente branching, merging y looping para workflows no-lineales complejos |
| **Primary focus** | Extractive question-answering y semantic search | LLM orchestration general-purpose: advanced RAG, agents y cualquier workflow custom multi-step |

> [!tip] *Table 3.1* es la "Rosetta Stone" del capítulo: resume las decisiones de diseño deliberadas que hacen de Haystack 2.0 un framework más potente, transparente y developer-friendly para la era de la IA generativa.

## Los building blocks: de components a agents

Haystack 2.0 introduce una jerarquía clara de abstracciones que escala de la unidad atómica de trabajo hasta los agentes. La jerarquía estricta es:

> **Component → Pipeline → SuperComponent → Tool → Agent**

### Components — las unidades atómicas de trabajo

Un **component** es una clase Python self-contained diseñada para ejecutar una **sola tarea específica** dentro de un pipeline: limpiar un documento, embeddar una query en un vector, recuperar documentos de una base de datos, o llamar a un LLM para generación. Su creación es notablemente simple y pythónica, definida por dos features clave:

- **El decorador `@component`** — cualquier clase Python se convierte en un Haystack component agregando `@component` sobre su definición. El decorador la registra en el framework y señala que puede usarse en un pipeline.
- **El método `run()`** — todo component debe tener un método `run()`, el entry point de su lógica donde se ejecuta la tarea principal.

Lo que hace robustos a los components es el concepto de **typed input/output sockets**, definidos con el decorador `@component.output` sobre la firma del `run`. Los sockets actúan como los puntos de conexión explícitos que definen exactamente qué tipo de dato espera recibir y producir un component — por ejemplo, un retriever con un input socket `query` de tipo `str` y un output socket `documents` de tipo `List`.

> [!note] Este **contrato explícito** impone type safety dentro del pipeline, hace transparente el data flow y permite a los IDEs dar mejor autocompletado y error-checking. Es una mejora significativa sobre el implicit dictionary-passing de Haystack 1.x, que a menudo llevaba a runtime errors y sesiones de debugging difíciles.

El notebook de [[Custom Components (Haystack)|custom components]] define un *prefixer document* que anota cada documento con un prefijo string dado por el usuario (`your-first-custom-component.ipynb`).

### Pipelines — orquestando components en grafos

Mientras los components ejecutan tareas individuales, los **pipelines** los orquestan en un workflow coherente. Un pipeline de Haystack 2.0 es un **DG** donde los components son los nodos y las conexiones entre sus sockets son las aristas. Construir uno es un proceso explícito de tres pasos en Python:

1. **Instanciar el pipeline**:
   ```python
   pipe = Pipeline()
   ```
2. **Agregar instancias de component** con un nombre único:
   ```python
   pipe.add_component(
       name="retriever",
       instance=InMemoryEmbeddingRetriever(document_store)
   )
   pipe.add_component(name="generator", instance=OpenAIGenerator())
   ```
3. **Conectar los components** — el output socket de uno al input socket de otro:
   ```python
   pipe.connect("retriever.documents", "generator.documents")
   ```

Esta sintaxis `connect` explícita es la piedra angular de la transparencia de Haystack 2.0: no deja ambigüedad sobre cómo se mueven los datos por el sistema. El framework además provee una herramienta crucial de debugging y visualización: el método **`.draw()`**. Llamar `pipe.draw("my_pipeline.png")` genera una imagen que representa visualmente la estructura del grafo (un diagrama [[Mermaid]]), mostrando todos los components y sus conexiones. La Figura 3.1 muestra un RAG pipeline simple que aumenta un LLM grounding-lo con dummy documents (notebook `your-first-pipeline.ipynb`).

![[03-fig-3.1-rag-pipeline.png]]
*Figure 3.1 – Mermaid diagram of a prompt*

### SuperComponents — encapsular y reusar complejidad

A medida que los pipelines se vuelven más complejos, ciertos patrones de components conectados aparecen repetidamente. Por ejemplo, un workflow estándar de document indexing podría consistir siempre en un file converter, un document cleaner, un document splitter, un embedder y un document writer, conectados en secuencia. Reconstruir ese subgrafo en cada pipeline nuevo sería tedioso y propenso a errores.

Para resolverlo, Haystack 2.0 introduce el concepto de [[SuperComponent]]: un **pipeline pre-empaquetado y reutilizable que se trata como un único component** dentro de un pipeline mayor. Encapsula un subgrafo complejo exponiendo solo los sockets de input/output necesarios al exterior. Todo el workflow de indexing anterior podría envolverse en un único `IndexingSupercomponent` que exponga un input socket simple (ej. `file_paths`) y un output socket (ej. `documents_written_count`), agregable con una sola línea:

```python
pipe.add_component(
    name="indexer",
    instance=IndexingSupercomponent())
```

> [!tip] Los SuperComponents simplifican dramáticamente la definición del pipeline principal y promueven un diseño limpio y modular — son una herramienta potente para gestionar complejidad y crear librerías de funcionalidades high-level reutilizables.

En el notebook se ve un patrón más simple: definen un **preprocessor pipeline** que limpia y splitea documentos en chunks (Figura 3.2), lo abstraen como un **preprocessor SuperComponent** y lo conectan al **prefixer custom component** visto antes. El pipeline final contiene solo dos components — el preprocessor SuperComponent y el prefixer component (Figura 3.3) — (notebook `supercomponents.ipynb`).

![[03-fig-3.2-preprocessor-pipeline.png]]
*Figure 3.2 – Mermaid graph of the preprocessor pipeline*

![[03-fig-3.3-supercomponent-prefixer.png]]
*Figure 3.3 – Mermaid graph of the preprocessor SuperComponent and the prefixer custom component*

### Agents y tools — habilitar el razonamiento LLM-driven

Un **tool** es esencialmente un *wrapper semántico* alrededor de un component de Haystack o de un pipeline entero abstraído como SuperComponent. Equipa la lógica determinista subyacente con:

- **Name** — un identificador corto y único (ej. `internal_knowledge_search`).
- **Description** — una explicación en lenguaje natural de las capacidades, inputs y outputs del tool (ej. *Searches the internal knowledge base. Input is a string query*).

Cuando un **agent** corre, entra en un **reasoning loop**: examina las descriptions de los tools disponibles, formula un plan, selecciona el tool apropiado, genera el input y lo ejecuta. El output (la *observation*) se realimenta al loop. El capítulo analiza las dos caras de esta jerarquía:

> [!note] **El beneficio (reliability)** — esta jerarquía asegura que el action step del agente se respalda en un engine robusto y type-safe. Como el tool es un wrapper alrededor de un pipeline Haystack validado, el agente orquesta unidades de trabajo **deterministas y confiables**, no scripts frágiles.

> [!warning] **El drawback (context opacity)** — como se exploró en el [[02 - Diving Deep into Large Language Models|Cap 2]], el [[Context Engineering|context engineering]] requiere control preciso del estado y la memoria del agente para prevenir context rot. El agent component de Haystack **encapsula el reasoning loop internamente** (black box), ocultando el estado y dificultando implementar las estrategias avanzadas de memory curation o el state management complejo que sí provee un framework como [[LangGraph]].

## Key features y advantages

La evolución arquitectónica de Haystack 2.0 lo hace el engine ideal para la tool layer de un stack de IA moderno. Estos beneficios derivan directamente de sus principios de diseño core — explicit graphs, typed contracts y modularidad:

- **Transparencia y debuggability** — la ventaja más significativa para production engineering. Las conexiones explícitas entre typed sockets eliminan la ambigüedad del opaque data passing: se ve exactamente de dónde viene y a dónde va cada dato. Complementado por `.draw()`, que convierte el debugging de una adivinanza en una tarea de inspección estándar.
- **Flexibilidad vía DGs** — la abstracción DG desbloquea workflows de complejidad arbitraria. Aunque los RAG simples son lineales, el framework soporta nativamente patrones no-lineales como conditional routing (ej. *If PDF, use OCR; if Text, skip*) y parallel branches, manejando edge cases con gracia en vez de romperse.
- **Escalabilidad y deployment con [[Hayhooks]]** — deepset introdujo Hayhooks, un framework complementario que despliega pipelines Haystack como **REST endpoints** serializando y cargando los pipelines serializados. Esto convierte a Haystack en el *capability provider* perfecto: tools deterministas y sofisticados que corren como microservicios independientes.
- **Arquitectura agent-supporting** — Haystack 2.0 está arquitecturado para ser backend de sistemas ágenticos. Internamente, sus abstracciones de SuperComponent y tool permiten envolver cualquier component —o un RAG pipeline complejo entero— en una unidad callable única con descripción clara. Externamente, combinado con Hayhooks, esos pipelines se vuelven tools consumibles por agentes de otros frameworks vía el [[Model Context Protocol (MCP)]], el estándar open source para conectar aplicaciones de IA a sistemas externos.
- **Extensibilidad** — el decorador `@component` provee una interfaz mínima para envolver lógica custom (se profundiza en el Cap 5). Esta baja barrera de entrada fomenta el patrón de diseño **micro-component**, donde la lógica específica se encapsula en bloques reutilizables en vez de scripts monolíticos.
- **Ecosistema comprehensivo** — Haystack integra con el MLOps landscape amplio, previniendo vendor lock-in: soporta todos los model providers mayores (OpenAI, Hugging Face, Cohere, Azure, Google, Amazon), vector DBs production-grade (Pinecone, Weaviate, Qdrant) y frameworks de evaluación ([[Ragas]], [[DeepEval]]).

## Use cases y aplicaciones

### Building RAG pipelines

[[RAG]] es la piedra angular de las aplicaciones LLM modernas: permite que los modelos respondan basándose en una knowledge base privada, reduciendo hallucinations y dando info up-to-date y grounded. Un concepto clave es la **separación de concerns en dos pipelines distintos**:

- **Indexing pipeline** — maneja la ingesta y normalización de la knowledge base. Transforma raw inputs (PDFs no estructurados, audio, JSON estructurado) en vectores recuperables almacenados en `DocumentStore`. Aunque tradicionalmente se trata como un batch job offline, la arquitectura de custom components de Haystack 2.0 permite correrlos en **real time** — por ejemplo, usando custom consumer components que disparan la ingestión directamente desde event streams como **Kafka**.
- **Query pipeline** — el pipeline online y real-time que responde requests. Toma una query, recupera contexto grounded del `DocumentStore` y usa un LLM para sintetizar una respuesta. Aunque a menudo lo dispara un usuario, en la arquitectura del libro frecuentemente sirve como un **search tool llamado por un agente**.

Al construir RAG moderno, una de las decisiones arquitectónicas más importantes es *cómo* recuperar la información. Para entender por qué el **hybrid retrieval** es necesario, hay que ver primero las limitaciones de usar un único método de búsqueda.

#### Semantic search: sparse retrievers (BM25)

Los sistemas tradicionales se apoyan en **keyword-based** o **sparse retrieval**. El algoritmo más común es [[BM25]], que matchea las palabras exactas de la query con las de los documentos, ranqueando según la frecuencia de términos con un sistema de pesos refinado que considera term frequency y document length.

- **Strengths** — rápido, eficiente y muy bueno cuando la query usa el mismo lenguaje que el documento fuente. Excelente para encontrar product codes, error messages o nombres únicos, sin necesidad de training. Excele donde el wording preciso importa (legal research, documentación técnica).
- **Weaknesses** — el **vocabulary mismatch problem** (Sourcely, 2025): buscar *AI safety concerns* puede no encontrar un doc sobre *risks of artificial intelligence* porque los keywords exactos no coinciden. Estos sistemas saben *qué son* las palabras, pero no *qué significan* — son brittle ante la variedad lingüística.

#### Lexical search: dense retrievers (embeddings)

Para resolver el vocabulary mismatch se desarrolló el **dense retrieval**. Usa un embedding model para convertir queries y documents en vectores numéricos ([[Embeddings|embeddings]]) que capturan el significado semántico, y luego encuentra los documentos cuyo vector está geométricamente más cerca del de la query en ese espacio de alta dimensión.

- **Strengths** — excelente entendiendo matices, contexto e intención; encuentra documentos conceptualmente relacionados aunque los keywords difieran por completo.
- **Weaknesses** — al capturar el significado amplio, a veces pasa por alto términos literales específicos. Una búsqueda de un error code preciso como `HTTP 404` podría devolver docs sobre errores de servidor generales en vez del único que contiene ese string crítico exacto.

> [!note] Los dense retrievers son potentes pero requieren más recursos: el embedding inicial de todos los documentos es una tarea upfront significativa. Para escalar a millones de documentos usan vector DBs especializadas con algoritmos **approximate nearest neighbor (ANN)**.

#### Tabla 3.2 — Sparse vs. dense retrieval

| Feature | Sparse (BM25) | Dense (Embeddings) |
|---|---|---|
| **Core principle** | Lexical term matching (weighted keyword counting) | Semantic concept matching (vector proximity) |
| **Representation** | High-dimensional, sparse vector (inverted index) | Low-dimensional, dense vector |
| **Strengths** | Alta precisión en keywords, computacionalmente barato, sin training | Maneja sinónimos y conceptos, entiende sintaxis, robusto al vocabulary mismatch |
| **Weaknesses** | Vocabulary mismatch, sin comprensión semántica, brittle | Puede perder keywords específicos, computacionalmente caro, requiere training data |
| **Computational profile** | Indexing rápido; querying muy rápido | Indexing lento (requiere embedding); querying rápido (con ANN) |
| **Ideal use cases** | Legal search, product codes, queries con jerga o nombres específicos | Question answering general, exploración de temas, búsqueda conceptual |

> [!tip] Las debilidades de sparse y dense son **complementarias**: donde una falla, la otra acierta. Por eso los sistemas RAG más robustos combinan ambas. El hybrid retrieval no es solo un upgrade técnico, es un **principio de diseño core** para un sistema que se adapta a las formas variadas en que la gente busca información.

#### Hybrid retrieval

El patrón arquitectónico más común y flexible es el **pipeline approach**, donde el código orquesta el proceso en tres pasos:

1. **Parallel fetch** — la query se envía simultáneamente a un sparse retriever (BM25) y a un dense retriever; cada uno devuelve su propia lista ranqueada de candidatos.
2. **Fusion** — las dos listas se mergean en un set único. Lo simple es combinar y deduplicar; lo avanzado y efectivo es [[Reciprocal Rank Fusion (RRF)]], que re-rankea los documentos según su posición en cada lista, evitando el problema complejo de comparar dos scoring systems distintos.
3. **Reranking** — la lista combinada pasa a un modelo **reranker** final (típicamente un [[cross-encoder]], ej. `BAAI/bge-reranker-base`) que mira la query y cada documento *juntos*, dando un relevance score mucho más preciso. Es computacionalmente intensivo pero crucial para filtrar los candidatos a los más relevantes.

### Building advanced RAG pipelines

El ejemplo (detallado en el Cap 4) usa el patrón **client-side fusion**: la query fluye por dos branches paralelos que luego se unen, se rerankean y se envían a un generator:

- **Sparse retrieval branch** — el keyword precision engine; ejecuta una búsqueda BM25 por los términos exactos de la query.
- **Dense retrieval branch** — maneja la semantic search; el embedder convierte la query en vector y el embedding-based retriever encuentra documentos conceptualmente similares.
- **Fusion component** — toma las listas de ambos branches, las mergea y remueve duplicados.
- **Re-ranking component** — el precision engine; un cross-encoder mira query + cada documento candidato juntos, produciendo un ranking final altamente preciso.
- **Generation stage** — la lista final de alta calidad se formatea en un prompt vía un *prompt builder*, que se envía a un *LLM generator* para sintetizar la respuesta final.

### Developing agentic systems

La introducción de agents y tools en Haystack 2.0 representa un shift fundamental: mueve el rol del developer de ser un *pipeline builder* para humanos a un **capability provider** para agentes, sin importar el patrón arquitectónico elegido.

- Para developers que aprovechan las **capacidades ágenticas nativas** de Haystack, el shift permite crear un set de tools robustos y bien descritos, y confiar en el sistema single/multi-agent interno para orquestarlos dinámicamente — simplificando la complejidad y manejando queries imprevistas sin reprogramar cada edge case.
- Para developers que prefieren el **control de estado granular de [[LangGraph]]**, serializar pipelines Haystack y desplegarlos vía Hayhooks o como MCP servers transforma a Haystack de librería local a un engine robusto de microservicios deterministas.

> [!warning] Este shift introduce nuevos desafíos. El debugging deja de ser solo chequear el data flow entre components: implica **analizar el reasoning trace del LLM** (la secuencia de thoughts y tool choices) para entender por qué se comportó de cierta manera. El prompt engineering también se expande — ya no es solo craftear un buen prompt para la respuesta final, sino escribir **tool descriptions claras y efectivas** que guíen exitosamente el razonamiento del agente.

#### RAG as a tool

La composabilidad de Haystack 2.0 hace trivial pasar de pipelines a agentes. Todo el hybrid RAG pipeline state-of-the-art puede envolverse en un único tool creando una instancia `tool` con:

- **Name** — `internal_knowledge_search`.
- **Description** — *Use this tool to search our internal database of company documents. It is the best source for information about our products, internal policies, and historical project data. The input should be a clear, specific question.*
- **Instance** — el objeto `hybrid_rag_pipeline` en sí.

Así, el proceso multi-step de hybrid retrieval queda abstraído en una **capacidad callable única** para un agente.

#### Building a multi-tool agent

El verdadero poder de los agentes se realiza cuando tienen acceso a **múltiples tools distintos**. Se crea un segundo tool para búsqueda web en vivo:

1. Crear un pipeline simple usando un **Serper web search component** (integración del Google Search API de bajo costo).
2. Envolver ese pipeline en otro tool: **Name** `web_search`; **Description** (*Use this tool to find up-to-the-minute information from the public internet. It is best for current events, competitor news, or general knowledge questions that are not specific to our company*); **Instance** = el objeto `web_search_pipeline`.

Ante una query compleja como *Summarize our internal Q3 performance report and compare our main product's features against our top competitor's latest release*, el agente razona en un loop tipo [[ReAct]]:

1. **Thought** — *The query has two parts. The first part is about an internal report. The `internal_knowledge_search` tool is described as being best for this.*
2. **Action** — el agente llama `internal_knowledge_search` con input `Q3 performance report`.
3. **Observation** — recibe el resumen del reporte desde el RAG pipeline.
4. **Thought** — *Now I need to find information about the competitor's latest release. This is external, recent information. The `web_search` tool is best for this.*
5. **Action** — el agente llama `web_search` con input `competitor X latest product release features`.
6. **Observation** — recibe una lista de resultados de búsqueda web.
7. **Thought** — *I now have all the information I need. I will synthesize the internal report summary and the competitor feature list into a final answer.*
8. **Final answer** — genera una respuesta comprehensiva que aborda ambas partes de la query original.

> [!tip] Este enfoque se generaliza fuera de Haystack diseñando sistemas multi-agent donde cada agente se empareja con uno o más pipelines Haystack desplegados como **REST endpoints** o servidos vía un **MCP server**.

## Deep dive en la arquitectura de Haystack 2.0

### Data classes

Antes de los components hay que entender las data structures core que fluyen entre ellos. Son el "lifeblood" del pipeline:

- **`Document`** — la data class más fundamental. Encapsula una pieza de datos procesable y almacenable. Puede contener texto (content) o datos binarios (blob), junto con un ID único, metadata custom, un relevance score, un dense vector embedding, y `sparse_embedding` para escenarios de hybrid retrieval.
- **`ByteStream`** — representa datos binarios crudos (imagen, PDF) antes de convertirse a texto. Contiene los bytes y metadata asociada, incluyendo `mime_type` (ej. `image/png`).
- **`ChatMessage`** — para aplicaciones conversacionales y agentes. Objeto estructurado (no un simple string) que puede contener contenido multimodal: texto, imágenes (`ImageContent`), tool calls (`ToolCall`) y tool results (`ToolCallResult`). Define explícitamente el role del emisor (*user*, *assistant*, *system* o *tool*).
- **`StreamingChunk`** — clase especializada para respuestas real-time. Encapsula un único segmento de contenido streameado (ej. un token) con metadata y posibles tool call deltas, habilitando UX fluida de baja latencia.
- **`Answer`** — familia de clases para el output final del pipeline: `GeneratedAnswer` (respuesta generada por un LLM, con el answer string y la lista de `Document` fuente usados) y `ExtractedAnswer` (usada en extractive QA, un span de texto extraído directamente de un documento fuente).

### Document stores

`DocumentStore` es el **backend de base de datos** donde se almacenan los `Document` para retrieval. La arquitectura de Haystack separa claramente la **storage layer** (`DocumentStore`, configurado y gestionado *fuera* del pipeline) de la **access layer** (el retriever, usado *dentro* del pipeline para fetchear datos). Haystack soporta:

- **In-memory** — opciones para prototyping rápido.
- **Vector databases** — `ChromaDocumentStore`, `PineconeDocumentStore`, `WeaviateDocumentStore`, `QdrantDocumentStore`, `MilvusDocumentStore`.
- **Traditional search engines** — `ElasticsearchDocumentStore`, `OpenSearchDocumentStore`.
- **Other databases** — `AstraDocumentStore` (Cassandra), `MongoDBAtlasDocumentStore`, `Neo4jDocumentStore` (Graph).

> [!tip] Elegir el document store correcto es una decisión arquitectónica clave según la escala del proyecto y la infraestructura existente. Haystack también soporta la creación de document stores custom (guía: docs.haystack.deepset.ai/docs/creating-custom-document-stores).

### Component categories

Los components pre-definidos se organizan en categorías lógicas según su función en un workflow end-to-end típico (Figura 3.4), y la funcionalidad se expande vía custom components (Cap 5):

![[03-fig-3.4-component-categories.png]]
*Figure 3.4 – Component categories in an end-to-end workflow*

- **Data preprocessing** — primeros pasos del indexing pipeline, ingieren raw data y la preparan: **file converters**; **web component** (recupera contenido de URLs y devuelve `ByteStream`, ej. `LinkContentFetcher`); **preprocessing components** (normalizan whitespaces, remueven headers/footers, limpian líneas vacías, splitean en piezas); **audio components** (transcriben audio a texto).
- **Data embedding** — usan un modelo de deep learning para transformar texto en vectores. Haystack separa embedders de documentos (indexing) y de queries (searching) porque algunos modelos tienen settings óptimos distintos por tarea: **OpenAI** (`OpenAIDocumentEmbedder`); **Sentence Transformers** (`SentenceTransformersDocumentEmbedder`, `SentenceTransformersTextEmbedder`, cargan cualquier modelo compatible del Hugging Face Hub); **otros providers** (Cohere, Hugging Face). El **`DocumentWriter`** es el paso final del indexing pipeline: toma una lista de `Document` y los escribe al `DocumentStore` con políticas para duplicados (skip, overwrite, fail).
- **Data retrieval** — el corazón de cualquier app de search/RAG. Los **retrievers** fetchean un subconjunto pequeño de documentos relevantes del `DocumentStore` según una query (soportan embedding-based y keyword-based). Los **rankers** mejoran la calidad reordenando los documentos que devuelve un retriever — clave en advanced RAG para asegurar que los más relevantes pasen al paso final.
- **LLM generation** — interactúan con LLMs y construyen prompts: **generators** (`OpenAIGenerator`, `HuggingFaceTGIGenerator`, `AnthropicGenerator`; manejan las API calls a los providers, toman un prompt y devuelven texto); **builders** (`PromptBuilder` —crea prompts desde templates sustituyendo variables como query y docs—, `AnswerBuilder` —parsea objetos `Answer` estructurados del texto crudo, a menudo con regex—, `ChatPromptBuilder` —builder especial para apps tipo chat).
- **Routing** — esenciales para pipelines complejos y no-lineales habilitados por el DG. **Routers** (rutean queries/documents a los components que mejor los manejan) y **joiners** (unen documents, data structures e incluso pipelines). Son la "plumbing" que habilita DAGs sofisticados.
- **Agentic** — components high-level para sistemas ágenticos: **`Agent`** (el reasoning engine core: toma una query y una lista de tools, y los orquesta para producir una respuesta) y **`ToolInvoker`** (ejecuta las tool calls preparadas por los language models).

## Incorporando Haystack a tu workflow

### Assessing your current data system

Antes de escribir código, hacer un análisis fundacional del proyecto en cuatro frentes:

1. **Definir objetivos NLP** — ¿Q&A interno, chatbot de soporte, o un agente autónomo? El objetivo determina la complejidad del pipeline.
2. **Evaluar fuentes de datos** — ¿estructurado en DBs o no-estructurado en PDFs/Word/HTML? Define los converters y preprocessing components.
3. **Assess infrastructure/compatibility** — on-premises vs cloud; mapear los sistemas de DB existentes y su compatibilidad con las integraciones de `DocumentStore`; asegurar soporte de infra para vector DBs como Pinecone o Weaviate.
4. **Data dynamics y governance** — ¿cada cuánto cambian los datos (reportes trimestrales estáticos vs streams real-time)? Evaluar la sensibilidad de los datos y establecer data governance y seguridad upfront.

### Designing your NLP pipeline (visual-first)

La naturaleza explícita y graph-based de Haystack 2.0 se presta a un diseño visual-first en tres pasos:

1. **Sketch the graph** — dibujar el workflow en un whiteboard o herramienta de diagramas: cada paso un nodo (component), flechas para el flujo de datos. Ej.: un hybrid RAG query pipeline muestra la query splitándose en dos paths de retriever paralelos que mergean en un joiner, fluyen por un ranker y finalmente a un generator.
2. **Translate the sketch to code** — con el grafo visual de guía: instanciar cada component, instanciar el pipeline, usar `pipe.add_component()` por cada nodo y `pipe.connect()` por cada flecha.
3. **Verify with `.draw()`** — tras definir el pipeline, correr `pipe.draw("pipeline_design.png")` y comparar con el sketch inicial. Esta verificación simple caza errores lógicos de conexión temprano.

### A recommended development roadmap

Para quienes recién empiezan con Haystack 2.0, conviene construir complejidad incrementalmente en cinco pasos:

1. **Build the indexing pipeline** — empezar por la preparación de datos: tomar los source files, procesarlos, embeddarlos y escribirlos al `DocumentStore`. Fundamental para cualquier RAG.
2. **Build a naive semantic RAG pipeline** — el query pipeline más simple posible: embed query → retrieve por similitud de vector → generate. Da un baseline end-to-end funcional.
3. **Enhance to a hybrid RAG pipeline** — agregar un path BM25 paralelo, joinear los resultados y sumar un reranker. En esta etapa se tiene un RAG state-of-the-art.
4. **Evaluate naive vs hybrid** — usar un framework como [[Ragas]] para generar un knowledge graph desde la knowledge base, crear personas, seleccionar query strategies, armar pares Q&A y evaluar sistemáticamente contra las respuestas reales (hands-on en caps posteriores).
5. **Graduate to an agent-ready tool** — envolver el hybrid RAG en un tool y validarlo localmente con un agente Haystack estándar. Una vez validado, el move final a producción (y el foco arquitectónico del libro) es serializar el pipeline, desplegarlo como REST microservice vía [[Hayhooks]] y entregar ese endpoint a un orquestador stateful como [[LangGraph]].

### Troubleshooting common issues (Tabla 3.3)

| Common issue | Likely cause | Possible fix |
|---|---|---|
| Data ingestion and processing errors | File paths incorrectos o configuraciones de chunk size poco razonables en preprocessing components | Atender los logs de converter y preprocessor para identificar el error. Asegurar paths correctos y ajustar chunk sizes. |
| Pipeline connection errors: `PipelineConnectError` | Conectar sockets con tipos incompatibles o conectar un socket inexistente | Usar `.draw()` para inspeccionar visualmente las conexiones y verificar que coincidan con las definiciones de los components. |
| Model and performance bottlenecks | Modelos pesados (embedders, rankers) no inicializados antes de procesar, o latencia natural alta de LLM generation | Llamar `warm_up()` en los components pesados antes de procesar requests. Para latencia de LLM, considerar streaming. |
| Unexpected agent behavior | Una tool description mal redactada o ambigua que confunde el razonamiento del agente | Inspeccionar el reasoning trace del agente (thoughts, tool selections, observations) en los logs. Refinar el prompt en lenguaje natural que define la capacidad del tool. |
| Data security and privacy risks | Hardcodear API keys/credenciales u obviar requisitos de data residency y compliance | Gestionar credenciales de forma segura vía variables de entorno o un secrets manager. Atender el compliance al usar providers cloud. |

## Citas

> "Developers no longer rely on implicit data passing; instead, they explicitly instantiate a `Pipeline` object, add `Component` instances, and strictly define the flow of data by connecting typed sockets."
> "This explicit connect syntax is the cornerstone of Haystack 2.0's transparency. It leaves no ambiguity about how data moves through the system."
> "A SuperComponent is a pre-packaged, reusable pipeline that can be treated as a single component within a larger pipeline."
> "Because the tool is a wrapper around a validated Haystack pipeline, the agent is orchestrating reliable, deterministic units of work rather than fragile scripts."
> "The Haystack agent component encapsulates the reasoning loop internally. This black box approach hides the state, making it difficult to implement the advanced memory curation strategies or the complex state management that a framework such as LangGraph provides."
> "Hybrid retrieval is not just a technical upgrade; it's a core design principle for building a system that can adapt to the flexible and varied ways that people seek information."
> "It moves the developer's role from being a strict pipeline builder for humans to a capability provider for agents."

## Para aplicar

- **Setup del capítulo** — hay un `pyproject.toml` dedicado; abrir la carpeta `ch3` en una ventana standalone de VS Code e instalar dependencias: `cd ch3/` → `uv sync` → `source .venv/bin/activate`. Activar el venv como kernel del Jupyter notebook (Select Kernel → Python Environments → el path del venv, llamado `rag-with-haystack-ch3`, o la ruta relativa `.venv/bin/python`).
- **Notebooks de práctica** — `components.ipynb` (usar components pre-existentes), `your-first-custom-component.ipynb` (el prefixer custom component), `your-first-pipeline.ipynb` (pipeline simple + visualización), `supercomponents.ipynb` (preprocessor SuperComponent + prefixer).
- **Crear un component** — agregar `@component` sobre una clase Python con un método `run()`; definir typed input/output sockets para imponer type safety.
- **Construir un pipeline en 3 pasos** — `Pipeline()` → `add_component(name, instance)` → `connect("a.output_socket", "b.input_socket")`. Verificar siempre con `pipe.draw("...png")` contra el sketch inicial.
- **Diseñar visual-first** — sketch del grafo → traducir a código (un `add_component` por nodo, un `connect` por flecha) → `.draw()` para cazar errores de conexión temprano.
- **Seguir el roadmap incremental de 5 pasos** — indexing pipeline → naive semantic RAG → hybrid RAG (BM25 paralelo + joiner + reranker) → evaluar con [[Ragas]] → graduar a agent-ready tool (validar con agente Haystack, luego serializar y desplegar vía Hayhooks para LangGraph).
- **RAG production-grade** — usar hybrid retrieval (parallel fetch sparse+dense → fusion con RRF → reranking con cross-encoder, ej. `BAAI/bge-reranker-base`) como baseline.
- **Envolver un pipeline como tool** — crear una instancia `tool` con Name corto y único, Description clara en lenguaje natural (guía el razonamiento del agente) e Instance = el objeto pipeline. Para multi-tool, repetir (ej. `web_search` con un Serper web search component).
- **Performance** — llamar `warm_up()` en components pesados (embedders, rankers) antes de procesar; usar streaming para mitigar latencia de LLM.
- **Seguridad** — nunca hardcodear API keys; usar variables de entorno o un secrets manager; atender data residency/compliance al usar providers cloud.

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[02 - Diving Deep into Large Language Models]] — capítulo anterior (introduce la arquitectura híbrida LangGraph + Haystack, context engineering, hybrid search y RRF que este capítulo aterriza en components concretos).
- [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]] — capítulo siguiente (ensambla estos components en pipelines reales: indexación multi-fuente, naive/hybrid RAG, multimodal, async).
- [[05 - Haystack Pipeline Development with Custom Components]] — profundiza custom components, `warm_up()` y la generación de ground-truth con Ragas.
- [[06 - Building Reproducible and Production-Ready RAG Systems]] — evaluación cuantitativa (Ragas) del naive vs hybrid mencionada en el roadmap.
- [[07 - Deploying Haystack-Based Applications]] — despliegue productivo con Hayhooks como REST/MCP (introducido aquí).
- [[08 - Hands-On Projects]] — NER/sentiment/classification como tools y la orquestación LangGraph; concreta el "RAG as a tool" y el multi-tool agent de este capítulo.
- [[Haystack 2.0]] · [[Hayhooks]] · [[SuperComponent]] · [[Custom Components (Haystack)]] — los building blocks de la tool layer presentados aquí.
- [[RAG]] · [[Hybrid Search]] · [[BM25]] · [[Reciprocal Rank Fusion (RRF)]] · [[cross-encoder]] · [[Embeddings]] — el stack de retrieval.
- [[Model Context Protocol (MCP)]] · [[LangGraph]] — interoperabilidad y la orchestration layer complementaria.
- [[Ragas]] · [[DeepEval]] — frameworks de evaluación del ecosistema.
- [[ReAct]] — el reasoning loop (thought → action → observation) del multi-tool agent; candidato a nota propia.
