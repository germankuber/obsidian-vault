---
title: 04 - Bringing Components Together - Haystack Pipelines for Different Use Cases
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 4
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Bringing Components Together - Haystack Pipelines for Different Use Cases
  - Cap 4 - Haystack Pipelines
updated: 2026-06-12
---

# 04 - Bringing Components Together - Haystack Pipelines for Different Use Cases

> [!info] Capítulo 4 · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> El capítulo hands-on que ensambla los building blocks del [[03 - Introduction to Haystack by deepset|Cap 3]] en pipelines funcionales. Recorre el diseño con la filosofía de [[Haystack 2.0]] (directed multigraphs, branching con routers, `.draw()`), construye los tres pipelines fundamentales — **indexing** multi-fuente, **naive RAG** y **hybrid RAG con reranking** — los empaqueta como [[SuperComponent|SuperComponents]], extiende a **multimodal RAG** (imagen/texto/audio con CLIP y vision LLMs) y termina con **parallelization y async pipelines** para reducir latencia. Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · anterior [[03 - Introduction to Haystack by deepset]] · siguiente [[05 - Haystack Pipeline Development with Custom Components]].

## Resumen

Tras introducir [[Haystack 2.0]] conceptualmente en el [[03 - Introduction to Haystack by deepset|Cap 3]], este capítulo pasa a la **implementación concreta**: cómo conectar components a través de pipelines para casos de uso comunes y especializados. Cubre cinco bloques. Primero, **diseñar un pipeline con la filosofía de Haystack** — su arquitectura de **directed multigraphs** habilita flujos simultáneos, loops y branching; los 6 pasos para construir un pipeline (select/init → `Pipeline()` → `add_component` → `connect` → `run` → `draw`); el **branching con routers** (conditional, metadata, text-language); el data flow controlado; y la validación automática al conectar.

Segundo, **construir los tres pipelines fundamentales de RAG**: un **indexing pipeline** versátil que ingiere múltiples fuentes y formatos (web, TXT, PDF, CSV) usando `FileTypeRouter` para branching, con dos ramas (unstructured y structured) que se unen, embeddan y escriben al `DocumentStore`; un **naive RAG** (vectorize query → retrieve → augment & generate) como baseline "glass box"; y un **hybrid RAG con reranking** que corre dense + sparse ([[BM25]]) en paralelo, fusiona con `DocumentJoiner` y rerankea con un [[cross-encoder]] (`TransformersSimilarityRanker`). Tercero, **simplificar el uso vía [[SuperComponent|SuperComponents]]**: dos métodos (wrappear una instancia, o definir una clase `@super_component` con parámetros configurables). Cuarto, **multimodal RAG**: cruzar el *modality gap* con joint embeddings ([[CLIP]]) o modality translation (vision LLMs que extraen texto), más el procesamiento de audio (Whisper). Quinto, **parallelization y async pipelines** (`AsyncPipeline`) para reducir la latency acumulada de operaciones I/O-bound.

El hilo: Haystack convierte cada caso de uso en una composición de components sobre un grafo explícito; la flexibilidad (branching, multimodal, async) es nativa, y la transparencia (`.draw()`, validación automática) hace el sistema debuggable y production-ready. Los Caps 5 y 6 evaluarán cuantitativamente naive vs hybrid con [[Ragas]].

## Diseñar un pipeline con la filosofía de Haystack

### El foco en la flexibilidad

[[Haystack 2.0]] introduce una flexibilidad que va más allá de los pipelines tradicionales de query e indexing. Su arquitectura, construida como **directed multigraphs**, permite un array diverso de conexiones entre components — **flujos simultáneos, loops y branching** — sin las limitaciones de los data flows lineales. Esto habilita adaptar los pipelines a una variedad de casos: data cleaning, processing, indexing, [[RAG]], pipeline branching y serialization.

### Los 6 pasos para diseñar tu pipeline

