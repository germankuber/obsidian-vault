---
title: 05 - Haystack Pipeline Development with Custom Components
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 5
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Haystack Pipeline Development with Custom Components
  - Cap 5 - Custom Components
updated: 2026-06-12
---

# 05 - Haystack Pipeline Development with Custom Components

> [!info] Capítulo 5 · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> El capítulo que marca la transición de **usuario** a **arquitecto** de [[Haystack 2.0]]: cómo definir **custom components** (decorador `@component`, `__init__`, `run()`, `@component.output_types`), gestionar recursos pesados con el método de ciclo de vida **`warm_up()`**, y aplicarlos a un caso avanzado — generar un **knowledge graph** y un **dataset sintético de ground-truth** (con [[Ragas]] y query synthesizers single/multi-hop) para evaluar [[RAG]]. Cierra con la disciplina de **testing** (mocking, ciclo de vida, edge cases). Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · anterior [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]] · siguiente [[06 - Building Reproducible and Production-Ready RAG Systems]].

## Resumen

Tras ensamblar components predefinidos en el [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases|Cap 4]], este capítulo marca la transición crítica de ser **usuario** del framework a ser **arquitecto capaz de extenderlo** — construir custom components e integrarlos en pipelines sofisticados, llevando el desarrollo de RAG "de un arte a una práctica de ingeniería madura". Cubre tres bloques. Primero, **cómo definir custom components**: los cuatro requisitos (el decorador `@component`, el método `__init__` para configuración, el método `run()` que debe retornar un diccionario, y el decorador `@component.output_types`), el modelo de **typed sockets** (data contract explícito vs el dictionary-passing implícito de 1.x), y el método de ciclo de vida **`warm_up()`** para cargar recursos pesados (modelos, conexiones) exactamente una vez antes del primer `run()`.

Segundo, un **caso avanzado**: construir custom components para crear un **knowledge graph** desde documentos (PDF o web scrapeada) y usarlo para generar **pares pregunta-respuesta sintéticos** que sirvan de ground-truth para evaluar RAG. Se justifica por qué el knowledge graph como paso intermedio (los RAG estándar fallan en preguntas **multi-hop**; los grafos permiten reasoning sobre relaciones — *graph RAG*). Se presenta el framework **[[Ragas]]** (sus cuatro métricas: faithfulness, response relevancy, context precision, context recall) y su `TestsetGenerator` con tres synthesizers (single-hop, multi-hop specific, multi-hop abstract). Se implementan los components `KnowledgeGraphGenerator` y `SyntheticTestGenerator` (con el patrón *preferred path with fallback*), un component "bridge" `DocumentToLangChainConverter`, y se ensamblan en pipelines para PDF, web, y la unificación de múltiples fuentes con branching.

Tercero, **testing y debugging**: los cinco principios para validar custom components (mocking de dependencias externas, validar el ciclo de vida y estado, testear configuración, verificar componentes "bridge" y data contracts, y testear edge cases con graceful failure). El hilo: extender Haystack con la misma rigurosidad de ingeniería de software (interfaces explícitas, separación de concerns, tests) convierte el RAG en una disciplina debuggable y production-ready. El Cap 6 usará el dataset generado aquí para evaluar cuantitativamente el RAG con Ragas y observabilidad ([[Weights and Biases|W&B]]).

## Cómo definir custom components en Haystack

Los **custom components** dan la flexibilidad de implementar funcionalidad a medida — filtrar resultados, interactuar con software externo, o cualquier tarea que los components estándar no cubren. Con [[Haystack 2.0]], crearlos, reusarlos y compartirlos es más streamlined.

### Los cuatro requisitos clave

