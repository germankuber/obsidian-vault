---
title: 01 - Why Retrieval-Augmented Generation
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 1
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Why Retrieval-Augmented Generation?
  - Why Retrieval-Augmented Generation
  - Why RAG
updated: 2026-06-11
---

# 01 - Why Retrieval-Augmented Generation

> [!info] Capítulo 1 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> RAG le da al modelo generativo el conocimiento que le falta. El capítulo define el framework, sus tres configuraciones (naïve, advanced, modular), RAG vs fine-tuning, el ecosistema D/G/E/T, y construye los tres en Python sobre el nuevo paradigma **MAS-RAG**: dejar de llevar los datos a la IA y empezar a llevar la IA a los datos. Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · siguiente [[02 - RAG Embeddings in Oracle Vector Stores]].

## Resumen

Hasta los modelos generativos mejor entrenados **no pueden responder sobre información a la que no tienen acceso**, y muchas veces *no saben que no saben* — lo que produce outputs inexactos, a veces llamados alucinaciones, sesgo o salida incoherente. **RAG (Retrieval-Augmented Generation)** es el framework que le provee al modelo generativo el conocimiento que le falta: realiza tareas de recuperación de información optimizada, y el ecosistema de generación agrega esa información al input (query del usuario o prompt automatizado) para producir un output mejorado. RAG fue diseñado para LLMs (Lewis et al., 2020) y **empieza donde la IA generativa termina**.

La integración de RAG con modelos generativos evolucionó en **dos generaciones**. En la *primera generación (RAG tradicional)*, los project managers enfrentaban un landscape fragmentado: navegar plataformas de IA generativa junto a una capa adicional de vector stores externos. Ese enfoque solía requerir **sacar datos corporativos sensibles** de entornos seguros hacia sistemas externos, creando complejidad, latencia y riesgos de seguridad innecesarios. Ahora entramos en la era de **MAS-RAG (multi-agent systems for retrieval-augmented generation)**, un cambio conceptual revolucionario: **pasar de llevar los datos a la IA, a llevar la IA a los datos**. La transformación la encabezan plataformas como **Oracle**, que mediante *converged databases* donde la vector search es una capacidad nativa garantizan que los datos sensibles nunca dejan el límite de gobernanza seguro, resolviendo el desafío de **soberanía de datos (data sovereignty)** sobre infraestructura ya desplegada globalmente.

El capítulo recorre: la definición de RAG, las tres configuraciones (naïve, advanced, modular), la decisión estratégica RAG vs fine-tuning (basada en distinguir conocimiento **parametric** vs **non-parametric**), el ecosistema RAG en cuatro dominios (**Retriever D, Generator G, Evaluator E, Trainer T**), y una parte práctica donde se construye desde cero un programa Python que implementa los tres tipos de RAG simulando este entorno seguro y convergido. Cierra mostrando cómo el `RetrievalComponent` modular es el *stepping stone* hacia sistemas multi-agente (MAS), donde el componente de retrieval pasa a ser una **tool empuñada por un agente autónomo** que planifica, razona y actúa — *cognitive orchestration*, ya no mera recuperación.

![[01-fig-1.1.png|887]]
*Figure 1.1: The paradigm shift: bringing intelligence to data*

## ¿Qué es RAG? La analogía del estudiante

Cuando un modelo generativo no sabe responder con precisión, *alucina* o produce sesgo — en el fondo **todo se reduce a la imposibilidad de dar una respuesta adecuada cuando el entrenamiento del modelo no incluyó la información pedida**. Esa confusión deriva en secuencias aleatorias de los outputs más *probables*, no los más *precisos*.

La analogía del capítulo: pensate como un estudiante escribiendo un ensayo sobre un tema complejo. **Vos sos el modelo generativo (LLM)**: estás bien entrenado en leer y escribir, pero no conocés la información secreta y específica que el ensayo requiere.

- **La vieja forma (RAG tradicional)** — tenés que *conmutar*: dejar tu cuarto seguro y viajar a una biblioteca pública a buscar los libros, los fotocopiás y los traés a tu escritorio.
  - **El riesgo**: cada vez que salís del cuarto, arriesgás perder tus notas.
  - **El costo**: el tiempo de viaje (latencia) te frena.
  - **La complejidad**: necesitás un mapa (integraciones) para encontrar la biblioteca.