1. **Select and initialize components** — identificar los components apropiados (ej. un indexing pipeline necesita preprocessor, splitting, embedding y writing; un retriever pipeline con augmented generation necesita query embedder, retriever y prompt generator).
2. **Creating the pipeline object** — inicializar con la clase `Pipeline()`, el backbone del data flow.
3. **Adding components** — usar `.add_component(name, component)` para agregar components (aún no define el flujo, solo prepara el pipeline).
4. **Connecting components** — establecer conexiones con `.connect("producer_component.output_name", "consumer_component.input_name")`. Paso crucial que define el flujo y asegura que el output de un component alimente correctamente el input del siguiente; soporta data flows intrincados (branching y loops).
5. **Running the pipeline** — ejecutar con `.run({"component_1": {"mandatory_inputs": value}})`, especificando los inputs iniciales.
6. **Visualizing the pipeline** — generar un grafo Mermaid con `pipeline.draw(path="path/to/image.png")`.

> [!note] La funcionalidad del pipeline la determinan **los components elegidos y el orden** en que se conectan. El mismo workflow de 6 pasos se aplica a cualquier caso de uso.

### Leveraging branching

El **branching** permite manejar tipos de datos o requisitos de procesamiento diversos simultáneamente, dirigiendo datos específicos a components especializados. Los **loops** permiten procesamiento iterativo (error correction, data refinement). Los componentes clave para crear branches son los **routers**. Casos: procesar documentos de distintos idiomas o formatos de forma diferente. Hay tres tipos en los notebooks: **conditional router**, **metadata router**, **text language router**.

![[04-fig-4.1-conditional-router.png]]
*Figure 4.1 – Conditional router classifying prompts as factual, semantic, or complex*

> [!tip] El pipeline de la Figura 4.1 usa el component **`ConditionalRouter`** para inspeccionar cada query y elegir dinámicamente el mejor path de procesamiento. El router evalúa una serie de **condiciones Jinja2** que buscan patrones en el texto (ej. *when/who/what is the* → **factual**; *how does/compare/difference between* → **semantic**; la presencia de `and` → **complex**). Según la primera condición que matchea, la query se rutea a un output (factual/semantic/complex), cada uno conectado a un `PromptBuilder` dedicado que genera un estilo de prompt distinto (direct answer, comprehensive explanation, detailed analysis). Una ruta default final (`condition: {{ True }}`) asegura que toda query se maneje.

### Ensuring effective data flow

Los pipelines de Haystack enfatizan un **data flow controlado**: solo los components conectados intercambian datos. Esto optimiza la velocidad de procesamiento y simplifica el debugging aislando los data paths. La asunción core: cada component toma una **data structure específica como input** y retorna otra como output — por eso **el orden de conexión importa**.

> [!note] Ejemplo del flujo de tipos: los components que convierten texto de PDFs/websites/Markdown toman un archivo del tipo apropiado (ej. document path) como input y retornan **Haystack document objects**; esos document objects se usan como input de los components que vectorizan el contenido y lo guardan en un document store. En un retriever pipeline, Haystack asume que el input es una pregunta (string) y/o una lista de floats (la pregunta en formato vector), y que el output es el documento relevante.

### Visualization y validation

Visualizar el pipeline con **grafos Mermaid** (diagramas text-based para flowcharts, sequence/class/state diagrams) da claridad sobre la estructura y el data flow. En Haystack la **validación ocurre automáticamente al conectar components**, chequeando compatibilidad y configuración. El método `.draw()` crea el grafo Mermaid del data flow.

![[04-fig-4.2-audio-to-vectors-pipeline.png]]
*Figure 4.2 – Mermaid graph for a Haystack pipeline that transforms audio into vectors*

La Figura 4.2 muestra un indexing pipeline de audio (toma audio y guarda vectores de su contenido). Sus components:

- **`RemoteWhisperTranscriber`** — usa el entry point de OpenAI Whisper para transformar audio en texto.
- **`DocumentCleaner`** — remueve caracteres raros (non-standard/special/unexpected) del texto transcrito.
- **`DocumentSplitter`** — splitea el texto en chunks para vectorización.
- **`SentenceTransformerDocumentEmbedder`** — transforma chunks en representación numérica reteniendo el significado semántico.
- **`DocumentWriter`** — escribe los embeddings en un document store.

