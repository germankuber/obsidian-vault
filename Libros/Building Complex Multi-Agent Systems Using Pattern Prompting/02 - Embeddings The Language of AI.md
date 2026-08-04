---
title: Embeddings: The Language of AI
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 2
created: 2026-06-19
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/case-study
  - status/permanent
aliases:
  - Embeddings The Language of AI
  - Cap 2 - Embeddings
---

# Embeddings: The Language of AI

> [!info] Capítulo 2 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> Los [[Embeddings|embeddings]] son la tecnología que sostiene a GenAI: un formato de datos que usan los [[LLM|LLMs]] para representar el **significado** de palabras y frases de forma que las máquinas lo entiendan. El capítulo explica qué son los embeddings y cómo capturan significado vía **afinidad** (semantic islands, affinity groups, intersections), cómo calcular similitud semántica con **[[Cosine Similarity|cosine similarity]]** (angular distance), cómo **elegir un embedding model** (MTEB leaderboard + 9 factores), cómo **testearlos en código** (sentence-transformers + all-MiniLM-L6-v2), qué son las **[[Vector Database|vector databases]]** (data migration vs live query) y cómo tomar **decisiones de chunking** (chunk size, overlap, strategy). Los embeddings son el **hilo común** que cruza prompt, documentos, vector DB y LLM. Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape]] · siguiente [[03 - Building with GenAI Parameters, Tuning, and Project Phases]].

## Resumen

El capítulo presenta los **[[Embeddings|embeddings]]** como "el lenguaje de la IA": un tipo especial de formato de datos que los [[LLM|LLMs]] usan para representar el significado de palabras y frases de un modo que las máquinas pueden procesar. Su importancia es transversal — no se usan solo dentro del LLM, sino para analizar documentos, mejorar prompts y hacer la búsqueda y recuperación más precisa — y son accesibles vía APIs, herramientas open-source y [[Vector Database|vector databases]]. La tesis operativa es que, como los LLMs "hablan" en vector embeddings, entender cómo funcionan es esencial para construir prompts y seleccionar los datos que más ayudarán al LLM a responder.

El recorrido arranca con **qué son los embeddings**: cómo capturan las relaciones entre palabras de forma análoga a cómo los humanos aprenden el lenguaje (por exposición a millones de frases). Un embedding es un **vector muy grande** donde cada componente es la **afinidad** (un float de cerca de 0 = afinidad máxima a cerca de 1 = afinidad mínima) entre la palabra objetivo y otra del vocabulario. Con la **Tabla 2.1** (afinidades de la palabra "park") se ve que una palabra tiene múltiples significados semánticos. Luego introduce la **afinidad** propiamente: las afinidades se construyen contando co-ocurrencias en cientos de miles de frases (excluyendo function words como "the", "to", "a"), y se ilustra con la pregunta "how can I go to the park?", que se resuelve filtrando **semantic islands** (Fig 2.1), **affinity groups** (Fig 2.2) y la **intersección** de grupos (Fig 2.3) hasta llegar a "you can drive your car or ride your bike to the park". Cierra con un **recap de 5 puntos**.

Como inspeccionar tablas de afinidad a mano no escala, el capítulo pasa a **calcular similitud semántica programáticamente** con **[[Cosine Similarity|cosine similarity]]**: cada embedding es un vector que define un punto, y el **ángulo** entre dos vectores (angular distance) mide su similitud (Fig 2.4); el coseno es muy eficiente incluso con miles de dimensiones. Después aborda **cómo seleccionar un embedding model** (empezando por el **Hugging Face / MTEB leaderboard**) y enumera **9 factores**: Max tokens, Memory requirements, Dimensions, Cost, Training documents, Zero-shot, Language, Compatibility, Size of the model. Sigue con un ejemplo práctico de **testear embeddings en código** (sentence-transformers, sklearn, all-MiniLM-L6-v2, tres frases).

