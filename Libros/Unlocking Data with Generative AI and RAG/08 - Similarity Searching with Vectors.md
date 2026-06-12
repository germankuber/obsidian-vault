---
title: 08 - Similarity Searching with Vectors
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 8
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Similarity Searching with Vectors
  - Similarity Search
updated: 2026-06-12
---

# 08 - Similarity Searching with Vectors

> [!info] Capítulo 8 · *Unlocking Data with Generative AI and RAG* — Packt (ISBN 9781835887905) · Keith Bourne
> Este capítulo es **la R (Retrieval) de RAG**: cómo se encuentran, dentro del vector store, los vectores más parecidos a la query. Recorre cuatro áreas — **indexing, métricas de distancia, algoritmos de similitud y servicios de vector search** — sobre el [[Vector Space|espacio vectorial]], distinguiendo [[Semantic Search|búsqueda semántica]] de la [[Keyword Search|búsqueda por keyword]], midiendo similitud con [[Euclidean Distance|Euclidean]], [[Dot Product|dot product]] y [[Cosine Similarity|cosine]], combinando ambos mundos con [[Hybrid Search|búsqueda híbrida]] ([[BM25]] + [[Reciprocal Rank Fusion|RRF]]) y eligiendo entre [[k-NN]]/[[ANN]] e índices ([[LSH]], [[KD-tree]], [[Ball Tree]], [[Product Quantization|PQ]], [[HNSW]]). Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[07 - The Key Role Vectors and Vector Stores Play in RAG]] · siguiente [[09 - Evaluating RAG Quantitatively and with Visualizations]].

## Resumen

Tras el capítulo 7 (qué son los vectores y dónde viven), el capítulo 8 ataca **el otro lado del par**: cómo el sistema usa esos vectores para **recuperar** lo relevante — es decir, **la R de RAG**. El recorrido cubre cuatro áreas: **indexing, distance metrics, similarity algorithms y vector search services**. Primero ordena el vocabulario en una **jerarquía** (un *vector search* usa distintos *similarity algorithms*, que a su vez usan distintas *distance metrics*: no son sinónimos). Luego introduce el **[[Vector Space|espacio vectorial]]** (también *embedding space* o *latent space*), donde cada dimensión es una feature y los textos parecidos quedan **cerca**. Sobre esa base separa la **[[Semantic Search|búsqueda semántica]]** (que compara *significado* vía vectores) de la **[[Keyword Search|búsqueda por keyword]]** (que compara *palabras*), con el ejemplo de las dos reseñas de una manta vs un comentario sobre Taylor Swift.

El primer code lab (8.1) mide la similitud con las tres métricas más comunes en NLP — **[[Euclidean Distance|Euclidean (L2)]]**, **[[Dot Product|dot product]]** y **[[Cosine Similarity|cosine distance]]** — usando [[Sentence Transformers]] para embeddear las tres frases (384 dimensiones) y verificando numéricamente que las dos reseñas son cercanas y el comentario aleatorio está lejos. Después distingue los **paradigmas de búsqueda**: [[Dense Search|dense]] (semántica), [[Sparse Search|sparse]] (keyword, vía [[Bag of Words]] / [[BM25]]) e [[Hybrid Search|híbrida]] (la fusión de ambas). Dos code labs más construyen la búsqueda híbrida: 8.2 con una **función custom** que imita [[Reciprocal Rank Fusion|RRF]] sobre BM25 + dense retriever, y 8.3 con el **[[EnsembleRetriever]]** de [[LangChain]] (una sola línea). Finalmente expone los **algoritmos** ([[k-NN]] exacto vs [[ANN]] aproximado), las **técnicas de indexing** ([[LSH]], árboles [[KD-tree|KD]]/[[Ball Tree|Ball]], [[Product Quantization|PQ]], [[HNSW]]) y un panorama de **nueve servicios de vector search**. El capítulo siguiente pasa a **visualizar y evaluar** el pipeline RAG.

> [!note] El código del capítulo vive en `github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_08`, con un notebook por lab: `CHAPTER8-1_DISTANCEMETRICS.ipynb`, `CHAPTER8-2_HYBRID_CUSTOM.ipynb` y `CHAPTER8-3_HYBRID-ENSEMBLE.ipynb`.

## Métricas de distancia vs algoritmos de similitud vs vector search

