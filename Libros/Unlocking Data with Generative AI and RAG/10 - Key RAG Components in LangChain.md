---
title: 10 - Key RAG Components in LangChain
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 10
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Key RAG Components in LangChain
  - Componentes clave de RAG en LangChain
updated: 2026-06-12
---

# 10 - Key RAG Components in LangChain

> [!info] Capítulo 10 · *Unlocking Data with Generative AI and RAG* — Packt (ISBN 9781835887905) · Keith Bourne
> Una mirada en profundidad a los **componentes técnicos clave de RAG en [[LangChain]]**, recorridos en su orden de uso: **[[Vector Store|vector stores]] → [[Retriever|retrievers]] → [[LLM|LLMs]]**. La tesis operativa del capítulo: **LangChain te deja intercambiar (swap) cualquier componente sin reescribir el resto del pipeline**. Construye sobre el último código del [[08 - Similarity Searching with Vectors]] (code lab 8.3), salteando la evaluación del [[09 - Evaluating RAG Quantitatively and with Visualizations]]. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[09 - Evaluating RAG Quantitatively and with Visualizations]] · siguiente [[11 - Using LangChain to Get More from RAG]].

## Resumen

El capítulo abre la última parte técnica del libro entrando de lleno en los **componentes de RAG que provee [[LangChain]]**, recorridos en el mismo orden en que se usan en un pipeline: primero los **[[Vector Store|vector stores]]** (dónde viven los vectores), luego los **[[Retriever|retrievers]]** (cómo se consulta el store) y finalmente los **[[LLM|LLMs]]** (la etapa de generación). El hilo conductor es la **modularidad**: LangChain estandariza cada componente detrás de una interfaz común, de modo que se puede **swap-ear** un vector store por otro, un retriever por otro o un LLM por otro **sin tocar el resto del código** — la gran fortaleza del framework. El capítulo es eminentemente práctico, con **tres code labs** que reutilizan el código del **code lab 8.3** (`[[08 - Similarity Searching with Vectors]]`, el `EnsembleRetriever`) y se saltean el código de evaluación del cap. 9. Todo el código vive en github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_10.

El **code lab 10.1** muestra cómo LangChain integra **[[Chroma DB|Chroma]]**, **[[Weaviate]]**, **[[FAISS]]**, **[[pgvector]]** y **[[Pinecone]]** detrás de una única **vector store class**, y reescribe el mismo store con tres backends distintos (Chroma → FAISS → Weaviate), donde Weaviate exige un esquema explícito al estilo **[[GraphQL]]**. El **code lab 10.2** recorre la familia de **retrievers**: el básico (`as_retriever`) sobre embeddings densos, sus variantes (`similarity_score_threshold` y **[[MMR]]**), el **[[BM25]]** sparse, el **[[EnsembleRetriever]]** híbrido (con su parámetro `c` de reranking), el **[[WikipediaRetriever]]** y otros sobre fuentes públicas, y el **[[k-NN]]** — con la tesis contracorriente de que kNN, pese a ser de **1951**, sigue siendo más preciso que **[[ANN]]** para datasets de hasta ~1M de puntos. El **code lab 10.3** cubre los **LLMs**: **[[OpenAI]]** (`gpt-4o-mini`) frente a **[[Together AI]]** con **[[Llama 3]]** 70B y **[[Mixtral]]** 8x22B a una fracción del costo, y cierra con la interfaz **[[Runnable]]** común a todos los LLMs (**async / stream / batch**). El cap. 11 seguirá con los componentes **más chicos** de LangChain que dan soporte a estos.

## Code lab 10.1 – Vector stores en LangChain

Un **[[Vector Store|vector store]]** almacena e indexa las representaciones vectoriales de la knowledge base. [[LangChain]] integra muchos de ellos — **[[Chroma DB|Chroma]], [[Weaviate]], [[FAISS]] (Facebook AI Similarity Search), [[pgvector]] y [[Pinecone]]**, entre otros — detrás de una **vector store class** unificada: una interfaz común para **agregar documentos, hacer similarity search y recuperar**. Esa abstracción es lo que permite **cambiar de backend sin cambiar la lógica de retrieval**.

