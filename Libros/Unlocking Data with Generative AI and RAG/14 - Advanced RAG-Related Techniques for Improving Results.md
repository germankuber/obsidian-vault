---
title: "14 - Advanced RAG-Related Techniques for Improving Results"
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 14
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Advanced RAG-Related Techniques for Improving Results
  - Técnicas avanzadas de RAG
---

# 14 - Advanced RAG-Related Techniques for Improving Results

> [!info] Capítulo 14 (FINAL) · *Unlocking Data with Generative AI and RAG* — Keith Bourne, Packt (ISBN 9781835887905)
> El **último capítulo** del libro: técnicas avanzadas que van *más allá* del RAG fundamental. Repasa las tres técnicas ya usadas ([[Naive RAG]], [[Hybrid Search|hybrid/multi-vector RAG]] y [[Re-ranking]]), muestra dónde se quedan cortas, y suma tres code labs nuevos — [[Query Expansion]], [[Query Decomposition]] y [[MM-RAG]] (multi-modal) — cerrando con un catálogo de ~15 técnicas más. Código en github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_14. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[13 - Using Prompt Engineering to Improve RAG Efforts]] · (no hay siguiente — es el final).

## Resumen

Este es el **capítulo de cierre** del libro y su objetivo es elevar el RAG fundamental con técnicas avanzadas. Hasta acá se usaron tres enfoques: **naïve RAG** (el framework básico del cap. 2), **hybrid RAG** (multi-vector, del cap. 8) y **re-ranking** (también del cap. 8). El capítulo primero recuerda en qué consisten y por qué se quedan cortos, y luego agrega tres técnicas nuevas, cada una con su propio code lab: **query expansion** (lab 14.1 — amplía la *query* de entrada con un answer hipotético generado por el LLM, mejorando retrieval y generation), **query decomposition** (lab 14.2 — descompone la pregunta en sub-preguntas, con [[Chain-of-thought]] e [[IR-CoT]], y consolida; 100→67 docs deduplicados) y **MM-RAG / multi-modal RAG** (lab 14.3 — el último lab del libro, va más allá del texto: extrae texto+imágenes de un PDF con [[unstructured]]/[[OCR]], resume las imágenes con GPT-4o-mini y las recupera con un [[MultiVectorRetriever]]). Cierra con un catálogo de ~15 técnicas más agrupadas por etapa del pipeline ([[RAPTOR]], [[ColBERT]], [[HyDE]], step-back, cross-encoder re-ranking, [[RAG-Fusion]], [[Self-Reflective RAG]], [[Modular RAG]]…) y con la despedida del autor. Una idea transversal del capítulo: con query expansion y decomposition el **LLM entra en la etapa de retrieval** (antes solo aparecía en generation), así que el prompt engineering del cap. 13 ahora también pesa al recuperar.

## Naïve RAG y sus límites

El punto de partida son los tres enfoques de RAG que el libro ya venía usando: **naïve RAG**, **hybrid RAG** y **re-ranking**. El **[[Naive RAG]]** es el framework básico que integra retrieval + generation — exactamente el código de inicio del cap. 2 ([[02 - Code Lab - An Entire RAG Pipeline]]). Es simple pero de **flexibilidad y escalabilidad limitadas**.

> [!note] El naïve RAG recupera numerosos fragmentos de contexto **fragmentados**: cuanto más chicos son los chunks, mayor la fragmentación → menos contexto/captura semántica → recuperación menos efectiva.

Típicamente el naïve RAG usa **solo búsqueda semántica**, lo que lo expone a los límites de la semántica pura y es lo que motivó el paso a la **búsqueda híbrida**.

## Hybrid RAG / multi-vector RAG

El **[[Hybrid Search|hybrid RAG]]** expande el naïve RAG usando **múltiples vectores** en vez de una sola representación; por eso también se lo llama **multi-vector RAG**. Se exploró en el cap. 8 ([[08 - Similarity Searching with Vectors]]), tanto vía [[LangChain]] como recreado a mano. Puede combinar cualquier técnica de recuperación vectorial; en el lab se mezclaron **semantic + keyword**.

> [!note] La búsqueda **keyword** ayuda con contenido de **contexto débil**: nombres, códigos, acrónimos internos — donde la semántica sola falla.

