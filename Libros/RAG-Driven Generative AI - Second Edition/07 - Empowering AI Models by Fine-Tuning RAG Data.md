---
title: 07 - Empowering AI Models by Fine-Tuning RAG Data
libro: RAG-Driven Generative AI - Second Edition
autor: Denis Rothman
capitulo: 7
created: 2026-06-11
tags:
  - libros/rag-driven-generative-ai
  - type/case-study
  - status/permanent
aliases:
  - Empowering AI Models by Fine-Tuning RAG Data
  - Fine-Tuning RAG Data
updated: 2026-06-11
---

# 07 - Empowering AI Models by Fine-Tuning RAG Data

> [!info] Capítulo 7 · *RAG-Driven Generative AI - Second Edition* — Denis Rothman
> Cuando el volumen de datos RAG cruza un **umbral** inmanejable, los **datos estáticos** (ciencia dura, estables en el tiempo) pueden **fine-tunearse para reducir la masa de RAG**: convertir conocimiento *non-parametric* (recuperable) en *parametric* (en los pesos del modelo). Se prepara el dataset **SciQ** (question → answer+support) en JSONL, se fine-tunea **GPT-4o-mini** (cost-effective) vía la API de OpenAI, se monitorean los jobs y se analizan las métricas (training loss). Navegá: [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] · anterior [[06 - Operationalizing the Universal Context Engine]] · siguiente [[08 - Boosting RAG Performance with Human Feedback]].

## Resumen

Una organización que acumula datos RAG sin parar llega a un **umbral de datos non-parametric** (no pre-entrenados en el LLM) donde la masa se vuelve **extremadamente difícil de gestionar**: costos de storage, recursos de retrieval, capacidad de los modelos. Además, un modelo pre-entrenado tiene una **cutoff date** — ignora todo conocimiento del día siguiente en adelante (no se puede chatear sobre un diario publicado tras el cutoff), y ahí el retrieval es clave. Pero **no todos los dominios necesitan datasets enormes**: gigantes como Oracle/Google/Microsoft o dominios como los fallos legales de EE.UU. sí, pero muchas corporaciones no, y grandes porciones de **datos estáticos** (como las ciencias duras) **permanecen estables mucho tiempo**. Esos datos estáticos **pueden fine-tunearse para reducir el volumen de RAG requerido** — la decisión del *threshold* RAG vs fine-tuning del cap. 1.

El capítulo: (1) examina la **arquitectura de reducción de RAG vía fine-tuning** (los thresholds de *processing D2* y *storage D3* alcanzados para datos estáticos); (2) instala el entorno; (3) **prepara el dataset SciQ** (ciencia dura, del Allen Institute for AI / AI2, crowdsourced y human-controlled) — descarga desde Hugging Face vía Parquet, filtra por `support` y `correct_answer` presentes (10.481 preguntas), elimina los distractors, y convierte cada fila en pares **prompt-completion en formato JSONL** (system/user/assistant); (4) **fine-tunea GPT-4o-mini** (cost-effective, suficiente para esta tarea de completion factual de baja entropía) creando un fine-tuning job vía la API; (5) **monitorea los jobs** (DataFrame con job IDs, status, modelos; emails de success/failure); (6) **usa el modelo fine-tuneado** (completion con `temperature=0.0`, verificando que aprendió los datos — ej. "coriolis effect"); (7) **analiza las métricas** en la UI de OpenAI (job details, hiperparámetros, **training loss 1.1570**, traceability, usage/costos).

La conexión con el ecosistema RAG D·G·E·T del cap. 1: este capítulo usa **D1/D2** (collect+prepare el dataset), **E2** (human feedback — SciQ fue controlado por humanos), **T2** (fine-tuning), **G3/G4** (prompt engineering + generation), **E1** (métricas). El mensaje central: el fine-tuning **optimiza datos RAG en ciertos casos** (datos estáticos de alta calidad, parámetros correctos), pero no reemplaza al RAG para datos dinámicos.

![[07-fig-7.1.png]]
*Figure 7.1: Fine-tuning threshold reached for RAG data*

## Arquitectura de fine-tuning para datos RAG estáticos

Se cuestiona el uso de datos RAG non-parametric cuando exceden un **threshold manejable** (el principio de *RAG vs fine-tuning* del cap. 1). Los thresholds de **processing (D2)** y **storage (D3)** se alcanzan para los **datos estáticos** vs los dinámicos. El threshold depende de cada proyecto y de parámetros como:

