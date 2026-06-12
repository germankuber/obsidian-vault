---
title: 04 - Components of a RAG System
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 4
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Components of a RAG System
  - Componentes de un sistema RAG
updated: 2026-06-12
---

# 04 - Components of a RAG System

> [!info] Capítulo 4 · *Unlocking Data with Generative AI and RAG* — Packt (ISBN 9781835887905) · Keith Bourne
> Anatomía de un sistema [[RAG]]: el capítulo mapea las **tres etapas técnicas** del cap. 1 —[[Indexing]], [[Retrieval]] y [[Generation]]— sobre el código real del cap. [[02 - Code Lab - An Entire RAG Pipeline]], y suma los **componentes prácticos de desarrollo**: prompting, definir el LLM, UI y evaluación. La idea fuerza: el indexing es **pre-procesamiento offline** (antes de que el usuario abra la app), mientras retrieval y generation corren en **tiempo real** con cada consulta. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[03 - Practical Applications of RAG]] · siguiente [[05 - Managing Security in RAG Applications]].

## Resumen

Este capítulo es el plano (anatomía) de un sistema [[RAG]]: toma las tres etapas técnicas que el cap. [[01 - What Is Retrieval-Augmented Generation (RAG)]] presentó en teoría —**[[Indexing|indexing]], [[Retrieval|retrieval]] y [[Generation|generation]]**— y las **mapea sobre el código concreto** del cap. [[02 - Code Lab - An Entire RAG Pipeline]], línea por línea, para mostrar dónde vive cada componente. Pero no se queda en lo técnico: añade los **componentes prácticos** que toda aplicación RAG real necesita —**prompting, definir tu LLM, la UI y la evaluación**— y deja para capítulos posteriores el detalle de cada uno. El insight estructural más importante es que el **indexing es la primera etapa pero NO corre en tiempo real**: es **pre-procesamiento offline** que ocurre antes de que el usuario siquiera abra la app, mientras que retrieval y generation se disparan en tiempo real con cada consulta. A lo largo del capítulo aparecen restricciones prácticas clave: el límite de **8191 tokens** de `OpenAIEmbeddings()`, cómo OpenAI tokeniza con **byte pair encoding (BPE)** (~1 token ≈ 4 caracteres en inglés), el **chunk overlap** para no perder contexto entre chunks, el **costo** de las llamadas a la API de embeddings, y el caso revelador de que **GPT-4o conoce RAG pero ChatGPT 3.5 no** (lo interpreta como "Red, Amber, Green") por su corte de conocimiento en **enero de 2022** —demostración viva del valor de RAG. Cierra con la UI (pre/post-procesamiento, interfaz de salida, feedback con [[NLU]]) y la evaluación como componentes de primera clase, y anuncia que el próximo capítulo se dedica entero a la **seguridad**. El setup (instalar, importar, cuentas de OpenAI) se omite acá porque ya se cubrió en el cap. 2. El código vive en github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_04.

## Visión general de los componentes

El capítulo cubre las **tres etapas técnicas principales** que ya aparecieron en el cap. 1 —[[Indexing|indexing]], [[Retrieval|retrieval]] y [[Generation|generation]]— y les suma los **componentes prácticos de desarrollo** que hacen falta para construir una aplicación real: **prompting, definir tu LLM, la UI (interfaz de usuario) y la evaluación**. Capítulos posteriores profundizan en cada componente; acá el objetivo es ver el panorama completo y dónde encaja cada pieza en el código del cap. 2. El setup (instalación, imports, cuentas de OpenAI) **se saltea** porque ya se trató en el code lab del cap. [[02 - Code Lab - An Entire RAG Pipeline]].

![[04-fig-4.1-three-stages.jpg|475]]
*Figure 4.1 – The three stages of a RAG system*

## Indexing

![[04-fig-4.2-indexing-highlighted.jpg|790]]
*Figure 4.2 – The Indexing stage of RAG highlighted*