Finalmente explica las **[[Vector Database|vector databases]]** (lo que un RDBMS/NoSQL es a los datos estructurados, la vector DB lo es a los no estructurados): dos procesos — **data migration** (ingesta desde RDBMS/file server/web service en JSON/PDF/CSV/TXT, con chunking) y **live query** (Fig 2.5) — y la idea de que los embeddings existen **en tres lugares a la vez**: prompt, vector database y LLM (Fig 2.6). Termina con **chunking de documentos** (Fig 2.7): las decisiones de **chunk size**, **chunk overlap** y **chunking strategy** impactan fuerte en la calidad. El ejemplo del libro de Git (Figs 2.8/2.9/2.10) muestra que chunkear por capítulos sin overlap aísla contexto útil, y que un **50% de overlap** mantiene continuidad contextual — con el downside de información redundante (más costo de la llamada LLM + clutter en session memory). El cap. 3 retomará chunking, testing y RAG.

## What are embeddings?

Los LLMs aparentan entender el texto que escribimos y devolver justo lo que buscamos, pese a que los lenguajes humanos tardaron miles de años en desarrollarse y están llenos de ambigüedades y expresiones idiomáticas. No tenemos una sola palabra para un objeto, acción o concepto: no solo "talk", sino "mumble", "mutter", "confer". A veces una palabra tiene significados enteramente no relacionados — el ejemplo del libro: "I want to go to the **park**" vs "I will **park** my car" — y además existe la ironía, donde decimos lo opuesto a lo que queremos. Los LLMs sortean estas barreras en gran parte gracias a los **[[Embeddings|embeddings]]**.

> [!note] Los embeddings contienen información sobre las **relaciones entre palabras** además de representaciones de las palabras mismas. Pueden pensarse como un método para almacenar los significados de palabras y frases.

El paralelo con el aprendizaje humano es central: los humanos aprenden el lenguaje de chicos, cuando el cerebro está afinado para adquirirlo, y a través de la exposición a muchas frases internalizan gradualmente los matices. Del mismo modo, los LLMs aprenden los matices al ser entrenados con **millones o miles de millones** de frases de grandes colecciones de datos. Esto deja dos preguntas que el capítulo responde con embeddings: cómo representar significado semántico en sistemas de cómputo, y cómo comprimir el conocimiento de miles de millones de frases en unos pocos millones de parámetros que caben en el disco de una laptop.

> [!note] Los embedding spaces incluyen una **métrica** para medir cuán similares son las palabras. Conviene pensar esa distancia como una **normalized Euclidean distance**: las palabras consideradas similares se ubican más cerca que las disímiles.

### What do vectors look like?

Un embedding se crea a partir de una palabra o frase y se implementa como un **vector muy grande**. Cada componente (la columna *affinity* de la Tabla 2.1) asocia la palabra objetivo con otra palabra o frase de un **source vocabulary**. El valor de cada componente es un **floating-point** que va de **cerca de 0 (afinidad máxima) a cerca de 1 (afinidad mínima)**, midiendo la fuerza de la relación.

> [!warning] La convención de afinidad es **contraintuitiva**: un valor de **0.0001** significa una afinidad **extremadamente fuerte**, y un número muy grande como **0.9999** significa que las palabras apenas están relacionadas. (Esto choca con la prosa del libro, que más abajo describe "medium/high affinity" para Car/Bike/Picnic; reproducimos abajo tanto los números de la tabla como el texto del libro tal cual, sin reconciliarlos.)

#### Tabla 2.1 — Word affinities for the word "park"

| Word | Affinity |
|---|---|
| Car | 0.6545 |
| Bike | 0.59934 |
| Play | 0.33333 |
| Picnic | 0.2313 |
| Menu | 0.99990 |
| Cup | 0.099999 |
| Dog | 0.460000 |

La columna izquierda muestra las palabras chequeadas para afinidad con "park"; la derecha, la fuerza de cada asociación. El libro afirma textualmente que **"Car" y "Bike" tienen medium affinity** (probablemente derivada de frases sobre parkear una bici o un auto), y que **"Picnic" tiene high affinity** con "park" (un park como lugar donde caminamos, jugamos y hacemos picnic). Con esta tabla se ve que "park" tiene **al menos dos significados**: en otras palabras, tiene múltiples semantic meanings.

### What is affinity?

Los valores de los vector embeddings se crean **iterando sobre cientos de miles, millones o miles de millones de frases** en las que aparece la palabra, anotando las otras palabras que aparecen cerca, **siempre excluyendo function words** comunes como "the", "to" y "a".