Es especialmente útil en aplicaciones de **alta precisión**: redacción técnica, asistencia a investigación académica, documentación interna con referencias a código/entidades, y QA complejo.

## Re-ranking en hybrid RAG

El **[[Re-ranking]]** viene también del cap. 8: tras recuperar con semantic + keyword, se **re-rankean** los resultados según el ranking en ambos conjuntos — si un documento aparece en **ambos** y en qué posición quedó inicialmente en cada uno.

Hasta acá, entonces, **3 técnicas** (2 de ellas avanzadas). Este capítulo suma **3 más**: query expansion, query decomposition y MM-RAG.

## Code lab 14.1 – Query expansion

`CHAPTER14-1_QUERY_EXPANSION.ipynb`. La **[[Query Expansion|query expansion]]** mejora **tanto el retrieval como la generation**. A diferencia de la expansión del cap. 13 ([[13 - Using Prompt Engineering to Improve RAG Efforts]]), que actuaba sobre la **salida**, acá el objetivo es la **entrada**: aumentar el prompt original con keywords/frases extra → mejor comprensión para recuperar → mejor generación.

> [!note] El enfoque: enviar la query del usuario al LLM con un prompt para obtener una respuesta **inicial (hipotética)** SIN ningún contexto de RAG, y usarla para **ampliar el alcance** de la búsqueda.

Los imports:

```python
from langchain.prompts.chat import ChatPromptTemplate, HumanMessagePromptTemplate, SystemMessagePromptTemplate
```

- `ChatPromptTemplate` — combina templates al estilo chat.
- `HumanMessagePromptTemplate` — el mensaje humano, hidratado con la `user_query`.
- `SystemMessagePromptTemplate` — el mensaje de sistema.

El system message (verbatim): "You are a helpful expert environmental research assistant. Provide an example answer to the given question, that might be found in a document like an annual environmental report."

La función que genera el answer hipotético:

```python
def augment_query_generated(user_query):
    system_message_prompt = SystemMessagePromptTemplate.from_template(
        "You are a helpful expert environmental research assistant. Provide an example answer to the given question, that might be found in a document like an annual environmental report."
    )
    human_message_prompt = HumanMessagePromptTemplate.from_template("{query}")
    chat_prompt = ChatPromptTemplate.from_messages([
        system_message_prompt, human_message_prompt])
    response = chat_prompt.format_prompt(query=user_query).to_messages()
    result = llm(response)
    content = result.content
    return content
```

El código que arma el `joint_query` = query original + answer hipotético:

```python
original_query = "What are Google's environmental initiatives?"
hypothetical_answer = augment_query_generated(original_query)
joint_query = f"{original_query} {hypothetical_answer}"
print(joint_query)
```

Salida (excerpt verbatim):

```
What are Google's environmental initiatives?
In 2022, Google continued to advance its environmental initiatives… 1. **Carbon Neutrality and Renewable Energy**: …carbon-neutral status since 2007… 24/7 carbon-free energy by 2030… over 7 gigawatts of renewable energy… 2. **Data Center Efficiency**: …average power usage effectiveness (PUE) of 1.10… 3. **Sustainable Products and Services**… [TRUNCATED]
```

El answer **imaginado** explota los conceptos internos del LLM que se alinean con la query. Luego se pasa el `joint_query` por el pipeline ya existente:

```python
result_alt = rag_chain_with_source.invoke(joint_query)
retrieved_docs_alt = result_alt['context']
print(f"Original Question: {joint_query}\n")
print(f"Relevance Score: {result_alt['answer']['relevance_score']}\n")
print(f"Final Answer:\n{result_alt['answer']['final_answer']}\n\n")
print("Retrieved Documents:")
for i, doc in enumerate(retrieved_docs_alt, start=1):
    print(f"Document {i}: Document ID: {doc.metadata['id']} source: {doc.metadata['search_source']}")
    print(f"Content:\n{doc.page_content}\n")
```

El render del answer final como Markdown:

```python
from IPython.display import Markdown, display
markdown_text_alt = result_alt['answer']['final_answer']
display(Markdown(markdown_text_alt))
```

Salida (excerpt verbatim):

```
Google has implemented a comprehensive set of environmental initiatives… **1. Carbon Neutrality and Renewable Energy**… carbon-neutral since 2007… 24/7 carbon-free energy by 2030… over 7 gigawatts… **2. Data Center Efficiency**… PUE of 1.10… [TRUNCATED]… **3. Supplier Engagement**… **4. Technological Innovations**: next-generation geothermal power and battery-based backup power systems…
```

