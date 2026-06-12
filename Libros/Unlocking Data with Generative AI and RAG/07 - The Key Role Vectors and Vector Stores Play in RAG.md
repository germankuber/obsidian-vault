---
title: 07 - The Key Role Vectors and Vector Stores Play in RAG
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 7
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - The Key Role Vectors and Vector Stores Play in RAG
  - Vectores y Vector Stores en RAG
updated: 2026-06-12
---

# 07 - The Key Role Vectors and Vector Stores Play in RAG

> [!info] Capítulo 7 · *Unlocking Data with Generative AI and RAG* — Packt (ISBN 9781835887905), Keith Bourne
> Los [[Vectors|vectores]] son el "secret ingredient" de RAG: este capítulo explica **qué es** un vector, **cómo se crea** ([[Vectorization|vectorización]]) y **dónde se guarda** ([[Vector Store|vector stores]]). Los vectores se reparten en DOS capítulos — este = vectores + vector stores; el [[08 - Similarity Searching with Vectors|cap. 8]] = las búsquedas vectoriales (cómo se usan). El recorrido culmina en un code lab que atraviesa 50 años de vectorizadores ([[TF-IDF]] 1972 → [[Doc2Vec]] → [[BERT]] → OpenAI/GPT-4) y en los factores para elegir embedding y vector store. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[06 - Interfacing with RAG and Gradio]] · siguiente [[08 - Similarity Searching with Vectors]].

## Resumen

Si un [[LLM]] es el motor de RAG, los **[[Vectors|vectores]]** son su combustible: representaciones matemáticas del texto que habilitan **velocidad** (búsqueda rápida) y **precisión** (representación semántica). El capítulo abre distinguiendo **[[Embeddings|embeddings]] vs vectores** (los embeddings son el tipo concreto de vector que usa el NLP, y como los LLMs de RAG son NLP, sus vectores se llaman embeddings; en RAG los términos se usan indistintamente), define qué es un vector (algo con **magnitud Y dirección**, no solo números), y desmenuza sus **dimensiones**: el modelo **ada** de OpenAI devuelve **1.536 floats** de doble precisión (64-bit) por texto, sin importar si el texto es una pregunta corta o una sección larga. De ahí saltan dos ideas modernas: la **[[Quantization|cuantización]]** (compresión con pérdida de los parámetros del modelo) y el **[[Adaptive Retrieval|adaptive retrieval]]** habilitado por las **[[Matryoshka Embeddings|Matryoshka embeddings]]** (`text-embedding-3-large`), que acelera la búsqueda 30–90%.

Luego rastrea **dónde viven los vectores en tu código**: la vectorización ocurre en dos lugares (los datos al indexar, la query al recuperar) y un tercero al evaluar respuestas ([[09 - Evaluating RAG Quantitatively and with Visualizations|cap. 9]]); ambas vectorizaciones alimentan la **similarity search** que compara distancias contra el [[Vector Store|vector store]]. El capítulo insiste en que **el tamaño del texto que vectorizás importa** (chunks grandes → embedding diluido; chunks chicos → poco contexto) y en que **no todas las semánticas son iguales** (los vectorizadores son modelos NLP con calidades dispares; existen modelos de dominio y se pueden fine-tunear).

El corazón práctico es el **Code lab 7.1**, que recorre la evolución de los vectorizadores con código real y compara qué documento recupera cada uno frente a la misma query: **[[TF-IDF]]** (1972, vectores sparse, estadístico), **[[Doc2Vec]]** (neuronal, `vector_size` ajustable), **[[BERT]]** (primer transformer, 768 dims, local) y **OpenAI** (basado en GPT-3/GPT-4, miles de millones de parámetros). Cierra con los **factores para elegir vectorización** (calidad/[[MTEB]], costo, red, velocidad y la crítica **compatibilidad** entre embeddings) y un **panorama de vector stores** ([[Chroma DB]], [[LanceDB]], [[Milvus]], [[pgvector]], [[Pinecone]], [[Weaviate]]) con sus tres capas de arquitectura y las **6 consideraciones** para elegirlo. El [[08 - Similarity Searching with Vectors|cap. 8]] continúa con los algoritmos de similitud.