El [[Indexing|indexing]] es la **primera** etapa principal, pero se diferencia de retrieval y generation en algo crucial: estas dos corren en **tiempo real** cuando el usuario interactúa con la aplicación, mientras que el indexing típicamente ocurre **mucho antes** —se lo llama **pre-procesamiento offline (pre-processing offline)** porque se hace antes de que el usuario siquiera abra la app. El indexing *puede* hacerse en tiempo real, pero es mucho menos común.

> [!note] **Indexing = pre-procesamiento offline.** A diferencia de retrieval y generation (tiempo real, al interactuar el usuario), el indexing construye la infraestructura de búsqueda **antes** de que llegue cualquier consulta. Puede ser en tiempo real, pero es la excepción.

### Extracción de documentos

El primer paso del indexing es extraer los datos. En el código del cap. 2 esto lo hace `WebBaseLoader` de [[LangChain]] (nótese que el snippet del libro omite una coma después de `web_paths`, se reproduce fiel):

```python
loader = WebBaseLoader(
    web_paths=("https://kbourne.github.io/chapter1.html",)
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            class_=("post-content", "post-title",
                    "post-header")
        )
    ),
)
docs = loader.load()
```

Acá los datos son una **página web** (Fig. 4.3 muestra la página que se procesa), pero podrían ser PDF, Word u otros **datos no estructurados** ([[Unstructured Data|unstructured data]]) —muy populares en RAG: históricamente eran difíciles de acceder frente a los datos estructurados de SQL, y RAG cambió eso. Otros tipos de datos se cargan con otros **document loaders** de LangChain, tema que se profundiza en el cap. [[11 - Using LangChain to Get More from RAG]]. El document loader llena el componente **Documents**.

![[04-fig-4.3-web-page.jpg]]
*Figure 4.3 – The web page that we process*

### Splitting

El siguiente paso convierte los datos a un formato buscable (vectores); para eso hay que **partirlos (split) en chunks digeribles**. El [[SemanticChunker]] hace el split semántico:

```python
text_splitter = SemanticChunker(OpenAIEmbeddings())
splits = text_splitter.split_documents(docs)
```

El vectorizador `OpenAIEmbeddings()` tiene un input máximo de **8191 tokens**. Cada chunk debe quedar por debajo de ese límite; otros servicios de embeddings tienen sus propios límites.

> [!note] **Cómo tokeniza OpenAI (BPE).** OpenAI tokeniza con **byte-level byte pair encoding (BPE)**, generando **sub-palabras (subword tokens)**, no caracteres. El número de tokens depende del contenido: palabras/sub-palabras comunes = un solo token; palabras raras se parten en varios. En promedio, **~1 token ≈ 4 caracteres** en inglés (regla aproximada): "a"/"the" = 1 token, una palabra larga e infrecuente = varios tokens.

Si en lugar del `SemanticChunker` se usa un splitter con `chunk_size` + `chunk_overlap`, para el total **hay que sumar el overlap al chunk size**. Ejemplo con `RecursiveCharacterTextSplitter` (chunk_size 1000, chunk_overlap 200):

```python
text_splitter = RecursiveCharacterTextSplitter(
   chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)
```

> [!tip] El **[[Chunk Overlap|chunk overlap]]** evita perder contexto entre chunks: p. ej., una dirección cortada por la mitad en un documento legal queda preservada gracias al solape. Las opciones de splitter (incluido [[RecursiveCharacterTextSplitter]]) se desarrollan en el cap. [[11 - Using LangChain to Get More from RAG]].

### Vector store y embeddings

La última parte del indexing crea el vector store y guarda los [[Embeddings|embeddings]]:

```python
vectorstore = Chroma.from_documents(
    documents=splits,
    embedding=OpenAIEmbeddings())
retriever = vectorstore.as_retriever()
```