> [!warning] Caveat clave: la query expansion mete al **LLM dentro de la etapa de retrieval** (antes solo aparecía en generation). Por eso el **prompt engineering ahora también pesa en el retrieval**, no solo en la generación — conecta directo con el cap. 13 ([[13 - Using Prompt Engineering to Improve RAG Efforts]]).

Paper: arxiv.org/abs/2305.03653.

## Code lab 14.2 – Query decomposition

`CHAPTER14-2_DECOMPOSITION.ipynb`. La **[[Query Decomposition|query decomposition]]** mejora el question-answering y cae bajo el paraguas de **query translation** (mejora el retrieval). La idea: **descomponer** una pregunta en preguntas más chicas (secuenciales o independientes); un paso de **consolidación** da una respuesta final más amplia que la del naïve RAG. Otros enfoques de query translation: **RAG-Fusion** y **multi-query** (sub-questions), tratados más adelante. El paper de Google lo llama **Least-to-Most** / decomposition; LangChain lo llama query decomposition.

### CoT e IR-CoT

Dos conceptos clave:

> [!note] **[[Chain-of-thought]] (CoT)** = estructurar el prompt para imitar el razonamiento humano → mejores resultados en tareas de lógica, cálculo y decisión. **Interleaving retrieval** = alternar (ida y vuelta) entre prompts CoT y retrieval para recuperar info más relevante para el razonamiento posterior. Combinados = **[[IR-CoT]]** (Interleave Retrieval with CoT).

Secuencialmente: se rompe la query en sub-preguntas → se responde Q1 (recuperando docs), luego se recupera para Q2 **agregando la respuesta de Q1**, … hasta que la última respuesta = la respuesta final.

### La decompose chain

El import:

```python
from langchain.load import dumps, loads
```

(Serializa/deserializa un objeto Python desde/hacia un string — se usa para convertir cada `Document` a string antes del dedup, y de vuelta.)

El prompt de descomposición (verbatim):

```python
prompt_decompose = PromptTemplate.from_template(
    """You are an AI language model assistant.
    Your task is to generate five different versions of
    the
    given user query to retrieve relevant documents from a
    vector search. By generating multiple perspectives on
    the user question, your goal is to help the user
    overcome some of the limitations of the distance-based
    similarity search.  Provide these alternative
    questions
    separated by newlines.
    Original question: {question}"""
)
```

La cadena:

```python
decompose_queries_chain = (
    prompt_decompose
    | llm
    | str_output_parser
    | (lambda x: x.split("\n"))
)
```

Invocar + imprimir:

```python
decomposed_queries = decompose_queries_chain.invoke({"question": user_query})
print("Five different versions of the user query:")
print(f"Original: {user_query}")
for i, question in enumerate(decomposed_queries, start=1):
    print(f"{question.strip()}")
```

Salida (verbatim — la original + las 5 versiones):

```
What steps is Google taking to address environmental concerns?
How is Google contributing to environmental sustainability?
Can you list the environmental programs and projects Google is involved in?
What actions has Google implemented to reduce its environmental impact?
What are the key environmental strategies and goals of Google?
```

### Dedup + retrieval chain

La función de deduplicación:

```python
def format_retrieved_docs(documents: list[list]):
    flattened_docs = [dumps(doc) for sublist in documents for doc in sublist]
    print(f"FLATTENED DOCS: {len(flattened_docs)}")
    deduped_docs = list(set(flattened_docs))
    print(f"DEDUPED DOCS: {len(deduped_docs)}")
    return [loads(doc) for doc in deduped_docs]
```

Aplana la lista-de-listas y deduplica vía `dumps`→`set`→`loads`. Salida: **FLATTENED DOCS: 100** → **DEDUPED DOCS: 67**.

La retrieval chain:

```python
retrieval_chain = (
    decompose_queries_chain
    | ensemble_retriever.map()
    | format_retrieved_docs
)
```

```python
docs = retrieval_chain.invoke({"question":user_query})
```

→ **67 documentos**. Se reemplaza la cadena ensemble por `retrieval_chain`:

```python
rag_chain_with_source = RunnableParallel(
    {"context": retrieval_chain,
     "question": RunnablePassthrough()}
).assign(answer=rag_chain_from_docs)
```

Salida (excerpt verbatim):

```
Google has implemented a wide range of environmental initiatives… **1. Campus and Habitat Restoration**: …40 acres of habitat… ~4,000 native trees… oak woodlands, willow groves, wetland habitats. **2. Carbon-Free Energy**: …net-zero emissions and 24/7 carbon-free energy (CFE) by 2030… **3. Water Stewardship**… [TRUNCATED]
```

La cobertura es **más amplia** que la del naïve RAG. Paper: arxiv.org/abs/2205.10625.

## Code lab 14.3 – MM-RAG (Multi-Modal RAG)

`CHAPTER14-3_MM_RAG.ipynb`. El **último code lab del libro**: ir más allá del texto hacia **imágenes, video y audio**.

### Multi-modal y modality independence

> [!note] **[[Multimodal|Multi-modal]]** = manejar múltiples "modos" (texto, imágenes, video, audio…) — en la entrada, la salida, o ambas. Ej.: text→image, image→text (captioning); avanzado: un prompt "turn this image into a video that goes further into the waterfall adding the sounds of the waterfall" + una imagen → video + audio (4 modos).

Beneficios: salidas más ricas y context-rich; agentes conversacionales con multimedia; herramientas de creación de contenido; refleja la experiencia sensorial/cognitiva humana.

Sobre los **embeddings multi-modales**: los vector embeddings pueden representar **cualquier** dato, no solo texto; la vectorización es una representación matemática, el lenguaje de los modelos de **[[Deep Learning]] (DL)**. En el espacio vectorial los conceptos similares quedan más cerca; el concepto "seagull" debería representarse de forma similar sea la **palabra**, una **imagen**, un **video** o un **clip de audio** — esa igualdad cross-modal es la **[[Modality Independence|modality independence]]**.

> [!note] Lo clave: los embeddings multi-modales **preservan la similitud semántica a través de todas las modalidades**.

Y las imágenes empresariales **no son solo "fotos"**: son charts, flowcharts, texto convertido a imagen; el PDF del reporte ambiental de Google tiene imágenes bien diseñadas + charts que son imágenes.

### `unstructured`: extracción

Los **5 pasos** de MM-RAG: (1) extraer texto + imágenes de un PDF con el paquete open-source **[[unstructured]]**; (2) usar un LLM multi-modal para **resumir** las imágenes extraídas; (3) embeddear + recuperar los resúmenes de imagen (junto con el texto) con una referencia a la imagen cruda; (4) guardar los resúmenes de imagen en el multi-vector retriever con Chroma (texto crudo + imágenes + sus resúmenes); (5) pasar imágenes crudas + chunks de texto al LLM multi-modal para sintetizar la respuesta.

Instalación:

```bash
%pip install "unstructured[pdf]"
%pip install pillow
%pip install pydantic
%pip install lxml
%pip install matplotlib
%pip install tiktoken
!sudo apt-get -y install poppler-utils
!sudo apt-get -y install tesseract-ocr
```

- `unstructured[pdf]` — extrae info estructurada de PDFs/imágenes/HTML (acá solo soporte PDF).
- `pillow` — fork de PIL; formatos de imagen.
- `pydantic` — validación de datos vía type annotations.
- `lxml` — procesamiento de XML/HTML.
- `matplotlib` — plotting.
- `tiktoken` — tokenizer **BPE (Byte-Pair Encoding)** para modelos OpenAI.
- `poppler-utils` — herramientas CLI de PDF, usadas por unstructured para la extracción.
- `tesseract-ocr` — motor **[[OCR]] (optical character recognition)**, saca texto de las imágenes.

Los imports:

```python
from langchain.retrievers.multi_vector import MultiVectorRetriever
from langchain_community.document_loaders import UnstructuredPDFLoader
from langchain_core.runnables import RunnableLambda
from langchain.storage import InMemoryStore
from langchain_core.messages import HumanMessage
import base64
import uuid
from IPython.display import HTML, display
from PIL import Image
import matplotlib.pyplot as plt
```