> [!note] ¿Por qué importa la co-ocurrencia? Cuando dos palabras aparecen juntas con frecuencia (p.ej. "park" y "picnic"), el emparejamiento repetido **moldea y refina** el significado de una o ambas. Ver "park" junto a "picnic" en innumerables contextos establece una asociación semántica: un lugar donde frecuentemente hacemos picnic. El efecto **depende de la frecuencia**: un par de palabras debe ocurrir cientos de miles de veces para dejar un impacto medible; si fuera raro, la conexión sería demasiado débil.

El libro desglosa cómo los embeddings responden la pregunta **"how can I go to the park?"**. Empieza por la palabra **"go"**, que indica movimiento y por tanto es semánticamente similar a otras motion words: "ride", "drive", "travel", "move", "run". Los embeddings agrupan naturalmente palabras y frases con significados similares — **semantic islands**: frases sobre ir a un lugar caen en un grupo, las de comer o trabajar caen en otros.

![[B34134_2_1.png|460]]
*Figure 2.1 – Semantic islands: sentences with similar meanings cluster together in embedding space*

> [!note] La **propiedad clave** de los embeddings: cuando un texto se convierte a su representación de embedding, **aterriza en un grupo de frases con significados semánticos similares.** Como la respuesta aceptable "drive a car or ride a bike" contiene las motion words "drive" y "ride", aterriza en el mismo grupo. Así sabemos que la respuesta, si existe, está en ese grupo — aunque el grupo también contiene frases inútiles como "fly to Mars" o "paddle my boat" que hay que eliminar.

Para refinar, se nota que "park" **no** tiene afinidad fuerte con "boat", "paddle" o "Mars", pero **sí** con "bike" y "car". La palabra "park" pertenece además a otro **affinity group** (Fig 2.2) que contiene la pregunta "how can I go to the park?". Un embedding puede tener **miles** de estas afinidades, cada grupo iluminando similitudes entre frases.

![[B34134_2_2.png|425]]
*Figure 2.2 – Affinity groups: words and sentences cluster by shared semantic context*

Considerando las palabras con afinidad fuerte tanto a "go" como a "park", se llega al grupo aún más chico de frases en la **intersección** de ambos grupos (group intersections):

![[B34134_2_3.png|258]]
*Figure 2.3 – Embedding group intersections: the overlap between "Traveling" and "Park" groups narrows the answer*

Iterando sobre el vocabulario y examinando cada entrada de la affinity table para evaluar la fuerza de las asociaciones, y filtrando palabras y frases de baja afinidad, se llega a una frase como **"you can drive your car or ride your bike to the park"** que responde correctamente la pregunta.

> [!tip] Recap de la sección de afinidad (los 5 puntos del libro):
> - Las palabras son **símbolos** que pueden representar distintos significados (ejemplo "park": lugar de picnic vs acción de parkear un auto).
> - Una palabra adquiere matices de significado a través de la **asociación frecuente** con otra palabra.
> - Los LLMs se entrenan con miles de millones de frases y palabras, y encuentran las palabras en los **mismos contextos** que nosotros; por eso aprenden los mismos matices.
> - La similitud de frases y palabras se captura con **embedding vectors**, donde cada dimensión refleja la probabilidad de que una palabra del vocabulario aparezca cerca de la palabra objetivo.
> - La **cercanía** de los embedding vectors implica significado semántico compartido; se la usó para hallar la respuesta removiendo iterativamente frases de baja afinidad hasta quedar con un set chico de frases hiper-relevantes.

Inspeccionar embeddings a mano "would take far too long" — se necesita un método más eficiente, las similarity functions de la próxima sección.

## Calculating semantic similarity programmatically

Examinar una affinity table manualmente no es eficiente; para volver útiles en la práctica los procesos descriptos se necesita una **similarity function** altamente eficiente que compare programáticamente los embedding vectors de dos palabras. Hay varias funciones, pero la **[[Cosine Similarity|cosine similarity]]** es quizás la más usada.

Como un word embedding es un vector y todos los vectores definen un punto único en el espacio, se puede trazar una línea desde el origen hasta ese punto (Fig 2.4). Cada par de líneas define un **ángulo**, y ese ángulo entre los dos puntos representa la **"angular distance"**. La longitud de las líneas es otro factor en la distancia: algunas versiones del coseno **normalizan** los vectores a la misma longitud, pero incluso sin normalizar, el coseno del ángulo entre dos vectores funciona bien como similarity function.

