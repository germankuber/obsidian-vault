---
title: 02 - Code Lab - An Entire RAG Pipeline
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 2
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Code Lab - An Entire RAG Pipeline
  - Code Lab — An Entire RAG Pipeline
updated: 2026-06-12
---

# 02 - Code Lab - An Entire RAG Pipeline

> [!info] Capítulo 2 · *Unlocking Data with Generative AI and RAG* — Keith Bourne (Packt, ISBN 9781835887905)
> Code lab que construye un **pipeline [[RAG]] completo de punta a punta** con [[LangChain]], [[Chroma DB]] y [[OpenAI]]: las tres etapas ([[Indexing]] · [[Retrieval]] · [[Generation]]) materializadas en unas pocas líneas de código. Es la base sobre la que el resto del libro itera. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[01 - What Is Retrieval-Augmented Generation (RAG)]] · siguiente [[03 - Practical Applications of RAG]].

## Resumen

Este capítulo es un **laboratorio de código**: construye un pipeline [[RAG]] **entero y funcional** que sirve de fundamento para todo lo que el libro evoluciona después. El recorrido es lineal y reproducible: no hay interfaz (se usa un simple string como stand-in del input del usuario); se da de alta una cuenta de LLM en [[OpenAI]]; se instalan los paquetes de Python; se hace el **indexing** (crawl web con [[WebBaseLoader]], split semántico con [[SemanticChunker]], embedding e indexado en [[Chroma DB]]); se hace el **retrieval** vía [[Similarity Search]] vectorial; y se hace la **generation** inyectando el contexto recuperado en el prompt del LLM. Las herramientas centrales son **[[LangChain]]**, **[[Chroma DB]]** y las **APIs de [[OpenAI]]**. El punto clave que el autor martilla: todo el pipeline RAG es, en esencia, **unas pocas líneas / strings**, ensambladas con [[LCEL]] (LangChain Expression Language). Al terminar, ya podés construir una app RAG completa; el resto del libro existe para superar los problemas que aparecen cuando esa app se enfrenta al mundo real.

> [!note] Objetivo del capítulo: construir un pipeline RAG básico de extremo a extremo. Cubre indexing (crawl, split, embed), retrieval (similarity search) y generation (inyección de contexto en el prompt), usando LangChain + Chroma DB + OpenAI APIs.

## Requisitos técnicos y entorno

El código del capítulo vive en GitHub: github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_02. Necesitás un **entorno de notebook Jupyter con Python 3**.

El autor menciona sus propios setups: **Google Colab** y **Colab Enterprise** en Vertex AI. Advierte sobre el costo: estos entornos pueden superar los **$20/mes**, y si sos muy activo, incluso **$1.000/mes**. Como alternativa más económica propone correr **Docker Desktop en Mac** hospedando un **cluster local de Kubernetes**.

> [!note] El código asume un entorno **Jupyter**. Se puede correr como un `.py` con algunos cambios, pero el notebook te deja ejecutar **celda por celda**, lo que es ideal para seguir el lab paso a paso.

## Sin interfaz!

Este ejemplo **no tiene UI**. Las interfaces se cubren en el [[06 - Interfacing with RAG and Gradio|capítulo 6]]. Acá se usa una simple **variable string** como sustituto del input que normalmente vendría de un usuario.

## Configurar la cuenta del LLM (OpenAI)

El [[LLM]] es el **"cerebro"** del pipeline. [[OpenAI]] / ChatGPT es el más popular, pero existen muchísimos LLMs y **no siempre necesitás el más grande ni el más caro**. Como ejemplo de modelo especializado por dominio, el autor cita los LLMs **Meditron** (versiones de **Llama 2** fine-tuneadas para investigación médica). También señala que los LLMs pueden **chequearse entre sí** (para eso necesitás más de uno). La recomendación general: **comprá inteligentemente, comparando opciones**.

> [!tip] Buscá el LLM adecuado para tu caso: el más grande/caro no siempre es el mejor; modelos especializados por dominio (p. ej. **Meditron** sobre Llama 2) o varios LLMs chequeándose entre sí pueden encajar mejor.

Los pasos de setup (1–7):