- **La nueva forma (arquitectura Oracle MAS-RAG)** — nunca salís del cuarto. Te sentás en tu escritorio dentro de un *smart room* seguro (la **Oracle Converged Database**); cuando tenés una pregunta, las páginas relevantes de los libros se materializan instantáneamente sobre tu escritorio.
  - **La seguridad**: nunca cruzás el umbral; los datos nunca dejan el cuarto.
  - **La velocidad**: la información ya está con vos.
  - **La simplicidad**: no hay viaje, no hay mapa, no hay riesgo de fuga de datos.

## Naïve, advanced y modular RAG (las tres configuraciones)

Un framework RAG se apoya en dos componentes primarios: un **Retriever** y un **Generator**. El Generator puede ser cualquier LLM o plataforma multimodal foundation (GPT-5.2, Gemini, Llama, o sus innumerables variantes). El landscape de retrievers, en cambio, sufrió una **consolidación significativa**: el mercado explotó inicialmente con tools standalone (Pinecone, Chroma, LlamaIndex), pero el foco se desplazó hacia **plataformas convergidas**. La decisión crítica del project manager ya no es elegir una tool, sino elegir el **patrón arquitectónico correcto** entre las tres configuraciones (Gao et al., 2024).

- **Naïve RAG** — **no** involucra embedding ni vectorización compleja. Se apoya en métodos de recuperación tradicionales: **keyword search o queries SQL estándar**. En entornos corporativos suele ser **altamente eficiente para recuperar registros estructurados específicos** (una factura concreta, un patient ID) donde se requiere *exact matching* y no similitud semántica.
- **Advanced RAG** — introduce el poder de **vector search e index-based retrieval**. En el contexto moderno implica **integrated vectorization**: en vez de mandar datos a un índice externo, usa las capacidades de vector nativas de la base (como Oracle AI Vector Search) para hallar información semánticamente similar — encontrar reportes de *"network outage"* aunque el usuario busque *"connection failure"*. Procesa múltiples tipos de datos (estructurados y no estructurados) sin fricción.
- **Modular RAG** — el nivel más alto de sofisticación y el **stepping stone hacia Multi-Agent Systems (MAS)**. Abre el horizonte a escenarios que requieren **orquestación dinámica**: puede conmutar inteligentemente entre naïve RAG (para hechos precisos) y advanced RAG (para comprensión conceptual), o incluso disparar algoritmos de machine learning externos. Es la **capa de toma de decisión** que asegura que se use la tool correcta para la query correcta.

## RAG versus fine-tuning

RAG no siempre es alternativa al fine-tuning, ni el fine-tuning siempre reemplaza a RAG: **la decisión es estratégica**. Si metemos demasiado conocimiento estático en los contextos de RAG, el sistema se vuelve pesado y caro. A la inversa, **no podemos fine-tunear un modelo sobre datos dinámicos y siempre cambiantes** (pronóstico del clima diario, valores de bolsa en tiempo real, noticias corporativas confidenciales). La decisión se basa en distinguir dos tipos de conocimiento:

| Tipo de conocimiento | Qué es | Dónde vive | Ejemplos |
|---|---|---|---|
| **Parametric (el "cerebro" del modelo)** | Conocimiento **implícito**, encapsulado en los **weights y biases** del modelo. Una vez entrenado, los datos originales se pierden, reemplazados por representaciones matemáticas. | En los pesos del modelo (fine-tuning). | Enseñarle al modelo un *lenguaje* específico (jerga legal, sintaxis médica) o un estilo de output. |
| **Non-parametric (la base de datos Oracle)** | Almacena datos **explícitos y recuperables**; preserva la estructura y relaciones originales (a diferencia de los pesos). | En la **Oracle Converged Database** (RAG). | Acceder a hechos específicos (*"¿cuál es el balance de la Invoice #204?"*), updates en tiempo real, **estricta data sovereignty**. |