> [!note] Dos líneas para dos capítulos. Todo este capítulo gira alrededor de UNA línea de código — `vectorstore = Chroma.from_documents(documents=splits, embedding=OpenAIEmbeddings())` — y el cap. 8 gira alrededor de otra — `retriever = vectorstore.as_retriever()`. Que cada una merezca un capítulo entero mide lo importantes que son los vectores. Código: github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_07, archivo `CHAPTER7-1_COMMON_VECTORIZATION_TECHNIQUES.ipynb`.

## Fundamentos de los vectores en RAG

### Embeddings vs vectores

Los **[[Embeddings|embeddings]]** son un TIPO específico de representación vectorial usada en NLP. Como los LLMs de RAG son parte del NLP, en este contexto los vectores se llaman embeddings; pero los **[[Vectors|vectores]]** en general se usan en muchísimos campos y para representar muchos tipos de objetos, no solo texto. Dentro de RAG, los términos *embeddings*, *vectores*, *vector embeddings* y *embedding vectors* se usan de forma intercambiable.

### ¿Qué es un vector?

Un vector es una **representación matemática del texto**, lo que permite hacer operaciones matemáticas con él. Esa matematización trae dos beneficios asociados: **velocidad** (buscar texto más rápido) y **precisión** (capturar la representación semántica). La clave conceptual es que un vector no es solo una lista de números: tiene **magnitud Y dirección**.

> [!note] El villano *Vector* de *Despicable Me* lo define mejor que cualquier libro de texto: "I go by the name of… Vector. It's a mathematical term, a quantity represented by an arrow with both direction and magnitude."

### Dimensiones y tamaño

Las **dimensiones** de un vector son su tamaño: la cantidad de números de punto flotante que contiene. Podemos inspeccionar las primeras 5 dimensiones del embedding de una pregunta:

```python
question = "What are the advantages of using RAG?"
question_embedding = embedding_function.embed_query(question)
first_5_numbers = question_embedding[:5]
print(f"User question embedding (first 5 dimensions): {first_5_numbers}")
```

Salida:

```
User question embedding (first 5 dim): [-0.006319054113595048, -0.0023517232115089787, 0.015498643243434815, -0.02267445873596028, 0.017820641897159206]
```

El vector completo tiene **1.536 floats**, cada uno de 17–20 dígitos. Esa longitud de dígitos se relaciona con el formato de **punto flotante de doble precisión (64-bit)**, que permite distinciones de grano muy fino. La salida parece (y ES) una **lista de Python** — OpenAI devuelve una lista —, pero en ML lo típico es un **NumPy array** (se ven idénticos al imprimirse; punto ya tocado en [[01 - What Is Retrieval-Augmented Generation (RAG)]]).

> [!note] Fun fact — Cuantización. Como los embeddings, la **[[Quantization|cuantización]]** trabaja con floats de alta precisión, pero hace lo contrario: CONVIERTE los parámetros del modelo (pesos, activaciones) de alta precisión a un formato de MENOR precisión. Reduce memoria y cómputo → abarata el pre-entrenamiento, el entrenamiento, el fine-tuning Y la inferencia (hardware más chico/barato). Es una **técnica de compresión con pérdida (lossy)**: se pierde algo de información, y la precisión reducida puede costar accuracy frente al LLM original de alta precisión.

> [!tip] Al elegir un algoritmo de embedding, fijate en la longitud de los valores para asegurarte de que use floats de alta precisión si la accuracy te importa.

Contamos las dimensiones con `len()`:

```python
embedding_size = len(question_embedding)
print(f"Embedding size: {embedding_size}")
```

Salida:

```
Embedding size: 1536
```

Las 1.533 dimensiones extra (más allá de las 3 de un vector geométrico clásico) agregan precisión. Los vectorizadores modernos manejan cientos o miles de dimensiones; el modelo **ada** de OpenAI usa **1.536 por defecto** (fue entrenado a un tamaño fijo, así que truncarlo cambia el contexto capturado).

> [!note] Adaptive retrieval y Matryoshka embeddings. El nuevo `text-embedding-3-large` permite **cambiar el tamaño** del vector, lo que habilita el **[[Adaptive Retrieval|adaptive retrieval]]**: generás embeddings a varios tamaños, buscás primero en los vectores de MENOR dimensión (rápido) para acercarte, y después te *adaptás* a los vectores de MAYOR dimensión (lento) para finalizar. Puede aumentar la velocidad de búsqueda un **30–90%**. Se llaman **[[Matryoshka Embeddings|Matryoshka embeddings]]** por las muñecas rusas: idénticas pero de tamaño variable.