### Integraciones de terceros

Haystack desarrolló un framework para conectar (e incluso desarrollar) **integrations**: packages, LLM y embedding providers, frameworks de evaluación y observabilidad. La lista está en `haystack.deepset.ai/integrations`.

> [!note] Los pilares de Haystack al diseñar pipelines LLM-powered: **flexibilidad** (que el pipeline resuelva tu problema), **effective data flow** (transformar y mover datos entre components), **branching y loops** (procesar e iterar sobre documentos), **serialization** (guardar y compartir pipelines más allá de scripts Python) e **integrations** (modelos, vector DBs y tecnologías de punta).

## Construir pipelines con Haystack: indexing, naive RAG y hybrid RAG

En el corazón de todo sistema RAG hay dos workflows distintos pero co-dependientes: un **indexing pipeline offline** que prepara la knowledge base, y un **query pipeline online** que la usa para responder en tiempo real.

### Indexing pipeline: preparar tu knowledge base

El indexing pipeline es un proceso offline crítico: toma datos no/semi-estructurados de varias fuentes, los convierte a un formato estandarizado y los carga en el `DocumentStore`. **La calidad de los datos ingeridos impacta directamente la calidad del retrieval y de la respuesta final.** Se construye uno que maneja múltiples fuentes simultáneamente (web pages, TXT, PDF, CSV) usando **`FileTypeRouter`** para dirigir cada tipo al converter apropiado.

![[04-fig-4.3-indexing-pipeline.png]]
*Figure 4.3 – Indexing pipeline with structured and unstructured text*

La secuencia de operaciones: (1) **Fetch and convert** (ingerir de web URLs, TXT, PDFs, CSVs → Haystack document objects); (2) **Categorize** (split en dos streams: structured y unstructured); (3) **Preprocess** (limpiar texto, espacios, caracteres especiales); (4) **Join** (mergear los streams de los distintos converters en una lista); (5) **Preprocess** (limpiar y splitear en chunks semánticamente coherentes); (6) **Embed and write** (generar embeddings y escribir al `DocumentStore`).

#### Stage 1 — Gathering and sorting (FileTypeRouter)

Las raw materials (files y web pages) llegan al component más importante de este workflow: **`FileTypeRouter`**, que asigna a qué output socket va cada dato, con cuatro streams: `text/plain`, `application/pdf`, `text/html` (web scrapeada) y `text/csv`. El router mira cada archivo y lo "sortea" al lane correcto — esto es **branching**, parte clave del diseño flexible de Haystack:

- `haystack_intro.txt` → output `text/plain`
- `sample.pdf` → output `application/pdf`
- `llm_models.csv` → output `text/csv`

#### Stage 2 — Processing (las dos ramas especializadas)

- **Branch 1, unstructured data (Web, TXT, PDF)** — para *blobs* de texto: (1) **Fetch and convert** (`LinkContentFetcher` agarra la web page; `HTMLConverter`, `TextConverter`, `PDFConverter` convierten a Haystack document objects); (2) **Join** (`unstructured_doc_joiner` reúne los docs en una lista); (3) **Preprocess** (`text_cleaner` ordena el texto; `text_splitter` corta en chunks de **150 words**).
- **Branch 2, structured data (CSV)** — para tabular data: **Convert** (`csv_converter` lee el CSV entero en un documento); **Preprocess** (`csv_cleaner` remueve filas/columnas vacías —ej. triple commas—; `csv_splitter` con `split_mode="row-wise"` convierte **cada fila en un documento separado**, ej. `"Company: OpenAI, Model: GPT-4..."`).

