---
title: 01 - Introduction to Natural Language Processing Pipelines
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 1
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Introduction to Natural Language Processing Pipelines
  - Cap 1 - NLP Pipelines
updated: 2026-06-12
---

# 01 - Introduction to Natural Language Processing Pipelines

> [!info] Capítulo 1 · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> El capítulo fundante: presenta la **agentic reliability crisis** de 2026 y la tesis del libro — el camino a aplicaciones ágenticas confiables NO es el prompt engineering solo, sino la **aplicación rigurosa del procesamiento clásico de data pipelines**. Recorre qué son los data pipelines y su nuevo rol en la era ágentica, las técnicas de procesamiento de texto, la evolución de pipelines clásicos → NLP → LLM → ágenticos, y el ciclo continuo [[MLOps]]/[[AgentOps]]. Establece el patrón arquitectónico central: [[tool layer vs orchestration layer]]. Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · siguiente [[02 - Diving Deep into Large Language Models]].

## Resumen

El capítulo abre el libro plantando su **tesis fundacional**: en 2026 la industria llegó a un punto de inflexión donde la pregunta dejó de ser *"¿puede la IA hacer esto?"* y pasó a ser *"¿se puede confiar en esta IA?"*. El foco se corrió de raw performance a **reliability, scalability y governance**. El problema central es la **agentic reliability crisis**: un agente es tan bueno como los datos y tools que recibe, y un agente que consume datos defectuosos no es solo poco fiable — es un mecanismo para *escalar hallucinations*, amplificar fallos operativos y crear vulnerabilidades de seguridad. La respuesta del libro: la confiabilidad production-grade se construye aplicando las **prácticas data-centric clásicas** (ingeniería de data pipelines) como fundamento de los sistemas ágenticos, usando [[Haystack 2.0]] como framework central.

A partir de ahí el capítulo cubre cuatro grandes bloques. Primero, **qué son los data pipelines y su rol en la era ágentica**: la analogía del café (raw beans → ground & brewed), los cuatro principios del [[data mesh]], y el cambio fundamental — el nuevo *data consumer* ya no es un humano mirando un dashboard sino un **agente autónomo** que no tolera datos defectuosos (para un agente, *"data is not an insight; it is a command"*). Segundo, **texto como dato**: el overview de técnicas de preprocesamiento (Tabla 1.1) y el re-framing moderno de la NLP clásica como **tools ágenticos deterministas**. Tercero, los **componentes clave** de pipelines clásicos, NLP y LLM, profundizando en tokenization/embeddings (Tabla 1.2), la regla no-negociable del *mismo embedding model* en RAG, y los dos roles del LLM (especialista en la tool layer, generalista en la orchestration layer), incluyendo la distinción **shallow vs deep agent**. Cuarto, el **ciclo MLOps/AgentOps** de 2025 (Tabla 1.4 = roadmap del libro entero) que reemplaza el viejo proceso lineal por un loop continuo de evaluación, mejora y redespliegue.

El hilo conductor: el LLM es un sistema probabilístico no-determinista; rodearlo de pipelines deterministas, evaluación cuantitativa y deployment escalable es lo que lo vuelve confiable. Las prácticas clásicas de data science **no son reemplazadas** por los agentes — son su fundamento indispensable.

## Data pipelines y su rol en aplicaciones ágenticas

Un **data pipeline** es un conjunto de procesos que extraen y mueven datos de un sistema a otro, transformándolos en el camino. Sus etapas clave: **data collection, processing, storage, analysis, modeling y serving**. Automatizan el flujo de datos con el fin de transformar información cruda en un formato apto para extraer insights, que puede servirse como dashboard, reporte, análisis de forecasting o aplicaciones.

Para ubicar los data pipelines en un contexto arquitectónico más amplio, el capítulo recurre a los cuatro principios del **[[data mesh]]** (Martin Fowler, 2019):