Antes de cualquier código, el capítulo ordena tres términos que suelen confundirse. No son lo mismo: forman una **jerarquía**. Un **vector search** (la operación de alto nivel: encontrar los vectores más parecidos a la query) **usa** distintos **similarity algorithms** ([[k-NN]], [[ANN]]); y cada similarity algorithm **usa** distintas **distance metrics** ([[Cosine Similarity|cosine]], [[Euclidean Distance|Euclidean]], [[Dot Product|dot product]]). La Figura 8.1 lo dibuja con **dos opciones en cada nivel**, pero en la realidad hay muchas más en cada capa.

![[08-fig-8.1-hierarchy.jpg]]
*Figure 8.1 – Vector store, similarity algorithm, and distance metric hierarchy for two options each*

> [!note] La jerarquía: **vector search → similarity algorithm → distance metric**. No conflar los niveles: la distancia es la medida puntual entre dos vectores, el algoritmo es la estrategia para encontrar los vecinos, y el vector search es el servicio que orquesta todo sobre el store.

## Espacio vectorial

Un **[[Vector Space|espacio vectorial]]** es un constructo matemático: una colección de vectores en un espacio de **alta dimensionalidad**, donde cada **dimensión** representa una **feature**. La propiedad clave: textos con significado parecido quedan **más cerca** en ese espacio. También se lo llama **embedding space** o **latent space**.

La Figura 8.2 lo representa en **2D**: una **X** es la query, los **puntos pequeños** son los datos del dataset y los **cuatro puntos grandes** son los resultados recuperados (los más cercanos). El punto contra-intuitivo: algunos puntos pequeños *parecen* más cercanos en el dibujo 2D, pero esos embeddings originalmente tenían **1.536 dimensiones**; al agregar de vuelta las dimensiones faltantes, los puntos grandes resultan ser los matemáticamente más cercanos. Es como una "altura" que no se ve en 2D.

![[08-fig-8.2-vector-space-2d.jpg]]
*Figure 8.2 – 2D representation of embeddings in a vector space with the X representing a query and large dots representing the closest embeddings from the dataset*

> [!warning] No confíes en la intuición visual de un gráfico 2D: la cercanía real se calcula en el espacio completo (cientos o miles de dimensiones), y un punto que "se ve cerca" en la proyección puede estar lejos al sumar las dimensiones ocultas.

## Búsqueda semántica vs búsqueda por keyword

Los vectores capturan **significado**, así que la **[[Semantic Search|búsqueda semántica]]** (o **vector search**) encuentra documentos con **significado similar**, no solo con las mismas palabras. La **[[Keyword Search|búsqueda por keyword]]**, en cambio, matchea palabras específicas y **se pierde** la similitud semántica cuando dos textos dicen lo mismo con otras palabras.

El ejemplo de la manta lo ilustra. Dos reseñas que NO comparten palabras, pero significan lo mismo, vs un comentario aleatorio:

> "This blanket does such a great job maintaining a high cozy temperature for me!"
> "I am so much warmer and snug using this spread!"
> "Taylor Swift was 34 years old in 2024."

Semánticamente, las dos reseñas están **cerca** (hablan de abrigo/calidez), mientras que el comentario sobre Taylor Swift está **lejos**. Una búsqueda por keyword fallaría en emparejar las dos reseñas (no comparten términos); la semántica las une.

## Code lab 8.1 – Métricas de distancia semántica

El notebook `CHAPTER8-1_DISTANCEMETRICS.ipynb` instala [[Sentence Transformers]], carga un modelo y embeddea las tres frases:

```bash
%pip install sentence_transformers -q --user
```

```python
import numpy as np
from sentence_transformers import SentenceTransformer
```

```python
model = SentenceTransformer('paraphrase-MiniLM-L6-v2')
```

> [!tip] `'paraphrase-MiniLM-L6-v2'` es un modelo pequeño. Para producción, `'all-mpnet-base-v2'` puntúa **~50% más alto** en búsqueda semántica — probalo cuando la calidad importe.

```python
sentence = ['This blanket has such a cozy temperature for me!', 'I am so much warmer and snug using this spread!', 'Taylor Swift was 34 years old in 2024.']
```

```python
embedding = model.encode(sentence)
print(embedding)
embedding.shape
```

El `.shape` devuelve `(3, 384)`: **3 vectores de 384 dimensiones**.

> [!note] **Fun fact** — Podés usar `sentence_transformers` (sobre todo `all-mpnet-base-v2`) como **alternativa gratuita** a la API de embeddings de OpenAI para tu vector store. Según el ranking **[[MTEB]]**: `all-mpnet-base-v2` está **#94 de 303** en retrieval, el `ada` de OpenAI **#65**, y `text-embedding-3-large` **#14**. Además estos modelos son **fine-tuneables**, y al ser locales están **siempre disponibles / 100% confiables**.