> [!warning] **Defensas para datos del mundo real (production-ready).** Tres capas: (1) **`FileTypeRouter`** actúa como safety valve para formatos desconocidos (ej. `.png`), diviértiéndolos a una ruta `unclassified` no conectada (ignorándolos); (2) **`LinkContentFetcher`** es estricto por default y lanza excepciones en URLs rotas (puede crashear el pipeline) → configurarlo con `raise_on_failure=False` para saltear bad links; (3) archivos vacíos crashean al `DocumentSplitter` → se introduce el **`DocumentSanitizer`**, un custom quality gate que inspecciona cada documento y descarta los de contenido vacío. Juntas: routing (type safety) + error skipping (access resilience) + sanitization (content quality).

#### Stage 3 — Unifying, indexing y shelving

(1) **Unifying** — `final_doc_joiner` es el re-merge point: espera y reúne todos los docs procesados (chunks de Branch 1 + filas CSV de Branch 2) en una lista única. (2) **Indexing** — `doc_embedder` lee cada chunk (texto o fila CSV) y lo convierte en vector. (3) **Shelving** — el writer guarda los documentos indexados (con sus embeddings) en el `InMemoryDocumentStore`. El texto de un blog, un PDF y las filas de un CSV quedan almacenados lado a lado, listos para retrieval por significado.

### Naive RAG — un sistema de Q&A fundacional

El **naive RAG** es la implementación más directa: un proceso lineal de dos pasos — **retrieve** documentos relevantes a la query, **generate** una respuesta basada en ellos. Es el baseline esencial.

> [!note] **Arquitectura "glass box".** La insistencia de Haystack en conexiones explícitas (`pipe.connect()`) crea una arquitectura caja de cristal: cada paso es transparente y trazable. El flujo —del query embedding, por el retriever, al prompt builder, al generator— está explícitamente definido por el developer. Invaluable para debugging.

La secuencia: (1) **Vectorize query** (embedding model sobre la pregunta); (2) **Retrieve relevant context** (la pregunta embebida recupera info del `DocumentStore`); (3) **Augment** (un prompt generator instruye al LLM a responder usando la query + el contexto recuperado).

- **Stage 1 — Vectorize query** — la pregunta (ej. *"What is Haystack 2.0?"*) se envía a `SentenceTransformersTextEmbedder`; el `text_embedder` aplica **exactamente el mismo embedding model usado en indexing**; output: un "query vector".
- **Stage 2 — Retrieve relevant context** — el query vector se alimenta a `InMemoryEmbeddingRetriever`; el `retriever` lo compara contra todos los vectores del `DocumentStore` y encuentra los `top_k=3` documentos de significado más similar; output: una lista de document objects (chunks de web, texto o filas CSV, lo más relevante).
- **Stage 3 — Augment and generate** — el `PromptBuilder` recibe la pregunta original + la lista de documentos, y "aumenta" la pregunta insertando el contexto en un prompt mayor con este template:
  ```text
  Given the following information...
  Context:
  [Content from Document 1]
  [Content from Document 2]
  ...
  Question: [The original user question]
  Answer:
  ```
  El prompt aumentado se envía al LLM (`OpenAIGenerator`), que usa **solo** ese contexto para responder en lenguaje natural.

> [!warning] **La debilidad del naive RAG: su total dependencia de la similitud semántica.** Puede fallar con queries que dependen de keywords específicos, acrónimos o product codes mal representados en el embedding space. Ej.: una búsqueda de un error code puede no recuperar el único documento que contiene ese string exacto si el contexto semántico general no matchea. Esto motiva el hybrid retrieval.

### Hybrid RAG con reranking: lo mejor de ambos mundos

El **hybrid retrieval** combina dos paradigmas de búsqueda:

- **Sparse (lexical) retrieval** (ej. [[BM25]]) — excele en keyword matching (nombres, jerga, código presentes literalmente). Debilidad: el vocabulary mismatch problem (no entiende sinónimos ni relaciones conceptuales).
- **Dense (semantic) retrieval** (vector embeddings) — excele en significado, intención y conceptos; encuentra docs sin keywords compartidos. Debilidad: a veces pasa por alto términos literales precisos.

