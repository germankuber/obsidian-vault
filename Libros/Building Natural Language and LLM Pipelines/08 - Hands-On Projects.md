---
title: 08 - Hands-On Projects
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 8
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Hands-On Projects
  - Cap 8 - Hands-On Projects
updated: 2026-06-12
---

# 08 - Hands-On Projects

> [!info] Capítulo 8 · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> El capítulo que pivota de pipelines monolíticos a **tools discretas de alto rendimiento** orquestadas por un agente: [[Haystack 2.0]] queda como el **tool-builder** y se introduce **[[LangGraph]]** como el **agentic orchestrator**. Tras repasar el patrón "pipelines as tools" y las limitaciones del Haystack agent (loops manuales, rigidez stateless, **context rot**), arma tres mini-proyectos — [[Named-entity recognition (NER)|NER]], **text classification**/[[Sentiment analysis]], y el **Yelp Navigator** multi-agente — que materializan la **arquitectura híbrida de dos capas**: tool layer con Haystack + [[Hayhooks]], orchestration layer con LangGraph. Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · anterior [[07 - Deploying Haystack-Based Applications]] · siguiente [[09 - Future Trends and Beyond]].

## Resumen

Este capítulo cierra el arco constructivo del libro armando sistemas reales, y para hacerlo formaliza la tesis arquitectónica que venía cocinándose: dejar de pensar en un pipeline monolítico que lo hace todo, y pensar en **tools discretas y especializadas** que un **agente** orquesta. La frase que organiza el capítulo es explícita — **[[Haystack 2.0]] es el robusto tool-builder y [[LangGraph]] el orquestador de la capa agéntica**. El capítulo arranca recapitulando la jerarquía Haystack (**Components → Pipelines → SuperComponents → Agents**) y el patrón "pipelines as tools" del Cap 4: envolver un [[RAG]] pipeline en un [[SuperComponent]], wrappearlo en un `ComponentTool` y dárselo a un `Agent` junto a un **tool belt**. Pero enseguida expone las **limitaciones del built-in agent** de Haystack: su reasoning loop **thought → action → observation** es pre-built y opaco; escalar a multi-agente con loops manuales (`ConditionalRouter` + `ToolInvoker` + `MessageCollector`) degenera en "spaghetti of pipeline connections"; los pipelines son **directed data flow** rígido, no state machines dinámicas; y la memoria conversacional sufre **context rot** cuando el context window se llena.

La alternativa es **[[LangGraph]] 1.0** (anunciado septiembre 2025): no un agente sino una **librería low-level para construir agentes como grafos**, modelando workflows agénticos como **state machine** (state, nodes, edges). La diferencia filosófica con Haystack es nítida — el grafo de Haystack es **estructural** (cómo fluyen los datos entre componentes serializables), el de LangGraph es **lógico** (un state machine donde nodes y edges son funciones Python que mutan un state persistente). LangGraph aporta **cyclical flows** nativos, **explicit guardrails** (un guardrail es solo otro node), **observability** vía el state object central, y **context engineering** vía middleware + un **checkpointer** que persiste el state (agentes durables/resumibles). La Tabla 8.1 contrasta ambos. El modelo que el capítulo defiende es **híbrido**: Haystack construye pipelines robustos → [[Hayhooks]] los expone como REST endpoints stateless → LangGraph los usa como tools.

Sobre ese andamiaje vienen los tres mini-proyectos. **NER** identifica/clasifica entidades para darle al agente structured data (no puede actuar sobre texto crudo): se usa `NamedEntityExtractor` con `dslim/bert-base-NER` (tipos LOC/ORG/PER/MISC, basado en [[BERT]]) y un custom `NERPopulator` que filtra por confidence. **Text classification y sentiment** repasan binary/multiclass/multilabel: `TransformersZeroShotTextRouter` ([[Zero-shot classification]], modelo `deberta-v3-large-zeroshot-v2.0`, evaluado sobre un dataset BBC de 2.225 muestras → 91% accuracy, Tabla 8.2 + Figura 8.1) y `TransformersTextRouter` con `cardiffnlp/twitter-roberta-base-sentiment` para sentiment (LABEL 0/1/2 = negative/neutral/positive). Finalmente el **Yelp Navigator**: un sistema multi-agente LangGraph con **sequential routing + loops + supervisor approval** que coordina tres pipelines Haystack expuestos con Hayhooks (`business_search`, `business_details`, `business_sentiment`), wrappeados con el decorador `@tool` de LangChain, un `AgentState` (TypedDict) como memoria compartida, y un `StateGraph` con conditional edges y un supervisor node de QA (max 2 approval attempts). La conclusión arquitectónica: un sistema de dos capas donde los tools NO son objetos Python importados sino **microservicios independientes** (Hayhooks, escalables con Docker/Kubernetes) y la orquestación es un `StateGraph` cuyos tool nodes son simples **API clients HTTP**.

## Recap: The Haystack agent and pipelines as tools

El capítulo retoma la jerarquía de abstracción de [[Haystack 2.0]] que organiza todo el libro: **Components → Pipelines → SuperComponents → Agents**. Un **agent** es el reasoning engine en el centro; los **tools** son las acciones que puede ejecutar. El patrón clave (introducido en el [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases|Cap 4]]) es **"pipelines as tools"**: tomás un pipeline RAG completo, lo encapsulás en un [[SuperComponent]], lo wrappeás en un `ComponentTool`, y se lo das al agente.

