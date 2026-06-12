---
title: 11 - Using LangChain to Get More from RAG
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 11
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Using LangChain to Get More from RAG
  - LangChain document loaders, text splitters y output parsers
updated: 2026-06-12
---

# 11 - Using LangChain to Get More from RAG

> [!info] Capítulo 11 · *Unlocking Data with Generative AI and RAG* — Packt (ISBN 9781835887905) · Keith Bourne
> Después de los componentes "clave" del cap. 10, este capítulo recorre los componentes **menos conocidos pero igual de importantes** de [[LangChain]] para RAG: los [[Document Loader|document loaders]] (cargan y procesan documentos desde distintas fuentes), los [[Text Splitter|text splitters]] (dividen los documentos en chunks de recuperación) y los [[Output Parser|output parsers]] (estructuran la respuesta del LLM). Tres code labs muestran cada familia sobre el pipeline RAG que el libro viene construyendo. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[10 - Key RAG Components in LangChain]] · siguiente [[12 - Combining RAG with the Power of AI Agents and LangGraph]].

## Resumen

El cap. 10 disecó los componentes **clave** de RAG en [[LangChain]] (vector stores, retrievers, LLMs); el cap. 11 baja a los componentes de **soporte** que alimentan a esos clave y que suelen pasar desapercibidos pero son igual de importantes: **document loaders**, **text splitters** y **output parsers**. La columna sigue siendo [[LangChain]] + [[LCEL]] y la lógica de **piezas intercambiables** (swap-eables) del cap. 10. El código está en `github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_11`.

El **code lab 11.1** ([[Document Loader|document loaders]]) muestra cómo acceder, extraer y cargar los datos desde distintos formatos para convertirlos en un formato indexable: parte del mismo PDF (el reporte ambiental de Google 2023), lo convierte a HTML, Word y JSON para tener material de prueba, y luego carga cada formato con un loader distinto (`BSHTMLLoader`, `PyPDF2`, `Docx2txtLoader`, `JSONLoader`) — todos intercambiables, cada uno reemplaza la variable `docs`. Aparece el **gotcha de metadata**: algunos loaders agregan su propia metadata `source` que choca con la nuestra, y la solución es usar la clave `search_source` en lugar de `source` al crear los vector stores. Los document loaders importan, pero para el RAG basado en chunks hace falta un text splitter a continuación.

El **code lab 11.2** ([[Text Splitter|text splitters]]) explica **por qué** se divide: los documentos grandes pierden representación de su contexto en el embedding, y además casi todos los modelos de embedding tienen un límite de input chico — el de OpenAI es de **8191 tokens**, pasarle un documento más grande da error. La parte difícil es mantener juntos los textos semánticamente relacionados; la herramienta **ChunkViz** de **Greg Kamradt** lo visualiza. Recorre tres splitters: el `CharacterTextSplitter` (el más simple, corta cada N caracteres, parte a mitad de oración), el `RecursiveCharacterTextSplitter` (el recomendado genérico de LangChain, prueba una lista de separadores en orden para respetar párrafos y oraciones) y el `SemanticChunker` (experimental, divide por similitud semántica en el espacio de embeddings).

El **code lab 11.3** ([[Output Parser|output parsers]]) estructura la respuesta del LLM para pasarla al siguiente paso de la cadena o devolverla como salida final: el `StrOutputParser` (ya usado, devuelve un string) y el `JsonOutputParser` (devuelve JSON validado contra un modelo [[Pydantic]] `FinalOutputModel`). El lab fusiona las dos cadenas previas en un único `rag_chain` que primero puntúa relevancia, responde, y luego — si la relevancia es suficiente — formatea la salida como JSON. Cierra con un recap y apunta al cap. 12 sobre **LangGraph y agentes de IA**.

## Code lab 11.1 – Document loaders

Los [[Document Loader|document loaders]] son las herramientas con las que se **accede, se extrae y se carga** la data en el pipeline, convirtiendo los documentos a un formato indexable. El notebook es `CHAPTER11-1_DOCUMENT_LOADERS.ipynb`. Primero hay que instalar las librerías de soporte:

```bash
%pip install bs4
%pip install python-docx
%pip install docx2txt
%pip install jq
```

- **bs4** = Beautiful Soup 4 — parsing de HTML, ya usado en [[02 - Code Lab - An Entire RAG Pipeline]].
- **python_docx** — crear y actualizar archivos `.docx`.
- **docx2txt** — extraer texto e imágenes desde un `.docx`.
- **jq** — procesador liviano de JSON.

> [!tip] Reiniciá el kernel después de instalar paquetes nuevos para que los imports tomen las versiones recién instaladas.

### Convertir el PDF a otros formatos

Para tener material de prueba de cada loader, el lab convierte el mismo PDF (el reporte ambiental de Google 2023) en HTML, Word y JSON. Los imports y los paths:

```python
from bs4 import BeautifulSoup
import docx
import json
```

```python
pdf_path = "google-2023-environmental-report.pdf"
html_path = "google-2023-environmental-report.html"
word_path = "google-2023-environmental-report.docx"
json_path = "google-2023-environmental-report.json"
```

El bloque que extrae el texto del PDF con [[BeautifulSoup]] / docx / json y genera los tres archivos:

```python
with open(pdf_path, "rb") as pdf_file:
    pdf_reader = PdfReader(pdf_file)
    pdf_text = "".join(page.extract_text() for page in pdf_reader.pages)
    soup = BeautifulSoup("<html><body></body></html>", "html.parser")
    soup.body.append(pdf_text)
    with open(html_path, "w", encoding="utf-8") as html_file:
        html_file.write(str(soup))
    doc = docx.Document()
    doc.add_paragraph(pdf_text)
    doc.save(word_path)
    with open(json_path, "w") as json_file:
        json.dump({"text": pdf_text}, json_file)
```

A continuación, cada loader es **intercambiable**: reemplaza la variable `docs`, se usa de a uno por vez, y cada uno etiqueta una metadata `source` propia.

### HTML — BSHTMLLoader

`BSHTMLLoader` es un loader de HTML **local** (a diferencia del `WebBaseLoader` del cap. 2, que cargaba sitios web en vivo):

```python
from langchain_community.document_loaders import BSHTMLLoader
loader = BSHTMLLoader(html_path)
docs = loader.load()
```

### PDF — PyPDF2

Una versión simplificada con [[PyPDF2]] que parte el texto en párrafos por dobles saltos de línea:

```python
from PyPDF2 import PdfReader
docs = []
with open(pdf_path, "rb") as pdf_file:
    pdf_reader = PdfReader(pdf_file)
    pdf_text = "".join(page.extract_text() for page in pdf_reader.pages)
    docs = [Document(page_content=page) for page in pdf_text.split("\n\n")]
```

> [!note] Otros PDF loaders nombrados (a tener en cuenta según el caso): **PyPDF2** (el usado acá), **PyPDF**, **PyMuPDF**, **MathPix**, **Unstructured**, **AzureAIDocumentIntelligenceLoader** y **UpstageLayoutAnalysisLoader**.

### Word — Docx2txtLoader

```python
from langchain_community.document_loaders import Docx2txtLoader
loader = Docx2txtLoader(word_path)
docs = loader.load()
```

### JSON — JSONLoader

El `jq_schema='.text'` apunta al campo `text` del JSON generado antes:

```python
from langchain_community.document_loaders import JSONLoader
loader = JSONLoader(
    file_path=json_path,
    jq_schema='.text',
)
docs = loader.load()
```

### El gotcha de metadata

Algunos loaders agregan su **propia metadata** (por ejemplo una clave `source`), que entra en conflicto con la metadata custom que el pipeline ya venía manejando.

> [!warning] Para evitar el choque, en el paso de indexing / creación del vector store se usa la clave `search_source` en lugar de `source`:

```python
dense_documents = [Document(page_content=doc.page_content, metadata={"id": str(i), "search_source": "dense"}) for i, doc in enumerate(splits)]
sparse_documents = [Document(page_content=doc.page_content, metadata={"id": str(i), "search_source": "sparse"}) for i, doc in enumerate(splits)]
```