![[B34134_2_4.png|484]]
*Figure 2.4 – Angle between semantically similar words: smaller angle → greater similarity*

> [!note] El diagrama de la Fig 2.4 es solo **bidimensional**, mientras que un vector real tendría muchas más dimensiones. Los mismos principios aplican en alta dimensión: si tomás "Car" y buscás embeddings cuyo ángulo respecto de "Car" sea pequeño, recreás el grupo con otras formas de transporte como "bike"; tomando palabras cerca de "dog" surgen otros tipos de mascotas. El coseno es **muy eficiente**: incluso con miles de dimensiones, computa el ángulo sobre cada dimensión rápidamente.

Las similarity functions como el coseno suelen empaquetarse como una **API** disponible tras instalar un embedding model. Los embedding models son aplicaciones de IA que crean vector embeddings.

## Selecting an embedding model

Los ingenieros de software no necesitan crear embeddings desde cero: hay una gran selección de embedding models descargables, cada uno con APIs para convertir texto en embeddings, con características distintas y entrenados sobre datasets distintos. Antes de empezar un proyecto GenAI conviene seleccionar el modelo óptimo; un buen punto de partida es el **Hugging Face embedding model leaderboard** (MTEB): `https://huggingface.co/spaces/mteb/leaderboard`. Estos modelos soportan APIs para crear embedding vectors a partir de texto como tweets, prompts y documentos — enteros o en chunks.

> [!tip] Los **9 factores** para decidir el embedding model ideal:
> - **Max tokens** — el tamaño máximo de texto que el modelo convierte a embeddings. Se pueden crear embeddings de partes de un documento, documentos enteros, capítulos o frases sueltas. Si el chunk a guardar tiene muchos tokens (**un token ≈ 3 a 4 caracteres**), puede exceder el límite de la API; en ese caso, buscar un modelo con mayor token limit o chunkear el documento en piezas más chicas (se ve en el cap. 3).
> - **Memory requirements** — los embedding models se construyen entrenando sobre grandes cantidades de texto y pueden tener millones o miles de millones de parámetros que hay que cargar en memoria. Considerar temprano el costo de máquinas con suficiente memoria, tanto en test/dev como en producción.
> - **Dimensions** — el tamaño del vector. Mayor dimensionalidad permite a los embedding vectors capturar más sutileza y matiz.
> - **Cost** — los modelos pueden descargarse a máquinas locales, correrse en una cloud platform o invocarse vía un servicio de un proveedor de LLM (como ChatGPT). El costo de infraestructura y el de uso de API son consideraciones importantes.
> - **Training documents** — algunos modelos se entrenaron sobre documentos de dominios específicos (p.ej. healthcare), otros sobre datos generales de internet.
> - **Zero-shot** — los **zero-shot embeddings** pueden identificar la similitud semántica de una palabra, frase o imagen **nueva** sin haber sido entrenados sobre un ejemplo. Ideal para casos donde el LLM debe entender datos únicos o de dominio específico.
> - **Language** — algunos modelos soportan múltiples idiomas, otros pueden ser solo en inglés.
> - **Compatibility** — el modelo usado para crear el embedding de una **query sentence** debe coincidir con el usado al **ingestar** los documentos. Revisar la documentación de la vector database por restricciones sobre modelos compatibles.
> - **Size of the model** — al igual que con memory requirements, elegir un modelo con miles de millones de parámetros puede llevar a costos de hardware sustanciales.

## Testing embeddings in your code

Con la teoría cubierta, el capítulo pasa a lo práctico: usar lenguajes de programación para hallar similitudes de embeddings en lugar de inspeccionar vectores a mano. Los embedding models pueden integrarse en el código para comparar strings — como un **prompt** y una pieza de texto — y estimar cuán probable es que ese texto sea útil para el LLM al generar una respuesta. Esta técnica sirve a los desarrolladores para **diseñar y testear prompts**, fine-tunearlos y mejorar la precisión de la respuesta, y para construir intuición sobre cómo funcionan los embeddings.

El siguiente snippet muestra un ejemplo de código de embedding model para comparar **tres strings**:

```python
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

# Step 1: Define your sentences
sentences = [
    "how do I go to the park",
    "you can drive to the park",
    "I love seafood"
]

# Step 2: Load a pre-trained sentence embedding model
model = SentenceTransformer('all-MiniLM-L6-v2')  # small, fast, good quality

# Step 3: Get vector embeddings for each sentence
embeddings = model.encode(sentences)

# Step 4: Compute cosine similarity matrix
similarity_matrix = cosine_similarity(embeddings)

# Step 5: Print the similarity matrix
print("Cosine Similarity Matrix:")
print(similarity_matrix)

# Optional: Interpret pairwise similarities
pairs = [(0, 1), (0, 2), (1, 2)]
for i, j in pairs:
    print(f"Similarity between '{sentences[i]}' and '{sentences[j]}': "
          f"{similarity_matrix[i][j]:.4f}")
```

> [!warning] En producción este código **no alcanzaría**: habría que hacer **semantic search** muy eficientemente sobre quizás miles de documentos, requiriendo un sistema mucho más escalable y robusto — una **[[Vector Database|vector database]]**, que como las relacionales y NoSQL está optimizada para consultar grandes volúmenes rápido, pero a diferencia de ellas devuelve resultados por **similitud semántica**.

El ejemplo usa el modelo **all-MiniLM-L6-v2** ("small, fast, good quality"), de la librería **[[sentence-transformers]]**, y calcula la matriz de **[[Cosine Similarity|cosine similarity]]** con `sklearn` para las tres frases (dos sobre ir al park, una sobre seafood), interpretando las similitudes **pairwise** (pares 0-1, 0-2, 1-2).

## What are vector databases?

> [!note] Una **vector database** es a los datos no estructurados lo que un **RDBMS o NoSQL** es a los datos estructurados y semi-estructurados.

Las vector databases son útiles cuando se envía un prompt a un LLM y se quieren **fetch chunks de texto** para incluirlos con el prompt y que el LLM los considere al generar su respuesta — especialmente útil cuando tenés datos privados o de confianza que el LLM desconoce, o cuando querés más control sobre la respuesta especificando exactamente qué datos usar. En el ejemplo de código previo se usó un array de embeddings para devolver frases semánticamente similares a un string; las vector databases hacen lo mismo, pero con todas las features y eficiencias de un sistema de base de datos completo.

La Fig 2.5 muestra **dos procesos**: el primero **ingesta** documentos en la vector database, el segundo maneja **queries semánticas en tiempo real**.

![[B34134_2_5.png]]
*Figure 2.5 – Vector database: Data migration (A) and live query (B) processes*

> [!note] **Los dos procesos de la vector database:**
> - **Data migration (A)** — los datos se cargan desde donde están almacenados actualmente (a menudo un **RDBMS**, **file server**, **web service** u otro medio) y se ingestan en la vector database. Los documentos ingestados pueden venir en distintos formatos: **JSON, PDF, CSV o TXT**. Durante la ingesta, los datos pueden necesitar ser **"chunked"** en varias piezas pequeñas de texto.
> - **Live query (B)** — una vez ingestados, los datos están listos para ser fetcheados y enviados al LLM para construir una respuesta a un prompt al manejar requests de usuarios.

Las vector databases dependen fuertemente de los embeddings, que son el **lenguaje por defecto** de las arquitecturas GenAI y un **hilo común** que corre a través de prompts, documentos, vector databases y LLMs (Fig 2.6):

![[B34134_2_6.png]]
*Figure 2.6 – Vector embeddings as a common thread across prompts, vector databases, and LLMs*

> [!note] Los **prompts** se convierten a vector embeddings, lo que permite consultar la vector database por datos semánticamente similares al prompt. Esos datos se **combinan** con el prompt y se envían al LLM. El LLM puede usar esos vectores para acceder a su conocimiento (también almacenado como embeddings) y extraer más información útil. Podemos pensar a los vector embeddings como existiendo en **tres lugares simultáneamente**: el **prompt**, la **vector database** y el **LLM**.

## Chunking documents