> [!note] Un agente Haystack se define con **tres elementos**: un **LLM** (el chat generator), un **system prompt** (su misión y reglas), y un **tool belt** (la lista de component tools disponibles).

Definir un SuperComponent (un hybrid RAG ya armado) como tool:

```python
tool_name = "internal_document_search"
tool_description = (
    "Use this tool to search and answer questions about internal knowledge, including information about Haystack, LLM models, and AI frameworks. This is the primary source for any questions related to company-specific data."
)
internal_search_tool = ComponentTool(
    name=tool_name,
    component=hybrid_rag_sc,
    description=tool_description,
)
```

Un segundo tool, esta vez de web search:

```python
web_search_component = SerperDevWebSearch(api_key=<>)
web_search_tool = ComponentTool(
    name="web_search",
    component=web_search_component,
    description="Use this tool to search the public internet for current events, news, and general knowledge. It is best for information that is not specific to our internal documents.",
)
```

> [!tip] La `description` de cada `ComponentTool` no es cosmética: es el **interface document** que el LLM lee para decidir cuándo usar el tool. Escribila pensando en el reasoning del agente, no en un humano.

Con ambos tools en una lista (el **tool belt**) se instancia el `Agent`:

```python
# The agent needs a list of all available tools
tools = [internal_search_tool, web_search_tool]
agent_llm = OpenAIChatGenerator(model="gpt-4o-mini")
system_prompt = """
You are a helpful research assistant. Your goal is to answer the user's question accurately and comprehensively.
You have access to two tools:
1. internal_document_search: For questions about our internal knowledge base (Haystack, AI models, etc.).
2. web_search: For questions about current events or public information. First, think about which tool is most appropriate for the user's question.
Then, call that tool with the necessary query.
If the question requires information from both sources, you can call the tools sequentially. Finally, synthesize the information from the tools into a final answer for the user.
"""
# Instantiate the Agent
agent = Agent(chat_generator=agent_llm, tools=tools, system_prompt=system_prompt)
```

Y se lo corre con una query en lenguaje natural:

```python
complex_query = ("Using the internal documents, explain how people use AI, then investigate the latest trends in 2025 in AI from a web search.")
messages = ChatMessage.from_user(complex_query)
agent_result = agent.run(messages=[messages])
```

El flujo del tool-calling es **query → decide tool → call tool → get answer**. Una capa adicional permite definir agentes enteros como component tools y pasarlos a un **supervisor agent** (es el tutorial multi-agent de deepset). Esta arquitectura jerárquica es potente, pero presenta desafíos para las necesidades agénticas modernas — lo que motiva la sección siguiente.

## Limitations of built-in agents

El Haystack agent component es **high-level**: su reasoning loop sigue una lógica interna pre-built **thought → action → observation**. Eso es un **pro** para la simplicidad, pero un **con** cuando se necesita control complejo. El capítulo lo ilustra con un tool-calling agent pipeline (notebook `tool-calling.ipynb`):

- **Sets up a tool** — wrappea `SearchApiWebSearch` en un tool.
- **Builds a decision loop** — un pipeline con un LLM (`OpenAIChatGenerator`), un router y un tool invoker.
- **Execution flow** — (a) el LLM recibe `How is the weather in Berlin?`; (b) reconoce que necesita info externa y genera un **tool call request**; (c) el `ConditionalRouter` lo detecta y lo manda al `ToolInvoker`; (d) el `ToolInvoker` corre la web search; (e) un `MessageCollector` custom junta los resultados y los realimenta al LLM; (f) el LLM usa los resultados para la respuesta final.

Ese ejemplo deja ver **dos limitaciones core** al escalar a multi-agente:

> [!warning] **Complexity of manual loops.** Crear un loop simple ya exige wiring intrincado entre `ConditionalRouter` y `ToolInvoker`, más gestionar el historial con `MessageCollector`. Escalar a multi-agente donde los agentes **se hablan entre sí** (no solo responden al usuario) deriva en un "unmanageable spaghetti of pipeline connections".

> [!warning] **Rigidity of data flow.** Los pipelines Haystack están diseñados para **directed data flow**. Forzarlos a comportarse como **dynamic state machines** (donde los paths cambian según resultados intermedios) es posible pero rígido: obliga a meter lógica stateful en componentes que son stateless por diseño.

A esto se suma el **context rot**: la degradación que sufre el sistema cuando el context window se llena. En Haystack la memoria se maneja con componentes como `ConversationalMemory`, pero al crecer la conversación puede haber **overflow del context window** — esa es la definición de context rot. Existen técnicas para mitigarlo (resumir la memoria periódicamente, reformular la query según el historial), pero son **patrones que el developer debe construir y gestionar**, no features intrínsecas del runtime core.

> [!note] La conclusión es la bisagra del capítulo: para necesidades agénticas modernas (loops dinámicos, multi-agente, gestión explícita de estado y contexto) hace falta **otra arquitectura** — una pensada como state machine, no como data-flow graph.

## An alternative for agentic orchestration: LangGraph