Las tres métricas de distancia más comunes en NLP son **Euclidean (L2)**, **dot product** y **cosine distance**.

### Euclidean distance (L2)

La **[[Euclidean Distance|distancia Euclidean]]** es la distancia más corta entre dos puntos. **Menor valor = más similar** (más cerca). Usa resta elemento a elemento + la norma L2 de NumPy (`linalg.norm()` = raíz de la suma de cuadrados):

```python
def euclidean_distance(vec1, vec2):
    return np.linalg.norm(vec1 - vec2)
```

```python
print("Euclidean Distance: Review 1 vs Review 2:", euclidean_distance(embedding[0], embedding[1]))
print("Euclidean Distance: Review 1 vs Random Comment:", euclidean_distance(embedding[0], embedding[2]))
print("Euclidean Distance: Review 2 vs Random Comment:", euclidean_distance(embedding[1], embedding[2]))
```

Resultados: Review 1 vs Review 2 = **4.6202903**; Review 1 vs Random = **7.313547**; Review 2 vs Random = **6.3389034**. Las dos reseñas (menor valor) son las más similares.

### Dot product

El **[[Dot Product|dot product]]** (producto interno) técnicamente **no es** una métrica de distancia: mide la **magnitud de la proyección** de un vector sobre el otro (similitud, no distancia). **Mayor valor positivo = más similar**; menor o negativo = menos similar.

```python
print("Dot Product: Review 1 vs Review 2:", np.dot(embedding[0], embedding[1]))
print("Dot Product: Review 1 vs Random Comment:", np.dot(embedding[0], embedding[2]))
print("Dot Product: Review 2 vs Random Comment:", np.dot(embedding[1], embedding[2]))
```

Resultados: Review 1 vs Review 2 = **12.270497**; Review 1 vs Random = **-0.7654616**; Review 2 vs Random = **0.95240986**. El valor más alto (las dos reseñas) marca la mayor similitud.

### Cosine distance

La **[[Cosine Similarity|cosine distance]]** mide la diferencia **direccional** entre los vectores. **Menor valor = más similar**. Combina el dot product con las magnitudes (cosine similarity) y luego hace `1 - |cosine similarity|`:

```python
def cosine_distance(vec1, vec2):
    cosine = 1 - abs((np.dot(vec1, vec2) / (np.linalg.norm(vec1) * np.linalg.norm(vec2))))
    return cosine
```

```python
print("Cosine Distance: Review 1 vs Review 2:", cosine_distance(embedding[0], embedding[1]))
print("Cosine Distance: Review 1 vs Random Comment:", cosine_distance(embedding[0], embedding[2]))
print("Cosine Distance: Review 2 vs Random Comment:", cosine_distance(embedding[1], embedding[2]))
```

Review 1 vs Review 2 = **0.4523802399635315** (similitud moderada); las otras dos comparaciones dan alta disimilitud. Como remata el libro:

> "Sorry Taylor Swift… you are not the semantic equivalent of a warm blanket!"

### Tabla 8.1 — Las tres métricas en el ejemplo de la manta

| Métrica | Dirección | Review 1 vs Review 2 | Review 1 vs Random | Review 2 vs Random |
|---|---|---|---|---|
| **Euclidean (L2)** | menor = más similar | **4.6202903** | 7.313547 | 6.3389034 |
| **Dot product** | mayor = más similar | **12.270497** | -0.7654616 | 0.95240986 |
| **Cosine distance** | menor = más similar | **0.4523802** | (alta disimilitud) | (alta disimilitud) |

> [!tip] Otras métricas mencionadas (no las más comunes en NLP): **Lin similarity, Jaccard similarity, Hamming distance, Manhattan distance, Levenshtein distance**. En NLP, las tres de arriba son las dominantes.

## Paradigmas de búsqueda: dense, sparse, hybrid

### Dense search

La **[[Dense Search|dense search]]** (semántica) usa **vector embeddings** para devolver objetos semánticamente similares. Sus límites: rinde mal si el modelo fue entrenado en **otro dominio**, y rinde mal con **referencias** como números de serie, códigos, IDs o nombres (texto con poco "significado").

### Sparse search

La **[[Sparse Search|sparse search]]** (keyword) hace **keyword matching**. Un **sparse embedding** cuenta las ocurrencias de cada palabra del vocabulario → el vector queda **casi todo en ceros** (de ahí "sparse"). El enfoque **[[Bag of Words]]** cuenta la frecuencia de palabras en query y datos y devuelve el de mayor coincidencia. El algoritmo de keyword más popular es **[[BM25]]** (Best Matching 25), basado en **[[TF-IDF]]** (cap. 7): las palabras **raras** puntúan más alto y las frecuentes pesan menos.