- **El decorador `@component`** — el mecanismo primario que registra una clase Python en el framework, señalando que puede instanciarse y usarse en un pipeline. Hace la clase "visible" al pipeline engine, que gestiona su ciclo de vida y orquesta su ejecución.
- **El método `__init__`** — el constructor del component; su rol es **configuración y dependency injection**. Acá se pasan y guardan como instance attributes los parámetros estáticos que no cambian entre runs (API keys, model names, conexiones a servicios externos). Hacerlo configurable vía `__init__` es clave para su reusabilidad.
- **El método `run()`** — todo component debe tenerlo; es el entry point de la lógica y el corazón de su funcionalidad. **Regla crítica: `run()` siempre debe retornar un diccionario Python** — las keys corresponden a los output sockets y los valores son los datos que pasan downstream.
- **El decorador `@component.output_types`** — se agrega sobre el `run()`; especifica los tipos y nombres de output del component. Los nombres y tipos declarados deben alinearse con el diccionario que retorna `run()`.

> [!note] **Typed sockets: el data contract explícito.** El avance arquitectónico de Haystack 2.0 es pasar del dictionary-passing implícito de 1.x a un modelo type-safe basado en "sockets". El contrato de datos del component (qué acepta y qué produce) se define directa y explícitamente en el código. **Inputs**: se definen declarándolos como argumentos tipados en la firma del `run()` (el engine valida automáticamente que el dato matchee el type hint, cazando errores temprano). **Outputs**: se definen con el decorador `@component.output_types()` sobre el `run()` (cada key debe matchear una key del diccionario retornado). El método `draw()` —que genera el grafo visual— es la manifestación última de esta filosofía: convierte código abstracto en un diagrama concreto y verificable.

### Component breakdown: Prefixer

El `Prefixer` es un component custom simple que demuestra patrones de diseño robustos: **appendea un prefijo a cada documento** en una lista de objetos `Document`. La lógica dentro del `run()` itera cada `Document` y hace dos acciones clave:

- **Immutable processing** — crea un **nuevo** objeto `Document` (best practice: evita modificar el dato original in-place, que causa side effects inesperados en pipelines complejos).
- **Metadata preservation** — copia explícitamente la metadata del documento original al nuevo (para no perder source, ID, etc. durante la transformación).

El contenido del nuevo documento se setea a `f"{prefix}{doc.content}"` y se agrega a una lista `modified_documents`. Antes de integrarlo, se valida standalone:

```python
prefixer_instance = Prefixer()
result = prefixer_instance.run(
    documents=documents, prefix="This is a prefixed document: ")
```

Se verifica que el result sea un diccionario `{'documents': [...]}` con cada contenido prefijado y la metadata intacta. Luego se integra en un pipeline que alimenta el output del `Prefixer` directo a un `DocumentWriter`:

```python
# Create an in-memory document store
document_store = InMemoryDocumentStore()
# Initialize our custom component
prefixer = Prefixer()
# Create a document writer component
writer = DocumentWriter(document_store=document_store)
# Create the pipeline
pipeline = Pipeline()
```

Se agregan y conectan (el output socket `documents` del prefixer al input socket `documents` del writer):

```python
pipeline.add_component("prefixer", prefixer)
pipeline.add_component("writer", writer)
pipeline.connect("prefixer.documents", "writer.documents")
```

Y se ejecuta, con los inputs estructurados para alimentar al `prefixer`:

```python
pipeline.run(
    {"prefixer": {
        "documents": documents,
        "prefix": "This is a prefixed document: "}
    }
)
```

Finalmente se verifica consultando el `document_store` con `.filter_documents()`, confirmando que los documentos almacenados son los procesados por el `Prefixer`.

### Gestionar estado y recursos pesados: el método warm_up()

> [!warning] **El dilema del "dónde cargo mis recursos".** Tareas de inicialización caras y de una sola vez (descargar un modelo de varios GB, abrir una conexión persistente a DB) no van bien ni en `__init__` ni en `run()`. Cargar en `__init__` hace el component **lento de inicializar** (solo crear la instancia dispara una descarga larga). Cargar en `run()` lo hace **extremadamente ineficiente** (el `run()` se llama por cada batch → recargarías el modelo entero de disco cada vez, agregando latencia masiva).

La solución es el método de ciclo de vida **`warm_up()`**: un hook especial que el pipeline engine llama **exactamente una vez por instancia de component**, justo antes de la primera invocación de su `run()`. Separa la lógica en tres etapas:

- **Configuration — `__init__(self, ...)`** — liviano y rápido; solo guarda parámetros de configuración (model name, API key). No carga recursos pesados.
- **Initialization — `warm_up(self)`** — acá ocurre el trabajo pesado: cargar el modelo, abrir la conexión, etc., una sola vez al arrancar el pipeline; guarda el recurso cargado en la instancia.
- **Processing — `run(self, ...)`** — la lógica principal; se llama repetidamente y es rápida (asume que `warm_up()` ya corrió y usa los recursos pre-cargados).

El ejemplo: un component `LocalEmbedderText` (adaptado a `LocalEmbedderDocs` para objetos `Document`) que carga un modelo `SentenceTransformer`:

```python
class LocalEmbedderDocs:
    def __init__(
        self, model_name:
        str = "sentence-transformers/all-MiniLM-L6-v2"
    ):
        self.model_name = model_name
        # The model is not loaded yet
        self.model: Optional = None
```

El `warm_up()` chequea si el modelo ya está cargado; si no (`self.model is None`), lo carga; si se llama una segunda vez, no hace nada (previene reload innecesario — **idempotencia**):

```python
def warm_up(self):
    """
    Loads the SentenceTransformer model.
    This is called only once
    before the first run.
    """
    if self.model is None:
        self.model = SentenceTransformer(self.model_name)
```

El `run()` hace una **safety check** (verifica que `self.model` no sea `None`, sino lanza un `RuntimeError`) y luego la **execution** (extrae el texto de cada `Document`, usa el modelo pre-cargado para generar embeddings, retorna el diccionario estándar).

> [!tip] **No llamás `warm_up()` vos mismo.** Cuando ejecutás `pipeline.run(...)`, Haystack inspecciona automáticamente todos los components y, para cada uno que tenga un `warm_up()`, lo llama **una vez** antes de que fluyan los datos. Así, para cuando el PDF se convirtió/limpió/spliteó, el `LocalEmbedderDocs` ya está "warmed up" y su modelo cargado en memoria. Esta separación de concerns es lo que hace a los pipelines de Haystack flexibles y performantes a la vez.

## Implementación avanzada: knowledge graph + synthetic data

Se construye una serie de custom components para crear un **knowledge graph** desde documentos y usarlo para generar pares pregunta-respuesta que evalúen un RAG.

> [!note] **Por qué un knowledge graph como paso intermedio.** Generar preguntas directo desde text chunks tiene limitaciones: los RAG estándar sobre chunks aislados **fallan con preguntas multi-hop** (que requieren conectar info dispersa en varios documentos; Su et al. 2020; Neo4j 2025). Los **knowledge graphs** guardan los datos como una red de **nodos** (entidades: personas, empresas, conceptos) y **edges** (relaciones), permitiendo traversals y reasoning complejos. Al representar *cómo* se conecta la info (no solo que existe), dan el scaffold para generar preguntas que de verdad testean el reasoning del RAG — el enfoque **graph RAG**, state-of-the-art para sistemas más inteligentes y explicables.

### El framework Ragas: fundamento para evaluar RAG

[[Ragas]] es una librería Python open source con un toolkit comprehensivo para evaluar pipelines RAG. Su filosofía: evaluar los dos componentes principales de un RAG — el **retriever** y el **generator** — tanto en aislamiento como como unidad. Su ventaja: evaluación de alta calidad con métricas LLM-based, muchas **sin necesidad de datasets ground-truth anotados por humanos**. Su evaluación se centra en cuatro dimensiones:

- **Faithfulness** — mide la consistencia factual de la respuesta generada contra el contexto recuperado (medida directa de hallucination del LLM).
- **Response relevancy** — cuán relevante es la respuesta generada a la pregunta del usuario.
- **Context precision** — el ratio "signal-to-noise" del contexto recuperado (si los chunks son relevantes a la query).
- **Context recall** — si el contexto recuperado contenía toda la info necesaria para responder.