**[[LangGraph]] 1.0** (anunciado en septiembre de 2025) es una librería para construir apps stateful multi-agent. No es un agente: es una **librería low-level para construir agentes como grafos**, que da el control explícito que el Haystack Agent abstrae. Modela los workflows agénticos como **state machine** con tres conceptos:

- **State-machine architecture** — primitivas **state, nodes y edges**. El developer define la lógica como código Python, decidiendo cómo cada **node** modifica el **state** central y qué **edge** (next node) tomar.
- **Complex logic** — para loops, lógica condicional o múltiples agentes: si necesitás un `if-else` multi-step o retry mechanisms custom, lo **escribís en Python**. Es el framework al que se recurre cuando hace falta mucho más control que las abstracciones estándar.
- **Low-level abstraction** — intencionalmente "low-level"; no abstrae la arquitectura del agente, lo que da máximo poder para reasoning paths bespoke, multi-agente y human-in-the-loop.

> [!note] **Diferencia filosófica Haystack vs LangGraph.** El grafo de Haystack es **estructural**: define cómo fluyen los datos entre componentes serializables. El de LangGraph es **lógico**: define un state machine donde nodes y edges son **funciones Python** que modifican un state persistente.

En un notebook simple, el state extiende `MessagesState` con campos adicionales (por ejemplo un contador de tool calls). Los beneficios concretos de LangGraph para construir agentes:

- **Cyclical flows** — con nodes y edges explícitos, los ciclos son nativos vía conditional edges + lógica Python. El reasoning loop NO está oculto: **es el grafo** (se cablea el LLM node → tool node → de vuelta al LLM node).
- **Explicit guardrails** (la ventaja clave) — un guardrail es simplemente **otro node en el grafo**. Se pueden poner múltiples guardrails en distintas capas: al inicio del grafo antes de llamar al LLM (asegurar que la query es aceptable), y después del tool call (asegurar que la respuesta es compliant). Se profundiza en el [[10 - Epilogue - The Architecture of Agentic AI|Epílogo]].
- **Observability** — el state object central da un **trace step-by-step** de la ejecución.
- **Context engineering** — LangChain 1.0 + LangGraph 1.0 están enfocados en [[Context Engineering|context engineering]] vía **middleware** para prevenir **context pollution**. El **checkpointer system** persiste el state → agentes durables/resumibles sin re-feedear todo el historial, y aporta patrones explícitos para gestionar el historial (trimming de mensajes para quedar dentro de los token limits).

### Tabla 8.1 — Haystack 2.0 Agent vs LangGraph 1.0 StateGraph

| Feature | Haystack 2.0 Agent Component | LangGraph 1.0 StateGraph |
|---|---|---|
| **Primary metaphor** | Batteries-included component | Low-level state machine |
| **Control flow** | Pre-built, internal reasoning loop | Explicit, developer-defined graph (nodes/edges) |
| **Cyclical logic (reflection, re-planning)** | Not natively supported; loop is fixed | Natively supported; developer explicitly defines the cycles |
| **Guardrail implementation** | External (via pipeline routing) or for debugging (via Breakpoint) | Internal (as an explicit node and conditional_edge in the graph) |
| **State management** | Implicitly managed inside the agent | Explicitly defined and managed via a central State object |
| **Best for** | Simple, tool-calling RAG and conversational agents | Complex, stateful, multi-step agentic workflows |

*Table 8.1 – Haystack 2.0 and LangGraph 1.0 agent implementation*

> [!tip] La tabla resume la regla de decisión: Haystack Agent para RAG conversacional simple con tool-calling; **LangGraph para workflows agénticos complejos, stateful y multi-step** donde necesitás ciclos, guardrails internos y estado observable.

### El modelo híbrido del capítulo

> [!note] **La arquitectura que el capítulo defiende.** [[Haystack 2.0]] construye los pipelines robustos → [[Hayhooks]] los expone como **REST endpoints stateless** → [[LangGraph]] los usa como **tools** para workflows agénticos. Haystack es el engine especializado para tareas data-intensive; Hayhooks encapsula la maquinaria RAG en un web service callable; **LangGraph es el orquestador agéntico high-level (el stateful agent runtime)**.

El "brain" LangGraph **no necesita saber CÓMO se hace el RAG**, solo **CUÁNDO llamar al RAG tool** (la Hayhook API). Esto permite desarrollar, escalar y versionar de forma **independiente** la lógica del agente (LangGraph) y los pipelines (Haystack). El sistema final del capítulo (Yelp Navigator) hace cinco cosas: (1) NER sobre la query, (2) usar las entidades para poblar/fetchear la Yelp Reviews API, (3) extraer reviews y websites, (4) sentiment analysis sobre las reviews, y (5) un summary final vía agentes autónomos.

## Mini-project: Named-entity recognition (NER)

[[Named-entity recognition (NER)|NER]] identifica y clasifica entidades (nombres, locations, organizations) en texto. Por ejemplo, sobre `"Schedule a meeting with Sarah at Blue Bottle Coffee in San Francisco for next Friday."` extrae **Person**: Sarah · **Organization**: Blue Bottle Coffee · **Location**: San Francisco · **Date**: Next Friday.

> [!note] Un agente **no puede actuar sobre texto crudo**: necesita **structured data**. Ante `What's the weather like in London tomorrow?`, antes de llamar `get_weather` el agente usa NER para obtener `{"LOC": "London", "DATE": "tomorrow"}` — exactamente los parámetros que alimenta a `get_weather`.