> [!tip] Considerá adaptive retrieval / Matryoshka al optimizar un pipeline RAG de producción con uso intensivo.

## Dónde viven los vectores en tu código

### Vectorización en dos lugares

La [[Vectorization|vectorización]] ocurre en **dos lugares**: (1) al vectorizar los datos originales del sistema RAG (etapa de [[Indexing|indexing]]); (2) al vectorizar la query del usuario (etapa de [[Retrieval|retrieval]]). Ambas alimentan la similarity search. El retriever de LangChain es el objeto que ejecuta esa búsqueda por similitud:

```python
rag_chain_with_source = RunnableParallel(
    {"context": retriever, "question": RunnablePassthrough()}
).assign(answer=rag_chain_from_docs)
```

Hay un **tercer lugar** donde aparece la vectorización — la evaluación de las respuestas RAG —, cubierto en el [[09 - Evaluating RAG Quantitatively and with Visualizations|cap. 9]].

### Los vector stores almacenan los vectores

Un **[[Vector Store|vector store]]** (normalmente, aunque no siempre, una vector database) está optimizado para guardar y servir vectores. PODRÍAS construir RAG sin uno, pero perderías las optimizaciones de memoria, cómputo y precisión de búsqueda.

> [!note] "Vector database" vs "vector store". Existen herramientas que NO son bases de datos pero cumplen el mismo propósito de servir vectores. El libro las agrupa a todas como **vector stores** (consistente con la documentación de LangChain), aunque "vector database" sea el término más popular; usa *vector store* por precisión.

### La similitud compara los vectores

La **similarity search** es una operación matemática que mide la **distancia** entre el embedding de la query del usuario y los embeddings de los datos originales. Hay múltiples algoritmos (cubiertos en el [[08 - Similarity Searching with Vectors|cap. 8]]) y devuelve los embeddings más cercanos, ordenados de más cerca a más lejos.

> [!note] En el código del libro hay una relación **1:1** entre chunk y embedding. Pero en muchas apps (p. ej. un chatbot de Q&A con contenido largo) los chunks llevan un **foreign key ID** que apunta a una pieza de contenido más grande, para poder recuperar el contenido completo y no solo el chunk. La arquitectura varía según la aplicación.

## El tamaño del texto que vectorizás importa

La query `What are the advantages of using RAG?` es corta, así que un vector de 1.536 dims la representa de sobra. Los datos, en cambio, vienen de un `WebBaseLoader` y se parten con el **[[Semantic Chunker|SemanticChunker]]**:

```python
text_splitter = SemanticChunker(embedding_function)
splits = text_splitter.split_documents(docs)
```

El tercer chunk (`splits[2]`) ilustra el tamaño de las secciones almacenadas:

```
splits[2]:
Image and video generative models, covered in Chapter 16, Going Beyond the LLM, are foundation models too. Foundation models are large models trained on vast quantities of data and adaptable to a wide range of downstream tasks. The GPT architecture is the foundation model behind ChatGPT, which was then fine-tuned for Chat.
```

El **Semantic Chunker** usa embeddings para partir por semántica/contexto, no por un tamaño arbitrario. El hecho contra-intuitivo clave: **sin importar el tamaño del texto de entrada, el embedding SIEMPRE tiene el mismo tamaño** — la query corta y la sección larga son ambas de 1.536 dims —; y aun así la similarity search detecta las similitudes semánticas pese a esa disparidad de tamaño ("esto es lo que hace que a los matemáticos les encante la matemática"). Pero el tamaño SÍ importa entre chunks:

> [!warning] Trade-off de tamaño de chunk. Contenido más **grande** → embedding más **diluido**. Contenido más **chico** → menos contexto para hacer match. Hay que encontrar un balance delicado entre tamaño del chunk y representación del contexto. Más técnicas de chunking en el [[11 - Using LangChain to Get More from RAG|cap. 11]] (splitters de LangChain).

## No todas las semánticas son iguales

Un error común es usar el primer algoritmo de vectorización que aparece y asumir que es el mejor. Estos algoritmos son a su vez **modelos NLP grandes**, que varían en capacidad/calidad como los LLMs. Ejemplo: los modelos viejos no distinguían `bark` (ladrido) de `bark` (corteza); los nuevos usan el contexto. Los modelos de vectorización **de dominio específico** (p. ej. entrenados con papers científicos) pueden superar a los genéricos en ese dominio.