Se usa **[[Chroma DB]]** como base/almacén de vectores (vector database/store) más los embeddings de OpenAI. Todo esto se hace **offline** y se almacena para futuras consultas/recuperaciones. Chroma y `OpenAIEmbeddings` son **solo una opción cada uno**. Los vectores, los stores y las búsquedas se profundizan en los caps. [[07 - The Key Role Vectors and Vector Stores Play in RAG]] y [[08 - Similarity Searching with Vectors]].

### Por qué el retriever vive en Indexing

¿Por qué definir el retriever (`vectorstore.as_retriever()`) cae en **Indexing** y no en Retrieval? Porque el retriever es **el mecanismo desde el cual se recupera**, pero la recuperación no se *aplica* hasta que llega la consulta del usuario. El indexing **construye la infraestructura**; al final tenés un retriever **listo y esperando**.

> [!note] Definir el retriever es parte del **Indexing** aunque su nombre evoque "retrieval": en indexing se arma el mecanismo (offline), y la recuperación recién se ejecuta en tiempo real con la consulta del usuario. La Fig. 4.5 es una representación más precisa del estado al final del indexing.

![[04-fig-4.4-creating-retriever.jpg|476]]
*Figure 4.4 – Creating a retriever in the Indexing stage of the RAG process*

![[04-fig-4.5-vectors-indexing.jpg|451]]
*Figure 4.5 – Vectors during the Indexing stage of the RAG process*

## Retrieval y Generation

![[04-fig-4.6-retrieval-generation.jpg|643]]
*Figure 4.6 – Vectors during the Indexing stage of the RAG process*

En el código, [[Retrieval|retrieval]] y [[Generation|generation]] están **combinados**: el `rag_chain` al invocarse atraviesa ambas etapas. Acá se las separa conceptualmente.

### Pasos de retrieval

Solo hay **dos lugares** donde se procesa la recuperación real. El primero es el post-procesamiento:

```python
# Post-processing
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)
```

El segundo es el primer paso de la cadena: `{"context": retriever | format_docs, "question": RunnablePassthrough()}`. El orden de invocación es:

```python
rag_chain.invoke("What are the Advantages of using RAG?")
```

La cadena completa, con la recuperación como primer eslabón:

```python
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
```

La consulta del usuario va al `retriever`, que ejecuta una **[[Similarity Search|similarity search]]** y devuelve una **lista** de strings de contenido contextualmente similares. Acá aparece el "glitch": los placeholders `{question}` y `{context}` del prompt esperan **STRINGS**, pero el retriever devuelve una **LISTA** de strings de contenido → por eso hace falta `format_docs`. El retriever es en realidad una **mini-cadena**: `retriever | format_docs` (el pipe `|`). Ese `format_docs` es el **post-procesamiento dentro de Retrieval**. El placeholder `{question}` no tiene este problema (ya es un string) → se usa `RunnablePassThrough` para pasarlo tal cual.

¿Dónde ocurre la conversión consulta → vector? En `retriever = vectorstore.as_retriever()`: el método **`.as_retriever()`** convierte la consulta del usuario en un embedding con el mismo formato que los demás y luego ejecuta la recuperación.

> [!warning] **Costo de los embeddings.** Usar `OpenAIEmbeddings()` envía los embeddings a la API de OpenAI → genera cargos. Acá es solo un embedding. OpenAI cobra actualmente **$0.10 por 1M de tokens**; la consulta "What are the Advantages of using RAG?" = **10 tokens** → cuesta **$0.000001**. El autor lo menciona por transparencia sobre cualquier costo, por pequeño que sea.

### Etapa de generación

La [[Generation|generation]] es la etapa **final**: el LLM genera la respuesta a partir del contenido recuperado. Tiene dos partes de código —el prompt y el LLM:

```python
prompt = hub.pull("jclemens24/rag-prompt")
llm = ChatOpenAI(model_name="gpt-4o", temperature=0)
```