Al ingestar documentos en una vector database, suele ser aconsejable **chunkearlos** en secciones pequeñas. Para un PDF o Word, los chunks pueden corresponder a capítulos, sub-capítulos, frases, etc. El chunking agrega eficiencia: generalmente **no** querés recuperar documentos enteros de la vector database. Aun si un documento está dentro del max token limit del LLM, hay que considerar el **costo** y los problemas de cargar documentos grandes, que pueden crear dificultades al manejar la memoria usada para cachear el **historial de conversación**. El proceso de chunking es además la oportunidad de aplicar **data architecture thinking** y usar el conocimiento del dominio y del corpus para mejorar la precisión y calidad de las respuestas del LLM.

![[B34134_2_7.png]]
*Figure 2.7 – Chunking a document with sections A, B, and C into a vector database with overlap*

Como se ve en la Fig 2.7, un documento con secciones A, B y C se chunkea en una vector database. Nótese el **overlap** entre chunks: la mayoría de las vector databases soportan configurar un **porcentaje de overlap** entre chunks adyacentes. Cómo se chunkea un documento tiene un **gran impacto** en los resultados de la búsqueda vectorial.

### Chunking decisions

Las decisiones tomadas durante la ingesta de datos son críticas para el éxito de un proyecto. Los parámetros de configuración más comunes son el **chunk size**, la **chunking strategy** y el **chunk overlap**, y cada uno puede tener un impacto significativo en la calidad de las respuestas. Hay reglas generales, pero como en la mayoría de los procesos, la mejor forma de optimizarlos es **iterando y testeando** (el testing se ve en el cap. 3).

> [!tip] Dos factores importantes al configurar la database e ingestar datos:
> - **Chunk size y overlap** — cada query puede devolver múltiples chunks de texto seleccionados por ser semánticamente similares al contenido del prompt. Esos chunks suelen embeberse en el prompt y pueden señalarse a la atención del LLM especificando su uso en el prompt. El **chunk size y overlap óptimos** para una respuesta dada son la combinación que devuelve la **mínima cantidad de datos** necesaria para que el LLM produzca la respuesta óptima.
> - **Chunking strategy** — hay varias librerías para chunkear documentos, y cada una puede soportar múltiples formatos de documento además de distintas estrategias de cómo se realiza el chunking.

### Chunking strategies

Las decisiones de chunking varían por tipo de documento; el capítulo se enfoca en chunkear **PDFs** (de los más comunes y complejos). Una vez entendidos los trade-offs al chunkear PDFs, los demás tipos resultan fáciles.

El libro usa un ejemplo familiar: agregar un **libro sobre Git** a una vector database. Git tiene muchos comandos poco usados — no todos usamos regularmente comandos como **`reflog`, `range-diff`, `rev-parse` o `--show-toplevel`** — así que es útil consultar la database en lenguaje natural, por ejemplo: **"record when the tips of branches and other references were updated"**. Suponé que el libro tiene la estructura de capítulos de la Fig 2.8.

![[B34134_2_8.png]]
*Figure 2.8 – Git book: chapter structure used to illustrate chunking strategies*

Por los nombres de los capítulos y nuestro conocimiento de Git, inferimos que el contenido de cada capítulo está **solo levemente relacionado** con el de los otros. Esto nos inclinaría a **chunkear por capítulos con poco o ningún overlap**, como en la Fig 2.9.

![[B34134_2_9.png]]
*Figure 2.9 – Git book chunking by chapter with no overlap*

> [!warning] Pero considerá: alguien pregunta **"how do I create a branch in git?"**. La información relevante sobre crear una rama podría estar en **Chunk One**, mientras que **Chunk Two** también contiene contenido útil aunque menos directamente relacionado — p.ej. sugerencias de **branching strategy** (cuándo crear feature branches), que también le interesarían a quien crea una rama. El embedding vector de "how do I create a branch in git?" **podría no matchear** ningún texto del Chunk Two, porque la chunking strategy aisló todo lo semánticamente similar a la query en Chunk One, que era **demasiado chico** para contener todo el contexto útil.

La solución es hacer que **Chunk One y Chunk Two se solapen un 50%**:

![[B34134_2_10.png]]
*Figure 2.10 – Chunking with 50% overlap: contextual continuity is maintained between chunks*