- **Domain-oriented decentralized data ownership** — los datos los gestiona quien está más cerca de ellos; los pipelines especializados (ej. los microservicios Yelp del Cap 8) los construyen domain experts, no un equipo centralizado.
- **Data as a product** — los pipelines se tratan como productos de alta calidad y descubribles. En el libro, el producto es un **microservicio containerizado** que sirve datos confiables a un agente autónomo.
- **Self-serve data infrastructure as a platform** — tooling estandarizado para que los developers construyan, desplieguen y escalen sus pipelines NLP sin reinventar la infraestructura subyacente.
- **Federated computational governance** — los pipelines descentralizados siguen estándares globales de seguridad e interoperabilidad, como el [[Model Context Protocol (MCP)]] (Cap 9).

> [!note] **La analogía del café.** Los granos enteros son los *raw data*; para consumirse deben molerse y prepararse. Así como el tiempo y el método de brewing cambian drásticamente el perfil de sabor, las técnicas y tools de transformación de datos influyen no solo en la calidad y precisión de los insights, sino en *quién* se beneficia del producto final.

### Serving the perfect cup — el nuevo data consumer

En el procesamiento de datos, el **data consumer** juega el rol del cliente que pide el café. Pero en 2025 y más allá, ese consumer cada vez más **no es un humano mirando un dashboard sino un agente de IA autónomo**, con requisitos mucho más exigentes. Un humano tolera una "mala taza de café" (datos algo defectuosos, un dashboard lento, un gráfico ambiguo) y aún extrae valor con su intuición. **Un agente no puede.**

> [!warning] Para un agente autónomo, *"data is not an insight; it is a command"*. Datos defectuosos, no verificados o ambiguos no llevan a un reporte ligeramente incorrecto, sino a **fallos críticos**: cascading hallucinations (un dato malo dispara una cadena de razonamiento incorrecto), desperdicio de compute en tareas fallidas, o vulnerabilidades severas como **excessive agency** (el agente toma una acción dañina no intencionada por input manipulado — Lasso Security, 2025; OWASP LLM06:2025).

El rol del data pipeline en la era ágentica cambió de raíz: su propósito ya no es solo transformar datos crudos en un formato apto para extraer insights, sino **transformarlos en un formato apto para razonamiento ágentico confiable**.

### The pipeline as a production-grade product

El libro enseña a construir el data pipeline como, literalmente, un **data product que sirve a agentes** — no es solo una metáfora:

- En el **Cap 7** el pipeline se serializa y expone como microservicio self-serve con REST API usando [[Hayhooks]].
- En el **Cap 8** el agente de IA (construido con un orquestador como [[LangGraph]]) se vuelve el *consumer* de ese producto, llamando su API endpoint como un tool.
- El **Cap 9** estandariza esto vía [[Model Context Protocol (MCP)|MCP]] (Google, 2024).
- El **Epílogo** mejora sistemáticamente al agente tratando los pipelines como assets separados y optimizados.

> [!tip] Esto convierte al data pipeline en la **foundational reliability layer** de todo el sistema ágentico. El pipeline es el barista que garantiza la calidad del café (datos) antes de que el cliente (agente) lo consuma. La garantía se da vía **evaluación cuantitativa** ([[Ragas|RAGAS]], Cap 6) y **observabilidad continua** ([[Weights and Biases|Weights & Biases]], Cap 6).

## Texto como dato — overview de técnicas de procesamiento

El texto es una de las fuentes de datos más abundantes y ricas (social media, reviews, papers, documentación corporativa, news, web). Pero, a diferencia de los datos estructurados que caben en tablas y schemas, el texto es **inherentemente no estructurado**. Para obtener insights hay que transformarlo en un formato que un LLM pueda procesar — por ejemplo, chunkeándolo (**tokenization**) y vectorizándolo con un embedding model. Una vez transformado, sirve para traducción, sentiment analysis, topic modeling, information retrieval (Q&A) y text classification.

Antes de almacenar texto o usarlo con un modelo, hay que **preprocesarlo**: limpiar, normalizar y estructurar el lenguaje natural.

### Tabla 1.1 — Overview of text-processing techniques