> [!tip] La segunda capacidad de Ragas (la más crítica acá) es la **synthetic test data generation**: pares pregunta-respuesta grounded en la documentación, generados con distintos personas y estrategias — resuelve el bottleneck de crear manualmente preguntas complejas multi-hop.

El `TestsetGenerator` de Ragas: (1) **knowledge graph creation** (ingiere documentos y construye un grafo que mapea entidades y relaciones); (2) **query synthesis** (usa el grafo para alimentar query synthesizers — de ahí la columna `synthesizer_name` en los CSV generados):

- `SingleHopSpecificQuerySynthesizer` — preguntas directas, fact-based.
- `MultiHopSpecificQuerySynthesizer` — preguntas complejas que requieren conectar facts específicos de distintas partes del texto.
- `MultiHopAbstractQuerySynthesizer` — preguntas de reasoning de alto nivel que requieren sintetizar info entre múltiples conceptos.

Se mimetiza este diseño con dos custom components: `KnowledgeGraphGenerator` y `SyntheticTestGenerator`.

### Implementando el KnowledgeGraphGenerator

Transforma una lista plana de documentos en una red estructurada e interconectada usando Ragas. Ingiere los documentos como nodos iniciales y aplica **transforms** (con `apply_transforms`) que los actualizan a una web rica de datos mapeando entidades y conexiones:

```python
kg = KnowledgeGraph()
for doc in documents:
        kg.nodes.append(
            Node(
                type=NodeType.DOCUMENT,
                properties={
                    "page_content": doc.page_content,
                    "document_metadata": doc.metadata}))
default_transforms_config = default_transforms(
         documents=documents, llm=transformer_llm,
         embedding_model=embedding_model)
apply_transforms(kg, default_transforms_config)
```

> [!note] **Los dos modelos y sus roles.** El **LLM** es el reasoning engine: escanea el contenido de los nodos para extraer entidades (personas, conceptos, organizaciones) e identificar las relaciones que las conectan (construye los **edges**). El **embedding model** da la comprensión semántica para analizar el texto y medir similitud entre conceptos, asegurando conexiones contextualmente precisas. El resultado se puede guardar en JSON (con el component `KnowledgeGraphSaver`) o pasar como estructura de datos Python.

### Implementando el SyntheticTestGenerator

Transforma el knowledge graph en un dataset de evaluación diverso. Usa un `query_distribution` configurado para instanciar synthesizers específicos (definidos en un método `create_query_synthesizers`) que traversan los nodos y edges para generar preguntas single-hop y multi-hop.

> [!warning] **Patrón preferred path with fallback.** Un component resiliente no debe crashear si la generación por grafo falla o da vacío. La lógica se envuelve en un `try/except`: si el path del knowledge graph lanza una excepción, degrada elegantemente a generación document-based:
> ```python
> try:
>     testset = self._generate_from_knowledge_graph(knowledge_graph)
>     method = "knowledge_graph"
> except Exception as kg_error:
>     logger.warning(f"Knowledge graph generation failed: {kg_error}. Falling back to document-based generation.")
>     testset = self._generate_from_documents(documents)
>     method = "documents_fallback"
> ```

La lógica de cada path se encapsula en métodos separados para aislar fallos. La generación vía grafo usa la clase `TestsetGenerator` de Ragas:

```python
def _generate_from_knowledge_graph(self, knowledge_graph: KnowledgeGraph):
    """Generate tests using a knowledge graph."""
    try:
        generator = TestsetGenerator(
            llm=self.llm,
            embedding_model=self.embeddings,
            knowledge_graph=knowledge_graph
        )
```

El método fallback `_generate_from_documents` también usa `TestsetGenerator` pero parsea los documentos directo. El LLM y embedding models pueden ser cualquiera de los generators soportados por Haystack (los notebooks muestran OpenAI y Ollama).

### Assembling: knowledge graph + synthetic data desde PDFs

El pipeline se divide en cuatro stages: (1) **Ingestion and preprocessing**; (2) **Format bridging**; (3) **Knowledge graph creation**; (4) **Synthetic data generation**.