- **Volumen de RAG a procesar** — embeber data requiere recursos humanos y de máquina; incluso sin embeber, apilar datos estáticos (estables mucho tiempo) **no tiene sentido**.
- **Volumen de RAG a almacenar y recuperar** — si se sigue apilando, mucho **se solapa (overlap)**.
- **Las recuperaciones que consumen recursos** — aun en sistemas open source, hay un número creciente de recursos a gestionar.

Cuando se alcanza el threshold, el fine-tuning puede ser buena solución.

### Fine-tuning dentro del ecosistema RAG (D·G·E·T)

![[07-fig-7.2.png]]
*Figure 7.2: Fine-tuning components of the RAG ecosystem*

Los componentes del ecosistema RAG (cap. 1) que este capítulo usa (los otros quedan en gris):
- **Collecting (D1) + preparing the dataset (D2)** — descargar y procesar el dataset **SciQ** (ciencia dura, human-crafted, crowdsourced) de Hugging Face.
- **Human feedback (E2)** — SciQ fue **controlado por humanos** y actualizado: simula cómo feedback humano confiable puede fine-tunearse para aliviar el volumen de RAG; las *explanations* de SciQ podrían incluso venir de evaluaciones humanas de modelos (como en el cap. 5).
- **Fine-tuning (T2)** — fine-tunear el cost-effective **GPT-4o-mini**.
- **Prompt engineering (G3) + generation/output (G4)** — engineering de prompts según OpenAI y display del output.
- **Metrics (E1)** — la interfaz de métricas de OpenAI.

## Instalar el entorno

`openai==2.26.0` (estable a marzo 2026, compatible con los workflows de supervised learning de OpenAI — fine-tuning donde **ejemplos human-generated ground truth** guían el comportamiento). Modelos fine-tuneables actuales: `gpt-4.1-2025-04-14`, `gpt-4.1-mini-2025-04-14`, `gpt-4.1-nano-2025-04-14`.

> [!note] Por qué ciencia dura: baja entropía
> Se elige **hard science human-controlled** porque **limita la entropía** del proceso: las preguntas científicas tienen un **único hecho definitivo** (no un rango de posibilidades creativas), constriñendo la incertidumbre y el ruido que el modelo debe navegar. Supervised learning = **dar ejemplos y obtener un resultado aceptable**.

```python
try:
    import openai
except:
    !pip install openai==2.26.0
    import openai
```

API key vía Colab secrets (`userdata.get("API_KEY")` → env var → `client = OpenAI()`). Además: `jsonlines==4.0.0` (escribir el training data en formato JSONL que OpenAI requiere, convirtiendo filas del DataFrame en JSON línea-a-línea), y `!pip install -U datasets` de Hugging Face (el flag `-U` actualiza `pyarrow` para leer Parquet).

## Preparar el dataset SciQ

Dos pasos: descargar+procesar columnas, y stream a JSONL. **Fine-tunear requiere preparación cuidadosa o el job falla.**

### Descargar y visualizar

SciQ fue desarrollado por el **Allen Institute for AI (AI2)** para scientific reasoning research; se publica en Hugging Face bajo `allenai`. Se descarga directo vía Pandas (`pd.read_parquet` — formato binario eficiente, carga instantánea a un DataFrame sin manejar archivos locales) y se **filtra** por `support` y `correct_answer` presentes:

```python
import pandas as pd
url = "https://huggingface.co/datasets/allenai/sciq/resolve/main/data/train-00000-of-00001.parquet"
df = pd.read_parquet(url)
filtered_dataset = df[(df["support"] != "") & (df["correct_answer"] != "")]
print("Number of questions with support: ", len(filtered_dataset))   # -> 10481
```

Dos razones del método: **direct access** (path más rápido y confiable a la fuente autoritativa de AI2) y **quality control** (solo filas con `support` —la explicación/razonamiento de por qué la respuesta es correcta— y `correct_answer`; ejemplos completos de alta calidad donde el modelo aprende a razonar con el contexto científico).

> [!warning] Eliminar los distractors (son ruido)
> SciQ está diseñado para tests de multiple-choice → incluye **3 distractor answers incorrectas** por pregunta. Para fine-tuning **son ruido**: queremos que el modelo aprenda a **generar** la respuesta correcta + explicación, no a **seleccionar** de una lista. Se eliminan con `.drop()`.

```python
df_view = pd.DataFrame(filtered_dataset)
columns_to_drop = ['distractor3', 'distractor1', 'distractor2']
df_view = df_view.drop(columns=columns_to_drop)
df_view.head()   # visual check: question, correct_answer, support aligned
```