| Técnica | Característica clave | Ejemplo | Aplicación |
|---|---|---|---|
| **Tokenization** | Romper el texto en piezas más chicas | `Hello, my name is` → `Hell-o my-na-me is` | Word frequency |
| **Stop word removal** | Quitar palabras que no cargan significado; la remoción no debe alterar el contexto | "the", "is" | Mejorar la eficiencia del análisis de texto |
| **Stemming y lemmatization** | Reducir palabras a su forma base o raíz | `Running` → `run` | Estandarización de texto |
| **Part-of-speech tagging** | Identificar las partes gramaticales de cada palabra | Noun, verb, adjective | Entender contexto y significado de palabras/oraciones |
| **Named-entity recognition (NER)** | Identificar y clasificar entidades nombradas en categorías | Personas, organizaciones, lugares | Extraer información de grandes volúmenes de texto |
| **Text normalization** | Convertir el texto a un formato estándar | Todo minúsculas/mayúsculas, reemplazar espacios con `_` | Asegurar consistencia |
| **TF-IDF** | Medida estadística de cuán importante es una palabra en un corpus de documentos | Ver fórmula abajo | Information retrieval y text mining |
| **Text embeddings** | Convertir palabras/oraciones/documentos en vectores de números, capturando relaciones semánticas | Word, sequence y contextual embedding | Entender contexto y semántica de las palabras |

*Table 1.1 – Overview of text-processing techniques*

El ejemplo de TF-IDF: si la palabra "cat" aparece 20 veces en un corpus de 1,000 términos, y el corpus tiene 10 documentos de los cuales 2 contienen "cat", su TF-IDF se calcula según la fórmula:

![[01-fig-tfidf-formula.png]]
*TF-IDF — ejemplo de cálculo (Table 1.1)*

> [!note] **Tokenization y text embeddings son el puente entre lenguaje natural y lógica de máquina.** La tokenization rompe el texto continuo en unidades discretas que un modelo puede trackear; los text embeddings transforman esas unidades en vectores numéricos que capturan el significado semántico real y las relaciones entre palabras. Sin estos pasos, un LLM no puede ver los patrones del lenguaje ni razonar. Estandarizar y vectorizar los datos crea el **fundamento determinista** para enfrentar la agentic reliability crisis.

### Modern re-framing: NLP clásica como tools ágenticos confiables

> [!warning] Un misconception común de 2025 es que el poder de los LLMs vuelve obsoletas las técnicas de NLP clásica. **Es al revés.** Aunque un LLM potente puede hacer NER o sentiment analysis, a menudo NO es la forma más confiable, cost-effective o gobernable de hacerlo. Un LLM es un sistema **probabilístico y no-determinista**; un clasificador fine-tuned o un NER rule-based es **predecible, rápido y barato**.

Esa distinción —razonamiento probabilístico vs ejecución determinista— motiva una **separación clara de concerns arquitectónicos**: diseñar pipelines deterministas con NLP clásica y usarlos como tools de un agente potenciado por LLM. Esto da lo mejor de ambos mundos: la fiabilidad de los enfoques testeados + el matiz y procesamiento dinámico del LLM.

De acá surge el patrón arquitectónico central del libro, el **[[tool layer vs orchestration layer]]**. Un sistema ágentico moderno (Cap 8) se compone de dos capas distintas:

- **The orchestration layer** — el cerebro o reasoning engine (ej. una state machine de [[LangGraph]]) que gestiona la lógica de alto nivel y decide *qué hacer*.
- **The tool layer** — un set de tools especializados y de alto rendimiento que el orquestador llama para ejecutar tareas específicas.

> [!tip] Acá renacen las pipelines de NLP clásica: lejos de ser artefactos de una era previa, técnicas como tokenization, stemming y NER se vuelven los **building blocks primarios de los tools ágenticos robustos**. En el Cap 8 se construyen pipelines clásicas de NER y sentiment como microservice tools, y el LLM se trata como orquestador que delega — mucho más robusto, debuggable y gobernable que pedirle a un único LLM monolítico que haga todo.

## Componentes clave en pipelines de datos, NLP y LLM