NER (Grishman & Sundheim, 1996) consiste en clasificar entidades en tipos semánticos pre-definidos. Categorizar **Apple** como organization vs fruit rutea la query a un stock API vs una grocery DB. Según el survey de Keraghel et al. (2024), los métodos evolucionaron de **feature-engineering** a **deep-learning**:

- **Feature-engineering**, en tres categorías:
  - **Supervised learning** — datasets fully labeled; accurate pero costoso y labor-intensive.
  - **Unsupervised learning** — usa patrones estadísticos o knowledge bases/diccionarios sin labeled data; escala bien pero con menor precisión.
  - **Semi-supervised learning** — híbrido: usa poco labeled data para hacer "bootstrap" sobre texto sin labelar.
- **Deep learning** — modelos transformer como **[[BERT]]**, que usan attention mechanisms para captar contexto profundo.

Aplicaciones clave (con sus citas): **Information extraction** (Weston, 2019), **Information retrieval** (Banerjee, 2019), **Document summarization** (Roha, 2023), **Virtual assistants** (Park, 2023), **Question answering** (Mollá, 2006) y **Language translation** (Li, 2020b).

> [!warning] La limitación central de NER es la **ambigüedad**: una palabra como "bank" puede ser institución financiera o la orilla de un río. Homónimos y polisemia exigen resolver el sentido por contexto.

### Enhancing NER with entity linking

Para resolver la ambigüedad se usa **entity linking**: desambiguar las entidades y linkearlas a entradas de un knowledge base (Wikipedia, una ontología). El proceso es: **identificar candidate entities → analizar el contexto circundante → desambiguar**. Por ejemplo "Jaguar" puede ser el animal o la marca de auto según el contexto; una vez identificada, se linkea a un **unique identifier**.

### Haystack's Entity Extractor API

Haystack ofrece dos componentes para NER:

- **`NamedEntityExtractor`** — out-of-the-box, con backend **Hugging Face** o **spaCy**.
- **`LLMMetaDataExtractor`** — context-driven, usa un LLM.

El mini-project usa `NamedEntityExtractor` con backend Hugging Face y el modelo **`bert-base-NER`** (`dslim/bert-base-NER`), un [[BERT]] fine-tuned y SOTA para NER. Reconoce **cuatro tipos**: location (`LOC`), organizations (`ORG`), person (`PER`) y miscellaneous (`MISC`). Almacena las entidades como **metadata** en los documentos.

### Building a pipeline for NER from web search results

El pipeline define un custom component **`NERPopulator`** que procesa objetos `Document`, extrae sus entidades y las guarda como metadata, **filtrando las de bajo confidence score**. Los pasos del pipeline:

1. **Web search** — `SearchApiWebSearch`, con dominios permitidos (Britannica).
2. **Link content fetcher**.
3. **HTML to document converter**.
4. **Document cleaner**.
5. **Named entity extractor** — `dslim/bert-base-NER`.
6. **Custom component** — `NERPopulator`.

El pipeline entero se abstrae como [[SuperComponent]] y se da como **tool** a un Haystack agent. Ante la query `Find entities about Nikola Tesla from Britannica and save them to a CSV file`, el agente usa el tool, procesa, extrae las entidades y devuelve un summary:

```text
Agent Response
I found some interesting entities related to Nikola Tesla from Britannica articles. Here's a summary of what was extracted:
Summary of Entities Found
People: 15 unique entities were identified.
Organizations: 13 organizations were recognized.
Locations: 16 locations were mentioned.
Miscellaneous Entities: 24 miscellaneous entities were noted.
Key Entities Discovered
People: Unfortunately, specific names related to Tesla weren't mentioned, but the articles included relevant political figures that might have been part of the contextual discussion.
Organizations: Notable organizations such as "House of Representatives" and "Encyclopædia Britannica."
Locations: Locations included United States locations like Virginia and New Jersey.
CSV File
The results have been saved to a CSV file. You can download it using the link below:
Download NER Results (contains link to CSV file)
```

> [!tip] Este pipeline NER se puede **serializar y desplegar con [[Hayhooks]]** (Cap 7), convirtiéndolo en un microservicio reutilizable por cualquier orquestador.

## Mini-project: Text classification and sentiment analysis

La **text classification** es una tarea foundational de NLP (sentiment, news categorization, emotion detection). Tiene tres tipos:

- **Binary classification** — 2 categorías mutuamente excluyentes (spam/no-spam, positive/negative).
- **Multiclass classification** — 3+ categorías excluyentes (género: sports/politics/entertainment).
- **Multilabel classification** — múltiples labels simultáneos (un comentario bajo *violence* Y *hate speech*).

El mini-project se enfoca en **binary y multiclass**.

### Haystack's text classification components

- **`TransformersTextRouter`** — rutea text strings a distintas conexiones según un category label pre-definido; usa un modelo Hugging Face con su set fijo de labels.
- **`TransformersZeroShotTextRouter`** — extiende lo anterior con [[Zero-shot classification]]: clasifica en categorías **sin un modelo fine-tuned** en esos labels específicos, manejando labels no vistos por la relación semántica entre el texto y los labels.

### Evaluating the TransformerZeroShotTextRouter