1. Andá a la sección de API de OpenAI: **openai.com/api/**.
2. Registrate (sign up).
3. Tené presente el costo.
4. Conseguí tu **API key** en platform.openai.com/docs/quickstart.
5. Elegí el tipo de permiso de la key.
6. Copiá la key y **mantenela en secreto**.
7. Comprá créditos por adelantado y **desactivá el auto-recharge**.

> [!warning] **Using OpenAI's API costs money! Use it sparingly!** Cada llamada a la API (LLM y embeddings) tiene costo real.

Tipos de permiso de la API key:

- **All** — lectura/escritura sobre todas las APIs.
- **Restricted** — control granular por API (read/write por cada una); habilitá al menos las APIs de **models** y **embeddings**.
- **Read Only** — lectura sobre todas las APIs.

> [!warning] La API key es **top secret**: cualquiera que la tenga **gasta tu dinero**. Comprá créditos por adelantado y dejá **OFF** el *"Enable auto recharge"* para poner un techo al gasto.

## Instalar los paquetes necesarios

Instalación con pip (reproducir verbatim):

```bash
%pip install langchain_community
%pip install langchain_experimental
%pip install langchain-openai
%pip install langchainhub
%pip install chromadb
%pip install langchain
%pip install beautifulsoup4
```

Descripción de cada paquete:

- **langchain_community** — paquete *community* de [[LangChain]], el framework open source para construir aplicaciones sobre LLMs.
- **langchain_experimental** — capacidades extra todavía **no estables** de LangChain.
- **langchain-openai** — integración LangChain ↔ [[OpenAI]] (ChatGPT-4 + embeddings).
- **langchainhub** — componentes/plantillas pre-construidos: agents, memory, utilities.
- **chromadb** — base de datos de embeddings / vector DB de alto rendimiento para [[Similarity Search]].
- **langchain** — el core: prompting, memory, agents, integraciones.

> [!tip] Después de instalar, **reiniciá el kernel**. Snippet de reinicio del kernel de IPython:

```python
import IPython
app = IPython.Application.instance()
app.kernel.do_shutdown(True)
```

## Imports

Bloque de imports (reproducir verbatim):

```python
import os
from langchain_community.document_loaders import WebBaseLoader
import bs4
import openai
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain import hub
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
import chromadb
from langchain_community.vectorstores import Chroma
from langchain_experimental.text_splitter import SemanticChunker
```

Qué aporta cada import:

- **os** — variables de entorno (para la API key).
- **[[WebBaseLoader]]** — carga páginas web como *documents*.
- **bs4 / BeautifulSoup 4** — web scraping / parseo de HTML (extraer título, contenido, headers).
- **openai** — SDK de [[OpenAI]].
- **ChatOpenAI + OpenAIEmbeddings** — implementaciones del [[LLM]] y de los [[Embeddings]] para OpenAI dentro de LangChain.
- **hub** — componentes pre-construidos del [[LangChain]] Hub.
- **StrOutputParser** — parsea el output del LLM; asume que es un string y lo devuelve tal cual.
- **RunnablePassthrough** — pasa la pregunta **sin modificar**.
- **chromadb** — el cliente de [[Chroma DB]].
- **Chroma** — la interfaz de LangChain hacia la vector DB Chroma.
- **[[SemanticChunker]]** — splitter de texto **experimental** que parte texto largo en chunks **preservando la coherencia/contexto semántico**.

## Conexión con OpenAI

Se setea la API key (para ChatGPT **y** para el servicio de embeddings — la misma key sirve para ambos):

```python
os.environ['OPENAI_API_KEY'] = 'sk-###'
openai.api_key = os.environ['OPENAI_API_KEY']
```

> [!warning] Acá la key va hardcodeada solo a fines didácticos. **Important:** usá un método seguro para manejar la key; el enfoque seguro se cubre en el [[05 - Managing Security in RAG Applications|capítulo 5]].

## Indexing

El [[Indexing]] prepara los datos para que puedan recuperarse. Suele hacerse **"offline" / por adelantado**, pero puede ser en **tiempo real** para conjuntos de datos pequeños que cambian rápido. Tiene 4 pasos:

1. Cargar y crawlear la web.
2. Partir (split) en chunks para [[Chroma DB]].
3. Embedding e indexado de los chunks.
4. Agregar chunks + embeddings a Chroma.

### Web loading & crawling

[[WebBaseLoader]] descarga y parsea la página (reproducir verbatim):

```python
loader = WebBaseLoader(
    web_paths=("https://kbourne.github.io/chapter1.html",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            class_=("post-content", "post-title", "post-header")
        )
    ),
)
docs = loader.load()
```

La página de ejemplo está adaptada del ejemplo de LangChain, basado a su vez en lilianweng.github.io/posts/2023-06-23-agent/.

Qué hace `load()` internamente:

- **I.** Hace los HTTP requests a las URLs de `web_paths`.
- **II.** Parsea el HTML con BeautifulSoup, limitado a lo que indica `parse_only`.
- **III.** Extrae el texto.
- **IV.** Crea objetos **Document** con metadata (p. ej. la URL de origen).

> [!warning] `post-content`, `post-title` y `post-header` son **clases CSS** de esta página: el `SoupStrainer` **no funcionará en páginas que no las tengan**. Si cambiás de página (o de clases), **reiniciá el kernel**, o vas a mezclar el contenido viejo con el nuevo.

### Splitting

El split parte los documents en chunks. Acá se usa [[SemanticChunker]] (reproducir verbatim):

```python
text_splitter = SemanticChunker(OpenAIEmbeddings())
splits = text_splitter.split_documents(docs)
```

[[SemanticChunker]] es **context-aware**: corta respetando el significado, a diferencia del chunking de longitud fija arbitraria, que puede partir contenido importante por la mitad. Es **experimental** y está en desarrollo. En el [[11 - Using LangChain to Get More from RAG|capítulo 11]] se lo enfrenta contra **RecursiveCharacterTextSplitter**.

![[02-fig-2.1-web-page-to-process.jpg]]
*Figure 2.1 – A web page that we will process*

> [!warning] **Costo:** `SemanticChunker` usa `OpenAIEmbeddings()`, que **cuesta dinero**. Los modelos de embeddings de OpenAI cuestan **$0.02–$0.13 por millón de tokens**. El modelo por defecto (si no especificás) es **text-embedding-ada-002**, a **$0.02 por millón de tokens**. Para evitar el costo podés caer de vuelta a **RecursiveCharacterTextSplitter** (ver [[11 - Using LangChain to Get More from RAG|cap. 11]]).

### Embedding e indexado de los chunks

[[Chroma DB]] es a la vez un **vector store** y una **vector database** (callback al vocabulario del [[01 - What Is Retrieval-Augmented Generation (RAG)|cap. 1]]). Se elige porque corre fácil en local, es bueno para demos pero igual potente. Otras opciones (y vectorización gratis) se ven en el [[07 - The Key Role Vectors and Vector Stores Play in RAG|capítulo 7]].

```python
vectorstore = Chroma.from_documents(documents=splits, embedding=OpenAIEmbeddings())
retriever = vectorstore.as_retriever()
```

`Chroma.from_documents` recibe `documents=splits` y `embedding=OpenAIEmbeddings`. Internamente, en 3 pasos:

- Itera los **Document**.
- Genera el **embedding** de cada uno.
- Guarda **texto + embedding** en Chroma.

El **retriever** se crea con `as_retriever()` y provee la interfaz para las búsquedas por [[Similarity Search]].

Snippet de prueba opcional del retriever:

```python
query = "How does RAG compare with fine-tuning?"
relevant_docs = retriever.get_relevant_documents(query)
relevant_docs
```

## Retrieval & Generation

El [[Retrieval]] y la [[Generation]] se combinan en **una sola cadena de LangChain**, usando componentes pre-construidos del **[[LangChain]] Hub** (plantillas de prompt) + el LLM elegido + **[[LCEL]]** (LangChain Expression Language). Los 6 pasos del flujo:

1. Tomar la query del usuario.
2. Vectorizarla.
3. [[Similarity Search]]: buscar los vectores más cercanos + su contenido.
4. Pasar el contenido recuperado al prompt template = **hydrating** (hidratar).
5. Pasar el prompt hidratado al [[LLM]].
6. Presentar la respuesta.

> [!note] **Hydrating** (hidratar) = rellenar los placeholders del [[Prompt Template]] (`context`, `question`) con el contenido real recuperado, produciendo el prompt completo que recibe el LLM.

### Prompt templates del LangChain Hub

Se trae un [[Prompt Template]] pre-construido del Hub y se imprime:

```python
prompt = hub.pull("jclemens24/rag-prompt")
print(prompt)
```

Convención de nombres del componente: **jclemens24** = el repo, **rag-prompt** = el componente.

Lo que imprime `print(prompt)` (verbatim):

```text
input_variables=['context', 'question'] messages=[HumanMessagePromptTemplate(prompt=PromptTemplate(input_variables=['context', 'question'], template="You are an assistant for question-answering tasks. Use the following pieces of retrieved-context to answer the question. If you don't know the answer, just say that you don't know.\nQuestion: {question} \nContext: {context} \nAnswer:"))]
```

El template, más legible:

```text
You are an assistant for question-answering tasks. Use the following pieces of retrieved-context to answer the question. If you don't know the answer, just say that you don't know.
Question: {question}
Context: {context}
Answer:
```

> [!note] `jclemens24/rag-prompt` es **una de más de 30 opciones**. Podés navegarlas en smith.langchain.com/hub/search?q=rag-prompt, o usar tu propio prompt.

### format_docs

```python
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)
```

Qué hace: extrae el `page_content` de cada **Document**, los une con **dos newlines**, y produce el string `context`.

> [!note] Acá un *"document"* = **una sección/chunk pequeño**, no el documento crawleado entero.

Por qué hace falta: las cadenas de [[LangChain]] necesitan funciones cortas para **encajar inputs/outputs** entre pasos — el retriever devuelve una **lista de objetos**, pero el prompt necesita un **string**.

### Definir el LLM

```python
llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)
```

`gpt-4o-mini` es más nuevo pero salió con un **gran descuento**, lo que mantiene bajo el costo de inferencia. Podés cambiar `model_name` por `gpt-4`, etc.

### La cadena LCEL

El pipeline RAG completo, ensamblado con [[LCEL]]:

```python
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
```

Los 3 pasos:

1. **Retrieval** (es su propia sub-cadena) — el dict con las claves `"context"` y `"question"`. `"context"` = `retriever | format_docs`: el pipe `|` los encadena — el **retriever** corre la [[Similarity Search]] → set de matches → **format_docs** los convierte en un único string → se asigna a `context`. `"question"` = `RunnablePassthrough()`: pasa la pregunta **sin modificar**, como el string que ya es. El paso siguiente espera un **dict de dos claves string**.
2. **`| prompt`** — hidrata el prompt (rellena los placeholders `context` + `question` → prompt completo). Luego **`| llm`**: ChatGPT (gpt-4o-mini / 4o) genera una respuesta a partir del prompt hidratado.
3. **`| StrOutputParser()`** — la API del LLM devuelve JSON con data extra; `StrOutputParser` la **descarta** y devuelve la respuesta como **string plano**.

> [!note] Todo este pipeline RAG es **solo unas pocas líneas / strings** de largo. Esa es la idea central del capítulo.

## Enviar una pregunta para RAG

```python
rag_chain.invoke("What are the advantages of using RAG?")
```

El string es la **pregunta** que se inyecta; pasa por `RunnablePassthrough`; en producción vendría de una UI.

## Output final

Output crudo del LLM (verbatim):

```text
The advantages of using Retrieval Augmented Generation (RAG) include:

1. **Improved Accuracy and Relevance:** RAG enhances the accuracy and relevance of responses generated by large language models (LLMs) by retrieving and incorporating specific information from databases or datasets in real time, ensuring outputs are based on both the model's pre-existing knowledge and the most current and relevant data provided.

2. **Customization and Flexibility:** RAG allows for the customization of responses based on domain-specific needs by integrating a company's internal databases into the model's response generation process, enabling tailored interactions that meet unique business requirements.

3. **Expanding Model Knowledge Beyond Training Data:** RAG enables models to access and utilize information that was not included in their initial training datasets, effectively expanding the knowledge base of the model without the need for retraining.
```

Versión renderizada (3 ventajas):

- **Improved Accuracy and Relevance** — recupera e incorpora info específica de bases/datasets en tiempo real; el output combina el conocimiento previo del modelo con datos actuales y relevantes.
- **Customization and Flexibility** — personaliza respuestas según el dominio integrando las bases de datos internas de la empresa al proceso de generación.
- **Expanding Model Knowledge Beyond Training Data** — permite usar info que no estaba en el entrenamiento, expandiendo el conocimiento sin necesidad de reentrenar.

> [!tip] **Trade-off costo/modelo:** ¿podría un modelo más barato dar un "good enough"? Un prompt de tipo *"keep it brief"* puede producir la misma respuesta corta con uno u otro modelo → ¿por qué pagar más? El más grande/caro no siempre hace falta.

Lo que el LLM **realmente ve** (prompt hidratado, verbatim con el marcador de truncado):

```text
You are an assistant for question-answering tasks. Use the following pieces of retrieved-context to answer the question. If you don't know the answer, just say that you don't know.
Question: What are the Advantages of using RAG?
Context: Can you imagine what you could do with all of the benefits mentioned above, but combined with all of the data within your company, about everything your company has ever done, about your customers and all of their interactions, or about all of your products and services combined with a knowledge of what a specific customer's needs are? You do not have to imagine it, that is what RAG does! ... [TRUNCATED FOR BREVITY!]
Answer:
```

> [!note] El contexto es **grande** = toda la info relevante del documento crawleado (chapter1.html). El **cómo** de la similarity search se detalla en el [[08 - Similarity Searching with Vectors|capítulo 8]].

## Código completo

El programa final, completo y ejecutable (de la instalación a la invocación):

```python
%pip install langchain_community
%pip install langchain_experimental
%pip install langchain-openai
%pip install langchainhub
%pip install chromadb
%pip install langchain
%pip install beautifulsoup4

import os
from langchain_community.document_loaders import WebBaseLoader
import bs4
import openai
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain import hub
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
import chromadb
from langchain_community.vectorstores import Chroma
from langchain_experimental.text_splitter import SemanticChunker

os.environ['OPENAI_API_KEY'] = 'sk-###'
openai.api_key = os.environ['OPENAI_API_KEY']

# Indexing: web loading and crawling
loader = WebBaseLoader(
    web_paths=("https://kbourne.github.io/chapter1.html",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            class_=("post-content", "post-title", "post-header")
        )
    ),
)
docs = loader.load()

# Indexing: splitting
text_splitter = SemanticChunker(OpenAIEmbeddings())
splits = text_splitter.split_documents(docs)

# Indexing: embedding and storing
vectorstore = Chroma.from_documents(documents=splits, embedding=OpenAIEmbeddings())
retriever = vectorstore.as_retriever()

# Retrieval and generation
prompt = hub.pull("jclemens24/rag-prompt")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

rag_chain.invoke("What are the advantages of using RAG?")
```

## Citas

> "Using OpenAI's API costs money! Use it sparingly!"
> "You are an assistant for question-answering tasks. Use the following pieces of retrieved-context to answer the question. If you don't know the answer, just say that you don't know."

## Para aplicar

- **Dar de alta OpenAI con freno de gasto** — sacá la API key en platform.openai.com, usá permiso **Restricted** (al menos models + embeddings), comprá créditos por adelantado y dejá **OFF** el auto-recharge.
- **Manejar la key de forma segura** — no la hardcodees en producción; ver el método seguro en [[05 - Managing Security in RAG Applications|cap. 5]].
- **Reiniciar el kernel** tras instalar paquetes y cada vez que cambiás de página/clases CSS en el loader.
- **Controlar el costo del splitting** — `SemanticChunker` consume embeddings de pago; si querés evitarlo, usá `RecursiveCharacterTextSplitter` ([[11 - Using LangChain to Get More from RAG|cap. 11]]).
- **Bajar el costo de inferencia** — elegí un modelo barato como `gpt-4o-mini` con `temperature=0`; subí a `gpt-4` solo si el caso lo justifica.
- **Construir tu primer RAG** — cloná Chapter_02 del repo de GitHub y corré el notebook celda por celda.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[01 - What Is Retrieval-Augmented Generation (RAG)]] — capítulo anterior (los fundamentos y las 3 etapas que acá se vuelven código) · [[03 - Practical Applications of RAG]] — capítulo siguiente (profundiza las aplicaciones del cap. 1 y agrega código para citar fuentes).
- [[06 - Interfacing with RAG and Gradio]] — la UI ausente acá.
- [[05 - Managing Security in RAG Applications]] — el manejo seguro de la API key.
- [[07 - The Key Role Vectors and Vector Stores Play in RAG]] — opciones de vector store y vectorización gratis.
- [[08 - Similarity Searching with Vectors]] — el detalle de la búsqueda por similitud que sustenta el retrieval.
- [[11 - Using LangChain to Get More from RAG]] — SemanticChunker vs RecursiveCharacterTextSplitter.
- [[RAG]] · [[LangChain]] · [[Chroma DB]] · [[Embeddings]] · [[Vector Database]] · [[Indexing]] · [[Retrieval]] · [[Generation]] · [[LCEL]] · [[SemanticChunker]] · [[OpenAI]] · [[Similarity Search]] · [[Prompt Template]] · [[WebBaseLoader]] · [[Fine-tuning]] · [[LLM]] — conceptos núcleo del code lab. (Candidatos a nota propia en negrita: **LCEL** · **SemanticChunker** · **WebBaseLoader** · **Chroma DB**.)