La columna `question` será el **user prompt**; `correct_answer` + `support` se combinan para formar el **model completion** ideal.

### Formatear a JSONL (prompt-completion)

Se estructura en el formato JSONL preciso que la API requiere, iterando el DataFrame y asignando roles **system/user/assistant**. El `correct_answer` se combina con el `support` para que el modelo aprenda a dar **tanto el hecho como el razonamiento**:

```python
import json, jsonlines
import pandas as pd
items = []
for idx, row in df_view.iterrows():
    detailed_answer = row['correct_answer'] + " Explanation: " + row['support']
    items.append({
        "messages": [
            {"role": "system", "content": "Given a science question, provide the correct answer with a detailed explanation."},
            {"role": "user", "content": row['question']},
            {"role": "assistant", "content": detailed_answer}
        ]
    })

output_file = '/content/QA_prompts_and_completions.json'
with jsonlines.open(output_file, 'w') as writer:
    writer.write_all(items)
print(f"✅ Successfully created {output_file} with {len(items)} examples.")
```

Verificación: `pd.read_json(dfile, lines=True)`. El archivo `QA_prompts_and_completions.json` queda listo para subir a OpenAI.

## Fine-tunear el modelo

Se sube el training file y se crea el fine-tuning job. **Hiperparámetros en `'auto'`** (OpenAI los determina): `n_epochs`, `batch_size`, `learning_rate_multiplier`.

```python
from openai import OpenAI
import jsonlines
client = OpenAI()

result_file = client.files.create(
    file=open("QA_prompts_and_completions.json", "rb"), purpose="fine-tune"
)
param_training_file_name = result_file.id

ft_job = client.fine_tuning.jobs.create(
    training_file=param_training_file_name,
    model="gpt-4.1-mini-2025-04-14"
)
print(ft_job)
```

El output da el `FileObject` (id, bytes, `status='processed'`, `purpose='fine-tune'`) y el `FineTuningJob` (ej. `id='ftjob-JqLq1qdQ5Vo2jWQFhGL1IUoq'`, `fine_tuned_model=None` hasta terminar). Status posible: `validating_files` (OpenAI chequea el training file). El `created_at` es un **Unix timestamp** (segundos desde 1/1/1970 UTC). Modelo = GPT-4o-mini (versión más chica y cost-effective de GPT-4).

## Monitorear los fine-tuning jobs

Se consultan los últimos jobs y se arma un **DataFrame de monitoreo** (job IDs, created_at, status, model, training_file, error_message, fine_tuned_model):

```python
from openai import OpenAI
import pandas as pd
client = OpenAI()
response = client.fine_tuning.jobs.list(limit=3)   # increase to include history

job_ids, created_ats, statuses, models = [], [], [], []
training_files, error_messages, fine_tuned_models = [], [], []
for job in response.data:
    job_ids.append(job.id)
    created_ats.append(job.created_at)
    statuses.append(job.status)
    models.append(job.model)
    training_files.append(job.training_file)
    error_messages.append(job.error.message if job.error else None)
    fine_tuned_models.append(job.fine_tuned_model if hasattr(job, 'fine_tuned_model') else None)

df = pd.DataFrame({
    'Job ID': job_ids, 'Created At': created_ats, 'Status': statuses,
    'Model': models, 'Training File': training_files,
    'Error Message': error_messages, 'Fine-Tuned Model': fine_tuned_models
})
df['Created At'] = pd.to_datetime(df['Created At'], unit='s')
df = df.sort_values(by='Created At', ascending=False)
df
```

El `Status` informa los pasos: validating files, running, failed, succeeded. Se recupera el modelo más reciente con un **flag `generation`** que controla si se disparan las completions:

```python
generation = False  # False until fine-tuned, True when fine-tuned
non_empty_models = df[df['Fine-Tuned Model'].notna() & (df['Fine-Tuned Model'] != '')]
if not non_empty_models.empty:
    first_non_empty_model = non_empty_models['Fine-Tuned Model'].iloc[0]
    print("The latest fine-tuned model is:", first_non_empty_model)
    generation = True
else:
    first_non_empty_model = 'None'
    print("No fine-tuned models found.")
# -> The latest fine-tuned model is: ft:gpt-4.1-mini-2025-04-14:personal::CwQpKy8Y
```

> [!note] El flag `generation` evita gastos
> `generation==False` mientras se entrena. Si se halla un modelo → `generation=True` dispara las completions. Si **no** se halla → `generation=False` **no corre la API** en el resto del notebook, **evitando usar modelos que no se están entrenando** (y costos).