> [!note] Fun fact — Podés **fine-tunear modelos de embedding** (no solo LLMs). Mejor entendimiento del dominio → mejor similarity search → mejor RAG.

> [!tip] Fine-tuneá los modelos de embedding para tu dominio cuando la calidad de la recuperación lo justifique.

## Code lab 7.1 — Técnicas comunes de vectorización

El lab recorre la evolución de los vectorizadores a lo largo de décadas. Instalación:

```bash
%pip install gensim --user
%pip install transformers
%pip install torch
```

### TF-IDF (Term Frequency–Inverse Document Frequency)

**[[TF-IDF]]** tiene raíces en **1972**, cuando **Karen Spärck Jones** (científica de la computación británica autodidacta) introdujo la **IDF (inverse document frequency)**.

> [!quote] "the specificity of a term can be quantified as an inverse function of the number of documents in which it occurs." — Karen Spärck Jones

El ejemplo de Shakespeare: a lo largo de sus **37 obras**, `Romeo` es la palabra de mayor puntaje (frecuente pero solo en UN documento, *Romeo and Juliet*) → **df = 1**, **idf = 1.57** (la más alta); `sweet` aparece en TODAS las obras → **df = 37**, **idf = 0** (no informativa).

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
tfidf_documents = [split.page_content for split in splits]
tfidf_vectorizer = TfidfVectorizer()
tfidf_matrix = tfidf_vectorizer.fit_transform(tfidf_documents)
vocab = tfidf_vectorizer.get_feature_names_out()
tf_values = tfidf_matrix.toarray()
idf_values = tfidf_vectorizer.idf_
word_stats = list(zip(vocab, tf_values.sum(axis=0), idf_values))
word_stats.sort(key=lambda x: x[2], reverse=True)
print("Word\t\tTF\t\tIDF")
print("----\t\t--\t\t---")
for word, tf, idf in word_stats[:10]:
    print(f"{word:<12}\t{tf:.2f}\t\t{idf:.2f}")