Esta sección muestra el camino evolutivo de los pipelines, de procesamiento clásico de datos a los sistemas ágenticos modernos.

### Classic data y NLP pipelines (el fundamento)

Un data pipeline general ingiere datos de fuentes (APIs, IoT, archivos), los procesa y los almacena en bases de datos o data warehouses; luego se usan para análisis, reporting y visualización. Hay un componente **iterativo** en las etapas de análisis y visualización, que suelen revelar si los datos o algoritmos son insuficientes para el objetivo deseado.

![[01-fig-1.1-stages-data-pipeline.png]]
*Figure 1.1 – Stages in a general data pipeline*

Construir pipelines de NLP clásica es similar, pero con diferencias clave por la naturaleza no estructurada del texto. Los pasos:

1. **Text acquisition** — reunir texto crudo de fuentes diversas (social media, reviews, documentación, web).
2. **Cleaning** — refinar el input crudo para asegurar datos de alta calidad y sin ruido.
3. **Tokenization** — el paso fundacional de romper el texto continuo en unidades discretas (palabras, caracteres, sub-words).
4. **Feature extraction** — convertir el texto en representaciones numéricas (scores TF-IDF o embeddings) para capturar significado semántico.
5. **Model training and inference** — usar esas features para entrenar o usar modelos especializados y deterministas para tareas como clasificación o sentiment analysis.

> [!note] Decisiones clave: opciones de almacenamiento (ej. **Elasticsearch**), modos de procesamiento (**real-time vs batch**), y la importancia crítica del **version control** (Git para código, **DVC** para datos) para sistemas reproducibles y escalables. Estas pipelines clásicas y deterministas establecen el fundamento reproducible sobre el que se construye el resto.

### Text tokenization y embeddings

La **tokenization** rompe el texto continuo en unidades discretas (palabras, caracteres, sub-words, frases). Es fundacional tanto para métodos tradicionales (ej. la clase `CountVectorizer` de Scikit-learn) como para la arquitectura detrás de los modelos state-of-the-art (el [[Transformer]]). Una vez tokenizado, cada token se mapea a un entero o ID único — proceso conocido como **encoding**. Para tareas avanzadas, los tokens se representan como **vectores** (word vectorization) que capturan significado y relaciones.

El ejemplo canónico con un vocabulario de 4 palabras en un espacio 3D:

```text
"king"  = [0.8, 0.6,  0.1]
"queen" = [0.8, 0.6, -0.1]
"man"   = [0.4, 0.2,  0.1]
"woman" = [0.4, 0.2, -0.1]
```

- **Semantic similarity** — palabras con significados similares quedan cerca en el espacio vectorial (ej. "king" y "queen" tienen vectores similares).
- **Relationships** — la diferencia entre vectores captura relaciones: la diferencia "king" − "queen" (`[0, 0, 0.2]`) es como la diferencia "man" − "woman" (`[0, 0, 0.2]`).

#### Tabla 1.2 — Types of text tokenization

| Tipo | Característica | Drawbacks |
|---|---|---|
| **Word tokenization** | Splitea el texto por espacios o delimitadores (tokens a nivel de palabra) | Sufre con palabras fuera del vocabulario (**out-of-vocabulary**, OOV) |
| **Character tokenization** | Rompe el texto en caracteres individuales | Maneja OOV, pero la longitud de input/output crece rápido → cuesta aprender la relación entre caracteres |
| **Sub-word tokenization** | Divide el texto en unidades menores con significado (n-gram characters) | La efectividad varía según la técnica usada |
| **Byte pair encoding (BPE)** | Sub-word tokenization que fusiona iterativamente pares frecuentes de caracteres/secuencias | A veces da tokenización subóptima de ciertos inputs o palabras/frases |
| **SentencePiece** | Tokenizer/detokenizer unsupervised y data-driven para tareas de generación con redes neuronales | El word splitting puede perder significado semántico; los caracteres especiales se tratan como tokens separados |

*Table 1.2 – Types of text tokenization*