Se elige el zero-shot router porque permite **labels custom dinámicos**. Se evalúa sobre un dataset comercial de Kaggle (BBC) con **2.225 muestras** en 5 categorías: **Politics** (Label 0), **Sport** (Label 1), **Technology** (Label 2), **Entertainment** (Label 3), **Business** (Label 4). El modelo es `MoritzLaurer/deberta-v3-large-zeroshot-v2.0` y se llama `warm_up()` antes de usarlo. El dataframe resultante tiene las columnas `text`, `label` (ground truth), `category label` (clasificación numérica) y `output branch` (mapeo numérico → label). Se usa el **classification report** de scikit-learn:

### Tabla 8.2 — Classification report

| | Precision | Recall | F1-score | Support |
|---|---|---|---|---|
| **Politics** | 0.80 | 0.97 | 0.88 | 417 |
| **Sport** | 0.96 | 1.00 | 0.98 | 511 |
| **Technology** | 0.96 | 0.86 | 0.91 | 401 |
| **Entertainment** | 0.90 | 0.95 | 0.93 | 386 |
| **Business** | 0.97 | 0.79 | 0.87 | 510 |
| **Accuracy** | — | — | 0.91 | 2225 |
| **Macro avg** | 0.92 | 0.92 | 0.91 | 2225 |
| **Weighted avg** | 0.92 | 0.92 | 0.91 | 2225 |

*Table 8.2 – Classification report*

> [!note] **Análisis.** Accuracy global del **91%**. **Sport** es la mejor categoría (recall 1.00, F1 0.98). **Business** tiene el recall más bajo (0.79 — pierde ~21% de los casos) aunque su precision es alta (0.97): cuando dice "Business" casi siempre acierta, pero deja business sin clasificar.

![[08-fig-8.1-confusion-matrix.png]]
*Figure 8.1 – Confusion matrix for text classification of news articles*

> [!tip] La confusion matrix confirma la alta accuracy, sobre todo en **Sport** y **Technology**, mientras **Politics** y **Business** concentran las misclassifications.

### Setting up a text classification pipeline

El pipeline define un custom component **`NewsClassifier`** que aplica el modelo a cada documento y guarda el label en metadata. Los pasos:

1. **Web search** — `SearchApiWebSearch`, dominio Yahoo Finance.
2. **Link content fetching** — `LinkContentFetcher`.
3. **HTML to document converter** — `HTMLToDocument`.
4. **Document cleaner** — `DocumentCleaner`.
5. **News classifier** — `NewsClassifier`.

Para la query "Elon Musk" el pipeline extrae metadata como esta:

```text
{'content_type': 'text/html', 'url': 'https://ca.finance.yahoo.com/news/musk-ascends-political-force-beyond-204319247.html', 'labels': 'Politics'}  {'content_type': 'text/html', 'url': 'https://ca.finance.yahoo.com/news/elon-musk-spacex-face-federal-184331986.html', 'labels': 'Politics'}  {'content_type': 'text/html', 'url': 'https://ca.finance.yahoo.com/news/elon-musk-becomes-first-person-184648913.html', 'labels': 'Business'}  {'content_type': 'text/html', 'url': 'https://ca.finance.yahoo.com/news/elon-musks-net-worth-tops-143636708.html', 'labels': 'Business'}  {'content_type': 'text/html', 'url': 'https://ca.finance.yahoo.com/news/openai-fires-back-elon-musk-212307411.html', 'labels': 'Technology'}
```

### Overview of sentiment analysis

El [[Sentiment analysis]] determina el **tono emocional** de un texto (positive/negative/neutral). Tiene tres granularidades:

- **Document-level** — todo el documento → una categoría.
- **Sentence-level** — cada oración por separado.
- **Aspect-based** — aspectos/features específicos (en una review: "battery life", "design", "performance").

Aplicaciones: **Customer feedback**, **Social media monitoring**, **Market research**, **Political analysis** y **Brand monitoring**.

### Haystack's classification component for sentiment

Para sentiment se usa `TransformersTextRouter` con el modelo pre-entrenado **`cardiffnlp/twitter-roberta-base-sentiment`**, llamando `warm_up()` antes de usarlo.

### Setting up a sentiment analysis pipeline

El pipeline fetchea Yelp business reviews vía **RapidAPI** y enriquece cada review con su sentiment label. Define **dos componentes custom**:

- **Component 1: `YelpReviewFetcher`** — hace el API call al Yelp Business Reviews endpoint y convierte la respuesta en objetos `Document`, cada uno con el review text + metadata (rating, URL).
- **Component 2: `BatchSentimentProcessor`** — un wrapper que internamente usa `TransformersTextRouter` con el modelo RoBERTa `cardiffnlp/twitter-roberta-base-sentiment`; loopea cada doc, lo clasifica como `LABEL 0/1/2` (= negative/neutral/positive), mapea a strings human-readable y devuelve los docs enriquecidos con el sentiment en metadata.

El input del pipeline:

```python
url = https://yelp-business-reviews.p.rapidapi.com/reviews/RJNAeNA-209sctUO0dmwuA
querystring = {"sortBy": "lowestRated"}
headers = {
    "x-rapidapi-key": RAPID_API_KEY,
    "x-rapidapi-host": "yelp-business-reviews.p.rapidapi.com"
}
pipeline_input = {
    "review_fetcher": {"url": url, "headers": headers,
    "querystring": {"sortBy": "highestRated"}}
}
pipeline_result = sentiment_pipeline.run(pipeline_input)
```