- **Stage 1 — Ingestion and preprocessing** — `PyPDFToDocument` (entry point; ingiere el PDF local → `List[HaystackDocument]`, content-agnostic); `DocumentCleaner` (remueve whitespace y líneas vacías que introducen ruido); `DocumentSplitter` configurado con `by="sentence"` y `split_length=5` (decisión crítica: chunks manejables y semánticamente coherentes para un grafo preciso).
- **Stage 2 — The bridge (`DocumentToLangChainConverter`)** — un component "bridge" custom: los components nativos de Haystack producen `List[HaystackDocument]`, pero **Ragas espera `List[LangChainDocument]`**. Su `run` itera los Haystack documents, copia `.content` y `.meta`, e instancia nuevos `LangChainDocument`.
- **Stage 3 — `KnowledgeGraphGenerator`** — `__init__` liviano (guarda LLM y flag `apply_transforms`, prepara los wrappers de Ragas); el `run` recibe documentos, los agrega como nodos aislados, y si `apply_transforms` es `True`, usa Ragas + el LLM para construir la inteligencia graph RAG (entidades, relaciones, edges); retorna el grafo terminado.
- **Stage 4 — `SyntheticTestGenerator`** — consume el grafo y genera el dataset final con el patrón preferred-path-with-fallback. El `__init__` guarda `testset_size`, `llm_model` y el `query_distribution`:
  ```python
  query_distribution=[("single_hop", 0.25),
      ("multi_hop_specific", 0.25),
      ("multi_hop_abstract", 0.5)]
  ```

Los tres tipos de query synthesizer:

- **`single_hop`** (25%) — queries factuales directas, respondibles desde una sola pieza de info. Ej.: *"Who is Christopher Ong in the context of ChatGPT research?"*
- **`multi_hop_specific`** (25%) — requieren conectar facts específicos de secciones distintas. Ej.: *"What are the key improvements in Haystack 2.0 compared to Haystack 1.0, particularly regarding the handling of loops and customizable components?"*
- **`multi_hop_abstract`** (50%) — las más complejas; reasoning amplio que sintetiza varios conceptos abstractos. Ej.: *"What are the primary functions of ChatGPT usage at work, and how does the quality of interactions reflect user satisfaction?"*

El `run` del `SyntheticTestGenerator` acepta inputs de dos upstream components (`KnowledgeGraphGenerator` y `DocumentToLangChainConverter`); usa el grafo (preferido, única forma de generar multi-hop confiable) o el fallback document-based (single-hop). Output: un `pd.DataFrame` que pasa al `TestDatasetSaver`.

### Integrar los custom components en un pipeline (PDF)

```python
# Built-in Haystack components
pdf_converter = PyPDFToDocument()
doc_cleaner = DocumentCleaner(...)
doc_splitter = DocumentSplitter(...)
# Our custom components from the .py files
doc_converter = DocumentToLangChainConverter()
kg_generator = KnowledgeGraphGenerator(apply_transforms=True)
test_generator = SyntheticTestGenerator(testset_size=10, ...)
test_saver = TestDatasetSaver("data_for_eval/...")
```

Se agregan y conectan, con la clave de diseño de que `test_generator` recibe inputs de **ambos** `doc_converter` (fallback) y `kg_generator` (preferido):

```python
pipeline = Pipeline()
pipeline.add_component("pdf_converter", pdf_converter)
pipeline.add_component("doc_cleaner", doc_cleaner)
pipeline.add_component("doc_splitter", doc_splitter)
pipeline.add_component("doc_converter", doc_converter)
pipeline.add_component("kg_generator", kg_generator)
pipeline.add_component("test_generator", test_generator)
pipeline.add_component("test_saver", test_saver)

pipeline.connect("pdf_converter.documents", "doc_cleaner.documents")
pipeline.connect("doc_cleaner.documents", "doc_splitter.documents")
pipeline.connect("doc_splitter.documents", "doc_converter.documents")
pipeline.connect("doc_converter.langchain_documents", "kg_generator.documents")
pipeline.connect("doc_converter.langchain_documents", "test_generator.documents")
pipeline.connect("kg_generator.knowledge_graph", "test_generator.knowledge_graph")
pipeline.connect("test_generator.testset", "test_saver.testset")
```