**Emails de OpenAI**: el de **success** trae dos datos críticos — el **Job ID** (`ftjob-...`, para logs/métricas) y el **new model ID** (`ft:gpt-4.1-mini-...::CwQpKy8Y`, **el más importante**: hay que copiarlo exacto para *usar* el modelo). El de **failure** avisa que falló pero **no explica por qué** (deriva a la UI de fine-tuning; en este caso fue un *hard limit* de billing). Ante fallo: verificar el training data (inconsistencias, missing values, labels incorrectos) y que el JSONL respete el schema de OpenAI.

## Usar el modelo fine-tuneado

Se prueba con una pregunta del dataset original para verificar que aprendió. `temperature=0.0` (sin creatividad para esta tarea de ciencia dura) y `max_output_tokens=200` (suficiente para confirmar, sin gastar tokens):

```python
prompt = "What phenomenon makes global winds blow northeast to southwest or the reverse in the northern hemisphere and northwest to southeast or the reverse in the southern hemisphere?"

if generation==True:
    response = client.responses.create(
        model=first_non_empty_model,
        input=[
            {"role": "system", "content": "Given a question, reply with a complete explanation for students."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.0,
        max_output_tokens=200
    )
else:
    print("Error: Model is None, cannot proceed with the API request.")
```

Se extrae el texto (`response.output_text`) y se formatea con `textwrap.fill`. El output confirma que **los datos se tomaron en cuenta** (aprendió el formato "answer Explanation: ..."):

```text
coriolis effect Explanation: Global winds blow northeast to
southwest or the reverse in the Northern Hemisphere. In the
Southern Hemisphere, they blow northwest to southeast or the
reverse. This is due to the Coriolis effect.
```

La respuesta es satisfactoria — aunque puede no serlo siempre y requerir más trabajo del dataset (mejor data, mayor volumen) **incrementalmente** hasta el goal.

> [!tip] Reusar el modelo sin re-entrenar (sin costos extra)
> Para verificar el modelo más tarde sin disparar un nuevo job: (1) correr solo *Installing the environment* (librería + API key, **no** las celdas del dataset); (2) saltar *Step 2 (Fine-tuning)* enteramente; (3) ir directo a *Step 3 (Monitoring)* — el código detecta automáticamente el `first_non_empty_model` del historial de la API; (4) correr las celdas de generación del Step 4. El nombre del modelo se puede guardar y usar en otro programa.

## Analizar jobs y métricas (UI de OpenAI)

En `platform.openai.com/finetune`: lista de jobs (todos / success / failed). Los **job details**:

![[07-fig-7.8.png]]
*Figure 7.8: Example view*

- **Status** (completed), **Job ID**, **Base model** (gpt-4o-mini), **Output model** (el modelo resultante), **Created at**.
- **Trained tokens** — total de tokens procesados (gauge del alcance del training).
- **Epochs** — pasadas completas sobre el training data; más epochs = mejor learning, **pero demasiados → overfitting**.
- **Batch size** — ejemplos por iteración; más chico = más updates y learning refinado, pero más lento.
- **LR multiplier** — cuánto se ajusta el learning rate del base model; más chico = updates más conservadores a los pesos.
- **Seed** — semilla del RNG; garantiza **reproducibilidad** (mismos resultados con mismas condiciones).

![[07-fig-7.9.png]]
*Figure 7.9: Loss metrics for a fine-tuned model*

**Training loss** — métrica confiable del performance durante el training: el **error promedio del modelo sobre el training dataset**. Valores más bajos = mejor fit. Un training loss de **1.1570** sugiere que el modelo aprendió a predecir bien su training data.

![[07-fig-7.10.png]]
*Figure 7.10: Training loss during the training job*

**Traceability completa** (mensajes del job): `Created fine-tuning job` → `Validating training file` → `Files validated, moving to queued` → `Fine-tuning job started` → `Checkpoint created at step 807` → `New fine-tuned model created` → `Evaluating against usage policies` → `model enabled for sampling` → `job successfully completed`. También hay una interfaz de **usage** (`platform.openai.com/usage`) para monitorear **costo por período y modelo**.

Conclusión: el fine-tuning **optimiza datos RAG en ciertos casos** — si se entrena con data de alta calidad y los parámetros correctos.

## Citas

> "a pretrained generative AI model is trained up to a cutoff date. The model ignores new knowledge starting the very next day."
> "large portions of static data, like those in the hard sciences, can remain stable for a long time. Such static data can be fine-tuned to reduce the volume of RAG data required."
> "the difference between learning (parametric) and referencing (non-parametric)." (principio del cap. 1 que este capítulo aplica)
> "fine-tuning can optimize RAG data in certain cases when necessary."