> [!warning] Se pidieron las **highest-rated** reviews pero el resultado salió **negative** — un recordatorio de que el rating numérico y el sentiment del texto no siempre coinciden.

Dos sample Documents enriquecidos:

```python
Document(id=7, content: 'I just want to start out by saying that we were a table of bartenders, servers, and nurses. We under...', meta: {'rating': 1, 'url': 'https://www.yelp.com/biz/RJNAeNA-209sctUO0dmwuA?hrid=H-rbvyF3NfZH58GyiQHOyw', 'sentiment': 'negative'})
Document(id=8, content: 'Definitely a tourist trap, I went here after reading that they had been voted best cheese curds in W...', meta: {'rating': 1, 'url': 'https://www.yelp.com/biz/RJNAeNA-209sctUO0dmwuA?hrid=TGWg_3KffA-vqWP9EVRvfw', 'sentiment': 'negative'})
```

## Mini-project: Orchestrating tools with agentic systems (Yelp Navigator)

El proyecto culminante es un sistema multi-agente que maneja queries como `"Find me 3 Mexican restaurants in Austin, Texas, and analyze their customer reviews to tell me which one has the best service."`. **Ningún pipeline solo lo resuelve**: requiere encadenar (1) **Search** (encontrar los businesses), (2) **Details** (fetchear data específica como website content), (3) **Sentiment analysis** (leer reviews), (4) **Summarization** (agregar en la respuesta final) y (5) **Quality assurance** (asegurar que el reporte cumple los requisitos). Como las queries varían en nivel de detalle, hace falta un sistema que **detecte el nivel necesario, elija el tool y delegue**.

### Architecture: Sequential routing with loops and approval

El modelo es **sequential routing**, donde cada agente determina el next step, más un **supervisor node** que actúa como capa de QA: decide si el reporte cumple los requisitos y lo aprueba, o lo manda de vuelta al agente apropiado. Componentes clave:

- **Haystack pipelines (los tools)** — 3 pipelines (search, info retrieval, sentiment) expuestos como REST APIs vía [[Hayhooks]].
- **Clarification node** — asegura que la query tiene info suficiente (location + detail level).
- **Worker nodes** — agentes LangChain especializados, cada uno con **UN** tool Haystack (ej. el search node solo accede al endpoint `business_search`).
- **Summarization node**.
- **Supervisor approval** — el orquestador que usa un LLM para decidir si el reporte es satisfactorio.

![[08-fig-8.2-multiagent-architecture.png]]
*Figure 8.2 – Multi-agent architecture*

Los **cinco pasos del proyecto**: (1) exponer los pipelines como REST, (2) wrappear los endpoints como tools LangGraph, (3) definir el agent state, (4) construir los workers + supervisor, (5) construir el grafo.

### Step 1: Exposing pipelines as tools

Vía [[Hayhooks]], en tres stages:

- **Serialization** — `pipeline.dumps()` produce un YAML con el blueprint completo (componentes + conexiones).
- **Storage** — el YAML se guarda en un directorio (ej. `pipelines/`).
- **Exposure** — `uv run hayhooks run --pipelines-dir pipelines`; Hayhooks escanea, carga cada YAML y crea un REST endpoint (ej. `http://localhost:1416/business_search/run`).

Las **tres pipelines especializadas**:

- **Pipeline 1: Business search with NER** (`business_search`) — extrae entidades (business type, location) y devuelve businesses relevantes.
- **Pipeline 2: Business details with website content** (`business_details`) — enriquece con el texto de los websites.
- **Pipeline 3: Reviews with sentiment analysis** (`business_sentiment`) — fetchea reviews y hace sentiment.

Cada pipeline sigue la misma estructura: component definition (`components.py`), build + serialize (`build_pipeline.py`), wrap + load (`pipeline_wrapper.py`), con un script `build_all_pipelines.sh`. Para correr todo:

```bash
./build_all_pipelines.sh
uv run hayhooks run --pipelines-dir pipelines
```

Esto instancia el server en `http://localhost:1416/` (docs en `http://localhost:1416/docs`).

> [!warning] El free plan de **RapidAPI** tiene un límite de **50 requests** — tenerlo en cuenta al testear el Yelp Navigator.

### Step 2: Wrapping endpoints as tools

Se usa el decorador **`@tool`** de LangChain: una función Python que hace un HTTP POST al endpoint Hayhooks y devuelve el JSON. `@tool` **auto-genera un JSON schema** desde el nombre, el docstring y los type hints, que actúa como el **interface document** que el LLM lee:

```python
@tool
def search_businesses(query: str) -> Dict[str, Any]:
    """Search for businesses using natural language query.

    Args:
        query: Natural language search query (e.g., '
        Mexican food in Austin, Texas')

    Returns:
        Dictionary containing business search results with
         names, ratings, locations, etc.
    """
    try:
        response = requests.post(
            f"{BASE_URL}/business_search/run",
            json={"query": query},
            timeout=30
        )
```

### Step 3: Defining the agent state

El **state** es la **memoria compartida** que persiste durante toda la ejecución del grafo: un `TypedDict`. Para el Yelp Navigator incluye **user intent data** (`clarified_query`, `clarified_location`, `detail_level`), **workflow control** (flags como `clarification_complete`) y **agent outputs** (un dict con los resultados parciales de cada worker):