- `MultiVectorRetriever` — combina vectorstore + docstore.
- `UnstructuredPDFLoader` — extrae texto+imágenes de un PDF vía unstructured.
- `RunnableLambda` — envuelve una función como runnable en una cadena (envuelve `split_image_text_types` + `img_prompt_func`).
- `InMemoryStore` — docstore key-value en memoria.
- `HumanMessage` — (del 14.1) para los prompts de resumen de imagen.
- `base64` — codifica imágenes como base64.
- `uuid` — **UUIDs** (universally unique identifiers) para los IDs de doc.
- `HTML`/`display` — muestran imágenes base64 en el notebook.
- `Image` de PIL; `matplotlib.pyplot as plt`.

Variables — **GPT-4o-mini** (la "o" = **omni** = multi-modal):

```python
llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)
short_pdf_path = "google-2023-environmental-report-short.pdf"
embedding_function = OpenAIEmbeddings()
```

> [!warning] `OpenAIEmbeddings` **NO** soporta embeddings multi-modales (no va a embeddear la imagen de una gaviota ≈ la palabra "seagull"). El workaround: embeddear la **descripción** de la imagen, no la imagen en sí — sigue siendo multi-modal.

> [!tip] Usá un **PDF corto** (`...-short.pdf`) y **reducí las imágenes** para ahorrar costo de API.

El PDF loader:

```python
pdfloader = UnstructuredPDFLoader(
    short_pdf_path,
    mode="elements",
    strategy="hi_res",
    extract_image_block_types=["Image","Table"],
    extract_image_block_to_payload=True,
    # converts images to base64 format
)
pdf_data = pdfloader.load()
```

- `mode="elements"` — extrae elementos individuales (bloques de texto, imágenes).
- `strategy="hi_res"` — identifica el layout del documento para más info; otras opciones: `auto`/`fast`/`ocr_only` (`fast` es más rápido pero menos efectivo).
- `extract_image_block_types=["Image","Table"]` — qué tipos de elemento extraer como base64.
- `extract_image_block_to_payload=True` — incluye las imágenes extraídas como base64 en la metadata (sin guardar archivos).

> [!note] `strategy="hi_res"` identifica el layout del documento para extraer más información que `fast`; tarda 1–5 minutos.

Explorar la data (NOTA: el libro tiene una errata en la condición — `== NarrativeText"]` sin la comilla de apertura; lo correcto sería `== "NarrativeText"]`):

```python
texts = [doc for doc in pdf_data if doc.metadata["category"] == "NarrativeText"]
images = [doc for doc in pdf_data if doc.metadata["category"] == "Image"]
print(f"TOTAL DOCS USED BEFORE REDUCTION: texts: {len(texts)} images: {len(images)}")
categories = set(doc.metadata["category"] for doc in pdf_data)
print(f"CATEGORIES REPRESENTED: {categories}")
```

Salida:

```
TOTAL DOCS USED BEFORE REDUCTION: texts: 78 images: 17
CATEGORIES REPRESENTED: {'ListItem', 'Title', 'Footer', 'Image', 'Table', 'NarrativeText', 'FigureCaption', 'Header', 'UncategorizedText'}
```

> [!tip] Una app robusta podría usar también `Title`/`Footer`/`Header` y otros tipos de elemento.

Reducir las imágenes para ahorrar costo:

```python
if len(images) > 3:
    images = images[:3]
print(f"total documents after reduction: texts: {len(texts)} images: {len(images)}")
```

→ `total documents after reduction: texts: 78 images: 3` (3 imágenes cuestan ~6× menos que 17).

### Resúmenes de imágenes

La función de prompt de resumen de imagen:

```python
def apply_prompt(img_base64):
    # Prompt
    prompt = """You are an assistant tasked with summarizing images for retrieval. \
    These summaries will be embedded and used to retrieve the raw image. \
    Give a concise summary of the image that is well optimized for retrieval."""
    return [HumanMessage(content=[
        {"type": "text", "text": prompt},
        {"type": "image_url","image_url": {"url": f"data:image/jpeg;base64,{img_base64}"},},
    ])]
```

La imagen ya está en base64 en la metadata (gracias a `extract_image_block_to_payload`). Preparar los resúmenes:

```python
text_summaries = [doc.page_content for doc in texts]
# Store base64 encoded images, image summaries
img_base64_list = []
image_summaries = []
for img_doc in images:
    base64_image = img_doc.metadata["image_base64"]
    img_base64_list.append(base64_image)
    message = llm.invoke(apply_prompt(base64_image))
    image_summaries.append(message.content)
```