Y se usan en la cadena como `| prompt | llm`. Detalle importante: la `question` se usa **dos veces** —en Retrieval como base de la similarity search, y en Generation como **variable de entrada del prompt**.

## Prompting

Los **prompts son fundamentales** para toda la GenAI, no solo para RAG. En el code lab el prompt se trae del **[[LangChain Hub]]**, descrito como un lugar para:

> "discover, share, and version control prompts."

Está bien **empezar con prompts pre-diseñados** y luego escribir los propios. El prompt es el eslabón de la cadena **después** de Retrieval (en `rag_chain` aparece como `| prompt`). Las **entradas (inputs) del prompt = las salidas del paso anterior**. Se pueden inspeccionar:

```python
prompt = hub.pull("jclemens24/rag-prompt")
prompt.input_variables
```

Salida: `['context', 'question']` —coincide con las claves del diccionario de la cadena. Con `print(prompt)` se ve el objeto completo:

```python
input_variables=['context', 'question'] messages=[HumanMessagePromptTemplate(prompt=PromptTemplate(input_variables=['context', 'question'], template="You are an assistant for question-answering tasks. Use the following pieces of retrieved-context to answer the question. If you don't know the answer, just say that you don't know.\nQuestion: {question} \nContext: {context} \nAnswer:"))]
```

Desglose del objeto:
- **`input_variables`** — las variables que el template espera (`context`, `question`).
- **`messages[]`** — una lista de mensajes (acá uno solo).
- **`HumanMessagePromptTemplate`** — inicializado con un **[[Prompt Template|PromptTemplate]]** (que a su vez tiene `input_variables` + el string `template`).

El **template string** (verbatim, con los placeholders `{question}`/`{context}`):

```
You are an assistant for question-answering tasks. Use the following pieces of retrieved-context to answer the question. If you don't know the answer, just say that you don't know.
Question: {question} 
Context: {context} 
Answer:
```

> [!tip] El `Answer:` final, **sin nada después**, le indica al LLM que produzca una respuesta a continuación: es un patrón prevalente y efectivo en prompting. Y es buena práctica **arrancar desde prompts pre-construidos del [[LangChain Hub]]** antes de escribir los propios.

En resumen: un **prompt** = un objeto que se enchufa a la cadena con inputs que rellenan un template → el prompt resultante que se pasa al LLM para la inferencia (la etapa de preparación de la Generation).

## Definir el LLM

El **LLM es central** en todo el sistema; en `rag_chain` aparece como el eslabón `| llm`:

```python
llm = ChatOpenAI(model_name="gpt-4o", temperature=0)
```

Es una instancia de **`ChatOpenAI`** (de `langchain_openai`), usando GPT-4o. A los LLMs se los alimenta vía `invoke`; se los podría llamar **directamente**:

```python
llm_only = llm.invoke("Answering in less than 100 words,
    what are the Advantages of using RAG?")
print(llm_only.content)
```

El **ejemplo clave**: GPT-4o **conoce** RAG, pero **ChatGPT 3.5 NO**. Con GPT-3.5, la respuesta (verbatim) interpreta "RAG" como **"Red, Amber, Green"** (reporte de estado de proyectos):

```
RAG (Red, Amber, Green) status reporting allows for clear and straightforward communication of project progress or issues. It helps to quickly identify areas that need attention or improvement, enabling timely decision-making. RAG status also provides a visual representation of project health, making it easy for stakeholders to understand the current situation at a glance. Additionally, using RAG can help prioritize tasks and resources effectively, increasing overall project efficiency and success.'
```

> [!warning] **El corte de conocimiento como trampa.** El corte de ChatGPT 3.5 es **enero de 2022**; en ese momento el sentido GenAI de "RAG" todavía no era lo bastante popular, por eso lo confunde con "Red, Amber, Green". Esto **demuestra el valor de RAG**: agregás tus datos para cubrir lo que el modelo no sabe. Tip: probá la frase completa "Retrieval Augmented Generation (RAG)" o un modelo más nuevo con corte más reciente para una mejor respuesta.