## Para aplicar

- **Identificar el threshold RAG vs fine-tuning** — cuando la masa de datos non-parametric se vuelve inmanejable (storage, retrieval, overlap), considerar fine-tunear los **datos estáticos** (estables mucho tiempo) para reducir el volumen de RAG; mantener RAG para los dinámicos.
- **Elegir dominios de baja entropía para fine-tuning** — ciencia dura / hechos definitivos limitan la incertidumbre; supervised learning con **ground truth human-generated**.
- **Filtrar por calidad antes de entrenar** — quedarse solo con filas que tienen `support` (explicación) y `correct_answer`; un job falla con data inconsistente o mal formateada.
- **Eliminar el ruido (distractors)** — quitar las opciones incorrectas de multiple-choice: el modelo debe **generar**, no seleccionar.
- **Combinar answer + support en el completion** — para que el modelo aprenda el **hecho Y el razonamiento**, no solo la respuesta.
- **Formatear en JSONL con roles system/user/assistant** — la estructura exacta que la API de chat/completion espera (`jsonlines`).
- **Fine-tunear el modelo más chico que alcance (GPT-4o-mini)** — cost-effective y suficiente para tareas de completion factual; dejar hiperparámetros en `'auto'` si no hay razón para fijarlos.
- **Usar un flag `generation` para evitar gastos** — no disparar la API hasta confirmar que hay un modelo entrenado; reusar el modelo del historial sin re-entrenar (saltar el Step 2).
- **Generar con `temperature=0.0` para tareas factuales** — sin creatividad; `max_output_tokens` acotado para verificar sin gastar.
- **Copiar y guardar el model ID exacto** (`ft:gpt-4.1-mini-...::xxxx`) — del email de success o del DataFrame; es lo que se usa para invocar el modelo.
- **Monitorear training loss y usage** — loss bajo = buen fit (cuidado overfitting con muchos epochs); el seed da reproducibilidad; revisar el costo por período/modelo.
- **Iterar el dataset incrementalmente** — si el output no satisface, mejorar la data / aumentar el volumen hasta el goal.

## Conexiones

- [[_RAG-Driven Generative AI - Second Edition|Mapa del libro]] — el MOC del libro.
- [[01 - Why Retrieval-Augmented Generation]] — cap. 1: este capítulo **aplica el principio RAG-vs-fine-tuning** (parametric vs non-parametric, *learning vs referencing*) y el ecosistema D·G·E·T (usa D1/D2, E1/E2, T2, G3/G4); el threshold del cap. 1 se materializa.
- [[05 - Building a Universal Context Engine]] — cap. 5: las *explanations* de SciQ podrían venir de evaluaciones humanas de modelos (human feedback E2 que el cap. 5 implementaba).
- [[08 - Boosting RAG Performance with Human Feedback]] — cap. 8: boost del RAG-driven generative AI con **human feedback** (hybrid-adaptive RAG); el `expert_feedback.txt` capturado allí alimenta un *fine-tuning loop*.
- [[06 - Operationalizing the Universal Context Engine]] — cap. 6: el patrón de carga segura de API key (`userdata` + `raise`) reusado.
- [[_RAG|RAG]] · [[Enterprise RAG Assistant]] · [[Chunking Strategies]] — el patrón RAG del vault; el fine-tuning como alternativa para reducir el volumen de datos RAG estáticos.
- [[_MLOps|MLOps]] · [[RLHF]] — fine-tuning supervisado (SFT), training loss, epochs/batch/LR, reproducibilidad por seed; el lifecycle de entrenamiento.
- [[Grounding]] · [[Hallucinations]] — datos estáticos verificados (ciencia dura, human-controlled) como grounding parametrizado; baja entropía reduce alucinaciones.
- [[Fine-tuning]] · [[LLM]] — el sujeto del capítulo (candidatos a nota propia si no existen aún).
- **SciQ dataset · Allen Institute for AI (AI2) · Static vs dynamic RAG data · Parametric vs non-parametric · Fine-tuning threshold · JSONL prompt-completion · Supervised fine-tuning (SFT) · GPT-4o-mini · Hugging Face datasets / Parquet · `jsonlines` · Distractors · Training loss · Epochs / batch size / LR multiplier / seed · Cutoff date · Coriolis effect (ejemplo)** — conceptos/técnicas del capítulo; candidatos a nota propia (varios ya existen en el vault, otros son semillas).