Se agregan dos retrievers (uno por método), se envía la query a ambos en paralelo, se mergean con `DocumentJoiner`, y se añade un **reranker** ([[cross-encoder]], más caro pero más preciso que los bi-encoders del retrieval inicial) que toma la query y cada documento candidato como **par**, permitiendo un análisis más profundo y context-aware. Ubicado tras la fusión, re-ordena la lista para pasar solo los más relevantes al LLM.

La estructura: (1) **Parallel retrieval** (query a `InMemoryEmbeddingRetriever` dense + `InMemoryBM25Retriever` sparse); (2) **Fusion** (resultados a `DocumentJoiner`); (3) **Reranking** (`TransformersSimilarityRanker` re-scorea y re-ordena); (4) **Generation** (`PromptBuilder` + `OpenAIChatGenerator` producen la respuesta).

- **Stage 1 — Parallel retrieval (los dos expertos)** — la pregunta se envía por dos paths simultáneos. **Path 1 (dense/semantic)**: `hybrid_question` → `SentenceTransformersTextEmbedder` → query vector → `embedding_retriever` encuentra sus `top_k=3` docs semánticamente similares. **Path 2 (sparse/keyword)**: el texto crudo de la pregunta va directo a `InMemoryBM25Retriever`; el `bm25_retriever` ignora el significado y encuentra sus `top_k=3` docs por matching de palabras exactas (ej. `Haystack` o `2.0`).
- **Stage 2 — Fusion** — las dos listas (hasta 6 docs en total) se alimentan a `DocumentJoiner`, que las mergea en una lista única de candidatos.
- **Stage 3 — Reranking (el "head librarian")** — la lista mergeada + el texto de la pregunta van a `TransformersSimilarityRanker` (modelo más caro pero más preciso) que hace un análisis context-aware mirando query + cada documento como par, re-scorea y re-ordena; output: los `top_k=3` documentos más relevantes, lo mejor de ambos métodos.
- **Stage 4 — Augment and generate** — igual que el naive RAG pero con los documentos rerankeados de mayor calidad: `PromptBuilder` inserta el contexto y `OpenAIGenerator` genera la respuesta final.

> [!tip] La metáfora del libro: el "semantic librarian" (dense) trabaja junto a un "keyword expert" ([[BM25]]), y un "head librarian" (el reranker) hace el análisis final profundo de sus resultados combinados. Haystack hace que correr estas ramas paralelas sea una capacidad nativa. En los **Caps 5 y 6** se evalúan naive vs hybrid con knowledge graphs, synthetic data y [[Ragas]].

## Simplificar el uso de pipelines con SuperComponents

Un [[SuperComponent]] (introducido en el [[03 - Introduction to Haystack by deepset|Cap 3]]) permite tratar un pipeline entero como un único component, simplificando estructuras complejas de input/output. Los pipelines `naive_rag_pipeline` y `hybrid_rag_pipeline` se pueden refactorizar por dos métodos.

### Método 1 — Wrappear una instancia de pipeline existente

Ideal cuando ya definiste el objeto pipeline y querés abstraerlo rápido sin modificar la clase subyacente. Se pasa el pipeline a la clase `SuperComponent` con `input_mapping` y `output_mapping`:

```python
from haystack import SuperComponent
# Naive RAG
naive_rag_sc = SuperComponent(
    pipeline=naive_rag_pipeline,
    input_mapping={
        "query": ["text_embedder.text", "prompt_builder.question"]
    },
    output_mapping={
        "llm.replies": "replies",
        "retriever.documents": "documents"
    }
)
# Hybrid RAG
hybrid_rag_sc = SuperComponent(
    pipeline=hybrid_rag_pipeline,
    input_mapping={
        "query": ["text_embedder.text", "bm25_retriever.query", "ranker.query", "prompt_builder.question"]
    },
    output_mapping={
        "llm.replies": "replies",
        "ranker.documents": "documents"
    }
)
```

La simplificación es dramática. El input original requería mapear cada component:

```python
naive_rag_pipeline.run({
    "text_embedder": {"text": question},
    "prompt_builder": {"question": question}
})
hybrid_rag_pipeline.run({
    "text_embedder": {"text": question},
    "bm25_retriever": {"query": question},
    "ranker": {"query": question},
    "prompt_builder": {"question": question}
})
```

Pero una vez wrappeado como SuperComponent, se ejecuta con una sola variable:

```python
naive_rag_sc.run(query=question)
hybrid_rag_sc.run(query=question)
```

> [!warning] Este método es rápido pero **limitado**: si querés cambiar el generator o el document store, tenés que modificar el pipeline manualmente. Para SuperComponents que toman variables como input, se usa el Método 2.

### Método 2 — Definir una clase SuperComponent

Definir una clase decorada con **`@super_component`** ofrece más poder y modularidad: trata el pipeline generator como un **blueprint customizable**, exponiendo parámetros de inicialización en el `__init__`. Permite instanciar la misma arquitectura RAG con configuraciones distintas — ej. la misma clase `NaiveRAGSuperComponent` para un pipeline con Elasticsearch y otro con in-memory document store, o swappear embedding models (ej. de `text-embedding-3-small` a un modelo local) pasando distintos argumentos.

El flujo de ejecución de la clase: (1) **definir la clase y su `__init__`** — aceptar argumentos como embedding model, LLM model y `top-k` (en vez de hardcodear); chequea credenciales (API keys) antes de construir el pipeline; (2) **`_build_pipeline`** — inicializar los components específicos (text embedder, retriever sobre Elasticsearch, prompt builder, generator —se puede elegir otro, ej. `OllamaGenerator` para un modelo open); (3) crear el `Pipeline()`, agregar components y conectarlos con `pipeline.connect()`; (4) **abstraer con input/output mappings** — un input mapping rutea una única query externa al embedder y al prompt builder; un output mapping expone solo lo que importa (la reply del LLM + los documentos recuperados).

> [!tip] Con el decorador `@super_component` y una clase dedicada, el pipeline pasa de un script estático a un **componente de software flexible** que encaja en aplicaciones mayores. En el [[06 - Building Reproducible and Production-Ready RAG Systems|Cap 6]] se evalúan los SuperComponents naive y hybrid con distintos embedding models.

## Multimodal pipelines: imagen, texto y audio

El conocimiento del mundo real suele estar en formatos no-textuales (imágenes, diagramas, audio). El **multimodal RAG** debe cruzar el **modality gap**: no se puede comparar matemáticamente una query de texto con un archivo binario de imagen directamente. Dos estrategias primarias: **joint embeddings** (mapear distintas modalidades a un vector space compartido) y **modality translation** (convertir señales visuales/audio a texto antes de indexar).

### Estrategia 1 — Joint embeddings con CLIP

Usa modelos como **[[CLIP]]** (Contrastive Language-Image Pretraining), entrenados para proyectar imágenes y texto al **mismo latent vector space** — permite buscar imágenes con lenguaje natural sin labeling explícito. Con un component document image embedder de `sentence-transformers` y un modelo como `sentence-transformers/clip-ViT-L-14`, se convierten imágenes locales en vectores geométricamente cercanos a los de sus descripciones textuales (crear text embedding de una query, image embeddings de muestras, cosine similarity, ver qué imagen matchea mejor).

El indexing pipeline para streams mixtos usa `FileTypeRouter` por MIME type:

- **Text branch** — PDFs → `PyPDFToDocument` converter + text embedder.
- **Image branch** — imágenes → `ImageFileToDocument` converter + image embedder.
- **The glue** — un custom component `ImagePathFixer`: los converters estándar pueden guardar solo filenames, pero los embedding components requieren paths absolutos; este resuelve los paths antes de vectorizar.

Resultado: un `InMemoryDocumentStore` unificado donde imágenes y texto coexisten, buscables con un único query embedding model.

### Estrategia 2 — LLM content extraction y vision RAG