```

Este modelo debe **entrenarse** sobre tu **corpus**. Sus vectores son **[[Sparse Vectors|sparse vectors]]** (vectores dispersos), a diferencia de los **[[Dense Vectors|dense vectors]]** (densos) de los labs posteriores — distinción detallada en el [[08 - Similarity Searching with Vectors|cap. 8]]. La salida:

| Word | TF | IDF |
|---|---|---|
| 000 | 1.41 | 2.95 |
| 1024 | 1.00 | 2.95 |
| 123 | 1.00 | 2.95 |
| 13 | 1.00 | 2.95 |
| 15 | 1.00 | 2.95 |
| 16 | 1.00 | 2.95 |
| 192 | 1.00 | 2.95 |
| 1m | 1.00 | 2.95 |
| 200 | 1.00 | 2.95 |
| 2024 | 1.00 | 2.95 |

El empate de 10 términos numéricos en `idf = 2.95` se debe a que el corpus es minúsculo. Luego, la recuperación con la query del usuario:

```python
tfidf_user_query = ["What are the advantages of RAG?"]
new_tfidf_matrix = tfidf_vectorizer.transform(tfidf_user_query)
tfidf_similarity_scores = cosine_similarity(new_tfidf_matrix, tfidf_matrix)
tfidf_top_doc_index = tfidf_similarity_scores.argmax()
print("TF-IDF Top Document:\n", tfidf_documents[tfidf_top_doc_index])
```

Usa **[[Cosine Similarity|cosine similarity]]** (cap. 8). El top doc de TF-IDF COINCIDE con el del retriever de OpenAI (ambos el chunk "Can you imagine what you could do…that is what RAG does").

> [!warning] No te dejes engañar: un algoritmo de **1972** no es realmente tan bueno como OpenAI. En datasets más grandes y queries complejas del mundo real se necesitan embeddings modernos. TF-IDF cuesta capturar contexto/semántica. (Adelanto: **[[BM25]]**, en el cap. 8, es un algoritmo de **keyword search** muy popular basado en TF-IDF.)

### Word2Vec, Sentence2Vec, Doc2Vec

**[[Word2Vec]]** fue uno de los primeros métodos de aprendizaje no supervisado para aprender vectores de palabras (significado/relaciones semánticas); **[[Doc2Vec]]** produce vectores de documentos enteros (contexto/tema global); **Sentence2Vec** opera a nivel de oración. Word2Vec sirve para similitud/analogía de palabras; Doc2Vec/Sentence2Vec para tareas a nivel documento (similitud, clasificación, retrieval). El libro usa **Doc2Vec** (documentos grandes). Entrenamiento:

```python
from gensim.models.doc2vec import Doc2Vec, TaggedDocument
from sklearn.metrics.pairwise import cosine_similarity
doc2vec_documents = [split.page_content for split in splits]
doc2vec_tokenized_documents = [doc.lower().split() for doc in doc2vec_documents]
doc2vec_tagged_documents = [TaggedDocument(words=doc, tags=[str(i)]) for i, doc in enumerate(doc2vec_tokenized_documents)]
doc2vec_model = Doc2Vec(doc2vec_tagged_documents, vector_size=100, window=5, min_count=1, workers=4)
doc2vec_model.save("doc2vec_model.bin")
```

> [!note] Fun fact — Ya entrenaste DOS modelos de lenguaje. TF-IDF y Doc2Vec son ambos modelos de lenguaje, así que a esta altura del lab ya entrenaste dos.

Uso:

```python
loaded_doc2vec_model = Doc2Vec.load("doc2vec_model.bin")
doc2vec_document_vectors = [loaded_doc2vec_model.dv[str(i)] for i in range(len(doc2vec_documents))]
doc2vec_user_query = ["What are the advantages of RAG?"]
doc2vec_tokenized_user_query = [content.lower().split() for content in doc2vec_user_query]
doc2vec_user_query_vector = loaded_doc2vec_model.infer_vector(doc2vec_tokenized_user_query[0])
doc2vec_similarity_scores = cosine_similarity([doc2vec_user_query_vector], doc2vec_document_vectors)
doc2vec_top_doc_index = doc2vec_similarity_scores.argmax()
print("\nDoc2Vec Top Document:\n", doc2vec_documents[doc2vec_top_doc_index])
```

Con **`vector_size=100`** el top doc es DIFERENTE (el chunk "Once you have introduced the new knowledge…fine-tuning…less reliable for factual recall"). Cambiando a `vector_size=1536`:

```python
doc2vec_model = Doc2Vec(doc2vec_tagged_documents, vector_size=100, window=5, min_count=1, workers=4)
# vs
doc2vec_model = Doc2Vec(doc2vec_tagged_documents, vector_size=1536, window=5, min_count=1, workers=4)
```

…el resultado pasa a COINCIDIR con OpenAI (el chunk "Can you imagine…"). Los resultados no son consistentes; más datos de entrenamiento lo mejoran. Doc2Vec es **basado en red neuronal** (tiene en cuenta las palabras circundantes), a diferencia de TF-IDF (medida estadística de keywords).

### BERT (Bidirectional Encoder Representations from Transformers)

**[[BERT]]** es totalmente neuronal y fue el primero en aplicar el **[[Transformer|transformer]]**, un paso mayor hacia los LLMs modernos (los modelos de ChatGPT también son transformers, con corpus más grande y otras técnicas). BERT puede ser un modelo **local autónomo** (sin API como OpenAI), ventaja en entornos con red limitada. Su rasgo definitorio es el **[[Self-Attention|self-attention]]** para capturar dependencias entre palabras; tiene múltiples capas transformer y fue pre-entrenado con Wikipedia + BookCorpus usando next-sentence-prediction.

```python
from transformers import BertTokenizer, BertModel
import torch
from sklearn.metrics.pairwise import cosine_similarity
bert_documents = [split.page_content for split in splits]
bert_tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
bert_model = BertModel.from_pretrained('bert-base-uncased')
bert_vector_size = bert_model.config.hidden_size
print(f"Vector size of BERT (base-uncased) embeddings: {bert_vector_size}\n")
bert_tokenized_documents = [bert_tokenizer(doc, return_tensors='pt', max_length=512, truncation=True) for doc in bert_documents]
bert_document_embeddings = []
with torch.no_grad():
    for doc in bert_tokenized_documents:
        bert_outputs = bert_model(**doc)
        bert_doc_embedding = bert_outputs.last_hidden_state[0, 0, :].numpy()
        bert_document_embeddings.append(bert_doc_embedding)
bert_user_query = ["What are the advantages of RAG?"]
bert_tokenized_user_query = bert_tokenizer(bert_user_query[0], return_tensors='pt', max_length=512, truncation=True)
bert_user_query_embedding = []
with torch.no_grad():
    bert_outputs = bert_model(**bert_tokenized_user_query)
    bert_user_query_embedding = bert_outputs.last_hidden_state[0, 0, :].numpy()