> [!warning] **La regla no-negociable de RAG: el mismo embedding model en indexing y querying.** Un pipeline [[RAG]] (Cap 4) tiene una **indexing phase** (los documentos se procesan y guardan en una vector database) y una **querying phase** (la pregunta del usuario se procesa en tiempo real). Un embedding model crea un **vector space** único de alta dimensión. Usar un modelo para crear el "mapa" (indexing) y otro distinto para la "brújula" (querying) es *"the equivalent of using a map of Paris to navigate the streets of Tokyo"* (Jabloun, 2025) — el mismatch de vector space hace que el pipeline **falle catastróficamente**.

Elegir el embedding model es una de las decisiones de data science más críticas, con consecuencias en reliability, performance y costo, y plantea tres preguntas:

1. **¿Es efectivo para mi dominio?** — que los embeddings capturen relaciones semánticas es una *asunción que debe probarse*. En el Cap 5 se construyen custom components (un **knowledge graph generator** y un **synthetic test set generator**) para crear un ground-truth dataset, y en el Cap 6 se usa [[Ragas|RAGAS]] para *puntuar cuantitativamente* el pipeline en métricas como **faithfulness** y **context recall**.
2. **Trade-off de MLOps/[[FinOps]]** — un embedding model más grande como OpenAI `text-embedding-3-large` puede dar ganancias marginales sobre `text-embedding-3-small`, pero con un **aumento de costo de 6.5×**. Para decidir, se usan herramientas de observabilidad como [[Weights and Biases|Weights & Biases]] (Cap 6) que trackean no solo scores sino el **dollar cost per query**.
3. **Information retrieval** — método clave usado en todo el libro; en el Cap 4 se profundiza el sparse y dense retrieval (combinar keyword-based con embeddings) para grounding en un corpus.

### LLMs

Un **LLM** es una categoría de modelos de deep learning para manejar y generar texto human-like, caracterizados por su vasta cantidad de parámetros (de cientos de millones a decenas de miles de millones). Su tamaño les permite capturar patrones del lenguaje y hacer un amplio rango de tareas NLP sin datos task-specific. La mayoría se basa en la arquitectura **[[Transformer]]** (Vaswani et al.), pre-entrenados en corpus masivos y luego fine-tuneables. Ejemplos conocidos: **GPT** de OpenAI (generative pre-trained transformer) y **[[BERT]]** de Google (bidirectional encoder representations from transformers).

> [!note] Una fortaleza de los LLMs es el **transfer learning** (aprovechar conocimiento de una tarea en otra). Sus desafíos: requieren recursos computacionales significativos (GPU/RAM) para training/fine-tuning/hosting (técnicas para reducir costo en el Cap 2), y pueden producir **hallucinations** (outputs incorrectos, sesgados o sin sentido). Técnicas como [[RAG]] (Lewis et al., 2021) permiten al LLM extraer info factualmente correcta de bases de datos conservando sus propiedades generativas.

### Modern LLM pipelines (el salto arquitectónico)

Un **LLM pipeline** no es solo un componente nuevo: es una **arquitectura nueva**. Es una secuencia estructurada de pasos de procesamiento donde el LLM actúa como *uno de muchos* componentes especializados. A diferencia de una interfaz de chat simple, automatiza el flujo de información — de retrieval y cleaning a construcción de prompt y generación de respuesta — asegurando que el output esté grounded en datos verificados. Para construirlos se usa [[Haystack 2.0]], un framework Python-native y explícito sobre una arquitectura de **directed graph (DG)** (Cap 3) que habilita parallel branches, conditional routing y data flows complejos.

Los dos patrones RAG (Cap 4):

- **Naive RAG** — el pipeline baseline: un único retriever semántico (embedding-based) + un generator.
- **Hybrid RAG** — el patrón avanzado production-ready: corre en paralelo sparse/lexical search (ej. [[BM25]]) y dense/semantic search, fusiona resultados y usa un **reranker** para seleccionar los mejores documentos antes de generar.

#### Tabla 1.3 — The evolution of pipeline architectures