Pero el punto real no es elegir un modelo más nuevo: es **pasar el prompt estructurado por Retrieval** en vez de llamar al LLM directamente → respuesta más informada. Se podría **terminar la cadena en el LLM**, pero su salida es un JSON con datos extra → por eso se la pipea a **`StrOutputParser()`**:

```python
StrOutputParser()
```

[[StrOutputParser]] es una utilidad de LangChain que **parsea la salida clave a un string** y descarta la info extra. La última línea pone todo en marcha:

```python
rag_chain.invoke("What are the Advantages of using RAG?")
```

donde la consulta se usa por **segunda vez** como variable de entrada del prompt. A futuro, la consulta vendrá de una **UI**.

## UI

La **UI (interfaz de usuario)** vuelve la app profesional y usable para usuarios que no ven el código: es el **punto primario de interacción**. Las interfaces avanzadas suman **[[NLU|natural language understanding (NLU)]]** —una forma de NLP enfocada en *entender*. En la práctica, se reemplaza la última línea `rag_chain.invoke("What are the Advantages of using RAG?")` por un **campo de entrada**, y se muestra la respuesta en una pantalla amigable; la UI en código se ve en el cap. [[06 - Interfacing with RAG and Gradio]]. Las interfaces van desde campos de texto hasta reconocimiento de voz; la clave es **capturar la intención del usuario en un formato procesable**. Una UI permite a los usuarios probar **cualquier** consulta.

### Pre-procesamiento

Incluso cuando el usuario solo escribe algo simple como `What is Task Decomposition?`, el **pre-procesamiento** vuelve la consulta más amigable para el LLM —mayormente en el prompt más otras funciones— todo **detrás de escena**.

### Post-procesamiento

La respuesta del LLM a menudo se **post-procesa** antes de mostrarse. La salida real del LLM (verbatim):

```python
AIMessage(content="The advantages of using RAG include improved accuracy and relevance of responses generated by large language models, customization and flexibility in responses tailored to specific needs, and expanding the model's knowledge beyond the initial training data.")
```

Tras pasar por `StrOutputParser()`, queda el string plano (verbatim):

```python
'The advantages of using RAG (Retrieval Augmented Generation) include improved accuracy and relevance, customization, flexibility, and expanding the model's knowledge beyond the training data. ...'
```

En una app profesional esto se mostraría **prolijamente en pantalla**, y podría incluir info extra como el **documento fuente** (el patrón de atribución del code lab del cap. [[03 - Practical Applications of RAG]]).

### Interfaz de salida

![[04-fig-4.7-chatgpt-ui.jpg]]
*Figure 4.7 – The ChatGPT 4 interface*

El string se pasa a la **interfaz de salida (display interface)**. Puede ser simple (como ChatGPT) o más robusta/conversacional (refinar consultas, hacer follow-ups, pedir más info). Algo común: **recolectar feedback** (utilidad/precisión) para mejorar continuamente —analizar las interacciones para entender mejor la intención, refinar la búsqueda vectorial y mejorar relevancia/calidad. Esto conduce a la evaluación.

> [!tip] Un **thumbs up/down** (pulgar arriba/abajo) es la forma más rápida de recolectar feedback a escala: simple para el usuario, masivo para el sistema.

## Evaluación

La **[[Evaluation|evaluación]]** es esencial para medir y mejorar el rendimiento; hay que enfocarse en **lo que más le importa a TUS usuarios**. Métricas: **accuracy (precisión), relevance (relevancia), response time (tiempo de respuesta) y user satisfaction (satisfacción del usuario)**. El feedback guía los ajustes al diseño, al manejo de datos y a la integración del LLM; la **evaluación continua** mantiene la calidad. El feedback del usuario puede ser:
- **Cualitativo (qualitative)** — formularios de entrada abiertos (open-ended).
- **Cuantitativo (quantitative)** — true/false, ratings, valores numéricos.