> [!note] Con **50% de overlap** es altamente probable que la query "How do I create a branch in git?" devuelva tanto la información de **branching strategy** como la del comando de crear rama. El área de overlap es grande y los dos chunks ahora **comparten 50% de sus embeddings**: los embeddings de la mitad inferior del Chunk One son los mismos que los de la mitad superior del Chunk Two. Se mantiene la **continuidad contextual** entre ambos chunks, permitiendo que el embedding matching funcione en los dos.

> [!warning] El **downside** de esta estrategia: mayor probabilidad de devolver **información redundante** para otras queries. La información redundante **aumenta el costo de la llamada al LLM** y agrega **clutter a la session memory**, dificultando el manejo de la sesión.

El cap. 3 revisitará el chunking al recorrer los procesos para construir una aplicación LLM.

## Citas

> "Think of embeddings as a special type of data format used by LLMs to represent the meaning of words and sentences in a way that machines can understand."

> "It helps to think of this distance metric as a normalized Euclidean distance in which words deemed similar are placed closer together than dissimilar words."

> "A value of 0.0001 would signify an extremely strong affinity. A very large number such as 0.9999 would mean the words are barely related at all."

> "The frequency with which words appear together is therefore a good indicator of a refined semantic meaning."

> "The key property of embeddings is that when text is converted into its embedding representation, it lands in a group of sentences with similar semantic meanings."

> "Vector databases depend heavily on embeddings, which are the default language of GenAI architectures and a common thread running through prompts, documents, vector databases, and LLMs."

> "We can think of vector embeddings as existing in three places simultaneously: the prompt, the vector database, and the LLM."

> "The downside to this strategy is a higher chance of returning redundant information for other queries. Redundant information increases the cost of the LLM call and adds clutter to session memory, making session management more difficult."

## Para aplicar

- **Elegir el embedding model en el MTEB leaderboard** — empezá por `https://huggingface.co/spaces/mteb/leaderboard` y evaluá contra los **9 factores** (Max tokens, Memory, Dimensions, Cost, Training documents, Zero-shot, Language, Compatibility, Size).
- **Garantizar compatibilidad query↔ingestión** — usá el **mismo modelo** para embedear la query y para ingestar documentos; revisá la doc de tu vector database por restricciones.
- **Calcular max tokens por chunk** — recordá que **1 token ≈ 3–4 caracteres**; si el chunk excede el límite de la API, buscá un modelo con mayor token limit o chunkeá más fino.
- **Testear prompts con cosine similarity** — usá `sentence-transformers` + `all-MiniLM-L6-v2` + `sklearn.cosine_similarity` para comparar prompt vs candidatos de texto y fine-tunear prompts antes de pasar a una vector DB.
- **Definir la chunking strategy según el dominio** — para contenido con capítulos poco relacionados (estilo libro de Git), chunkeá por capítulo; añadí **overlap (p.ej. 50%)** cuando la respuesta necesite contexto que cruza chunks adyacentes.
- **Balancear overlap vs costo** — más overlap mejora la continuidad contextual pero devuelve **información redundante** que encarece la llamada al LLM y ensucia la session memory; optimizá iterando y testeando.
- **Planificar memoria y costo temprano** — dimensioná máquinas con suficiente RAM para los parámetros del modelo en **test, dev y producción**, no solo en prod.

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro.
- [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape]] — capítulo anterior: introdujo embeddings, vector databases y LLMs en el survey del paisaje GenAI; este capítulo los profundiza.
- [[03 - Building with GenAI Parameters, Tuning, and Project Phases]] — capítulo siguiente: retoma chunking, testing, **[[RAG]]** architectures, prompt engineering y frameworks de evaluación/optimización de GenAI.
- [[Embeddings]] — el formato de datos que representa significado; hilo común de GenAI.
- [[Cosine Similarity]] — función de similitud por angular distance entre vectores.
- [[Vector Database]] — DB optimizada para búsqueda por similitud semántica (data migration vs live query).
- [[LLM]] · [[GenAI]] — los embeddings son su "lenguaje".
- [[RAG]] — fetch de chunks semánticamente similares para enriquecer el prompt (se profundiza en el cap. 3).
- **[[Chunking]]** — decisiones de chunk size / overlap / strategy (candidata a nota propia).
- **[[Zero-shot embeddings]]** · **[[sentence-transformers]]** · **[[MTEB]]** · **[[all-MiniLM-L6-v2]]** — herramientas y conceptos sembrados (candidatos a nota propia).