```python
class AgentState(MessagesState):
    """State shared across all nodes in the workflow.

    Inherits 'messages: Annotated[List[BaseMessage], add_messages]' from MessagesState.
   """
    # User intent (set by Clarification Node)
    user_query: str              # Original user question
    clarified_query: str         # What they're looking for
    clarified_location: str      # Where to search
    detail_level: str            # "general", "detailed", or "reviews"
    # Workflow control
    clarification_complete: bool  # True when ready to proceed to search
    next_agent: str              # Which node to call next (enables conditional routing)
    # Node results
    agent_outputs: Dict[str, Any]  # Results from each node (search, details, sentiment)
    # Final output
    final_summary: str  # User-friendly response
    # Quality control (for Supervisor Approval Node)
    approval_attempts: int      # How many times supervisor has reviewed (max 2)
    needs_revision: bool        # True if summary needs improvement
    revision_feedback: str      # What to improve
```

> [!note] El `AgentState` logra la coordinación vía tres mecanismos: **Context preservation** (los requisitos accesibles a todos los agentes), **Flow control** (gatekeeper: solo procede cuando la request se entiende) y **Data aggregation** (repositorio central de resultados parciales).

### Step 4: Building workers, a supervisor, and the graph

Hay **cuatro tipos de nodos**:

- **Clarification node** — clarifica el intent, asegura la info, pasa al Search Agent.
- **Worker node** — especialistas, cada uno con **UN** tool (ej. el search node usa solo `business_search`).
- **Summary node** — genera el reporte con nombres, teléfonos, websites y reviews.
- **Supervisor approval node** — actúa como **router puro, NO usa tools**: analiza el state y usa un LLM para decidir qué worker sigue o si terminó.

Se usan **conditional edges** para routing dinámico: en vez de un path fijo A→B, el grafo mira el output (ej. `"next_agent": "sentiment"`) y rutea dinámicamente. La construcción del grafo:

```python
# Build the graph
workflow = StateGraph(AgentState)
# Add nodes
workflow.add_node("clarification", clarification_wrapper)
workflow.add_node("search", search_wrapper)
workflow.add_node("details", details_wrapper)
workflow.add_node("sentiment", sentiment_wrapper)
workflow.add_node("summary", summary_wrapper)
workflow.add_node("supervisor_approval", supervisor_approval_wrapper)
```

```python
workflow.add_edge(START, "clarification")
# Clarification loops or moves to search
workflow.add_conditional_edges("clarification",
        route_after_clarification,
        {"clarification": "clarification", "search": "search"}
    )
```

```python
workflow.add_conditional_edges(
        "search",
        route_after_search,
        {"details": "details", "summary": "summary"}
    )
```

```python
workflow.add_conditional_edges(
        "details",
        route_after_details,
        {"sentiment": "sentiment", "summary": "summary"}
    )
```

```python
# Sentiment -> Summary (always)
workflow.add_edge("sentiment", "summary")
# Summary -> Supervisor Approval (always)
workflow.add_edge("summary", "supervisor_approval")
```

```python
# Supervisor Approval can route back to nodes or to END
workflow.add_conditional_edges(
        "supervisor_approval",
        route_from_supervisor_approval,
        {
            "search": "search",
            "details": "details",
            "sentiment": "sentiment",
            "summary": "summary",
            END: END
        }
```

Ejecución de `Find coffee shops in Portland and check their reviews`:

1. **Clarification node** — identifica `query="coffee shops"`, `location="Portland"`, `detail_level="reviews"`; rutea a search.
2. **Search node** — encuentra coffee shops vía Pipeline 1; como `detail_level="reviews"` → details node.
3. **Details node** — website info vía Pipeline 2; → sentiment node.
4. **Sentiment node** — reviews vía Pipeline 3; → summary node.
5. **Summary node** — agrega todo; → supervisor approval.
6. **Supervisor approval** — revisa la calidad; aprueba → `END`, o pide revisión → vuelve a un agente (**máximo 2 approval attempts**).

> [!tip] Una query más simple como `best places for sushi in California` corre un flujo más corto: **Clarification → Search → Summary → Supervisor approval**, salteando details y sentiment. El routing es dinámico según el detail level detectado.

> [!warning] **Tres failure modes** a monitorear: **LLM decision failure** (la secuencia se detiene en un reasoning node porque el LLM no generó un tool call válido), **Node timeout** (un componente como web search excede el time limit) y **Tool execution error** (el node se llama bien pero la función Python/API tira una excepción). El [[10 - Epilogue - The Architecture of Agentic AI|Epílogo]] profundiza el debug de nodos.

## Summary y conclusión arquitectónica

El capítulo construye **dos tipos de sistemas**:

- **High-performance tools (con Haystack)** — NER, text classification y sentiment: utilidades especializadas y modulares. Haystack excele en RAG knowledge-intensive.
- **Flexible orchestration (con LangGraph)** — el Haystack agent tiene limitaciones (un reasoning loop black-box sin flexibilidad de control flow); **[[LangGraph]] da control white-box explícito, lógica cíclica y state observable**.