> [!note] La **vector store class** de LangChain es una interfaz unificada (add docs · similarity search · retrieve) que abstrae el backend concreto. Por eso podés swap-ear el store entero y el resto del pipeline sigue funcionando.

La elección del store depende de **escalabilidad, performance de búsqueda y modo de despliegue**: **[[Pinecone]]** es fully managed, escalable y en tiempo real → orientado a **producción**; **[[FAISS]]** es open source, ideal para **desarrollo local y experimentación**; **[[Chroma DB|Chroma]]** es un punto de partida popular por su **facilidad de uso** e integración con LangChain. LangChain llama a estos componentes **integrations**: en la nav **Integrations** del sitio, la sección **Providers** lista Partner Packages, Featured Community Providers y un "Click here to see all providers", mientras que **Components** lista los **Vector stores** — actualmente **49** opciones (v0.2.0).

> [!tip] La fortaleza central de LangChain en este capítulo: podés **intercambiar cualquier vector store** (Chroma ↔ FAISS ↔ Weaviate ↔ pgvector ↔ Pinecone) y el resto del código de retrieval sigue funcionando sin cambios.

### Chroma

**[[Chroma DB|Chroma]]** es una base de datos vectorial **AI-native** y open source. Sus modos de despliegue son **in-memory**, **persistente** y **containerizado (Docker)**. Ofrece una API simple y dev-friendly (add / update / delete / query), **filtrado dinámico por metadata**, y **chunking + indexing built-in**. Su arquitectura combina capas de **indexing + storage + processing**, y soporta **[[MMR]] (maximum marginal relevance)** además del filtrado por metadata. El snippet actual que crea el store con Chroma:

```python
chroma_client = chromadb.Client()
collection_name = "google_environmental_report"
vectorstore = Chroma.from_documents(
    documents=dense_documents,
    embedding=embedding_function,
    collection_name=collection_name,
    client=chroma_client
)
```

### FAISS

**[[FAISS]]** (Facebook AI Similarity Search) es open source, de **Facebook AI**, de alta performance, y maneja **datasets grandes que no entran en memoria**. Tiene capas de indexing / storage / y una capa de processing opcional; usa indexing por **clustering + cuantización**; y soporta **aceleración por GPU**. Para usarlo, primero instalar (y **reiniciar el kernel**):

```python
%pip install faiss-cpu
```

Y reemplazar Chroma con:

```python
from langchain_community.vectorstores import FAISS
vectorstore = FAISS.from_documents(
    documents=dense_documents,
    embedding=embedding_function
)
```

Los parámetros `collection_name` y `client` **no aplican** a FAISS. Para datasets grandes, la GPU paraleliza las comparaciones de vectores → mucho más rápido:

```python
%pip install faiss-gpu
```

> [!tip] Para escalar FAISS a grandes volúmenes, usá `faiss-gpu`: la aceleración por GPU paraleliza las comparaciones de vectores y acelera enormemente la búsqueda a gran escala. Acordate de **reiniciar el kernel** tras instalar.

### Weaviate

Se usa la versión **EMBEDDED** de **[[Weaviate]]**: corre desde el código de la app (no como servidor standalone) y crea un **datastore permanente** en `persistence_data_path` — los datos persisten incluso después de que el cliente termina. Weaviate está inspirado en **[[GraphQL]]** (API RESTful + un lenguaje de consulta tipo GraphQL, con tipos de datos predefinidos: string, int, number, Boolean, date…) y soporta **operaciones por lotes (batch)** vía `client.batch`. Instalación:

```bash
%pip install weaviate-client
%pip install langchain-weaviate
```

Imports (el texto del libro dice por error "install FAISS" — ignorarlo; `tqdm` se necesita para las barras de progreso de carga de Weaviate):