Los textos se toman **directamente** como sus propios resúmenes (no se resumen — para ahorrar costo).

### Multi-vector retriever

El vector store:

```python
vectorstore = Chroma(
    collection_name="mm_rag_google_environmental",
    embedding_function=embedding_function
)
```

El multi-vector retriever:

```python
store = InMemoryStore()
id_key = "doc_id"
retriever_multi_vector = MultiVectorRetriever(
    vectorstore=vectorstore,
    docstore=store,
    id_key=id_key,
)
```

`InMemoryStore` = docstore con el contenido real por doc ID; `id_key="doc_id"` funciona como una **foreign key** entre los dos stores.

El helper `add_documents`:

```python
def add_documents(retriever, doc_summaries, doc_contents):
    doc_ids = [str(uuid.uuid4()) for _ in doc_contents]
    summary_docs = [
        Document(page_content=s, metadata={id_key: doc_ids[i]})
        for i, s in enumerate(doc_summaries)
    ]
    content_docs = [
        Document(page_content=doc.page_content, metadata={id_key: doc_ids[i]})
        for i, doc in enumerate(doc_contents)
    ]
    retriever.vectorstore.add_documents(summary_docs)
    retriever.docstore.mset(list(zip(doc_ids, doc_contents)))
```

Aplicarlo:

```python
if text_summaries:
    add_documents(retriever_multi_vector, text_summaries, texts)
if image_summaries:
    add_documents(retriever_multi_vector, image_summaries, images)
```

El helper `split_image_text_types`:

```python
def split_image_text_types(docs):
    b64_images = []
    texts = []
    for doc in docs:
        if isinstance(doc, Document):
            if doc.metadata.get("category") == "Image":
                base64_image = doc.metadata["image_base64"]
                b64_images.append(base64_image)
            else:
                texts.append(doc.page_content)
        else:
            if isinstance(doc, str):
                texts.append(doc)
    return {"images": b64_images, "texts": texts}
```

El helper `img_prompt_func`:

```python
def img_prompt_func(data_dict):
    formatted_texts = "\n".join(data_dict["context"]["texts"])
    messages = []
    if data_dict["context"]["images"]:
        for image in data_dict["context"]["images"]:
            image_message = {"type": "image_url",
                "image_url": {"url": f"data:image/jpeg;base64,{image}"}}
            messages.append(image_message)
    text_message = {
        "type": "text",
        "text": (
            f"""You are are a helpful assistant tasked with describing what is in an image. The user will ask for a picture of something.  Provide text that supports what was asked for. Use this information to provide an in-depth description of the aesthetics of the image. Be clear and concise and don't offer any additional commentary.
User-provided question: {data_dict['question']}
Text and / or images: {formatted_texts}"""
        ),
    }
    messages.append(text_message)
    return [HumanMessage(content=messages)]
```

### La cadena MM-RAG

```python
chain_multimodal_rag = ({"context": retriever_multi_vector
    | RunnableLambda(split_image_text_types),
    "question": RunnablePassthrough()}
    | RunnableLambda(img_prompt_func)
    | llm
    | str_output_parser
)
```

Los 4 componentes: (1) `context` = el retriever multi-vector `| split_image_text_types`, con `question` por passthrough; (2) `img_prompt_func` arma el mensaje multi-modal; (3) el `llm` (GPT-4o-mini); (4) `str_output_parser`.

Invocar:

```python
user_query = "Picture of multiple wind turbines in the ocean."
chain_multimodal_rag.invoke(user_query)
```

Salida (verbatim):

```
'The image shows a vast array of wind turbines situated in the ocean, extending towards the horizon. The turbines are evenly spaced and stand tall above the water, with their large blades capturing the wind to generate clean energy. The ocean is calm and blue, providing a serene backdrop to the white turbines. The sky above is clear with a few scattered clouds, adding to the tranquil and expansive feel of the scene. The overall aesthetic is one of modernity and sustainability, highlighting the use of renewable energy sources in a natural setting.'
```

El helper para ver la imagen:

```python
def plt_img_base64(img_base64):
    image_html = f'<img src="data:image/jpeg;base64,{img_base64}" />'
    display(HTML(image_html))
plt_img_base64(img_base64_list[1])
```