La diferencia fundamental entre un modelo entrenado/fine-tuneado y un sistema RAG se resume como la diferencia entre **aprender (learning, parametric)** y **referenciar (referencing, non-parametric)**. La elección depende de la cantidad de datos estáticos (parametric) vs dinámicos (non-parametric) que el sistema deba procesar: un sistema que depende demasiado del fine-tuning **no se adapta a los updates diarios**; uno que depende enteramente de RAG hasta para el entendimiento básico del dominio **se vuelve ineficiente**. No son mutuamente excluyentes: **RAG provee los hechos, el fine-tuning enseña al modelo cómo interpretarlos**. (El fine-tuning se explora específicamente en [[07 - Empowering AI Models by Fine-Tuning RAG Data|el cap. 7]].)

![[01-fig-1.2.png]]
*Figure 1.2: The strategic threshold of sovereign RAG versus fine-tuning*

## El ecosistema RAG (los cuatro dominios D · G · E · T)

RAG corre dentro de un ecosistema amplio. Sin importar cuántos frameworks de retrieval y generación se encuentren, todo se reduce a **cuatro dominios** y las preguntas críticas que los acompañan:

- **Data** — ¿De dónde vienen los datos? ¿Son confiables? ¿Son suficientes? Crucialmente, en la era MAS-RAG: ¿los datos **se quedan dentro del trust boundary corporativo seguro**?
- **Storage** — ¿Cómo se almacenan? En el enfoque tradicional, los datos estaban fragmentados entre bases SQL y vector stores externos. En el moderno: ¿podemos **almacenar vectores junto a los datos de negocio en una única converged database**?
- **Retrieval** — ¿Cómo se recupera el dato correcto? ¿Keyword matching simple (naïve RAG) o integrated vector search (advanced RAG)?
- **Generation** — ¿Cómo se selecciona el modelo generativo adecuado? ¿Cómo se canaliza de forma segura el dato privado recuperado hacia el modelo?

Los componentes del ecosistema se etiquetan con dominios **D, G, E, T** y subcomponentes numerados (D1, G3…) que mapean directo a la *Figure 1.3*:

![[01-fig-1.3.png]]
*Figure 1.3: The converged ecosystem*

### Retriever (D) — recolecta, procesa, almacena y recupera

En el paradigma Oracle MAS-RAG, todo este pipeline ocurre a menudo **sin que los datos dejen nunca el entorno seguro de la base**.

- **(D1) Data collection** — los datos de IA son diversos: un chunk de texto de un blog, un PDF técnico, un JSON ordenado, o multimedia. Buena parte es **no estructurada**. Plataformas convergidas como **Oracle Database 23ai** proveen tools ready-to-use para ingerir esa "jungla" de datos directamente, **eliminando la necesidad de pipelines ETL complejos hacia sistemas externos**.
- **(D2) Data processing** — los distintos tipos de datos se transforman en representaciones de features uniformes. Suele implicar **embedding**: convertir texto o imágenes en listas de números (vectores) que representan su significado.
- **(D4) Retrieval** — disparado por el user input (G1). Para recuperar rápido, el sistema busca directamente **dentro de la base**, combinando **SQL estándar (precisión)** con **Vector Search (significado)** — ej. encontrar "contracts related to Project X" (SQL) que "discuss liability risks" (Vector Search). Como datos y vectores conviven, la recuperación es instantánea y segura.

### Generator (G) — augmenta el input y genera

En el ecosistema RAG las líneas entre input y retrieval se difuminan: el user input (G1) interactúa con la retrieval query (D4) para augmentar el prompt antes de mandarlo al modelo.

- **(G1) Input** — automatizado vía agentes o ingresado por humanos vía UI (ej. una app Oracle APEX).
- **(G2) Augmentation** — el **mecanismo core de RAG**: toma la pregunta del usuario y le **agrega los datos seguros recuperados** de la base.
- **(G3) Prompt engineering** — junta el output del Retriever y el user input en una instrucción coherente que el LLM entiende.
- **(G4) Generation** — el prompt augmentado se manda al modelo seleccionado (GPT-5.2, Gemini, Llama 3), que genera una respuesta basada en los hechos seguros provistos.

### Evaluator (E) — métricas + humano