```python
import weaviate
from langchain_weaviate.vectorstores import WeaviateVectorStore
from weaviate.embedded import EmbeddedOptions
from langchain.vectorstores import Weaviate
from tqdm import tqdm
```

Crear el cliente embedded:

```python
weaviate_client = weaviate.Client(embedded_options=EmbeddedOptions())
```

Limpiar cualquier esquema remanente:

```python
try:
    weaviate_client.schema.delete_class(collection_name)
except:
    pass
```

El esquema al estilo GraphQL (clase `collection_name`, descripción "Google Environmental Report", propiedades `text` [text], `doc_id` [string], `source` [string]):

```python
weaviate_client.schema.create_class({
    "class": collection_name,
    "description": "Google Environmental Report",
    "properties": [
        {
            "name": "text",
            "dataType": ["text"],
            "description": "Text content of the document"
        },
        {
            "name": "doc_id",
            "dataType": ["string"],
            "description": "Document ID"
        },
        {
            "name": "source",
            "dataType": ["string"],
            "description": "Document source"
        }
    ]
})
```

Weaviate aplica una **validación de esquema más estricta** que Chroma → más control granular. Los documentos usan **`'doc_id'` y NO `'id'`** (`id` es una clave reservada internamente):

```python
dense_documents = [Document(page_content=text, metadata={"doc_id": str(i), "source": "dense"}) for i, text in enumerate(splits)]
sparse_documents = [Document(page_content=text, metadata={"doc_id": str(i), "source": "sparse"}) for i, text in enumerate(splits)]
```

> [!warning] **Gotcha del `id` reservado**: en Weaviate la metadata usa **`'doc_id'`, nunca `'id'`** — `id` está reservado internamente y reutilizarlo rompe la carga.

El vectorstore:

```python
vectorstore = Weaviate(
    client=weaviate_client,
    embedding=embedding_function,
    index_name=collection_name,
    text_key="text",
    attributes=["doc_id", "source"],
    by_text=False
)
```

La carga por lotes (preservando la estructura):

```python
weaviate_client.batch.configure(batch_size=100)
with weaviate_client.batch as batch:
    for doc in tqdm(dense_documents, desc="Processing documents"):
        properties = {
            "text": doc.page_content,
            "doc_id": doc.metadata["doc_id"],
            "source": doc.metadata["source"]
        }
        vector = embedding_function.embed_query(doc.page_content)
        batch.add_data_object(
            data_object=properties,
            class_name=collection_name,
            vector=vector
        )
```

En resumen: **Chroma** es más simple y de esquema flexible (embebible); **Weaviate** es más estructurado y rico en features (esquema explícito, múltiples backends, servidor standalone o cloud). Y de nuevo el punto central: LangChain permite **swap-ear cualquier vector store** y el resto del código funciona — una fortaleza clave.

### Tabla 10.1 — Comparación de vector stores

| Vector store | Tipo / origen | Esquema | Despliegue | Cuándo usarlo |
|---|---|---|---|---|
| **[[Chroma DB\|Chroma]]** | AI-native, open source | Flexible, implícito | In-memory / persistente / Docker | Punto de partida; facilidad de uso + integración LangChain |
| **[[FAISS]]** | Facebook AI, open source | Sin esquema (sólo vectores) | Local (CPU/GPU) | Dev local, experimentación, datasets que no entran en memoria |
| **[[Weaviate]]** | Open source, estilo [[GraphQL]] | Explícito, estricto (`create_class`) | Embedded / standalone / cloud | Producción estructurada, control granular, múltiples backends |
| **[[Pinecone]]** | Fully managed | — | Cloud managed | Producción, escalable, tiempo real |
| **[[pgvector]]** | Extensión de Postgres | SQL | Donde ya haya Postgres | Reutilizar infra Postgres existente |