| Pipeline | Principio core | Componentes clave | Use case primario | Capítulo |
|---|---|---|---|---|
| **Classic NLP** | Transformación determinista | Cleaner, splitter, TF-IDF, classifier, NER extractor | Feature extraction, text classification | 1, 8, Epílogo |
| **Naive RAG** | Retrieval semántico + generación | Document embedder, query embedder, retriever, prompt builder, LLM generator | Q&A simple sobre documentos | 4, 6, 7 |
| **Hybrid RAG** | Dense + sparse retrieval + reranking | BM25 retriever, embedding retriever, document joiner, ranker, LLM generator | Q&A confiable de alta precisión | 4, 6, 7 |
| **Agentic system** | Orchestration + tool use | Agent (orchestrator), component tool (ej. hybrid RAG, NER, sentiment) | Problem-solving dinámico multi-step | 8, Epílogo |

*Table 1.3 – The evolution of pipeline architectures*

#### El LLM como especialista en la tool layer

En este rol, el LLM actúa como un **engine altamente especializado y restringido** dentro de un data pipeline — el mejor ejemplo es el componente `Generator` de los pipelines RAG. No se le permite razonar libremente; su trabajo está estrictamente definido:

1. Recibe un prompt **aumentado** con contexto verificado y recuperado (la "data" del pipeline clásico).
2. Se le instruye sintetizar una respuesta *solo* usando ese contexto provisto, evitando que alucine o use su conocimiento general.

> [!tip] Al restringir el LLM así, se lo transforma de un engine creativo impredecible en un **tool confiable y determinista**. Ese pipeline (ej. el hybrid RAG) se empaqueta y despliega como microservicio containerizado con Docker y [[Hayhooks]] (Cap 7), estandarizable luego como **MCP Server**. En esta capa el LLM no es el "cerebro": es una parte especializada de alto rendimiento de una máquina confiable.

#### El LLM como generalista en la orchestration layer

Acá un LLM distinto actúa como el **cerebro u orquestador** de todo el sistema ágentico (Cap 8, con [[LangGraph]]). Su trabajo no es saber hechos ni sintetizar datos, sino **razonar, planificar y delegar**. Se le da: un goal de alto nivel del usuario (ej. *"Find me the best-rated restaurant and book a table"*), un "tool belt" de tools disponibles, y un prompt bien estructurado. Esos tools no son funciones simples: son los **microservicios robustos** de la tool layer (ej. el RAG pipeline desplegado, o un MCP Server).

El orquestador crea dinámicamente un plan tipo [[ReAct]]:

```text
Thought:      "The user wants a restaurant. I should first use the search tool to find options."
Action:       llama al tool elegido
Observation:  recibe una lista de restaurantes
Thought:      "Now I need to book one. I will use the information tool to get the address and then call the booking API tool."
Action:       llama los siguientes tools en secuencia
```

Hay varios patrones: un único LLM con múltiples tools especializados, múltiples LLMs cada uno con sus tools, y **arquitecturas supervisor-based** (un LLM delega a otros LLMs equipados con sus propios tools y prompts). Cuando se combinan LLMs + tool belt + descripciones detalladas + prompts que permiten planear/razonar/iterar, se llega a la definición de un **AI agent**.

#### Shallow vs deep agents

El "agentic system" de la Tabla 1.3 representa un fork crítico. La industria de 2025 distingue dos clases de agentes:

> [!warning] Un **shallow agent** es el patrón más común: un loop simple alrededor de un LLM, operando en un ciclo *receive, reason, respond*, usando a menudo solo su **context window como estado**. Efectivo para tareas simples y transaccionales, pero **falla** ante problemas complejos, multi-step o long-running: su dependencia del context window efímero lo hace propenso a context overflow, goal loss e incapacidad de recuperarse de errores — poco fiable para tareas enterprise.

Un **deep agent** es una arquitectura evolucionada, diseñada para reliability y complejidad — no una entidad monolítica sino un sistema jerárquico construido sobre tres pilares que son el foco del libro:

- **Hierarchical delegation** — no "hace" todo; delega tareas a sub-agents o tools especializados y confiables (el patrón tool vs orchestration). Las pipelines Haystack robustas, evaluadas y desplegadas de los Caps 4–7 son esos "tools".
- **Explicit planning** — a diferencia del "chain of thought" del shallow agent, usa un orquestador (ej. LangGraph) para crear y mantener un plan explícito y estructurado.
- **Persistent memory** — supera los límites del context window usando memoria externa, como las **vector databases** de los pipelines RAG.

> [!note] El argumento central del libro: **un deep agent confiable es imposible de construir sin una tool layer confiable.** El razonamiento de un deep agent es tan bueno como los tools a los que delega. Por eso las prácticas clásicas de data science —pipeline engineering, evaluación cuantitativa, deployment escalable— no son reemplazadas por los agentes: se vuelven su fundamento indispensable. El LLM de la orchestration layer da el razonamiento fluido y dinámico; pero construye sus planes sobre la roca sólida de los LLMs de la tool layer, restringidos por principios clásicos para ser confiables, verificables y escalables.

## El pipeline ágentico de 2025 — el ciclo MLOps/AgentOps

El ciclo de vida de un proyecto de IA generativa del pasado era a menudo **lineal** (de problem scoping a deployment final). En 2025 y más allá lo reemplaza un **loop continuo MLOps/[[LLMOps]]/[[AgentOps]]**, donde los sistemas en producción se evalúan, mejoran y redespliegan constantemente. El libro está estructurado para superar la etapa *"it works on my machine"* resolviendo reliability, scalability y security a nivel enterprise:

- **Reliability and feedback** — enfoque sistemático para evaluar y medir pipelines RAG (Caps 5 y 6); mejora de sistemas ágenticos vía context engineering y observabilidad con LangGraph y **LangSmith** (Cap 9).
- **Scalability and deployment** — serializar pipelines en un microservicio containerizado (Docker) servido por [[Hayhooks]], escalable vía Kubernetes (Cap 7).
- **Interoperability and security** (la actualización 2025 más crítica) — estandarizar tool layer y orchestration layer vía dos protocolos: [[Model Context Protocol (MCP)|MCP]] (Google, 2024) y el protocolo **[[A2A]]** (agent-to-agent, Google, 2025), en el Cap 9.
- **The SRE for AI approach** — el Epílogo demuestra que la reliability no es propiedad de la inteligencia del modelo, sino de la **arquitectura que lo rodea**.
- **Quantifiable success** — trackear costo de la aplicación y, vía A/B testing, optimizar el uso de tokens (Cap 5 + Epílogo).

### Tabla 1.4 — The book's roadmap: From data pipeline to reliable agent

| Pilar del ciclo | Desafío core | Tecnología/técnica clave | Capítulo |
|---|---|---|---|
| **Orchestration** | ¿Cómo construyo y conecto componentes de pipeline? | Haystack 2.0, DGs, LangGraph 1.0 | 3, 8, 9 |
| **Core application** | ¿Cómo construyo un RAG moderno de alto rendimiento? | Naive RAG, hybrid RAG (sparse + dense), reranking | 4 |
| **Reliability** | ¿Cómo creo datos "ground-truth" para testear mi pipeline? | Custom components, knowledge graph generator, synthetic test generator | 5 |
| **Reliability** | ¿Cómo pruebo cuantitativamente que mi RAG es confiable y cost-effective? | RAGAS (faithfulness, recall), observabilidad (W&B), FinOps | 6 |
| **Scalability** | ¿Cómo despliego mi pipeline para tráfico de producción? | Docker, Hayhooks (API serving), CI/CD, LangSmith, LangGraph | 7, 8, 9 |
| **Agentic pattern** | ¿Cómo gradúo de un pipeline a un agente inteligente? | Tool vs orchestration pattern, Haystack (como tool), LangGraph (como orquestador) | 4, 8, Epílogo |
| **Interoperability** | ¿Cómo conectan mis agentes y tools de forma estándar? | MCP, A2A protocol | 9 |
| **Self-improvement** | ¿Cómo aprende mi agente del fallo y el feedback? | Weights & Biases, context engineering, LangSmith | 6, 9, Epílogo |