El resumen de imagen correspondiente (`image_summaries[1]`, verbatim):

```
'Offshore wind farm with multiple wind turbines in the ocean, text "What\'s inside" on the left side.'
```

> [!note] La descripción de MM-RAG es **más rica** que el resumen → el LLM realmente puede "ver" la imagen. "You are officially multi-modal!"

## Otras técnicas avanzadas

El capítulo cierra con un catálogo de técnicas adicionales, agrupadas por etapa del pipeline. Fuente de técnicas nuevas: arxiv.org (buscar *RAG*, *retrieval augmented generation*, *vector search*).

### Indexing

- **Deep chunking** — usar modelos de DL/transformers para un chunking inteligente.
- **Training and utilizing embedding adapters** — módulos livianos que adaptan embeddings pre-existentes a una tarea/dominio sin re-entrenar todo.
- **Multi-representation indexing (proposition indexing)** — un LLM produce resúmenes/proposiciones del documento optimizados para retrieval.
- **[[RAPTOR]]** (Recursive Abstractive Processing for Tree Organized Retrieval) — maneja hechos low-level de un solo doc Y preguntas high-level cross-doc; embebe+clusteriza docs, resume cada cluster recursivamente → un **árbol de resúmenes** con conceptos cada vez más high-level, indexado junto con los docs de partida.
- **[[ColBERT]]** (Contextualized Late Interaction over BERT) — similitud **token-a-token** de mayor granularidad entre documento y query, vs un único vector de largo fijo que diluye el matiz semántico.

### Retrieval

La categoría más grande:

- **[[HyDE]]** (Hypothetical Document Embeddings) — genera un documento hipotético desde el conocimiento del LLM, lo embebe y recupera — mejor alineado con los docs del índice que la pregunta cruda.
- **Sentence-window retrieval** — recupera sobre oraciones más chicas, sintetiza sobre una ventana expandida alrededor de la oración.
- **Auto-merging retrieval** — fusiona chunks chicos en un chunk padre más grande para arreglar la fragmentación.
- **Multi-query rewriting** — reescribe la pregunta desde múltiples perspectivas, recupera cada una y toma la unión única.
- **Query translation step-back** — step-back prompting: genera una pregunta más high-level/abstracta como precondición; construye sobre CoT; útil cuando el conocimiento de fondo ayuda.
- **Query structuring (text-to-DSL)** — convierte las preguntas del usuario en queries estructuradas (DSL = domain-specific language).

### Post-retrieval/generación

- **[[Cross-encoder Re-ranking]]** — un modelo más costoso computacionalmente reevalúa/reordena los docs recuperados por relevancia antes de generar.
- **RAG-fusion query rewriting** ([[RAG-Fusion]]) — reescribe desde múltiples perspectivas, recupera cada una y aplica **[[Reciprocal Rank Fusion|reciprocal rank fusion]]** → un ranking consolidado.

### Pipeline completo

- **[[Self-Reflective RAG]]** — mecanismo de auto-reflexión + grafo lingüístico [[LangGraph]] → refina respuestas vía contexto/semántica más profundos; bueno para creación de contenido, QA y agentes conversacionales.
- **[[Modular RAG]]** — componentes intercambiables → arquitectura flexible; [[LangChain]] permite swap-ear LLMs/retrievers/vector stores → customizable, eficiente, potente.

### Tabla 14.1 — Técnicas avanzadas por etapa

| Técnica | Etapa | Qué hace |
|---|---|---|
| Deep chunking | Indexing | Chunking inteligente con modelos de DL/transformers. |
| Embedding adapters | Indexing | Módulos livianos que adaptan embeddings a tarea/dominio sin re-entrenar. |
| Multi-representation (proposition) indexing | Indexing | El LLM genera resúmenes/proposiciones optimizadas para retrieval. |
| [[RAPTOR]] | Indexing | Árbol recursivo de resúmenes de clusters; cubre hechos low-level y high-level. |
| [[ColBERT]] | Indexing | Similitud token-a-token de alta granularidad vs un vector fijo. |
| [[HyDE]] | Retrieval | Embebe un documento hipotético del LLM en vez de la query cruda. |
| Sentence-window retrieval | Retrieval | Recupera por oración, sintetiza sobre una ventana expandida. |
| Auto-merging retrieval | Retrieval | Fusiona chunks chicos en un chunk padre (anti-fragmentación). |
| Multi-query rewriting | Retrieval | Múltiples perspectivas de la pregunta → unión única. |
| Step-back | Retrieval | Pregunta más abstracta como precondición (sobre CoT). |
| Query structuring | Retrieval | Text-to-DSL: pregunta → query estructurada. |
| [[Cross-encoder Re-ranking]] | Post-retrieval | Modelo costoso reordena los docs por relevancia. |
| [[RAG-Fusion]] | Post-retrieval | Multi-perspectiva + reciprocal rank fusion → ranking consolidado. |
| [[Self-Reflective RAG]] | Pipeline completo | Auto-reflexión + LangGraph para refinar respuestas. |
| [[Modular RAG]] | Pipeline completo | Componentes intercambiables → arquitectura flexible (LangChain). |