> [!tip] La tabla deja claro el espectro: de **Chroma** (lo más simple y embebible) a **Pinecone** (lo más managed) — y LangChain las expone a todas tras la misma `vector store class`, así que la elección es de arquitectura, no de código.

## Code lab 10.2 – Retrievers en LangChain

Un **[[Retriever|retriever]]** consulta el vector store y devuelve los documentos más relevantes para una query. El capítulo arranca con los tres retrievers ya vistos (sobre Chroma) y sigue con varios nuevos.

### Dense (retriever básico)

El retriever básico envuelve el vector store con `as_retriever`:

```python
dense_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})
```

Es un **wrapper liviano** sobre el vector store: usa **[[Dense Search|vectores densos]]** + similitud (cosine/Euclidean), expone la interfaz consistente de LangChain y es fácil de swap-ear. Tiene dos capacidades de búsqueda: **similarity search** y **[[MMR]]**.

### Similarity score threshold

El default es similarity search. Para fijar un umbral de score:

```python
dense_retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={"score_threshold": 0.5}
)
```

### MMR (maximum marginal relevance)

**[[MMR]]** recupera ítems relevantes **evitando redundancia** (balancea relevancia + diversidad); se usa en information retrieval y en summarization:

```python
dense_retriever = vectorstore.as_retriever(search_type="mmr")
```

> [!note] **MMR (maximum marginal relevance)**: estrategia de retrieval que balancea **relevancia y diversidad**, devolviendo documentos relevantes pero no redundantes entre sí.

### BM25 (sparse)

**[[BM25]]** es una función de ranking para retrieval de texto **sparse**. El `BM25Retriever` de LangChain:

```python
sparse_retriever = BM25Retriever.from_documents(sparse_documents, k=10)
```

Calcula un score de relevancia a partir de los términos de la query + las frecuencias de términos del documento + las inverse document frequencies (**TF-IDF**); es **probabilístico** y devuelve los top-k por score BM25.

### Ensemble (híbrido)

El **[[EnsembleRetriever]]** combina varios métodos de retrieval + un algoritmo para fusionar resultados → búsqueda **híbrida** (del cap. 8):

```python
ensemble_retriever = EnsembleRetriever(
    retrievers=[dense_retriever, sparse_retriever],
    weights=[0.5, 0.5], c=0, k=10)
```

Combina el dense retriever de Chroma + el BM25 sparse (pesos iguales **0.5**). Envía la query a ambos, fusiona por peso y devuelve los top-k. El **[[Dense Search|dense]]** aporta similitud semántica; el **[[Sparse Search|sparse]]**, frecuencias de términos / keywords.

> [!note] El parámetro **`c`** del `EnsembleRetriever` es un parámetro de **reranking** que balancea los scores de retrieval originales contra los de reranking. **`c=0` → NO hay reranking**; un `c` distinto de cero agrega un paso adicional de reranking que vuelve a puntuar los documentos (vía un modelo/función de reranking separado, con features/criterios extra).

### Wikipedia y otros retrievers sobre fuentes públicas

El **[[WikipediaRetriever]]** es un retriever construido sobre una **fuente pública** (no tus documentos indexados):

> "Wikipedia is the largest and most-read reference work in history, acting as a multilingual free online encyclopedia written and maintained by a community of volunteers."

Instalación:

```bash
%pip install langchain_core
%pip install --upgrade --quiet wikipedia
```

Código:

```python
from langchain_community.retrievers import WikipediaRetriever
retriever = WikipediaRetriever(load_max_docs=10)
docs = retriever.get_relevant_documents(query="What defines the golden age of piracy in the Caribbean?")
metadata_title = docs[0].metadata['title']
metadata_summary = docs[0].metadata['summary']
metadata_source = docs[0].metadata['source']
page_content = docs[0].page_content
print(f"First document returned:\n")
print(f"Title: {metadata_title}\n")
print(f"Summary: {metadata_summary}\n")
print(f"Source: {metadata_source}\n")
print(f"Page content:\n\n{page_content}\n")
```