- **(E1)** Métricas matemáticas (como **cosine similarity**) que solo dan parte del cuadro.
- **(E2)** **Human feedback** — sigue siendo imprescindible; ningún sistema generativo elude la evaluación humana, que decide en última instancia si se puede confiar en el sistema. (Feedback loops adaptativos se implementan en [[08 - Boosting RAG Performance with Human Feedback|el cap. 8]].)

### Trainer (T) — fine-tuning

- **(T2) Fine-tuning** — opción de fine-tunear el modelo para entender lenguaje domain-specific, o fine-tunear las estrategias de retrieval. (Se explora en [[07 - Empowering AI Models by Fine-Tuning RAG Data|el cap. 7]].)

## Implementación en Python: naïve, advanced y modular RAG

El notebook (`RAG_Overview_db.ipynb` en el repo de GitHub) construye keyword matching, vector search e index-based retrieval **desde cero**, usando GPT-5.2 para generar respuestas a partir de queries y documentos recuperados. **Simula un entorno Oracle MAS-RAG**: en vez de conectar a una base enterprise viva, crea un *trust boundary* simulado usando **listas de Python**. Las siete secciones del programa:

1. **Environment** — setup de la integración con la API de OpenAI usando Google Colab secrets.
2. **Generator** — función usando GPT-5.2.
3. **The Data** — una lista de documentos (`db_records`) que simula el entorno Oracle Database 23ai.
4. **The Query** — un user input estratégico sobre seguridad de datos.
5. **Naïve RAG** — keyword search y matching.
6. **Advanced RAG** — vector search e index-based search.
7. **Modular RAG** — métodos de retrieval flexibles y dinámicos.

### El entorno (Environment)

Hay que **congelar (freeze) la versión de OpenAI** que se instala, porque en ecosistemas RAG los updates de librerías a veces introducen conflictos — congelarla asegura reproducibilidad:

```python
!pip install openai==2.14.0
```

A diferencia de la primera edición (que leía keys de archivos de texto en Google Drive), ahora se usan **Google Colab Secrets** para manejar credenciales sin exponerlas en el código. Primero se agrega `API_KEY` al Secrets manager (ícono de llave en el sidebar de Colab); luego:

```python
# Cell 2: Imports and API Key Setup
import os
from openai import OpenAI
from google.colab import userdata

# Load the API key from Colab secrets
try:
    api_key = userdata.get("API_KEY")
    if not api_key:
        raise userdata.SecretNotFoundError("API_KEY not found.")

    # Set environment variable
    os.environ["OPENAI_API_KEY"] = api_key
    client = OpenAI()
    print("OpenAI API key loaded successfully.")

except userdata.SecretNotFoundError:
    print('Secret "API_KEY" not found. Please add it to Secrets Manager.')
```

### El Generator

Función sobre la API de OpenAI (puede usarse cualquier LLM). Se upgradeó a **GPT-5.2** para máxima calidad de razonamiento. Nota: la **`temperature` se fija en `0.1`** — en un sistema RAG que maneja datos corporativos y políticas de seguridad, queremos al modelo preciso y factual, minimizando alucinaciones creativas.

```python
import openai
from openai import OpenAI

client = OpenAI()
gptmodel = "gpt-5.2"

def call_llm_with_full_text(itext):
    # Join all lines to form a single string
    text_input = '\n'.join(itext)
    prompt = f"Please elaborate on the following content:\n{text_input}"

    try:
        response = client.chat.completions.create(
            model=gptmodel,
            messages=[
                {"role": "system", "content": "You are an expert Natural Language Processing exercise expert."},
                {"role": "assistant", "content": "1.You can explain read the input and answer in detail"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.1  # Low temperature for precise, factual answers
        )
        return response.choices[0].message.content.strip()
    except Exception as e:
        return str(e)
```

Y una función de formateo para output legible (wrap a 80 columnas):

```python
import textwrap
def print_formatted_response(response):
    wrapper = textwrap.TextWrapper(width=80)
    wrapped_text = wrapper.fill(text=response)
    print("Response:")
    print("---------------")
    print(wrapped_text)
    print("---------------\n")
```

### Los datos (The Data) y la query