Y el código de test de salida se actualiza en consecuencia (notar que aun así imprime `doc.metadata['source']`, la metadata que aporta el loader):

```python
for i, doc in enumerate(retrieved_docs, start=1):
    print(f"Document {i}: Document ID: {doc.metadata['id']} source: {doc.metadata['source']}")
    print(f"Content:\n{doc.page_content}\n")
```

Los document loaders importan, pero en un RAG basado en chunks lo que sigue es un **text splitter**.

## Code lab 11.2 – Text splitters

Los [[Text Splitter|text splitters]] dividen los documentos en los chunks que después se recuperan. El notebook es `CHAPTER11-2_TEXT_SPLITTERS.ipynb`. Hay dos razones para dividir: los documentos grandes **pierden representación de su contexto** en el embedding (un solo vector no captura bien un texto largo), y además casi todos los modelos de embedding tienen un **límite de input chico**.

> [!note] El modelo de embedding de OpenAI tiene una longitud de contexto de **8191 tokens** — pasarle un documento más grande directamente **da error**.

La parte difícil del splitting es mantener juntos los textos **semánticamente relacionados**.

### ChunkViz

La herramienta **ChunkViz** de **Greg Kamradt** (`chunkviz.up.railway.app`) visualiza cómo distintos splitters dividen un texto. Con `chunk_size` **1000** y `chunk_overlap` **200**, el splitter recursivo captura **párrafos enteros** alrededor de un tamaño de chunk de ~**434** caracteres (Figura 11.1), mientras que el character splitter **corta a mitad de oración** en cualquier configuración (Figura 11.2) — los párrafos parciales son ruido para el LLM.

![[11-fig-11.1-recursive-splitter.jpg]]
*Figure 11.1 – Recursive Character Text Splitter captures whole paragraphs at 434 characters*

![[11-fig-11.2-character-splitter.jpg]]
*Figure 11.2 – Character splitter captures partial paragraphs at 434 characters*

> [!tip] Usá ChunkViz para **visualizar** cómo queda tu splitting antes de comprometerte con una estrategia: ahí se ve de inmediato si los chunks rompen oraciones o respetan párrafos.

### Character text splitter

El `[[CharacterTextSplitter]]` es el más simple: corta en chunks arbitrarios de N caracteres. Se mejora un poco indicando un separador (por ejemplo `\n`):

```python
from langchain_text_splitters import CharacterTextSplitter
text_splitter = CharacterTextSplitter(
    separator="\n",
    chunk_size=1000,
    chunk_overlap=200,
    is_separator_regex=False,
)
splits = text_splitter.split_documents(docs)
```

El primer chunk resultante (`split[0]`), verbatim, con su markup de saltos de línea:

```
Environmental 
Report
2023What's 
inside
About this report
This report provides an overview of Google's environmental 
sustainability efforts and performance. Unless otherwise 
specified, the data and metrics in this report are for our 
fiscal year ending December 31, 2022.
…
Help in
g people make  14
```

Y el chunk siguiente:

```
Highlights  6
Our sustainability strategy 7
…
Appendix  85
```

El splitter cuenta ~**1000** caracteres, busca el `\n` más cercano y corta ahí (puede caer a mitad de oración); el segundo chunk **retrocede 200** caracteres (el `chunk_overlap`). Los 4 parámetros:

- **Separators** — el default es `\n\n`; si tu documento no tiene dobles saltos de línea, **nunca divide** → hay que elegir un separador que encaje con el contenido.
- **Chunk size** — el conteo de caracteres objetivo (bastante consistente, no exacto).
- **Chunk overlap** — los caracteres que se solapan entre chunks consecutivos, para capturar el contexto de los bordes. Es como la **sliding window de las [[CNN]] (redes neuronales convolucionales)**.
- **Is separator regex** — si el separador se interpreta como una expresión regular.

> [!warning] La trampa del default `\n\n`: si tu documento no tiene dobles saltos de línea, el `CharacterTextSplitter` con la configuración por defecto **no divide nada**. Elegí siempre un separador adecuado al contenido real.