> [!note] **Sparse embedding** = vector de conteos sobre el vocabulario, mayoritariamente ceros. **[[BM25]]** = el algoritmo de keyword estándar, una mejora estadística sobre [[TF-IDF]] que pondera por rareza del término.

### Hybrid search

La **[[Hybrid Search|hybrid search]]** combina **dense + sparse** y **fusiona** los resultados rankeados de ambos mediante un sistema de scoring. Es lo que construyen los labs 8.2 y 8.3.

## Code lab 8.2 – Hybrid search con una función custom

El notebook `CHAPTER8-2_HYBRID_CUSTOM.ipynb` parte del notebook **blue-team del cap. 5** (`CHAPTER5-3_BLUE_TEAM_DEFENDS.ipynb`), NO del cap. 6 ni del 7 (ver [[05 - Managing Security in RAG Applications]]). Introduce elementos **nuevos que se arrastran a los capítulos siguientes**: un **PDF document loader** (en vez de páginas web), un documento **más grande** y un **text splitter** distinto. El objetivo: usar **[[BM25]]** para los sparse vectors + los dense vectors existentes → híbrido, y rerankear con **[[Reciprocal Rank Fusion|Reciprocal Rank Fusion (RRF)]]** (el lab arma una función que imita RRF).

### Cambios: PDF + RecursiveCharacterTextSplitter

Se quita `%pip install beautifulsoup4` y se instala:

```bash
%pip install PyPDF2 -q –user
%pip install rank_bm25
```

Se quitan los imports `from langchain_community.document_loaders import WebBaseLoader`, `import bs4` y `from langchain_experimental.text_splitter import SemanticChunker`; se agregan:

```python
from PyPDF2 import PdfReader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.docstore.document import Document
from langchain_community.retrievers import BM25Retriever
```

Se elimina el bloque de `WebBaseLoader` y se agregan variables:

```python
pdf_path = "google-2023-environmental-report.pdf"
collection_name = "google_environmental_report"
str_output_parser = StrOutputParser()
```

Procesamiento del PDF:

```python
pdf_reader = PdfReader(pdf_path)
text = ""
for page in pdf_reader.pages:
    text += page.extract_text()
```

Se reemplaza el `SemanticChunker` por un **[[RecursiveCharacterTextSplitter]]**:

```python
character_splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", ". ", " ", ""],
    chunk_size=1000,
    chunk_overlap=200
)
splits = character_splitter.split_text(text)
```

Se preparan los documentos y los dos retrievers (ambos con `k=10`):

```python
documents = [Document(page_content=text, metadata={"id": str(i)}) for i, text in enumerate(splits)]
```

```python
chroma_client = chromadb.Client()
vectorstore = Chroma.from_documents(
    documents=documents,
    embedding=embedding_function,
    collection_name=collection_name,
    client=chroma_client
)
dense_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})
sparse_retriever = BM25Retriever.from_documents(documents, k=10)
```

> [!note] **Fun fact** — Los **sparse embeddings NO se guardan en Chroma**: el `BM25Retriever` los mantiene en memoria. Además, esta [[Chroma DB]] es **efímera** (`chromadb.Client()`): se pierde al apagar el kernel. Para algo más robusto, usá `vectorstore.persist()` y guardalo en un archivo `sqlite`.

### La función hybrid_search y RRF

La función `hybrid_search` es la **más grande del libro**. Toma los top-k de ambos retrievers, deduplica IDs, calcula el **reciprocal rank** y los combina con pesos:

```python
def hybrid_search(query, k=10, dense_weight=0.5, sparse_weight=0.5):
    dense_docs = dense_retriever.get_relevant_documents(query)[:k]
    dense_doc_ids = [doc.metadata['id'] for doc in dense_docs]
    print("\nCompare IDs:")
    print("dense IDs: ", dense_doc_ids)
    sparse_docs = sparse_retriever.get_relevant_documents(query)[:k]
    sparse_doc_ids = [doc.metadata['id'] for doc in sparse_docs]
    print("sparse IDs: ", sparse_doc_ids)
    all_doc_ids = list(set(dense_doc_ids + sparse_doc_ids))
    dense_reciprocal_ranks = {doc_id: 0.0 for doc_id in all_doc_ids}
    sparse_reciprocal_ranks = {doc_id: 0.0 for doc_id in all_doc_ids}
    for i, doc_id in enumerate(dense_doc_ids):
        dense_reciprocal_ranks[doc_id] = 1.0 / (i + 1)
    for i, doc_id in enumerate(sparse_doc_ids):
        sparse_reciprocal_ranks[doc_id] = 1.0 / (i + 1)
    combined_reciprocal_ranks = {doc_id: 0.0 for doc_id in all_doc_ids}
    for doc_id in all_doc_ids:
        combined_reciprocal_ranks[doc_id] = dense_weight * dense_reciprocal_ranks[doc_id] + sparse_weight * sparse_reciprocal_ranks[doc_id]
    sorted_doc_ids = sorted(all_doc_ids, key=lambda doc_id: combined_reciprocal_ranks[doc_id], reverse=True)
    sorted_docs = []
    all_docs = dense_docs + sparse_docs
    for doc_id in sorted_doc_ids:
        matching_docs = [doc for doc in all_docs if doc.metadata['id'] == doc_id]
        if matching_docs:
            doc = matching_docs[0]
            doc.metadata['score'] = combined_reciprocal_ranks[doc_id]
            doc.metadata['rank'] = sorted_doc_ids.index(doc_id) + 1
            if len(matching_docs) > 1:
                doc.metadata['retriever'] = 'both'
            elif doc in dense_docs:
                doc.metadata['retriever'] = 'dense'
            else:
                doc.metadata['retriever'] = 'sparse'
            sorted_docs.append(doc)
    return sorted_docs[:k]
```

La lógica de **[[Reciprocal Rank Fusion|RRF]]**: se toman los top-k de ambos retrievers; se deduplican los IDs con `set`; el reciprocal rank es **`1.0 / (posición)`** — fijate que usa las **posiciones de ranking, NO los scores** de similitud. Esto es lo elegante: como cosine y TF-IDF dan scores **incomparables**, RRF los esquiva usando solo el rank, sin necesidad de normalizar. Luego hace una **suma ponderada** vía `dense_weight`/`sparse_weight`, ordena descendente y etiqueta cada doc con `score`, `rank` y `retriever` (`'both'`/`'dense'`/`'sparse'`), devolviendo los top-k.

> [!warning] **Limitación conocida de RRF/híbrido**: con pesos iguales, un resultado semántico **muy cercano** y un top keyword **no muy parecido** reciben el **mismo valor de ranking** (ambos en posición 1 → mismo `1.0`). Se mitiga ajustando los pesos, pero es un límite del esquema → testealo para tu caso.

Se cablea en la cadena:

```python
rag_chain_with_source = RunnableParallel(
    {"context": hybrid_search, "question": RunnablePassthrough()}
).assign(answer=rag_chain_from_docs)
```

Y el código de salida:

```python
user_query = "What are Google's environmental initiatives?"
result = rag_chain_with_source.invoke(user_query)
relevance_score = result['answer']['relevance_score']
final_answer = result['answer']['final_answer']
retrieved_docs = result['context']
print(f"\nOriginal Question: {user_query}\n")
print(f"Relevance Score: {relevance_score}\n")
print(f"Final Answer:\n{final_answer}\n\n")
print("Retrieved Documents:")
for i, doc in enumerate(retrieved_docs, start=1):
    doc_id = doc.metadata['id']
    doc_score = doc.metadata.get('score', 'N/A')
    doc_rank = doc.metadata.get('rank', 'N/A')
    doc_retriever = doc.metadata.get('retriever', 'N/A')
    print(f"Document {i}: Document ID: {doc_id} Score: {doc_score} Rank: {doc_rank} Retriever: {doc_retriever}\n")
    print(f"Content:\n{doc.page_content}\n")
```

### Resultados

Los IDs recuperados por cada retriever (casi sin solapamiento):

```
dense IDs:  ['451', '12', '311', '344', '13', '115', '67', '346', '66', '262']
sparse IDs: ['150', '309', '298', '311', '328', '415', '139', '432', '91', '22']
```

`Relevance Score: 5` y una Final Answer sobre las iniciativas ambientales de Google. Los tres primeros documentos del ranking fusionado:

- **Document 1** = ID **150**, score **0.5**, rank 1, retriever **sparse**.
- **Document 2** = ID **451**, score **0.5**, rank 2, retriever **dense**.
- **Document 3** = ID **311**, score **0.29…**, rank 3, retriever **both**.

La distribución de los 10 retrievers en el resultado: **sparse, dense, both, sparse, dense, sparse, dense, dense, sparse, sparse**. Los scores de ranking: **0.5, 0.5, 0.29, 0.25, 0.25, 0.17, 0.125, 0.1, 0.1, 0.83**.