> [!tip] No hay una técnica "ganadora" universal: cada una ataca una etapa distinta del pipeline. Para descubrir nuevas, explorá **arxiv.org** (RAG, retrieval augmented generation, vector search).

## Citas

> "You are officially multi-modal!"
> "It has been a pleasure to go on this RAG journey with you… Good luck in your future RAG endeavors…"

## Para aplicar

- **Query expansion** — antes de recuperar, pedile al LLM un answer hipotético SIN contexto y concatenalo a la query (`joint_query`) para ampliar el alcance de la búsqueda. Recordá: ahora el prompt engineering pesa también en el retrieval.
- **Query decomposition** — generá ~5 versiones/sub-preguntas de la query, recuperá cada una con `ensemble_retriever.map()`, aplaná y deduplicá con `dumps`→`set`→`loads` (100→67 docs) antes de generar.
- **MM-RAG** — usá un **PDF corto** y **reducí imágenes** (a 3) para ahorrar costo; extraé texto+imágenes con `unstructured` (`strategy="hi_res"`, `extract_image_block_to_payload=True`), resumí las imágenes con GPT-4o-mini, guardalas en un `MultiVectorRetriever` (Chroma + `InMemoryStore`) y embeddeá la **descripción** (no la imagen — OpenAIEmbeddings no es multi-modal).
- **Probá `fast` vs `hi_res`** en `UnstructuredPDFLoader` según el trade-off velocidad/calidad; explorá también los elementos `Title`/`Footer`/`Header`.
- **Catálogo de técnicas** — para mejorar un pipeline RAG existente, elegí la técnica por etapa (indexing/retrieval/post-retrieval/pipeline) y explorá arxiv para lo más reciente.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro (capítulo **final**).
- [[13 - Using Prompt Engineering to Improve RAG Efforts]] — capítulo anterior; la query expansion mete el prompt engineering en el retrieval, y la expansión de salida del cap. 13 contrasta con la expansión de entrada de acá.
- [[02 - Code Lab - An Entire RAG Pipeline]] — el [[Naive RAG]] base que este capítulo extiende.
- [[08 - Similarity Searching with Vectors]] — origen del [[Hybrid Search|hybrid/multi-vector RAG]], el [[Re-ranking]], [[Reciprocal Rank Fusion|RRF]] y el [[EnsembleRetriever]] que reusan los labs.
- [[07 - The Key Role Vectors and Vector Stores Play in RAG]] — vectores y espacio vectorial, base de la [[Modality Independence|modality independence]] multi-modal.
- [[12 - Combining RAG with the Power of AI Agents and LangGraph]] — [[LangGraph]] sustenta la [[Self-Reflective RAG]] y el [[Modular RAG]].
- [[Query Expansion]] · [[Query Decomposition]] · [[MM-RAG]] · [[Multimodal]] · [[Modality Independence]] — las técnicas nuevas del capítulo.
- [[Chain-of-thought]] · [[IR-CoT]] — el razonamiento que sustenta la query decomposition.
- [[HyDE]] · [[RAPTOR]] · [[ColBERT]] · [[Cross-encoder Re-ranking]] · [[RAG-Fusion]] · [[Self-Reflective RAG]] · [[Modular RAG]] — el catálogo de técnicas avanzadas (candidatas a nota propia).
- [[unstructured]] · [[OCR]] · [[MultiVectorRetriever]] · [[Deep Learning]] · [[LangChain]] — piezas técnicas del lab MM-RAG.