Otras notas: usa el objeto **[[Document]] de LangChain** vía `create_documents` (para strings crudos se usa `split_text`); `create_documents` **espera una lista** (envolvé un string suelto en `[]`); **splitting** y **[[Chunking|chunking]]** son términos intercambiables.

### Recursive character text splitter

El `[[RecursiveCharacterTextSplitter]]` es el splitter genérico **recomendado** por LangChain (es el que más se usa en los labs del libro). Divide recursivamente manteniendo el texto relacionado adyacente: recibe una **lista de caracteres** que prueba en orden hasta que los chunks sean suficientemente chicos. La lista default es `["\n\n", "\n", " ", ""]`; el libro le agrega `". "` para mantener juntos párrafos, oraciones (por `"\n"` y `". "`) y palabras:

```python
recursive_splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", ". ", " ", ""],
    chunk_size=1000,
    chunk_overlap=200
)
splits = character_splitter.split_documents(docs)
```

Divide primero por `"\n\n"` (párrafos); si un chunk supera 1000 → pasa al siguiente separador (`"\n"`), y así sucesivamente.

> [!note] Los 3 pasos del algoritmo recursivo: (1) busca el último espacio/salto de línea en el rango `[chunk_size - chunk_overlap, chunk_size]` para cortar en límites de palabra/línea; (2) divide en el chunk-anterior + el resto-posterior; (3) aplica recursivamente al texto restante hasta que todos los chunks entren en `chunk_size`.

> [!tip] El `RecursiveCharacterTextSplitter` es la recomendación genérica de LangChain y la opción por defecto sensata. Agregar `". "` a los separadores ayuda a no cortar oraciones a la mitad.

Es útil para documentos grandes frente a los límites de input del modelo. Aun así, sigue dividiendo por **separadores, no por semántica real** → no detecta dos párrafos que en realidad son un mismo hilo de pensamiento.

### Semantic chunker

El `[[SemanticChunker]]` es experimental (viene del primer code lab) y apunta a evitar el número arbitrario de `chunk_size`, dividiendo por **semántica**. La descripción de LangChain, verbatim:

> "First splits on sentences. Then (it) combines ones next to each other if they are semantically similar enough."

Por debajo: divide en oraciones → las agrupa de a tres → fusiona los grupos cuando son lo suficientemente similares en el espacio de [[Embeddings|embeddings]]:

```python
from langchain_experimental.text_splitter import SemanticChunker
embedding_function = OpenAIEmbeddings()
semantic_splitter = SemanticChunker(embedding_function, number_of_chunks=200)
splits = semantic_splitter.split_documents(docs)
```

Usa el mismo modelo de embedding de OpenAI (consume embeddings → **cuesta dinero**). El parámetro `number_of_chunks=200` define la granularidad deseada (más alto = más chunks, más finos; más bajo = menos chunks, más grandes).

> [!warning] El `SemanticChunker` es débil cuando la semántica es difícil de discernir: mucho código, direcciones, nombres o IDs de referencia internos confunden la agrupación por similitud.

> [!tip] Probá variar `chunk_size` / `chunk_overlap` / `number_of_chunks` según el splitter para ver cómo cambian los chunks resultantes.

### Tabla 11.1 — Comparación de text splitters

| Splitter | Cómo divide | Parámetro clave | Cuándo |
|---|---|---|---|
| [[CharacterTextSplitter]] | Cada N caracteres, cortando en el separador más cercano (puede ser a mitad de oración) | `separator` (default `\n\n`) | El más simple; cuando el contenido tiene un separador claro y consistente |
| [[RecursiveCharacterTextSplitter]] | Prueba una lista de separadores en orden hasta que el chunk entra en `chunk_size`, respetando párrafos/oraciones/palabras | `separators` (lista ordenada) | Recomendado genérico de LangChain; el default sensato para documentos grandes |
| [[SemanticChunker]] | Por similitud semántica: agrupa oraciones vecinas si son similares en el espacio de embeddings | `number_of_chunks` | Cuando se quiere evitar el chunk_size arbitrario y la semántica es discernible (mal con código/IDs) |