[[CLIP]] es bueno para similitud visual pero no lee texto detallado ni analiza charts complejos en una imagen. Para eso, **vision LLMs** (ej. GPT-4o). Se reemplaza el image embedder por **`LLMDocumentContentExtractor`**, que envía la imagen a un vision LLM para generar una descripción textual detallada; luego se embedda esa descripción con un text model de alto rendimiento (ej. `mixedbread-ai/mxbai-embed-large-v1`). Esto **transmuta imágenes en texto**, habilitando retrieval de texto estándar de alta precisión.

Los pasos del indexing: **Text branch** (PDFs → `PyPDFToDocument`, sin embedding aún); **Image branch** (imágenes → vision model que extrae info, sin embedding aún); **The glue** (`DocumentJoiner` une los docs procesados y aplica el mismo embedding text-based sobre los chunks).

> [!note] **Search by proxy, answer by source** — el patrón state-of-the-art del multimodal RAG. (1) **Retrieve** — se busca en el document store usando la **descripción textual** generada en indexing; (2) **Generate** — al encontrar el documento relevante, el pipeline pasa la **imagen binaria original** (referenciada en la metadata) al generator. El vision LLM mira la foto original y hace **fresh visual reasoning** sobre el material fuente, no se apoya en la descripción proxy. Máxima fidelidad: la descripción es solo un index key robusto, mientras la generación retiene la resolución completa de la modalidad original.

### Processing audio data

A diferencia de las imágenes espaciales, el audio debe **linealizarse** (transcribirse a texto) para ser compatible con vector search. El pipeline usa **`RemoteWhisperTranscriber`** para convertir audio (`.wav`, `.mp3`) en documentos de texto.

> [!tip] Paso crítico: la **segmentación** con `DocumentSplitter` configurado con `split_by="sentence"` (ver Figura 4.2). Rompe los transcripts largos en chunks semánticamente coherentes (ej. 10 oraciones), permitiendo al retrieval **pinpointear momentos específicos** de una conversación en vez de recuperar la grabación entera de una hora.

## Parallelization y asynchronous pipelines

La estructura sola no garantiza performance. Las apps RAG del mundo real **rara vez están limitadas por la velocidad del CPU**: están bound por operaciones **I/O** (input/output) — pasan la mayoría del tiempo *esperando* (que un embedding API retorne un vector, que una DB ejecute una búsqueda, que un LLM genere tokens).

> [!warning] En un pipeline **síncrono** estándar, los wait times son **acumulativos**: si un hybrid search requiere dense retrieval (0.5 s) + sparse retrieval (0.5 s), el usuario espera **1.0 s**. Haystack introduce **parallelization** y **asynchronous pipelines** para desacoplar operaciones independientes y ejecutarlas concurrentemente, reduciendo drásticamente la latencia total.

Se instancia la clase **`AsyncPipeline`** en vez de `Pipeline` (API similar para agregar/conectar components, pero execution graph fundamentalmente distinto). El `AsyncPipeline` analiza el directed multigraph para identificar branches independientes:

- En un **linear pipeline** (`A → B → C`) no hay room para paralelización (B depende de A).
- En un **branching pipeline** (`A → B` y `A → C`), B y C comparten dependencia de A pero son independientes entre sí. `AsyncPipeline` detecta esta topología y agenda B y C como `asyncio.Task` concurrentes.

Métodos especializados:

- **`run_async()`** — ejecuta el pipeline asincrónicamente; non-blocking, ideal para web servers (ej. FastAPI) o manejar múltiples queries concurrentes.
- **`run_async_generator()`** — el método de **streaming**; yield-ea outputs parciales a medida que los components completan (útil para debugging y user feedback: se ven los retrievers terminar antes de que el LLM empiece a generar).

> [!tip] Dominar la paralelización asegura pipelines no solo inteligentes sino **responsivos** para la interacción en tiempo real.

## Citas

> "At the heart of any effective RAG system are two distinct yet co-dependent workflows: an offline indexing pipeline responsible for preparing the knowledge base, and an online query pipeline that leverages this prepared data to answer user questions in real time."
> "The framework's insistence on explicit connections (pipe.connect()) creates a 'glass box' architecture. Every step in a Haystack pipeline is transparent and traceable."
> "Real-world RAG applications are rarely limited by CPU speed; they are bound by input/output (I/O) operations."