Y se ejecuta:

```python
pdf_sources = [Path("./data_for_indexing/howpeopleuseai.pdf")]
result = pipeline.run({"pdf_converter": {"sources": pdf_sources}})
```

### Tabla 5.1 — Sample synthetic data (PDF)

> Cada ejecución varía las queries por la naturaleza no-determinista del knowledge graph y la generación sintética.

| Question | Answer (Reference) | Strategy |
|---|---|---|
| "Who is Christopher Ong in the context of ChatGPT research?" | "Christopher Ong is one of the co-authors of the NBER Working Paper No. 34255 titled 'How People Use ChatGPT.' He is affiliated with Harvard University and OpenAI." | Single-hop-specific query synthesizer |
| "How does the usage of ChatGPT for Practical Guidance differ among users with varying education levels, particularly in relation to work-related messages and the types of requests made?" | "The usage... shows notable differences... 36% of Practical Guidance messages are requests for Tutoring or Teaching... In terms of work-related messages, 37% are sent by users with less than a bachelor's degree, compared to... 48% for users with some graduate education." | Multi-hop-specific query synthesizer |

*Table 5.1 – Sample synthetic data containing question, answer, and synthesizer*

> [!note] La primera es **single-hop** (respondible de una pieza). La segunda es **multi-hop**: requiere sintetizar tres conceptos (practical guidance usage, education levels, work-related messages) — exactamente el tipo de pregunta de alto valor que el knowledge graph está diseñado para producir.

### Assembling desde una web scrapeada

Acá brilla la modularidad de Haystack: para procesar una web en vez de un PDF, **solo se swappean los components de ingestion** — la lógica core (`DocumentCleaner`, `DocumentSplitter`, `DocumentToLangChainConverter`, `KnowledgeGraphGenerator`, `SyntheticTestGenerator`) queda idéntica:

- **Removido**: `PyPDFToDocument`.
- **Agregado**: `LinkContentFetcher` (recupera el contenido de la URL) + `HTMLToDocument` (convierte el HTML crudo a `Documents`).

> [!tip] La generación de knowledge graph es **content-agnostic**: funciona igual de bien sobre el HTML procesado de un blog que sobre el contenido de un PDF.

### Tabla 5.2 — Sample synthetic data (HTML)

| Question | Answer (Reference) | Strategy |
|---|---|---|
| "What does HyDE do in Haystack?" | "HyDE is an optimization technique that can be used in Haystack pipelines with custom components." | Single-hop-specific query synthesizer |
| "What are the key improvements in Haystack 2.0 compared to Haystack 1.0, particularly regarding the handling of loops and customizable components?" | "Haystack 2.0 introduces significant improvements... In Haystack 1.0, the pipeline graph was acyclic... Haystack 2.0 allows for cycles in the pipeline graph... Additionally, Haystack 2.0 emphasizes a technology-agnostic design... and supports the creation of custom components..." | Multi-hop-specific query synthesizer |

*Table 5.2 – Sample synthetic data containing question, answer, and synthesizer*

### Arquitectura avanzada: unificar múltiples fuentes

Un pipeline "branching" que ingiere, procesa y unifica múltiples tipos de datos simultáneamente — esencial cuando el conocimiento está fragmentado en silos (documentos internos + webs públicas). Crea un efecto "funnel" en tres stages:

1. **Parallel ingestion** — `file_router` dirige el PDF a `PyPDFToDocument`, mientras `LinkContentFetcher` manda el web stream a `HTMLToDocument`.
2. **Unification** — `DocumentJoiner` colecta los `Document` procesados de ambas fuentes.
3. **Unified processing** — de acá en adelante idéntico: la lista mergeada va a `doc_cleaner` → `doc_splitter` → `doc_converter`.

> [!note] Los custom components downstream (`KnowledgeGraphGenerator`, `SyntheticTestGenerator`) son **completamente ajenos** a que los datos vinieron de dos fuentes distintas. Gracias a la separación de concerns, reciben una lista unificada y crean un único knowledge graph y test dataset comprehensivo.