> [!note] El recursivo y el character splitter dividen por **estructura** (separadores), no por significado; solo el `SemanticChunker` divide por **semántica**, a costa de consumir embeddings.

## Code lab 11.3 – Output parsers

Los [[Output Parser|output parsers]] estructuran la respuesta del LLM, ya sea para pasarla al siguiente paso de la cadena o como salida final. El notebook es `CHAPTER11-3_OUTPUT_PARSERS.ipynb`. El lab muestra dos.

### String output parser

El `[[StrOutputParser]]` ya se venía usando; solo se asigna a una variable:

```python
from langchain_core.output_parsers import StrOutputParser
str_output_parser = StrOutputParser()
```

Devuelve la respuesta del LLM como **string** al siguiente eslabón de la cadena.

### JSON output parser

El `[[JsonOutputParser]]` devuelve **JSON**. Puede que no lo necesites: muchos modelos nuevos ya soportan structured output nativo (JSON / XML). Los imports:

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.pydantic_v1 import BaseModel, Field
from langchain_core.outputs import Generation
import json
```

El modelo [[Pydantic]] que define la estructura de salida:

```python
class FinalOutputModel(BaseModel):
    relevance_score: float = Field(description="The relevance score of the retrieved context to the question")
    answer: str = Field(description="The final answer to the question")
```

El parser, atado al modelo:

```python
json_parser = JsonOutputParser(pydantic_model=FinalOutputModel)
```

La función `format_json_output` (ubicada entre `extract_score` y `conditional_answer`):

```python
def format_json_output(x):
    # print(x)
    json_output = {"relevance_score": extract_score(x['relevance_score']), "answer": x['answer'],
    }
    return json_parser.parse_result([Generation(text=json.dumps(json_output))])
```

El `conditional_answer` actualizado, que ahora devuelve `format_json_output(x)` cuando el contexto es relevante:

```python
def conditional_answer(x):
    relevance_score = extract_score(x['relevance_score'])
    if relevance_score < 4:
        return "I don't know."
    else:
        return format_json_output(x)
```

### La cadena combinada

Las dos cadenas previas se fusionan en un único `rag_chain` ([[LCEL]]): recupera el contexto y la pregunta en paralelo, formatea los documentos, calcula en paralelo `relevance_score` y `answer` (cada uno con su propio prompt | llm | `str_output_parser`), y finalmente asigna `final_result` vía `conditional_answer`:

```python
rag_chain = (
    RunnableParallel({"context": ensemble_retriever, "question": RunnablePassthrough()})
    | RunnablePassthrough.assign(context=format_docs)
    | RunnableParallel(
        {
            "relevance_score": (
                RunnablePassthrough()
                | (lambda x: relevance_prompt_template.format(question=x['question'], retrieved_context=x['context']))
                | llm
                | str_output_parser
            ),
            "answer": (
                RunnablePassthrough()
                | prompt
                | llm
                | str_output_parser
            ),
        }
    )
    | RunnablePassthrough().assign(final_result=conditional_answer)
)
```

> [!note] Las dos cadenas anteriores quedan unidas en una sola; se sigue usando `str_output_parser` como antes, y el `JsonOutputParser` se aplica **adentro** de `format_json_output` (llamada desde `conditional_answer`). Esta simplificación **pierde** el contexto que las labs previas arrastraban — es solo un setup alternativo de la cadena, no necesariamente mejor.

El código de la corrida de prueba:

```python
result = rag_chain.invoke(user_query)
print(f"Original Question: {user_query}\n")
print(f"Relevance Score: {result['relevance_score']}\n")
print(f"Final Answer:\n{result['final_result']['answer']}\n\n")
print(f"Final JSON Output:\n{result}\n\n")
```

La salida tiene esta estructura: un **Relevance Score 5**; un Final Answer sobre las iniciativas ambientales de Google; y un Final JSON Output que es un dict con `'relevance_score': '5'` (string) + `'answer'` (string) a nivel raíz, más un dict anidado `'final_result'` con `'relevance_score': 5.0` (float) + `'answer'`:

```
Relevance Score: 5