En producción los datos vivirían en **Oracle Database 23ai** indexados con **AI Vector Search**; acá se *simula* ese entorno seguro con una lista Python `db_records` que establece la *ground truth* que el sistema debe recuperar. **El modelo solo sabe lo que está en esta base**: si cambian los registros, cambian las respuestas — eso demuestra cómo RAG ancla la IA en la realidad corporativa específica.

```python
db_records = [
    "Retrieval Augmented Generation (RAG) is a hybrid AI approach that combines neural language models with secure retrieval systems.",
    "The Oracle strategy for RAG represents a paradigm shift: instead of moving data to external vector stores, it brings AI vector search directly to the database.",
    "This methodology ensures that sensitive corporate data never leaves the secure governance boundary of the Oracle environment.",
    "At the core of this architecture is the Oracle Database 23ai, which integrates vector search natively alongside relational data.",
    # ... additional records ...
    "In summary, Oracle-based RAG represents the maturity of AI: merging the best of generative technologies with the security and reliability of enterprise data systems."
]
```

La query es **el junction entre Retriever y Generator** — diseñada para requerir conocimiento arquitectónico específico (no un trivial *"What is RAG?"*), disparando la recuperación de los conceptos *"Data Sovereignty"* y *"Paradigm Shifts"* plantados en `db_records`:

```python
# A query designed to trigger the retrieval of "Integrated Vectorization" and "Data Sovereignty" concepts
query = "How does the Oracle RAG approach differ from traditional vector stores regarding data security?"
```

### Naïve RAG — keyword search y matching

Eficiente con documentos bien definidos (facturas con IDs, documentos legales con títulos claros). Lógica: dividir query y cada record en sets de keywords, calcular la **intersección** (palabras comunes), elegir el record con mayor overlap. Luego: augmentar el input con el record recuperado, pedirle a GPT-5.2 que responda con ese contexto, y mostrar la respuesta.

```python
def find_best_match_keyword_search(query, db_records):
    best_score = 0
    best_record = None
    # Split the query into individual keywords
    query_keywords = set(query.lower().split())

    # Iterate through each record in db_records
    for record in db_records:
        # Split the record into keywords
        record_keywords = set(record.lower().split())
        # Calculate the number of common keywords
        common_keywords = query_keywords.intersection(record_keywords)
        current_score = len(common_keywords)
        # Update the best score and record if the current score is higher
        if current_score > best_score:
            best_score = current_score
            best_record = record

    return best_score, best_record
```

Resultado: el keyword search encuentra el record con más términos solapados ("Oracle", "RAG", "vector", "stores") con **Best Keyword Score: 5**, recuperando el record del *paradigm shift*:

```text
Best Keyword Score: 5
Response:
---------------
The Oracle strategy for RAG represents a paradigm shift: instead of moving data to
external vector stores, it brings AI vector search directly to the database.
---------------
```

**Augmented input** — concatenación de query + best matching record:

```python
augmented_input = query + " " + best_matching_record
print_formatted_response(augmented_input)
```

**Generation** — se llama a GPT-5.2 con el contexto augmentado; como el dato menciona explícitamente *"AI vector search directly to the database"*, el modelo explica con precisión las implicancias de seguridad:

```python
llm_response = call_llm_with_full_text(augmented_input)
print_formatted_response(llm_response)
```

> [!warning] Limitación de naïve RAG
> Funcionó porque la query usó **las palabras exactas** del record ("Oracle", "vector stores"). Si el usuario hubiera preguntado por *"keeping information safe"* en vez de *"data security"*, el keyword search podría **no encontrar** el record relevante si esas palabras exactas no estaban presentes. Con datasets más grandes y queries con sinónimos o descripciones conceptuales, hace falta un método más robusto → **advanced RAG**.

### Advanced RAG — vector search e index-based search

Al crecer los datasets, el keyword search falla: lucha con **sinónimos** (buscar "security" pero perder "governance") y contexto, y un scan lineal sobre millones de documentos es **computacionalmente prohibitivo**. La solución es **integrated vectorization**: transformar texto en representaciones numéricas (vectores) que capturan significado (Oracle Database 23ai lo hace nativo a escala). Dos sub-técnicas:

- **Vector search** — convierte `db_records` en vectores y calcula **cosine similarity** entre el vector de la query y los de los documentos para hallar lo más relevante por **significado semántico**, no por coincidencia exacta de palabras.
- **Index-based search** — en producción Oracle los vectores se guardan en un **Vector Index especializado (HNSW o IVF)**:
  - **HNSW (Hierarchical Navigable Small World)** — índice **basado en grafos** que halla los vectores más cercanos rápido, navegando redes layered small-world.
  - **IVF (Inverted File Index)** — **agrupa vectores en clusters** y solo busca en los más cercanos, haciendo la recuperación a gran escala rápida y eficiente.
  - En el notebook se simula **precomputando una matriz TF-IDF**, permitiendo recuperación veloz sin reprocesar el texto en cada query.

**Vector search** — `TfidfVectorizer` simula el proceso de embedding (en Oracle real se usaría un embedding model *dentro* de la base); **TF-IDF (term frequency-inverse document frequency)** crea representaciones numéricas ponderadas. `cosine_similarity` mide cuán cerca están dos vectores: **1.0 = match exacto; 0.0 = sin relación**.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

def calculate_cosine_similarity(text1, text2):
    vectorizer = TfidfVectorizer()
    tfidf = vectorizer.fit_transform([text1, text2])
    similarity = cosine_similarity(tfidf[0:1], tfidf[1:2])
    return similarity[0][0]

def find_best_match(text_input, records):
    best_score = 0
    best_record = None
    for record in records:
        current_score = calculate_cosine_similarity(text_input, record)
        if current_score > best_score:
            best_score = current_score
            best_record = record
    return best_score, best_record
```

Resultado: a diferencia del naïve (que cuenta palabras), el vector search calcula un peso de similitud específico → **Best Cosine Similarity Score: 0.243**, recuperando el mismo record del *paradigm shift*. El sistema identificó el principio arquitectónico correcto **aunque la métrica baja no lo refleje** — lo que muestra que **las métricas requieren supervisión humana**.

**Augmented input y generation** — la respuesta de GPT-5.2 es satisfactoria, explicando que con *Integrated Vectorization* los vectores residen en la misma base que los datos relacionales, por lo que **políticas de seguridad existentes (Row-Level Security) se aplican automáticamente** y el dato nunca cruza el governance boundary.

**Index-based search** — en el ejemplo anterior se calculaba el vector de cada documento en cada query (ineficiente). En la realidad se crea un **índice** precomputando la matriz:

```python
def setup_vectorizer(records):
    vectorizer = TfidfVectorizer()
    tfidf_matrix = vectorizer.fit_transform(records)
    return vectorizer, tfidf_matrix

def find_best_match_indexed(query, vectorizer, tfidf_matrix):
    query_tfidf = vectorizer.transform([query])
    similarities = cosine_similarity(query_tfidf, tfidf_matrix)
    best_index = similarities.argmax()  # Get the index of the highest score
    best_score = similarities[0, best_index]
    return best_score, best_index
```

```python
# Create the index once
vectorizer, tfidf_matrix = setup_vectorizer(db_records)