bert_similarity_scores = cosine_similarity([bert_user_query_embedding], bert_document_embeddings)
bert_top_doc_index = bert_similarity_scores.argmax()
print("BERT Top Document:\n", bert_documents[bert_top_doc_index])
```

Diferencia clave: aquí NO fine-tuneamos sobre nuestros datos (BERT viene pre-entrenado), aunque hacerlo es lo recomendado. Salida:

```
Vector size of BERT (base-uncased) embeddings: 768
```

…y un top doc pobre ("Or if you are developing in a legal field…Vector Store or Vector Database?"), que no es el mejor chunk: necesita fine-tuning para este dominio (sobre todo siendo un modelo local más chico vs una API grande). Nótese el tamaño de vector de **768**.

### OpenAI y otros servicios de embedding a gran escala

`'bert-base-uncased'` tiene **110M de parámetros**; `'bert-large-uncased'` tiene **340M** (3×, puede crashear un kernel débil) — millones, NO miles de millones:

```python
bert_tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
bert_model = BertModel.from_pretrained('bert-base-uncased')
# vs
bert_tokenizer = BertTokenizer.from_pretrained('bert-large-uncased')
bert_model = BertModel.from_pretrained('bert-large-uncased')
```

El modelo de embedding de OpenAI se basa en **GPT-3** (**175 mil millones** de parámetros); los más nuevos, en **GPT-4** (~**un billón** / one trillion). BERT se entrenó con **3.300 millones de palabras**; el corpus de GPT-3 ronda los **17 billones de palabras (17 trillion words, 45 TB)**. OpenAI ofrece tres modelos de embedding: el más viejo **`text-embedding-ada-002`** (GPT-3, usado aquí para ahorrar costo), y los basados en GPT-4 **`text-embedding-3-small`** y **`text-embedding-3-large`** (ambos soportan Matryoshka/adaptive retrieval). Otras nubes: **GCP** `text-embedding-preview-0409` (lanzado el 9 de abril de 2024; en ese momento el único otro modelo cloud a gran escala con soporte Matryoshka); **AWS** con modelos **Titan** + embeddings de **Cohere**; se esperaba pronto **Titan Text Embeddings V2**, también con Matryoshka previsto.

## Factores para elegir una opción de vectorización

### Calidad del embedding

No te apoyes solo en métricas genéricas. El **[[MTEB|Massive Text Embedding Benchmark (MTEB)]]** da `text-embedding-ada-002` = **61.0%** y `text-embedding-3-large` = **64.6%** — pero un gap de 3.6% en el benchmark NO significa 3.6% mejor (ni mejor en absoluto) para TU app.

> [!warning] No confíes ciegamente en benchmarks genéricos. Probá varios modelos en tu propio sistema RAG y comparalos con las técnicas de evaluación del [[09 - Evaluating RAG Quantitatively and with Visualizations|cap. 9]]; un modelo de dominio o auto-entrenado puede ganar.

### Costo

El modelo de embedding más caro de OpenAI cuesta **$0.13 por millón de tokens** → una página de 800 tokens = **$0.000104** (~1% de un centavo). Pero a volúmenes empresariales el costo trepa a miles o decenas de miles de dólares incluso en proyectos pequeños; existen APIs más baratas, y tu propio modelo solo cuesta hardware/hosting.

### Disponibilidad de red

Las redes fallan: una API de embedding hosteada (OpenAI) puede quedar inaccesible aun cuando tu UI funciona. Un modelo local en tu entorno evita ese punto de falla.

### Velocidad

Una API hosteada agrega latencia de red; pero lo local no siempre es más rápido (depende del tiempo de inferencia del modelo). Considerá latencia de red, tiempo de inferencia, hardware y generación por lotes.

### Compatibilidad de embeddings

> [!warning] Compatibilidad (MUY importante). Al comparar embeddings (query vs almacenados) DEBEN haber sido creados por el **mismo modelo de embedding** — cada modelo genera firmas vectoriales únicas, incluso modelos del mismo proveedor (los tres de OpenAI son mutuamente incompatibles). NO podés usar un modelo local de respaldo cuando la red se cae si tu store fue construido con una API propietaria: quedás casado con ese modelo/API (OpenAI no ofrece embeddings locales). Cambiar/actualizar el modelo de embedding al escalar = costo mayor (regenerar TODOS los embeddings), lo que puede empujarte hacia un modelo local.

## Empezar con vector stores

Los [[Vector Store|vector stores]] junto con otros data stores son el combustible del motor RAG.

### Fuentes de datos (más allá del vector)

El ejemplo básico no usa una DB extra (la página web es una fuente de datos no estructurada), pero las apps crecen y necesitan DBs — SQL, data lakes. Las arquitecturas posibles (**RDBMS**, NoSQL, NewSQL, data warehouses, data lakes) el libro las abstrae como la **data source**. Tu elección de vector store está fuertemente influida por la arquitectura de datos existente y las skills de tu equipo. Ejemplo: un equipo **PostgreSQL** debería considerar la extensión **[[pgvector]]** (convierte tablas de Postgres en vector stores reutilizando indexado/SQL familiares).

> [!tip] Si tu equipo ya está en Postgres, considerá pgvector antes que adoptar una DB vectorial nueva.

> [!note] Fun fact — SharePoint. SharePoint es un **CMS** que guarda cantidades masivas de datos no estructurados (PDF/Word/Excel/PowerPoint) = una base de conocimiento enorme. La GenAI explota bien lo no estructurado; APIs sofisticadas extraen el texto antes de vectorizar → suele ser una de las PRIMERAS fuentes de datos RAG en grandes empresas.

### Vector stores y sus 3 capas de arquitectura

Los vector stores (alias vector databases / vector search engines) están optimizados para operaciones de vectores de alta dimensión (no filas/columnas) → búsqueda por similitud rápida. Es posible (pero subóptimo) construir RAG sin uno.

> [!note] Las 3 capas de arquitectura. **Indexing layer** — organiza los vectores para queries rápidas (particionado en árbol, p. ej. **KD-trees**, o hashing, p. ej. **locality-sensitive hashing**). **Storage layer** — gestiona los datos en disco/memoria para rendimiento y escalabilidad. **Processing layer (opcional)** — transformaciones de vectores, cómputo de similitud y analítica en tiempo real.

(Se reitera la nota de terminología: "vector database" es el término popular, pero el libro usa *vector store* por precisión y consistencia con LangChain.)

### Opciones comunes de vector store

| Vector store | Tipo | Rasgo distintivo | Consideración |
|---|---|---|---|
| **[[Chroma DB]]** | Open source | Rápido; API simple; filtrado dinámico de colecciones; chunking/indexado integrados; self-hostable | Menos features avanzadas (búsqueda distribuida, múltiples indexados, hybrid search) |
| **[[LanceDB]]** | Open source | Búsqueda por similitud eficiente; **hybrid search** (vector+keyword); múltiples métricas de distancia + **[[HNSW]]** (ANN); integración LangChain | Comunidad más chica |
| **[[Milvus]]** | Open source | Escalable, cloud-native (Kubernetes); indexado multi-vector; sistema de plugins; distribuido + escalable horizontalmente | Más setup/gestión |
| **[[pgvector]]** | Extensión de PostgreSQL | Aprovecha el ecosistema/madurez de Postgres; hybrid search; perf reciente a la par de DBs dedicadas | Ideal solo si ya estás en Postgres |
| **[[Pinecone]]** | Managed / serverless | Indexado en tiempo real; hybrid search; distribuido; múltiples algoritmos de indexado; setup mínimo | Puede ser más caro |
| **[[Weaviate]]** | Open source (search engine) | Modelo de datos semántico basado en esquema; CRUD, validación, autorización; módulos ML (clasificación de texto, similitud de imágenes); API GraphQL; indexado en tiempo real | Más setup/configuración |

> [!note] Estas opciones se centran en vector stores integrados con LangChain; el panorama evoluciona rápido — chequeá la documentación de LangChain para las opciones vigentes.

## Elegir un vector store — las 6 consideraciones

Factores generales: escala de datos, rendimiento de búsqueda (velocidad + accuracy), complejidad de las operaciones de vectores, escalabilidad, facilidad de integración y APIs/docs/comunidad robustas. Las **6 consideraciones clave**:

- **Compatibilidad con la infraestructura existente** — que integre con tus DBs/warehouses/lakes actuales y las skills del equipo (p. ej. pgvector para un equipo Postgres).
- **Escalabilidad y rendimiento** — que soporte el crecimiento de datos y la performance; DBs distribuidas como Milvus o Elasticsearch-con-plugin-vectorial para gran escala.
- **Facilidad de uso y mantenimiento** — curva de aprendizaje, docs, soporte; los servicios managed como Pinecone simplifican operaciones; los self-hosted como Weaviate dan control/flexibilidad.
- **Seguridad de datos y cumplimiento** — features de seguridad, controles de acceso, cifrado; cumplir GDPR/HIPAA.
- **Costo y licenciamiento** — precio por volumen de datos / operaciones de búsqueda; open source = menor costo inicial pero más expertise in-house; managed = fees más altos pero más simple.
- **Ecosistema e integraciones** — client libraries/SDKs/APIs por lenguaje; compatibilidad con herramientas NLP/ML; masa crítica de comunidad.

> [!tip] No hay un vector store único para todo. Reevaluá la elección periódicamente a medida que el sistema crece.

## Citas

> "I go by the name of… Vector. It's a mathematical term, a quantity represented by an arrow with both direction and magnitude."

> "the specificity of a term can be quantified as an inverse function of the number of documents in which it occurs."

## Para aplicar

- **Inspeccioná dimensiones y precisión** — usá `embed_query(...)[:5]` y `len(question_embedding)` para verificar el tamaño (1.536 con ada) y que los floats sean de alta precisión.
- **Probá adaptive retrieval / Matryoshka** — con `text-embedding-3-large` generá embeddings a varios tamaños y buscá primero en los chicos (+30–90% de velocidad) al optimizar producción.
- **Balanceá el tamaño de chunk** — ni tan grande que diluya el embedding ni tan chico que pierda contexto; usá [[Semantic Chunker|SemanticChunker]] y mirá más splitters en el cap. 11.
- **Comparand vectorizadores en tu propio RAG** — corré TF-IDF, Doc2Vec, BERT y OpenAI sobre tus datos y compará el top doc; no te fíes solo del MTEB.
- **Fine-tuneá embeddings de dominio** — para mejorar la similarity search en tu campo.
- **Fijá un único modelo de embedding** — query y store deben usar el mismo; planificá el costo de re-embeddear si lo cambiás; un modelo local evita el lock-in de API.
- **Elegí el vector store por tu infra** — pgvector si estás en Postgres; managed (Pinecone) para simplicidad; aplicá las 6 consideraciones y reevaluá al escalar.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[06 - Interfacing with RAG and Gradio]] — capítulo anterior (UI con Gradio) · [[08 - Similarity Searching with Vectors]] — capítulo siguiente: los algoritmos de **similarity search**, **[[BM25]]**, **[[Cosine Similarity|cosine similarity]]** y la distinción **sparse vs dense vectors**; la línea `as_retriever()`.
- [[01 - What Is Retrieval-Augmented Generation (RAG)]] — el punto de lista-de-Python vs NumPy array; los vectores como vocabulario base.
- [[09 - Evaluating RAG Quantitatively and with Visualizations]] — el tercer lugar donde se vectoriza (evaluación) y la comparación de modelos de embedding.
- [[11 - Using LangChain to Get More from RAG]] — más técnicas de chunking / splitters de LangChain.
- Chapter 16 "Going Beyond the LLM" — modelos generativos multimodales (mencionado en `splits[2]`).
- [[Vectors]] · [[Embeddings]] · [[Vectorization]] · [[Vector Store]] — núcleo conceptual del capítulo.
- [[Adaptive Retrieval]] · [[Matryoshka Embeddings]] · [[Quantization]] — ideas modernas sobre dimensionalidad y precisión. (**candidatos a nota propia**)
- [[TF-IDF]] · [[Word2Vec]] · [[Doc2Vec]] · [[BERT]] · [[Transformer]] · [[Self-Attention]] — la evolución de los vectorizadores. (**candidatos a nota propia**)
- [[Cosine Similarity]] · [[Sparse Vectors]] · [[Dense Vectors]] · [[BM25]] · [[HNSW]] — conceptos de similitud (se desarrollan en el cap. 8). (**candidatos a nota propia**)
- [[Chroma DB]] · [[pgvector]] · [[Pinecone]] · [[Milvus]] · [[Weaviate]] · [[LanceDB]] — opciones de vector store. (**candidatos a nota propia**)
- [[MTEB]] · [[Fine-tuning]] · [[LangChain]] — calidad de embedding, mejora de dominio y capa de orquestación.