Final Answer:
<respuesta sobre las iniciativas ambientales de Google…>

Final JSON Output:
{'relevance_score': '5', 'answer': '<…>', 'final_result': {'relevance_score': 5.0, 'answer': '<…>'}}
```

> [!warning] Es difícil confiar en que un LLM devuelva siempre un formato determinado. Un sistema robusto embebe el parser **más adentro** de la cadena y agrega más chequeos de formato; este lab es una demo liviana, no una solución de producción.

## Citas

> "First splits on sentences. Then (it) combines ones next to each other if they are semantically similar enough."

## Para aplicar

- **Convertí y cargá según la fuente** — usá el [[Document Loader|document loader]] que matchee el formato del origen: `BSHTMLLoader` (HTML local), `PyPDF2`/`PyPDF`/`PyMuPDF`/`Unstructured`/`AzureAIDocumentIntelligenceLoader`/`UpstageLayoutAnalysisLoader` (PDF), `Docx2txtLoader` (Word), `JSONLoader` con `jq_schema` (JSON). Son intercambiables: solo cambia cómo se llena `docs`.
- **Resolvé el choque de metadata con `search_source`** — si un loader agrega su propia clave `source`, usá `search_source` en la metadata custom al crear los vector stores para no perder la tuya.
- **Empezá por el `RecursiveCharacterTextSplitter`** — es la recomendación genérica de LangChain; agregale `". "` a los separadores para no cortar oraciones; subí a `SemanticChunker` solo si la semántica del texto es clara (y aceptás el costo de embeddings).
- **Respetá el límite de 8191 tokens** — dimensioná `chunk_size` + `chunk_overlap` para no superar la longitud de contexto del modelo de embedding de OpenAI.
- **Visualizá con ChunkViz** (`chunkviz.up.railway.app`) antes de fijar la estrategia de chunking, y probá variar `chunk_size` / `chunk_overlap` / `number_of_chunks`.
- **Reiniciá el kernel** después de cada `%pip install`.
- **Estructurá la salida con un parser** — `StrOutputParser` para pasar strings entre eslabones; `JsonOutputParser` + un modelo [[Pydantic]] (`FinalOutputModel`) cuando necesitás JSON validado, pero embebé el parser bien adentro y agregá chequeos porque el LLM no garantiza el formato.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[10 - Key RAG Components in LangChain]] — capítulo anterior: los componentes **clave** (vector stores, retrievers, LLMs) que estos componentes de soporte alimentan · [[12 - Combining RAG with the Power of AI Agents and LangGraph]] — capítulo siguiente: **LangGraph y agentes de IA**.
- [[02 - Code Lab - An Entire RAG Pipeline]] — de ahí vienen `bs4`/[[BeautifulSoup]] y el `WebBaseLoader` (loader de sitios en vivo, contraparte del `BSHTMLLoader` local de este capítulo).
- [[07 - The Key Role Vectors and Vector Stores Play in RAG]] y [[08 - Similarity Searching with Vectors]] — el [[SemanticChunker]] y el [[RecursiveCharacterTextSplitter]] (y el límite de **8191 tokens**) atan con los embeddings y el PDF loader que esos capítulos profundizan.
- [[Document Loader]] · [[Text Splitter]] · [[Output Parser]] — las tres familias de componentes de soporte de LangChain (candidatos a nota propia).
- [[CharacterTextSplitter]] · [[RecursiveCharacterTextSplitter]] · [[SemanticChunker]] · [[Chunking]] · [[Chunk Overlap]] — splitting de documentos.
- [[StrOutputParser]] · [[JsonOutputParser]] · [[Pydantic]] · [[Document]] — parsing de salida y estructura de datos.
- [[BeautifulSoup]] · [[PyPDF2]] · [[Embeddings]] · [[LCEL]] · [[Runnable]] · [[RunnableParallel]] · [[LangChain]] · [[CNN]] — librerías y abstracciones que el capítulo usa.