## Para aplicar

- **Setup del capítulo** — abrir la carpeta `ch4` en una ventana standalone de VS Code:
  ```bash
  cd ch4/
  uv sync
  source .venv/bin/activate
  ```
  Kernel Jupyter `rag-with-haystack-ch4` (path `.venv/bin/python`).
- **Diseñar pipelines en 6 pasos** — select/init components → `Pipeline()` → `add_component(name, component)` → `connect("a.output", "b.input")` → `run({...})` → `draw(path=...)`. Verificar siempre el grafo Mermaid contra el diseño previsto.
- **Branching con routers** — usar `ConditionalRouter` (condiciones Jinja2), `MetadataRouter` o `TextLanguageRouter` para rutear queries/documentos a paths especializados; `FileTypeRouter` por MIME type en indexing.
- **Indexing production-ready** — manejar fuentes mixtas con `FileTypeRouter` (branching structured/unstructured); blindar con `raise_on_failure=False` en `LinkContentFetcher` y un `DocumentSanitizer` custom para descartar documentos vacíos; CSV con `csv_splitter` en `split_mode="row-wise"`.
- **RAG: empezar naive, graduar a hybrid** — naive (vectorize → retrieve → augment/generate) como baseline glass-box; hybrid (dense + [[BM25]] en paralelo → `DocumentJoiner` → `TransformersSimilarityRanker` reranker → generate) para production. Usar el **mismo embedding model** en indexing y query.
- **Empaquetar como [[SuperComponent]]** — Método 1 (wrappear instancia con `SuperComponent(pipeline, input_mapping, output_mapping)`) para abstraer rápido; Método 2 (clase `@super_component` con params en `__init__`) para reusabilidad y configuración (swappear document store/embedding/generator).
- **Multimodal RAG** — joint embeddings con [[CLIP]] (`clip-ViT-L-14`) para búsqueda visual por texto; o vision LLM + `LLMDocumentContentExtractor` con patrón *search by proxy, answer by source* para máxima fidelidad; audio vía `RemoteWhisperTranscriber` + `DocumentSplitter(split_by="sentence")`.
- **Optimizar latencia con async** — usar `AsyncPipeline` para correr branches independientes concurrentemente; `run_async()` para web servers (FastAPI), `run_async_generator()` para streaming de outputs parciales.

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[03 - Introduction to Haystack by deepset]] — capítulo anterior (los building blocks: components, pipelines, SuperComponents, routers que este capítulo ensambla en pipelines reales).
- [[05 - Haystack Pipeline Development with Custom Components]] — capítulo siguiente (custom components como el `DocumentSanitizer`/`ImagePathFixer` usados aquí; genera knowledge graph + synthetic data).
- [[06 - Building Reproducible and Production-Ready RAG Systems]] — evalúa cuantitativamente los SuperComponents naive vs hybrid de este capítulo con [[Ragas]] y distintos embedding models.
- [[07 - Deploying Haystack-Based Applications]] — despliega estos pipelines como microservicios (Hayhooks); el async/FastAPI anticipado aquí.
- [[Haystack 2.0]] · [[SuperComponent]] · [[Custom Components (Haystack)]] — los building blocks.
- [[RAG]] · [[Hybrid Search]] · [[BM25]] · [[cross-encoder]] · [[Embeddings]] · [[Reciprocal Rank Fusion (RRF)]] — el stack de retrieval implementado aquí.
- [[CLIP]] — joint embeddings multimodales; candidato a nota propia.
- [[Ragas]] — evaluación de naive vs hybrid (Caps 5-6).
- **FileTypeRouter** · **ConditionalRouter** · **DocumentJoiner** · **TransformersSimilarityRanker** · **AsyncPipeline** · **modality gap** · **search by proxy, answer by source** — componentes y conceptos clave del capítulo; candidatos a nota propia.