La ejecución con ambos tipos de input:

```python
pdf_file = Path("./data_for_indexing/howpeopleuseai.pdf")
web_urls = ["https://www.bbc.com/news/articles/c2l799gxjjpo",
    "https://www.brookings.edu/articles/how-artificial-intelligence-is-transforming-the-world/"]

# Run pipeline with both input types
result = pipeline.run({
    "file_router": {"sources": [pdf_file]},
    "link_fetcher": {"urls": web_urls }
})
```

### Tabla 5.3 — Sample synthetic data (HTML + PDF)

| Question | Answer (Reference) | Strategy |
|---|---|---|
| "How does Alexa utilize artificial intelligence in its functionality?" | "Alexa is a voice-controlled virtual assistant that uses artificial intelligence to process large amounts of data, identify patterns, and follow detailed instructions." | Single-hop-specific query synthesizer |
| "What are the implications of legal liability in the context of improving data access for AI development?" | "The implications of legal liability in the context of improving data access for AI development are significant. A body of case law indicates that liability is determined by the facts and circumstances of a situation, which can lead to various penalties, including civil fines or imprisonment…" | Multi-hop abstract query synthesizer |

*Table 5.3 – Sample synthetic data containing question, answer, and synthesizer*

## Testing y debugging de custom components

Mover el RAG "de un arte a una práctica de ingeniería madura" requiere código robusto, predecible y debuggable. Los custom components, sobre todo en pipelines multi-stage, deben testearse rigurosamente como cualquier software de producción. Cinco principios clave:

1. **Isolate component logic with mocking** — el principio más importante: aislar la unidad de lógica. Los components dependen de servicios externos (OpenAI API) o recursos pesados (modelos de GB); los tests no deben ser lentos, costar dinero ni fallar por red. Se usa **mocking** para simular dependencias.
   > [!tip] En `test_synthetic_test_components.py`, `@patch('synthetic_test_components.TestsetGenerator')` reemplaza el `TestsetGenerator` real (que llamaría al LLM) por un mock. Y `@patch.dict(os.environ, {'OPENAI_API_KEY': 'test-key'})` simula la presencia de la API key, permitiendo inicializar sin un secret real. Tests rápidos, repetibles y ejecutables en cualquier entorno.
2. **Validate the component life cycle and state** — los components tienen un ciclo (configured → initialized/warmed-up → running). En `test_warmup_components.py`: `test_component_initialization` confirma que el modelo NO está cargado al crear (`embedder.model` es `None`); `test_run_with_valid_texts` confirma que llamar `run()` antes de `warm_up()` lanza `RuntimeError`; `test_warm_up_loads_model` verifica que `warm_up()` carga el modelo y confirma **idempotencia** (segunda llamada no recarga).
3. **Test configuration and initialization** — verificar que la configuración vía `__init__` funcione. Ej.: `test_component_initialization` en `test_synthetic_test_components.py` valida que sin argumentos el `testset_size` default sea `10`, y con un custom `20` se guarde correctamente.
4. **Verify "bridge" components and data contracts** — el `DocumentToLangChainConverter` tiene un único job: transformar `List[HaystackDocument]` → `List[LangChainDocument]`; su test verifica esa transformación de data contract.
5. **Test edge cases and graceful failure** — un pipeline robusto no crashea ante inputs inesperados. Tests como `test_empty_document_handling` y `test_empty_document_list_handling` chequean el caso crítico: *¿qué pasa si el component recibe una lista vacía?* No debe crashear — debe **fallar elegantemente** y retornar un output vacío predecible, dejando que el pipeline continúe o termine limpio.

> [!note] Con estos principios se completa el viaje de usuario a arquitecto: no solo construir custom components domain-specific, sino implementar las prácticas de ingeniería que los hacen robustos, mantenibles y production-ready.

## Citas