# Search the index
best_similarity_score, best_index = find_best_match_indexed(query, vectorizer, tfidf_matrix)
best_matching_record = db_records[best_index]
```

`tfidf_matrix` es el **vector index**: un mapa numérico pre-calculado de toda la base. El record recuperado es **idéntico** al vector search estándar, pero la recuperación es una simple operación matricial, **computacionalmente mucho más escalable**. (El score difiere ligeramente porque el índice calcula los pesos de palabras sobre **toda la base** en vez de un solo par.) Por eso Oracle Database 23ai usa vector indexes como HNSW para manejar **billones de vectores con latencia de milisegundos**.

### Modular RAG — la clase `RetrievalComponent`

¿Keyword, vector o index-based search? En un sistema rígido elegís una en desarrollo. En **modular RAG la arquitectura elige la mejor tool para cada query específica** — fundamento de **Oracle AI Vector Search**, que soporta **hybrid search** (SQL para filtrado exacto + vectores para entendimiento semántico, simultáneamente). Se construye una clase `RetrievalComponent` que actúa como **agente primitivo**: encapsula los datos, gestiona el estado (índices) y ejecuta la estrategia apropiada según la config. Es **self-contained** (guarda su copia de los datos en `self.documents` y sus modelos en `self.vectorizer`), operando independiente de variables globales — patrón crítico para sistemas MAS-RAG escalables.

```python
class RetrievalComponent:
    def __init__(self, method='vector'):
        self.method = method
        # Initialize vectorizer only if needed for advanced methods
        if self.method == 'vector' or self.method == 'indexed':
            self.vectorizer = TfidfVectorizer()
            self.tfidf_matrix = None
        self.documents = []

    def fit(self, records):
        self.documents = records  # Encapsulate data within the component
        if self.method == 'vector' or self.method == 'indexed':
            self.tfidf_matrix = self.vectorizer.fit_transform(records)

    def retrieve(self, query):
        if self.method == 'keyword':
            return self.keyword_search(query)
        elif self.method == 'vector':
            return self.vector_search(query)
        elif self.method == 'indexed':
            return self.indexed_search(query)

    def keyword_search(self, query):
        best_score = 0
        best_record = None
        query_keywords = set(query.lower().split())
        for doc in self.documents:
            doc_keywords = set(doc.lower().split())
            common_keywords = query_keywords.intersection(doc_keywords)
            score = len(common_keywords)
            if score > best_score:
                best_score = score
                best_record = doc
        return best_record

    def vector_search(self, query):
        query_tfidf = self.vectorizer.transform([query])
        similarities = cosine_similarity(query_tfidf, self.tfidf_matrix)
        best_index = similarities.argmax()
        return self.documents[best_index]  # Return from encapsulated data

    def indexed_search(self, query):
        # Uses the pre-computed matrix for speed
        query_tfidf = self.vectorizer.transform([query])
        similarities = cosine_similarity(query_tfidf, self.tfidf_matrix)
        best_index = similarities.argmax()
        return self.documents[best_index]
```

El método `retrieve` actúa como **dispatcher**: según `self.method` rutea la query al algoritmo correcto (`keyword`, `vector` o `indexed`). `fit` carga la knowledge base y pre-computa la matriz TF-IDF si hace falta (simula indexar en Oracle). Esto permite que **un solo objeto despache dinámicamente** queries entre los tres algoritmos.

**Modular RAG en acción** — se instancia con estrategia `vector` para capturar el significado semántico de *data security*:

```python
# Initialize the component with 'vector' strategy
retrieval = RetrievalComponent(method='vector')
# 'Train' the component with our simulated Oracle data
retrieval.fit(db_records)
# Execute retrieval
best_matching_record = retrieval.retrieve(query)
print_formatted_response(best_matching_record)