*Table 1.4 – The book's roadmap: From data pipeline to reliable agent*

> [!tip] **Repositorio del libro** — hay un repo comprehensivo organizado por capítulo con Jupyter notebooks, scripts, sample Docker files y un capstone final: `github.com/PacktPublishing/Building-Natural-Language-and-LLM-Pipelines`.

## Citas

> "An AI agent is only as good as the data and tools it is given."
> "the path to building production-grade, trustworthy agentic applications is not through prompt engineering alone. It is through the rigorous, systematic application of classic data pipeline processing."
> "For an autonomous agent, data is not an insight; it is a command."
> "Its new, mission-critical purpose is to transform raw data into a format suitable for reliable agentic reasoning."
> "the equivalent of using a map of Paris to navigate the streets of Tokyo"
> "a reliable deep agent is impossible to build without a reliable tool layer."
> "reliability is not a property of model intelligence, but of the architecture surrounding it."

## Para aplicar

- **Tratar el data pipeline como un data product que sirve agentes** — no un script: serializarlo y exponerlo como microservicio REST con [[Hayhooks]] (Cap 7), consumible por un orquestador como [[LangGraph]] (Cap 8) y estandarizable vía MCP (Cap 9).
- **Aplicar la separación tool vs orchestration** — diseñar pipelines deterministas con NLP clásica (tokenization, NER, sentiment) como tools, y dejar que el LLM orquestador delegue; más robusto, debuggable y gobernable que un LLM monolítico.
- **Respetar la regla no-negociable de RAG** — usar exactamente el **mismo embedding model** en indexing y querying para no romper el vector space ("map of Paris / streets of Tokyo").
- **Elegir el embedding model con datos, no por defecto** — probar su efectividad en el dominio con un ground-truth dataset (knowledge graph + synthetic test generator, Cap 5) y RAGAS (faithfulness, context recall, Cap 6); evaluar el trade-off FinOps (ej. `large` vs `small` = 6.5× costo) trackeando el dollar cost per query con W&B.
- **Preferir NLP clásica determinista para tareas acotadas** — un clasificador fine-tuned o NER rule-based es predecible, rápido y barato vs un LLM probabilístico.
- **Versionar código y datos** — Git para código, **DVC** para datos; decidir storage (ej. Elasticsearch) y modo (real-time vs batch) por reproducibilidad y escalabilidad.
- **Diseñar para deep agents, no shallow** — usar hierarchical delegation + explicit planning (LangGraph) + persistent memory (vector DBs) en vez de depender solo del context window.
- **Adoptar el loop MLOps/AgentOps** — evaluar, mejorar y redesplegar continuamente; instrumentar observabilidad (LangSmith, W&B) y estandarizar interoperabilidad (MCP, A2A) desde el diseño.

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[02 - Diving Deep into Large Language Models]] — capítulo siguiente (profundiza los LLMs, las técnicas de reducción de costo, context engineering y la arquitectura híbrida que este capítulo introduce).
- [[03 - Introduction to Haystack by deepset]] — el framework de la tool layer (Haystack 2.0, DG, components/pipelines) anticipado aquí.
- [[tool layer vs orchestration layer]] — el patrón arquitectónico central del libro, introducido en este capítulo.
- [[data mesh]] · [[Model Context Protocol (MCP)]] · [[A2A]] — contexto arquitectónico e interoperabilidad.
- [[RAG]] · [[BM25]] · [[Embeddings]] · [[Transformer]] · [[BERT]] — el stack técnico de retrieval y modelos.
- [[Haystack 2.0]] · [[Hayhooks]] · [[LangGraph]] · [[ReAct]] — los frameworks de tool y orchestration layer.
- [[Ragas]] · [[Weights and Biases]] · [[FinOps]] · [[MLOps]] · [[AgentOps]] · [[LLMOps]] — evaluación, observabilidad y operación.
- **agentic reliability crisis** · **shallow vs deep agent** · **excessive agency** — conceptos clave del capítulo; candidatos a nota propia.