Un **thumbs up/down** da feedback rápido a escala. La evaluación en código se ve en el cap. [[09 - Evaluating RAG Quantitatively and with Visualizations]].

## Resumen del capítulo

No es una lista exhaustiva de componentes, sino los que están presentes en **todo sistema RAG exitoso**. Los sistemas RAG **evolucionan constantemente**: agregá los componentes que entreguen lo que **TUS usuarios** necesitan. Recap de las **3 etapas** (indexing offline → retrieval → generation) **+ UI + evaluación**. El próximo capítulo se dedica **enteramente a la seguridad** tal como se relaciona con RAG → cap. [[05 - Managing Security in RAG Applications]].

## Citas

> "discover, share, and version control prompts."

## Para aplicar

- **Tratá el indexing como pre-procesamiento offline** — construilo antes de que el usuario abra la app; retrieval y generation son los que corren en tiempo real.
- **Respetá el límite de 8191 tokens** de `OpenAIEmbeddings()` al chunkar; con `chunk_size` + `chunk_overlap`, sumá el overlap al chunk size para el total.
- **Usá chunk overlap** para no perder contexto entre chunks (p. ej. una dirección cortada a la mitad).
- **Arrancá desde prompts del LangChain Hub** (`hub.pull(...)`) y después escribí los propios; inspeccioná `prompt.input_variables` para verificar que casen con las claves de la cadena.
- **Pasá el prompt estructurado por Retrieval al LLM** en vez de llamarlo directo; pipeá la salida a `StrOutputParser()` para quedarte con un string limpio.
- **Verificá el corte de conocimiento del modelo** — si el modelo no conoce un término reciente (como RAG en GPT-3.5, corte enero 2022), es señal de que RAG aporta valor; probá un modelo con corte más nuevo.
- **Sumá una UI con un campo de entrada** y mostrá la respuesta post-procesada; recolectá feedback con un thumbs up/down.
- **Definí métricas de evaluación** centradas en tus usuarios (accuracy, relevance, response time, user satisfaction) y evaluá de forma continua.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[03 - Practical Applications of RAG]] — capítulo anterior: dónde aplicar RAG.
- [[05 - Managing Security in RAG Applications]] — capítulo siguiente: dedicado entero a la seguridad de RAG.
- [[02 - Code Lab - An Entire RAG Pipeline]] — el código que este capítulo diseca componente por componente; el setup (instalar/importar/cuentas OpenAI) vive allí.
- [[06 - Interfacing with RAG and Gradio]] · [[09 - Evaluating RAG Quantitatively and with Visualizations]] · [[11 - Using LangChain to Get More from RAG]] · [[07 - The Key Role Vectors and Vector Stores Play in RAG]] · [[08 - Similarity Searching with Vectors]] — capítulos que profundizan UI, evaluación, document loaders/splitters, vectores y similarity search.
- [[RAG]] · [[Indexing]] · [[Retrieval]] · [[Generation]] — las tres etapas + los componentes prácticos.
- [[Embeddings]] · [[Chroma DB]] · [[SemanticChunker]] · [[RecursiveCharacterTextSplitter]] · [[Tokens]] · [[Byte Pair Encoding]] · [[Chunk Overlap]] — el indexing y sus restricciones prácticas.
- [[LangChain]] · [[LangChain Hub]] · [[Prompt Template]] · [[Similarity Search]] · [[StrOutputParser]] — la orquestación de la cadena.
- [[NLU]] · [[Evaluation]] — la UI y la evaluación como componentes de primera clase. (Candidatos a nota propia en negrita si aún no existen: **[[Byte Pair Encoding]]** · **[[Chunk Overlap]]** · **[[NLU]]** · **[[StrOutputParser]]** · **[[Evaluation]]** · **[[RecursiveCharacterTextSplitter]]**.)