> [!note] Juzgar híbrido vs dense aquí es **subjetivo** — la evaluación **objetiva** llega en el cap. 9 ([[09 - Evaluating RAG Quantitatively and with Visualizations]]). El híbrido tuvo **cobertura más amplia**; en cambio, el dense capturó un detalle ("1 billion people / 1 gigaton by 2030") que el híbrido omitió.

## Code lab 8.3 – Hybrid search con el EnsembleRetriever de LangChain

El notebook `CHAPTER8-3_HYBRID-ENSEMBLE.ipynb` continúa desde 8.2 y reemplaza toda la función custom por **una línea** usando el **[[EnsembleRetriever]]** de [[LangChain]]:

```python
from langchain.retrievers import EnsembleRetriever
```

Se renombra `documents` separándolo en `dense_documents` + `sparse_documents`, etiquetando la fuente en la metadata:

```python
dense_documents = [Document(page_content=text, metadata={"id": str(i), "source": "dense"}) for i, text in enumerate(splits)]
sparse_documents = [Document(page_content=text, metadata={"id": str(i), "source": "sparse"}) for i, text in enumerate(splits)]
```

El retriever combinado:

```python
ensemble_retriever = EnsembleRetriever(retrievers=[dense_retriever, sparse_retriever], weights=[0.5, 0.5], c=0)
```

Toma ambos retrievers, los **pesos**, y un valor **`c`** = una constante que se suma al rank para balancear la importancia de los ítems mejor rankeados frente a los de más abajo. Su default es **60**; acá se pone en **0** (la función custom no tenía `c`, equivalente a 0). Se borra toda la función `hybrid_search` y se actualiza la cadena:

```python
rag_chain_with_source = RunnableParallel(
    {"context": ensemble_retriever, "question": RunnablePassthrough()}
).assign(answer=rag_chain_from_docs)
```

Código de salida nuevo:

```python
user_query = "What are Google's environmental initiatives?"
result = rag_chain_with_source.invoke(user_query)
relevance_score = result['answer']['relevance_score']
final_answer = result['answer']['final_answer']
retrieved_docs = result['context']
print(f"Original Question: {user_query}\n")
print(f"Relevance Score: {relevance_score}\n")
print(f"Final Answer:\n{final_answer}\n\n")
print("Retrieved Documents:")
for i, doc in enumerate(retrieved_docs, start=1):
    print(f"Document {i}: Document ID: {doc.metadata['id']} source: {doc.metadata['source']}")
    print(f"Content:\n{doc.page_content}\n")
```

`Relevance Score: 5` + una Final Answer (iMasons Climate Accord, ReFED, The Nature Conservancy, RE-Source Platform, eficiencia de los data centers…). Los tres primeros: **Document 1** = ID **344** source **dense**; **Document 2** = ID **150** source **sparse**; **Document 3** = ID **309** source **dense**.

> [!warning] El `EnsembleRetriever` da un resultado **≈ a la función custom**, pero **el dense gana los empates** (en la custom ganaba el sparse). Y su metadata `"source"` **no puede mostrar `'both'`** — una ventaja que sí tenía la función custom.

> [!tip] Si la búsqueda híbrida es central para tu caso, considerá un vector DB/servicio con **más flexibilidad de ranking**: por ejemplo, **[[Weaviate]]** te deja elegir entre **dos** algoritmos de ranking, mientras que LangChain solo ofrece el RRF integrado.

## Algoritmos de búsqueda semántica

### k-NN (k-nearest neighbors)

El **[[k-NN|k-NN]]** es **fuerza bruta**: calcula la distancia entre la query y **todos** los vectores del dataset, los ordena y devuelve los `k` más cercanos (p. ej. `k=5`). Es **exacto** y muy preciso, pero **se degrada** al crecer los datos: su **complejidad temporal es O(n · d)** (n = instancias, d = dimensionalidad) — **lineal** → inviable con millones/billones de vectores. Brilla con datasets **chicos** (más preciso que ANN). El autor lo usó para **25.000–30.000 embeddings a 256 dimensiones**, con una mejora de **2–6%** en las métricas de retrieval (cap. 9, [[09 - Evaluating RAG Quantitatively and with Visualizations]]). Puede usar cualquier métrica de distancia (la más común, Euclidean; también Manhattan/city-block o cosine).

> [!warning] El **O(n · d)** de k-NN es su talón de Aquiles: con datasets grandes y/o consultas en near-real-time, la búsqueda lineal no escala.

### ANN (Approximate Nearest Neighbors)