> "This chapter marks a critical transition in proficiency: from being a user of the Haystack framework to becoming an architect capable of extending it."
> "A critical rule is that the run() method must always return a Python dictionary."
> "Standard RAG systems that rely on vector search over isolated text chunks often struggle with complex, multi-hop questions."
> "By representing how information is connected, not just that it exists, a knowledge graph provides the necessary scaffold to generate questions that genuinely test a RAG system's reasoning capabilities."
> "Our custom components, especially in a complex, multi-stage pipeline, are no different from any other piece of production software: they must be rigorously tested."

## Para aplicar

- **Setup del capítulo** — seguir las Setup Instructions del folder `ch5` del repo (`github.com/PacktPublishing/Building-Natural-Language-and-LLM-Pipelines/tree/main/ch5#setup-instructions`).
- **Definir un custom component** — los 4 requisitos: `@component` sobre la clase; `__init__` liviano (solo config/API keys, sin recursos pesados); `run()` que retorna un **diccionario** (keys = output sockets); `@component.output_types()` sobre el `run()` (alineado con el diccionario). Declarar inputs como argumentos tipados en `run()`.
- **Best practices del Prefixer** — immutable processing (crear nuevos `Document`, no mutar) + metadata preservation (copiar `.meta`).
- **Recursos pesados con `warm_up()`** — 3 etapas: `__init__` liviano (guarda config), `warm_up()` (carga el modelo una vez, con check `if self.model is None` para idempotencia), `run()` rápido (safety check + execution). No llamar `warm_up()` manualmente: Haystack lo hace una vez al `pipeline.run()`.
- **Generar ground-truth con knowledge graph** — usar [[Ragas]] (`TestsetGenerator`) con un `KnowledgeGraphGenerator` (`apply_transforms=True`) y un `SyntheticTestGenerator` con `query_distribution` (ej. single 25% / multi-specific 25% / multi-abstract 50%) para producir preguntas multi-hop que testean reasoning, no solo fact retrieval.
- **Patrón preferred path with fallback** — envolver la generación por grafo en `try/except` que degrada a generación document-based; encapsular cada path en su método para aislar fallos.
- **Component "bridge"** — cuando dos libs esperan formatos distintos (Haystack `Document` vs LangChain `Document`), crear un converter custom que solo transforma el data contract.
- **Modularidad multi-fuente** — swappear solo los components de ingestion (PDF → web vía `LinkContentFetcher` + `HTMLToDocument`) o unificar fuentes con `FileTypeRouter`/`LinkContentFetcher` + `DocumentJoiner`; la lógica core queda content-agnostic.
- **Testear con los 5 principios** — mocking (`@patch`) de LLM/API keys; validar el ciclo de vida (model `None` antes de `warm_up`, `RuntimeError` si `run` antes, idempotencia); testear config (defaults vs custom); verificar bridge components; testear edge cases (lista vacía → graceful failure, no crash).

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]] — capítulo anterior (ensambla components predefinidos; usa custom components como `DocumentSanitizer`/`ImagePathFixer` que este capítulo enseña a construir).
- [[06 - Building Reproducible and Production-Ready RAG Systems]] — capítulo siguiente (usa el dataset ground-truth generado aquí para evaluar cuantitativamente naive vs hybrid RAG con [[Ragas]] y observabilidad con [[Weights and Biases]]).
- [[03 - Introduction to Haystack by deepset]] — el decorador `@component`, sockets y el método `warm_up()` introducidos conceptualmente allí.
- [[Haystack 2.0]] · [[Custom Components (Haystack)]] · [[SuperComponent]] — los building blocks extendidos aquí.
- [[Ragas]] — el framework de evaluación y synthetic data; central en este capítulo y el Cap 6.
- [[RAG]] · [[Embeddings]] · [[Knowledge graph]] · [[Graph RAG]] — el stack de retrieval y la técnica graph RAG.
- [[Weights and Biases]] — observabilidad, en el Cap 6.
- **knowledge graph generator** · **synthetic test generator** · **multi-hop query** · **warm_up() lifecycle** · **preferred path with fallback** · **bridge component** — conceptos y patrones clave del capítulo; candidatos a nota propia.