Salida (excerpt verbatim): el **Title** "Golden Age of Piracy"; el **Summary** describe el período **1650s–1730s**, con el buccaneering period (~1650 a 1680)…; y el **source** = en.wikipedia.org/wiki/Golden_Age_of_Piracy.

Otros retrievers sobre fuentes públicas: **PubMedRetriever** (investigación científica), **ArxivRetriever** (matemática / CS — más de **2 millones** de artículos académicos) y **KayAiRetriever** (finanzas — filings de la **SEC**, Securities and Exchange Commission / estados financieros de empresas públicas).

### kNN

Los algoritmos vistos hasta acá usaban **[[ANN]] (approximate nearest neighbor)**. El **[[k-NN]] (k-nearest neighbor)** data de **1951** pero sigue siendo **la forma MÁS EFECTIVA** de encontrar los vecinos más cercanos — más precisa que ANN (que todos los vendors de DBs / vector DBs / IR promocionan). ¿Por qué se promociona ANN? Porque kNN **no escala** a niveles enterprise grandes — pero es relativo: incluso **1 millón de data points con vectores de 1.536 dimensiones** es chico en la escala global enterprise, y kNN lo maneja con facilidad; muchos proyectos chicos que usan ANN podrían usar kNN. No hay un límite fijo (depende del entorno, los datos, las dimensiones, la conectividad): hay que **testearlo**. Por debajo de ~**1M de puntos / 1.536 dims** en un entorno capaz → considerá kNN; cambiá a ANN cuando el tiempo de procesamiento se vuelva demasiado largo. Código:

```python
from langchain_community.retrievers import KNNRetriever
dense_retriever = KNNRetriever.from_texts(splits, OpenAIEmbeddings(), k=10)
ensemble_retriever = EnsembleRetriever(
    retrievers=[dense_retriever, sparse_retriever],
    weights=[0.5, 0.5], c=0, k=10)
```

> [!warning] **kNN vs ANN — caveat de escala**: el ecosistema empuja ANN porque kNN no escala a millones de millones de vectores, pero por debajo de ~**1M de puntos / 1.536 dims** kNN suele ser **más preciso** y perfectamente viable. No asumas que necesitás ANN: medí y decidí.

Otros retrievers notables: un **time-weighted vector store retriever** (incorpora la **recencia**) y **Long-Context Reorder** (mejora los modelos de contexto largo que sufren con la información ubicada en el medio de los documentos recuperados).

## Code lab 10.3 – LLMs en LangChain

Sin el **[[LLM]]** (la etapa de **generación**) no hay RAG: el retrieval le da al LLM datos que no conoce, y el LLM responde la pregunta del usuario. La sinergia: RAG aporta **conocimiento externo** (factual, actualizado); los LLMs aportan **comprensión de query y contexto** → relación simbiótica.

### OpenAI

Instalar y configurar (pasos verbatim). Instalación:

```python
%pip install langchain-openai
```

Imports:

```python
import openai
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
```

Cargar el entorno y la clave:

```python
_ = load_dotenv(dotenv_path='env.txt')
```

```python
os.environ['OPENAI_API_KEY'] = os.getenv('OPENAI_API_KEY')
openai.api_key = os.environ['OPENAI_API_KEY']
```

Definir el LLM:

```python
llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)
```

`temperature=0` → respuestas **focalizadas / deterministas**. **gpt-4o-mini** es el más nuevo, capaz y **cost-effective** de la serie GPT-4, pero aún así cuesta **10×** lo que **gpt-3.5-turbo** (un modelo relativamente capaz). El más caro, **gpt-4-32k**, NO es tan rápido/capaz como gpt-4o-mini y tiene una ventana de contexto **4×** mayor.

> [!warning] **No asumas que el modelo más nuevo es el más caro.** Modelos más recientes (incl. gpt-5) pueden ser **a la vez más baratos y más capaces** → mantené la diligencia y pesá costo vs capacidad en cada release.