El **[[ANN|ANN]]** **sacrifica algo de precisión** a cambio de eficiencia: usa indexing, particionamiento y aproximación para **achicar el espacio de búsqueda**, logrando tiempos **sublineales** → escalable a datasets grandes. Se apoya en estructuras de indexing: **árboles jerárquicos** ([[KD-tree|KD-trees]], [[Ball Tree|Ball trees]]), **hashing** ([[LSH]] = Locality-Sensitive Hashing) y **grafos** ([[HNSW]] = Hierarchical Navigable Small World). k-NN es **exacto**; ANN es **aproximado** (puede perderse algún vecino verdadero). La elección depende de la app: exacto + chico → k-NN; masivo / near-real-time → ANN.

## Técnicas de indexing

El **indexing reduce la cantidad de vectores que hay que comparar**. Recordando el armado en capas: las **distance metrics** (cosine, Euclidean, dot product) las usan los **search algorithms** ([[k-NN]], [[ANN]]), que a su vez usan **técnicas de indexing** (LSH, KD-trees, Ball trees, PQ, HNSW). Librerías como **[[FAISS]]** y **[[pgvector]]** implementan varias.

### LSH (Locality-Sensitive Hashing)

**[[LSH]]** mapea vectores similares a los **mismos buckets de hash** con alta probabilidad; divide el espacio mediante funciones de hash; balancea precisión/eficiencia. Funciona como un **paso de preprocesamiento** para angostar el espacio de búsqueda.

### Árboles (KD-trees y Ball trees)

El **tree-based indexing** incluye dos variantes:

- **[[KD-tree|KD-trees]]** — árboles binarios de partición del espacio; dividen recursivamente el espacio k-dimensional y **podan** ramas irrelevantes.
- **[[Ball Tree|Ball trees]]** — particionan en **hiperesferas anidadas**; cada nodo es una hiperesfera; buenos para NN search en **alta dimensión**.

### PQ (Product Quantization)

**[[Product Quantization|PQ]]** combina compresión + indexing: **cuantiza** los vectores en **sub-vectores** representados con *code books* (el concepto de cuantización del cap. 7). Da **almacenamiento compacto** + cálculo aproximado eficiente de distancias; ideal para vectores de **alta dimensión** (recuperación de imágenes, recomendaciones).

### HNSW (Hierarchical Navigable Small World)

**[[HNSW]]** es **basado en grafos**: un grafo **multi-capa** jerárquico de nodos interconectados; escalable, resuelve el runtime de la fuerza bruta de k-NN. La parte **NSW** encuentra vectores bien posicionados como **puntos de partida**, y luego **traversa** hacia la query recalculando distancia, **salteando** grandes partes de los datos. La parte **H** (jerárquica) **apila capas** de small worlds navegables (analogía: avión → tren → búsqueda local).

> [!note] **Fun fact** — HNSW está inspirado en los **six degrees of separation** (dos personas cualesquiera están a ~6 conexiones de distancia), idea de un cuento de **Frigyes Karinthy** de **1929** (un juego para conectar a cualquier persona vía una cadena de otras cinco); también conocida como la regla de los **six handshakes**.

## Opciones de vector search

El **vector search** es la operación de encontrar, en el vector store, los vectores más similares a la query. El capítulo repasa **nueve** opciones:

| Servicio | Tipo | Algoritmo / indexing | Cuándo usarlo |
|---|---|---|---|
| **[[pgvector]]** | Extensión de Postgres | L2 + cosine; **exacto k-NN Y aproximado ANN**; índices **IVF** (Inverted File Index) + **HNSW** (IVF combinable con HNSW) | Si ya estás sobre Postgres |
| **[[Elasticsearch]]** | Motor de búsqueda/analytics open source | ANN vía **HNSW**; full-text + agregaciones + geospatial; distribuido/escalable; integración con [[LangChain]] | Búsqueda potente full-text + vector, asumiendo más setup |
| **[[FAISS]]** (Facebook AI Similarity Search) | Librería de Facebook | ANN vía **IVF/PQ/HNSW**; soporte **GPU**; billion-scale | Escala enorme, control bajo nivel (más setup manual) |
| **Google Vertex AI Vector Search** | Managed en GCP | ANN, probablemente vía **ScaNN** de Google; updates online; integra con BigQuery/Dataflow | Ecosistema GCP, totalmente managed (puede ser más caro) |
| **Azure AI Search** | Managed por Microsoft Azure | ANN; vector + keyword; sinónimos/autocomplete/faceted nav; integra con Azure ML | Ecosistema Azure (curva de aprendizaje más empinada) |
| **[[ANNOY]]** (Approximate Nearest Neighbors Oh Yeah) | Open source de Spotify | Random projections + binary space partitioning → un **bosque de árboles**; rápido; API simple | Simplicidad y velocidad; puede no escalar a datasets enormes |
| **[[Pinecone]]** | Vector DB fully managed para ML | Dense + sparse; ANN incl. **HNSW**; updates en tiempo real, escalado horizontal, replicación multi-región | Producción managed sin operar infra (más caro) |
| **[[Weaviate]]** | Motor de búsqueda open source | **HNSW**; API GraphQL; schema management, validación de datos, updates en tiempo real | Self-host o managed; **flexibilidad de ranking** (2 algoritmos) |
| **[[Chroma DB|Chroma]]** | Vector DB embebida open source (la que usa el libro) | ANN incl. **HNSW**; API Python simple; filtrado dinámico + metadata; in-memory o persistir a disco | Prototipos / labs; menos escalable que soluciones maduras |