> [!note] **Conclusión arquitectónica: un sistema híbrido de dos capas.** **Tool layer con Haystack + Hayhooks** — los tools NER/classifier/RAG NO son objetos Python importados por el agente, sino **microservicios independientes** desplegados con Hayhooks, escalables con Docker/Kubernetes. **Orchestration layer con LangGraph** — un `StateGraph` central cuyos tool nodes NO son `ComponentTool` wrappers sino simples **API clients que hacen HTTP requests** a los endpoints Hayhooks.

El próximo [[09 - Future Trends and Beyond|Cap 9]] cubre las limitaciones de los LLMs (accuracy, scalability, ética) y los campos emergentes (ética, ley, operations, security, decision-making).

## Citas

> "Haystack serves as our robust tool-builder, and LangGraph will be introduced as the agentic layer orchestrator."
> "Scaling this to a multi-agent system where agents need to talk to each other, rather than just responding to the user, leads to an unmanageable spaghetti of pipeline connections."
> "It is not an agent itself but a low-level library for building agents as graphs."
> "Haystack's graph is structural... LangGraph's is logical, defining a state machine where nodes and edges are Python functions that modify a persistent state."
> "The agent's reasoning loop is not hidden; it is the graph."
> "By combining Haystack's robust, specialized pipelines with LangGraph's reasoning capabilities, we have created a system that is far more capable than the sum of its parts."

## Para aplicar

- **Pipelines as tools** — envolver un pipeline RAG en un [[SuperComponent]], wrappearlo en un `ComponentTool` (name + description + component) y dar un **tool belt** (lista de tools) a un `Agent` junto a un LLM y un system prompt. La `description` es el interface document que el LLM lee.
- **Reconocer las limitaciones del Haystack agent** — el spaghetti de loops manuales (`ConditionalRouter` + `ToolInvoker` + `MessageCollector`), la rigidez del data flow stateless, y el **context rot** del context window. Cuando aparecen, es señal de pasar a LangGraph.
- **LangGraph para orquestación stateful** — modelar con **state / nodes / edges**; usar **conditional edges** para routing dinámico; implementar **guardrails como nodes** (al inicio y tras cada tool call); usar el **checkpointer** para durabilidad/resumibilidad y middleware para context engineering.
- **Arquitectura híbrida de producción** — **tool layer**: Haystack pipelines serializados (`dumps()`) + [[Hayhooks]] como microservicios REST. **Orchestration layer**: LangGraph `StateGraph` con tool nodes = API clients HTTP a los endpoints Hayhooks.
- **NER** — `NamedEntityExtractor` + `dslim/bert-base-NER` (tipos LOC/ORG/PER/MISC) + un custom `NERPopulator` que filtra por confidence score; guardar las entidades como metadata.
- **Text classification** — `TransformersZeroShotTextRouter` (labels custom dinámicos, `deberta-v3-large-zeroshot-v2.0`) cuando necesitás labels no vistos, o `TransformersTextRouter` con un modelo fine-tuned cuando los labels son fijos.
- **Sentiment** — `TransformersTextRouter` con `cardiffnlp/twitter-roberta-base-sentiment` (LABEL 0/1/2 = negative/neutral/positive), mapeando a strings human-readable. Siempre llamar **`warm_up()`** antes de usar el router.
- **Yelp Navigator** — serializar 3 pipelines (`business_search` / `business_details` / `business_sentiment`) con `dumps()`, exponer con Hayhooks (`hayhooks run --pipelines-dir`), wrappear los endpoints con `@tool` de LangChain (HTTP POST), definir un `AgentState` (TypedDict), y construir el `StateGraph` con clarification / worker / summary / supervisor nodes + conditional edges (max 2 approval attempts).
- **Monitorear los 3 failure modes** — LLM decision failure, node timeout y tool execution error. Recordar el límite de 50 requests del free plan de RapidAPI.

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[07 - Deploying Haystack-Based Applications]] — capítulo anterior; los pipelines desplegados con Hayhooks/REST se usan acá como **tools** del orquestador.
- [[09 - Future Trends and Beyond]] — capítulo siguiente; las limitaciones de los LLMs y los campos emergentes (ética, seguridad).
- [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]] — el patrón "pipelines as tools" y el hybrid RAG que se envuelve como SuperComponent.
- [[02 - Diving Deep into Large Language Models]] — context engineering, context rot y middleware.
- [[10 - Epilogue - The Architecture of Agentic AI]] — el Epílogo profundiza el debug de nodos, los guardrails y el Yelp Navigator V1/V2/V3.
- [[Haystack 2.0]] · [[Hayhooks]] · [[LangGraph]] · [[RAG]] · [[SuperComponent]] · [[Model Context Protocol (MCP)]] — el stack tool + orquestación.
- [[Named-entity recognition (NER)]] · [[Sentiment analysis]] · [[Zero-shot classification]] · [[BERT]] — las tools NLP clásicas del capítulo.
- [[Context Engineering]] · [[ReAct]] · [[Supervisor-worker pattern]] — la disciplina de contexto y los patrones de orquestación.
- **StateGraph** · **conditional edges** · **AgentState (TypedDict)** · **entity linking** · **TransformersZeroShotTextRouter** · **NamedEntityExtractor** · **checkpointer** · **Yelp Navigator** · **sequential routing model** — conceptos clave del capítulo; candidatos a nota propia.