### Together AI (Llama 3 + Mixtral)

**[[Together AI]]** da acceso developer-friendly a muchos modelos, con pricing difícil de superar y a menudo **$5.00 de créditos gratis** (sin tarjeta). Las API keys se sacan en api.together.ai/settings/api-keys → agregar como `TOGETHER_API_KEY` en env.txt. Los costos están en api.together.ai/models — por ejemplo **Meta Llama 3 70B instruct (`Llama-3-70b-chat-hf`) = $0.90 por 1M de tokens** (rivaliza con ChatGPT-4 a un costo de inferencia mucho menor) y **Mixtral mixture-of-experts = $1.20 por 1M**. Pasos (verbatim). Instalación:

```python
%pip install --upgrade langchain-together
```

Luego:

```python
from langchain_together import ChatTogether
_ = load_dotenv(dotenv_path='env.txt')
```

```python
os.environ['TOGETHER_API_KEY'] = os.getenv('TOGETHER_API_KEY')
```

Definir los dos modelos:

```python
llama3llm = ChatTogether(
    together_api_key=os.environ['TOGETHER_API_KEY'],
    model="meta-llama/Llama-3-70b-chat-hf",
)
mistralexpertsllm = ChatTogether(
    together_api_key=os.environ['TOGETHER_API_KEY'],
    model="mistralai/Mixtral-8x22B-Instruct-v0.1",
)
```

> [!tip] Probá los **$5.00 de créditos gratis** de Together AI con **[[Llama 3]]** 70B ($0.90/1M) o **[[Mixtral]]** 8x22B ($1.20/1M) para inferencia mucho más barata que OpenAI, manteniendo respuestas comparables o más robustas.

La cadena RAG con [[Llama 3]] (estructura [[LCEL]]):

```python
llama3_rag_chain_from_docs = (
    RunnablePassthrough.assign(context=(lambda x: format_docs(x["context"])))
    | RunnableParallel(
        {"relevance_score": (
            RunnablePassthrough()
            | (lambda x: relevance_prompt_template.format(question=x["question"], retrieved_context=x["context"]))
            | llama3llm
            | StrOutputParser()
        ), "answer": (
            RunnablePassthrough()
            | prompt
            | llama3llm
            | StrOutputParser()
        )}
    )
    | RunnablePassthrough().assign(final_answer=conditional_answer)
)

llama3_rag_chain_with_source = RunnableParallel(
    {"context": ensemble_retriever, "question": RunnablePassthrough()}
).assign(answer=llama3_rag_chain_from_docs)
```

Invocación y print:

```python
result = llama3_rag_chain_with_source.invoke(user_query)
retrieved_docs = result['answer']['context']

print(f"Original Question: {user_query}\n")
print(f"Relevance Score: {result['answer']['relevance_score']}\n")
print(f"Final Answer:\n{result['answer']['final_answer']}\n\n")
print("Retrieved Documents:")
for i, doc in enumerate(retrieved_docs, start=1):
    print(f"Document {i}: Document ID: {doc.metadata['doc_id']} source: {doc.metadata['source']}")
    print(f"Content:\n{doc.page_content}\n")
```

Salida de Llama 3 (excerpt verbatim):

> "Google's environmental initiatives include: 1. Empowering individuals to take action… [TRUNCATED] 10. Engagement with external targets and initiatives… RE-Source Platform, iMasons Climate Accord, and World Business Council for Sustainable Development."

La cadena con **[[Mixtral]]** (misma estructura, usando `mistralexpertsllm`):