> [!tip] No hay "el mejor": pgvector si ya tenés Postgres; FAISS para escala extrema con control bajo nivel; Pinecone/Weaviate managed para producción; Chroma para empezar y prototipar; los servicios cloud (Vertex AI, Azure AI Search) si vivís en ese ecosistema.

## Citas

> "This blanket does such a great job maintaining a high cozy temperature for me!"

> "I am so much warmer and snug using this spread!"

> "Taylor Swift was 34 years old in 2024."

> "Sorry Taylor Swift… you are not the semantic equivalent of a warm blanket!"

## Para aplicar

- **Probá `all-mpnet-base-v2`** — como alternativa gratuita y local a la API de embeddings de OpenAI (mejor que `paraphrase-MiniLM-L6-v2`); consultá el ranking [[MTEB]] para elegir.
- **Persistí Chroma** — la `chromadb.Client()` por defecto es efímera; usá `vectorstore.persist()` a un `sqlite` para no perder el store al apagar el kernel.
- **Tuneá los pesos dense/sparse** — empezá en 0.5/0.5 y ajustá según tus resultados; recordá que con pesos iguales RRF empata un top semántico fuerte con un top keyword flojo.
- **Elegí el algoritmo según escala** — k-NN (exacto) para datasets chicos (el autor usó 25.000–30.000 vectores a 256 dims con +2–6%); ANN (HNSW/LSH/PQ/árboles) cuando hay millones de vectores o latencia near-real-time.
- **Para híbrido serio, mirá Weaviate** — ofrece dos algoritmos de ranking; LangChain solo trae el RRF integrado y su `EnsembleRetriever` no distingue la fuente `'both'`.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[07 - The Key Role Vectors and Vector Stores Play in RAG]] — capítulo anterior: qué son los vectores y dónde viven; el cap. 8 los usa para **recuperar** (la R de RAG) y profundiza la dicotomía [[Sparse Vectors|sparse]]/[[Dense Vectors|dense]] y la [[Cosine Similarity]].
- [[09 - Evaluating RAG Quantitatively and with Visualizations]] — capítulo siguiente: **visualizar y evaluar** el pipeline RAG (de donde sale el +2–6% de k-NN y la evaluación objetiva de híbrido vs dense).
- [[05 - Managing Security in RAG Applications]] — el notebook blue-team (`CHAPTER5-3_BLUE_TEAM_DEFENDS.ipynb`) sobre el que se construyen los labs 8.2 y 8.3.
- [[06 - Interfacing with RAG and Gradio]] — el código del cap. 6, que los labs de este capítulo **NO** reutilizan (parten del cap. 5).
- [[11 - Using LangChain to Get More from RAG]] — los document loaders / text splitters de [[LangChain]] (PDF loader, `RecursiveCharacterTextSplitter`) se tratan en detalle.
- [[Semantic Search]] · [[Keyword Search]] · [[Vector Space]] · [[Euclidean Distance]] · [[Dot Product]] · [[Cosine Similarity]] — fundamentos de la similitud.
- [[Dense Search]] · [[Sparse Search]] · [[Hybrid Search]] · [[BM25]] · [[TF-IDF]] · [[Bag of Words]] · [[Reciprocal Rank Fusion]] · [[EnsembleRetriever]] — paradigmas de búsqueda y fusión.
- [[k-NN]] · [[ANN]] · [[LSH]] · [[KD-tree]] · [[Ball Tree]] · [[Product Quantization]] · [[HNSW]] — algoritmos de búsqueda e indexing.
- [[FAISS]] · [[pgvector]] · [[Pinecone]] · [[Weaviate]] · [[Chroma DB]] · [[Elasticsearch]] · [[ANNOY]] — servicios de vector search.
- [[Sentence Transformers]] · [[RecursiveCharacterTextSplitter]] · [[LangChain]] · [[MTEB]] — herramientas del code lab.