llm_response = call_llm_with_full_text(query + " " + best_matching_record)
print_formatted_response(llm_response)
```

La respuesta final de GPT-5.2 explica que el enfoque Oracle RAG cambia la *security posture*: manteniendo los embeddings dentro de la base se asegura **Zero Trust compliance**; a diferencia de métodos donde el dato debe exportarse (ETL) a vector DBs externas —creando puntos de fuga—, Oracle aplica **Row-Level Security (RLS)** y auditing a la propia vector search.

### Del modular RAG a los sistemas multi-agente

Este es el **stepping stone hacia MAS**: en una arquitectura multi-agente el `RetrievalComponent` que se construyó **se vuelve una tool empuñada por un agente autónomo**. Ese agente no espera a que le pregunten: **planifica y razona** — puede decidir correr una query SQL para chequear inventario real-time (datos estructurados), luego conmutar a vector search para interpretar manuales de compliance complejos (no estructurados), y finalmente sintetizar una respuesta que adhiere a governance corporativa estricta. **Esto ya no es solo retrieval, sino *cognitive orchestration*.** El imperativo nuevo: **los agentes deben operar donde viven los datos** — adoptando la filosofía de *traer la IA a los datos* (deployando agentes dentro del ecosistema de Oracle Database 23ai), se revierte la necesidad tradicional de sacar datos para "pensar" sobre ellos (latencia + riesgo de fuga).

## Citas

> "Even the most advanced and well-trained generative AI models simply cannot answer a question about information it has no access to, and often, these models don't know that they don't know."
> "We are moving from bringing data to the AI to bringing the AI to the data."
> "RAG begins where generative AI ends by providing the information an LLM model lacks to answer accurately."
> "the difference between a model trained from scratch (or fine-tuned) and a RAG system can be summed up as the difference between learning (parametric) and referencing (non-parametric)."
> "This is no longer just retrieval, but cognitive orchestration."

## Para aplicar

- **Elegir el patrón arquitectónico, no solo la tool** — entre naïve (exact matching para registros estructurados), advanced (vector search para similitud semántica) y modular (orquestación dinámica). La decisión crítica es el patrón, no Pinecone vs Chroma vs converged DB.
- **Decidir RAG vs fine-tuning por tipo de conocimiento** — RAG para datos **non-parametric** (dinámicos, propietarios, sensibles al tiempo, soberanos); fine-tuning para conocimiento **parametric** (lenguaje/jerga del dominio, estilo). No son excluyentes: **RAG da los hechos, fine-tuning enseña a interpretarlos**.
- **Preferir converged databases (vector + relacional juntos)** — para que el dato nunca cruce el governance boundary; heredás Row-Level Security y auditing sobre la vector search (Zero Trust), y eliminás ETL hacia vector stores externos.
- **Fijar `temperature=0.1` en RAG corporativo** — para outputs precisos y factuales, minimizando alucinaciones creativas.
- **Congelar (freeze) la versión de las librerías** (ej. `openai==2.14.0`) — los updates introducen conflictos; freezear asegura reproducibilidad.
- **Usar secret managers (Colab Secrets / env vars), nunca keys en código** — evolución de la práctica respecto de leer keys de archivos.
- **Precomputar el índice (TF-IDF / HNSW / IVF)** en vez de re-vectorizar en cada query — recuperación como operación matricial, escalable a billones de vectores con latencia de ms.
- **Encapsular el retrieval en un componente self-contained** (`self.documents` + `self.vectorizer`) con un dispatcher por estrategia — base escalable para que un agente MAS lo use como tool.
- **Recordar que las métricas requieren supervisión humana** — un cosine score bajo (0.243) puede recuperar el record correcto; el Evaluator combina métricas (E1) con human feedback (E2).

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[02 - RAG Embeddings in Oracle Vector Stores]] — cap. 2: pasar de la simulación Python a la realidad enterprise; cómo Oracle AI Vector Search embebe datos en vectores a escala (mencionado al cierre).
- [[08 - Boosting RAG Performance with Human Feedback]] — cap. 8: implementación de feedback loops adaptativos / human feedback (el Evaluator E2 referenciado aquí). *(En la 1ª edición esto era el cap. 5.)*
- [[07 - Empowering AI Models by Fine-Tuning RAG Data]] — cap. 7: fine-tuning del modelo para aliviar la base RAG (el Trainer T2 y RAG-vs-fine-tuning referenciados aquí). *(En la 1ª edición esto era el cap. 9.)*
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Hybrid Search]] · [[Chunking Strategies]] · [[Reranking]] — el patrón RAG del vault; *hybrid search* (SQL + vectores) es exactamente Oracle AI Vector Search; advanced RAG conecta con reranking.
- [[Change Data Capture]] · [[ACL Filtering en RAG]] — keeping data fresh y data sovereignty / governance boundary en el RAG del vault.
- [[Grounding]] · [[Hallucinations]] — RAG como mitigación de alucinaciones por falta de info en el training; grounding en hechos recuperables.
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — el `RetrievalComponent` como tool de un agente; modular RAG → MAS (cognitive orchestration).
- **MAS-RAG · Oracle Converged Database · Oracle Database 23ai · Oracle AI Vector Search · Data sovereignty · Parametric vs non-parametric knowledge · Naïve RAG · Advanced RAG · Modular RAG · Agentic RAG · Integrated vectorization · Embeddings · Vector Search · Cosine Similarity · TF-IDF · HNSW · IVF · Row-Level Security (RLS) · Zero Trust · Fine-tuning · LLM · Multi-Agent Systems (MAS)** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