```python
mistralexperts_rag_chain_from_docs = (
    RunnablePassthrough.assign(context=(lambda x: format_docs(x["context"])))
    | RunnableParallel(
        {"relevance_score": (
            RunnablePassthrough()
            | (lambda x: relevance_prompt_template.format(question=x["question"], retrieved_context=x["context"]))
            | mistralexpertsllm
            | StrOutputParser()
        ), "answer": (
            RunnablePassthrough()
            | prompt
            | mistralexpertsllm
            | StrOutputParser()
        )}
    )
    | RunnablePassthrough().assign(final_answer=conditional_answer)
)

mistralexperts_rag_chain_with_source = RunnableParallel(
    {"context": ensemble_retriever, "question": RunnablePassthrough()}
).assign(answer=mistralexperts_rag_chain_from_docs)
```

Invocación y print:

```python
result = mistralexperts_rag_chain_with_source.invoke(user_query)
retrieved_docs = result['answer']['context']

print(f"Original Question: {user_query}\n")
print(f"Relevance Score: {result['answer']['relevance_score']}\n")
print(f"Final Answer:\n{result['answer']['final_answer']}\n\n")
print("Retrieved Documents:")
for i, doc in enumerate(retrieved_docs, start=1):
    print(f"Document {i}: Document ID: {doc.metadata['doc_id']} source: {doc.metadata['source']}")
    print(f"Content:\n{doc.page_content}\n")
```

Salida de Mixtral (excerpt verbatim): describe los **tres pilares** (empowering individuals, working with partners and customers, operating sustainably); la meta de **1 gigaton para 2030**; el UNFCCC + el Paris Agreement (**< 2°C**); la RE-Source Platform + el Google.org Impact Challenge on Climate Innovation.

Para comparar, la respuesta original de **gpt-4o-mini** (excerpt verbatim): "Google's environmental initiatives include empowering individuals to take action, working together with partners and customers… [partnerships such as the RE-Source Platform]…". **Takeaway**: tanto Llama 3 como Mixtral dan respuestas expandidas, **similares o más robustas** que gpt-4o-mini, a un costo muy inferior.

### Tabla 10.2 — Costo / capacidad de los LLMs

| Modelo | Costo | Nota |
|---|---|---|
| **gpt-4o-mini** | (paga) | El más nuevo/capaz/cost-effective de GPT-4, pero **10×** el costo de gpt-3.5-turbo |
| **gpt-3.5-turbo** | (más barato) | Relativamente capaz; baseline de costo |
| **gpt-4-32k** | (el más caro) | NO tan rápido/capaz como gpt-4o-mini; contexto **4×** mayor |
| **[[Llama 3]] 70B** (`Llama-3-70b-chat-hf`) | **$0.90 / 1M tokens** | Vía [[Together AI]]; rivaliza con ChatGPT-4 a mucho menor costo |
| **[[Mixtral]] 8x22B** (`Mixtral-8x22B-Instruct-v0.1`) | **$1.20 / 1M tokens** | [[Mixture of Experts]] vía [[Together AI]] |

> [!tip] La tabla muestra que el LLM es **swap-eable por costo**: con LangChain podés cambiar de OpenAI a Together AI (Llama 3 / Mixtral) sin reescribir la cadena, obteniendo calidad comparable a una fracción del costo de inferencia.

### Extender el LLM: async / stream / batch

Todos los LLMs en LangChain implementan la interfaz **[[Runnable]]**:

> "All LLMs implement the Runnable interface, which comes with default implementations of all methods, ie. ainvoke, batch, abatch, stream, astream. This gives all LLMs basic support for async, streaming and batch."

Los tres métodos:

- **Async** — corre el método sync en un thread separado → las otras partes async siguen ejecutándose.
- **Stream** — devuelve un `Iterator` / `AsyncIterator` con **un solo ítem** = el resultado final; no es word-by-word, pero es compatible con integraciones que esperan un token stream.
- **Batch** — procesa múltiples inputs a la vez (sync = múltiples threads; async = `asyncio.gather`); la concurrencia se controla con **`max_concurrency`** en el `RunnableConfig`.

> [!note] La interfaz **[[Runnable]]** da a todo LLM soporte básico de **async, streaming y batch** vía implementaciones por defecto (`ainvoke`, `batch`, `abatch`, `stream`, `astream`). No todos los LLMs soportan todo nativamente → LangChain provee una **tabla comparativa**.

## Citas

> "Wikipedia is the largest and most-read reference work in history, acting as a multilingual free online encyclopedia written and maintained by a community of volunteers."

> "All LLMs implement the Runnable interface, which comes with default implementations of all methods, ie. ainvoke, batch, abatch, stream, astream. This gives all LLMs basic support for async, streaming and batch."

## Para aplicar

- **Swap-eá vector stores libremente** — gracias a la `vector store class` de LangChain, cambiar de [[Chroma DB|Chroma]] a [[FAISS]] o [[Weaviate]] es cuestión de pocas líneas; el resto del pipeline no cambia.
- **Reiniciá el kernel tras instalar** un vector store nuevo (`faiss-cpu`, `faiss-gpu`, etc.) antes de importarlo.
- **Usá `faiss-gpu` para escala** — la GPU paraleliza las comparaciones de vectores y acelera enormemente la búsqueda en datasets grandes.
- **Con Weaviate, usá `'doc_id'` en la metadata, nunca `'id'`** (reservado), y declará el esquema con `create_class` antes de cargar.
- **Probá el [[EnsembleRetriever]] con `c=0`** para híbrido sin reranking, o un `c≠0` si querés un paso extra de reranking.
- **Considerá [[k-NN]] por debajo de ~1M de puntos / 1.536 dims** — suele ser más preciso que [[ANN]]; testealo en tu entorno y pasá a ANN sólo cuando el tiempo de procesamiento crezca demasiado.
- **Aprovechá los $5.00 de crédito gratis de [[Together AI]]** con [[Llama 3]] 70B ($0.90/1M) o [[Mixtral]] 8x22B ($1.20/1M) para inferencia barata.
- **No asumas que el modelo más nuevo es el más caro** — pesá costo vs capacidad en cada release.
- **Apoyate en la interfaz [[Runnable]]** (async / stream / batch, con `max_concurrency`) para escalar la generación.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[09 - Evaluating RAG Quantitatively and with Visualizations]] — capítulo anterior (su código de evaluación se saltea acá) · [[11 - Using LangChain to Get More from RAG]] — capítulo siguiente (los componentes más chicos de LangChain que dan soporte a estos: document loaders, splitters…).
- [[08 - Similarity Searching with Vectors]] — la base de código (code lab 8.3 con el `EnsembleRetriever`), y donde se introdujeron [[BM25]], el híbrido, [[k-NN]] vs [[ANN]] y los servicios de vector search.
- [[07 - The Key Role Vectors and Vector Stores Play in RAG]] — donde aparecieron los [[Vector Store|vector stores]], [[FAISS]], [[pgvector]] y la elección de store como decisión de arquitectura.
- [[LangChain]] · [[LCEL]] · [[Runnable]] — la capa de orquestación y la interfaz común a todos los componentes.
- [[Vector Store]] · [[Chroma DB]] · [[FAISS]] · [[Weaviate]] · [[pgvector]] · [[Pinecone]] · [[GraphQL]] — vector stores y el esquema estilo GraphQL de Weaviate.
- [[Retriever]] · [[Dense Search]] · [[Sparse Search]] · [[MMR]] · [[BM25]] · [[EnsembleRetriever]] · [[WikipediaRetriever]] · [[k-NN]] · [[ANN]] — la familia de retrievers.
- [[OpenAI]] · [[Together AI]] · [[Llama 3]] · [[Mixtral]] · **[[Mixture of Experts]]** · [[dotenv]] — los LLMs y su configuración.
- **Candidatos a nota propia**: [[Together AI]] · [[Llama 3]] · [[Mixtral]] · [[Mixture of Experts]] · [[WikipediaRetriever]] · [[Runnable]] · [[Retriever]] · [[MMR]] (si aún no existen en el vault).
