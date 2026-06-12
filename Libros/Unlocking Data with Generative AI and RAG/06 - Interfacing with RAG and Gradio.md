---
title: 06 - Interfacing with RAG and Gradio
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 6
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Interfacing with RAG and Gradio
  - Interfaces para RAG con Gradio
updated: 2026-06-12
---

# 06 - Interfacing with RAG and Gradio

> [!info] Capítulo 6 · *Unlocking Data with Generative AI and RAG* — Packt (ISBN 9781835887905), Keith Bourne
> Hasta ahora la app [[RAG]] funcionaba con un prompt y una pregunta **hard-codeados**; este capítulo le da una **[[UI]] interactiva** con [[Gradio]] para que cualquier usuario la pruebe sin tocar el código. Guía práctica: levantar el entorno de Gradio, integrar la cadena RAG, construir una interfaz amigable y hostearla online de forma permanente y gratis. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[05 - Managing Security in RAG Applications]] · siguiente [[07 - The Key Role Vectors and Vector Stores Play in RAG]].

## Resumen

Desarrollar [[RAG]] es **construir aplicaciones**, y al principio uno codifica el prompt y la pregunta directamente en una variable — pero los usuarios reales necesitan una **interfaz**. Este capítulo es una guía práctica para volver interactiva la app RAG usando **[[Gradio]]** como **[[UI]]**: instalar el entorno, integrar la cadena RAG existente, armar una interfaz amigable y, finalmente, **hostearla online de forma permanente y gratis** vía [[Hugging Face Spaces]]. Gradio evita tener que hacer desarrollo web/mobile extenso, lo que facilita compartir y demostrar el sistema. El capítulo recorre cuatro temas: **¿Por qué Gradio?**, sus **beneficios**, sus **limitaciones** y un **code lab** que retoma la app del cap. 5 ([[05 - Managing Security in RAG Applications]]) y le añade la interfaz.

El argumento central: un data scientist domina ML, NLP, GenAI, LLMs y RAG, pero raramente domina además el **desarrollo web** (un campo muy técnico de por sí). Gradio cierra esa brecha levantando una UI apta para RAG "en minutos", en formato compartible y con autenticación básica. No reemplaza a un desarrollador web ni sirve para un sitio robusto de producción: es ideal para una **prueba de concepto (POC)** y para testear/demostrar con usuarios. El code lab materializa esto con tres piezas — la función `process_question`, la instancia `gr.Interface` y la llamada `demo.launch()` que **levanta un servidor web local** y, con `share=True`, una **URL pública**. La interfaz resultante muestra finalmente en pantalla el **relevance score** que se añadió en el cap. 5 y las **fuentes** que vienen arrastrándose desde el metadata del cap. 3 ([[03 - Practical Applications of RAG]]).

El código del capítulo vive en github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_06.

## ¿Por qué Gradio?

Los capítulos previos cubrieron temas de **data science** (ML, NLP, [[GenAI]], LLMs, [[RAG]]) que requieren tanta experiencia y tiempo de dominio que un data scientist típicamente **no** domina además el **desarrollo web** — un campo altamente técnico por sí mismo. Pero una **[[UI]]** es muy útil para **testear y demostrar** un sistema RAG. Acá entra **[[Gradio]]**: permite parar una UI muy rápido (frente a construir un frontend web completo), en un formato **compartible** y con **autenticación básica**.

> [!note] **[[Gradio]]** NO reemplaza a los desarrolladores web ni sirve para un sitio web robusto y completo, pero es **perfecto** para levantar una UI apta para RAG "en minutos". El libro se queda solo con los componentes necesarios para poner una app RAG en la web de forma compartible, y anima a explorar después las demás capacidades de Gradio.

## Beneficios de Gradio

[[Gradio]] aporta varias ventajas concretas:

- **Fácil para no-desarrolladores web** — permite armar la UI sin saber desarrollo web.
- **Open source** — la librería central de Gradio es **open source**: libre de usar, modificar y contribuir.
- **Integración con frameworks de ML** — se integra con los principales: **TensorFlow** ([[TensorFlow]]), **PyTorch** ([[PyTorch]]) y **Keras** ([[Keras]]).
- **Plataforma hosteada** — ofrece una plataforma para desplegar las interfaces de modelos y gestionar el acceso.
- **Colaboración** — funciones para compartir interfaces y recolectar feedback.
- **Integración con Hugging Face** — se integra con **[[Hugging Face]]** (fundada por ex-empleados de OpenAI), que aporta recursos a la comunidad GenAI: compartir modelos y hostear datasets.
- **Hugging Face Spaces** — **[[Hugging Face Spaces]]** permite hostear de forma **permanente** la demo de tu modelo de ML: un link de Gradio permanente y **gratis**.

> [!note] **[[Hugging Face Spaces]]**: hosting **permanente y gratuito** para la demo de tu modelo de ML — un link de Gradio que no expira, ideal para compartir el POC sin mantener un servidor propio.

## Limitaciones de Gradio

[[Gradio]] tiene dos límites importantes que conviene tener presentes:

- **No apto para producción** — NO sirve para apps a nivel **producción** que sirvan a cientos, miles o millones de usuarios; para eso hay que contratar a un experto en frontend. Es excelente para una **prueba de concepto ([[Proof of Concept|POC]])** o para testear con usuarios con interactividad básica.
- **Falta de flexibilidad** — está limitado en lo que se puede construir; bien para un POC, pero para features de UI sofisticadas es mucho más limitante que un framework web completo.

> [!warning] Seteá expectativas con tus usuarios: lo que ven es **solo una demo simple**. [[Gradio]] NO es para producción (cientos/miles/millones de usuarios → contratar un experto en frontend) y tiene **flexibilidad limitada** frente a un framework web completo.

## Code lab – Añadir una interfaz Gradio

El laboratorio retoma exactamente el código del cap. 5 ([[05 - Managing Security in RAG Applications]]) **excepto** las líneas del ataque prompt-probe del final. Sobre esa base, se le añade la interfaz [[Gradio]].

### Instalación e imports

Primero se instala `gradio` y se **remueve un paquete en conflicto** (`uvloop`):

```bash
%pip install gradio
%pip uninstall uvloop -y
```

La primera línea instala `gradio`; la segunda elimina el paquete `uvloop`, que entra en conflicto.

Luego, los nuevos imports:

```python
import asyncio
import nest_asyncio
asyncio.set_event_loop_policy(asyncio.DefaultEventLoopPolicy())
nest_asyncio.apply()
import gradio as gr
```

- `asyncio` — librería para escribir código concurrente mediante corutinas y event loops.
- `nest_asyncio` — permite **event loops anidados** en Jupyter.
- `asyncio.set_event_loop_policy(asyncio.DefaultEventLoopPolicy())` — fija la política de event loop por defecto.
- `nest_asyncio.apply()` — parchea asyncio para habilitar los loops anidados.
- `import gradio as gr` — importa Gradio bajo el alias **`gr`**.

### La función process_question

Esta función es la que se ejecuta al apretar **Submit** (se cablea más abajo en `gr.Interface`):

```python
def process_question(question):
    result = rag_chain_with_source.invoke(question)
    relevance_score = result['answer']['relevance_score']
    final_answer = result['answer']['final_answer']
    sources = [doc.metadata['source'] for doc in result['context']]
    source_list = ", ".join(sources)
    return relevance_score, final_answer, source_list
```

Toma la pregunta del usuario, invoca `rag_chain_with_source`, extrae del resultado el `relevance_score`, el `final_answer` y las `sources` (la `source` del metadata de cada documento del contexto), une las fuentes en un string separado por comas (`source_list`) y devuelve los **tres** valores.

### gr.Interface

Se instancia la interfaz declarando función, entradas y salidas (se conserva el salto de línea del libro en la descripción):

```python
demo = gr.Interface(
    fn=process_question,
    inputs=gr.Textbox(label="Enter your question",
        value="What are the Advantages of using RAG?"),
    outputs=[
        gr.Textbox(label="Relevance Score"),
        gr.Textbox(label="Final Answer"),
        gr.Textbox(label="Sources")
    ],
    title="RAG Question Answering",
    description=" Enter a question about RAG and get an answer, a 
        relevancy score, and sources."
)
```

- `fn` — la función que se llama en cada interacción (`process_question`, que dispara el pipeline RAG).
- `inputs` — un `gr.Textbox` para la pregunta (pre-cargado con `"What are the Advantages of using RAG?"`).
- `outputs` — tres `gr.Textbox` para **Relevance Score**, **Final Answer** y **Sources**.
- `title` + `description` — definen el encabezado de la interfaz.

### demo.launch y autenticación

La interfaz se lanza con:

```python
demo.launch(share=True, debug=True)
```

- **`share=True`** — genera una **URL pública** vía un servicio de tunneling: cualquiera con el link puede usar la app **sin** correr el código localmente.
- **`debug=True`** — activa el modo debug (mensajes de error detallados en la consola del navegador).

> [!note] `demo.launch(...)` es una línea especial: **inicia un servidor web local** que hostea el `gr.Interface(...)`. La celda corre **en perpetuidad** hasta que la detenés, y **no podés correr otras celdas** mientras está activa. La interfaz sigue viva hasta que la celda completa o se llama `gr.close_all()`.

Opcionalmente, se puede añadir una capa de **autenticación** básica:

```python
demo.launch(share=True, debug=True, auth=("admin", "pass1234"))
```

El parámetro **`auth`** agrega un gate de usuario/contraseña (`admin` / `pass1234`).

> [!warning] **Cambiá esas credenciales** sí o sí, y compartilas solo con los usuarios previstos. No es altamente seguro, pero es una barrera de acceso básica.

![[06-fig-6.1-gradio-interface.jpg]]
*Figure 6.1 – Gradio interface*

Tras el launch tenés un servidor web interactivo que toma input, lo procesa y devuelve nuevos elementos de interfaz — algo que antes requería experiencia en desarrollo web y ahora se levanta en minutos, dejándote enfocar en el código RAG. La pregunta pre-cargada se puede cambiar; preguntas irrelevantes hacen que el LLM responda `I don't know` (el guard de relevancia del cap. 5, [[05 - Managing Security in RAG Applications]]). Conviene probar tanto preguntas relevantes como irrelevantes.

### La interfaz en acción

Al lanzar en Colab, el log de arranque es:

```
Colab notebook detected. This cell will run indefinitely so that you can see errors and logs. To turn off, set debug=False in launch().
Running on public URL: https://pl09q9e4g8989braee.gradio.live
This share link expires in 72 hours.
```

> [!warning] El **share link expira en 72 horas**. Para un link permanente, hostear en [[Hugging Face Spaces]].

Hacer clic en el link abre la interfaz a pantalla completa en su propia ventana del navegador. **Submit** dispara el proceso RAG vía `result = rag_chain_with_source.invoke(question)`; la respuesta llega tras unos momentos.

![[06-fig-6.2-gradio-with-response.jpg]]
*Figure 6.2 – Gradio interface with response*

La interfaz de respuesta muestra **tres elementos**:

1. **Relevancy score** — el puntaje de relevancia añadido en el cap. 5 ([[05 - Managing Security in RAG Applications]]) como medida de seguridad para bloquear prompt injections; normalmente **NO** se muestra al usuario final — acá es solo un ejemplo de cómo exponer info extra.
2. **Final Answer** — la respuesta de **ChatGPT 4**, ya formateada en markdown → Gradio usa los saltos de línea para separar párrafos.
3. **Sources** — una lista de **cuatro** fuentes (cuatro las devuelve el retriever), provenientes del código que carga metadata añadido en el cap. 3 ([[03 - Practical Applications of RAG]]), recién ahora mostradas porque hay una UI. Las cuatro fuentes son **la misma** porque este ejemplo chico recupera de una sola fuente de datos; en apps reales se recuperarían muchas más fuentes → más entradas.

## Citas

> "Colab notebook detected. This cell will run indefinitely so that you can see errors and logs. To turn off, set debug=False in launch()."
> "Running on public URL: https://pl09q9e4g8989braee.gradio.live"
> "This share link expires in 72 hours."

## Para aplicar

- **Levantar la UI** — `%pip install gradio` y `%pip uninstall uvloop -y`; importar `asyncio` + `nest_asyncio` (con `set_event_loop_policy` y `nest_asyncio.apply()`) y `import gradio as gr`.
- **Cablear RAG a la interfaz** — escribir `process_question(question)` que invoque `rag_chain_with_source`, declarar `gr.Interface(fn=..., inputs=..., outputs=[...])` y lanzar con `demo.launch(...)`.
- **Compartir** — usar **`share=True`** para obtener una URL pública temporal (expira en 72 h); para permanente, hostear en [[Hugging Face Spaces]].
- **Proteger** — añadir **`auth=("usuario", "contraseña")`** como gate básico, cambiando las credenciales por defecto.
- **Testear** — probar preguntas relevantes e irrelevantes (estas últimas → `I don't know`); cerrar el servidor con **`gr.close_all()`** cuando termines.
- **Seguir aprendiendo** — visitar gradio.app y recorrer el **Quickstart** y la documentación.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[05 - Managing Security in RAG Applications]] — capítulo anterior; el code base que este capítulo continúa y el **relevance score** (guard de relevancia) que la UI exhibe.
- [[07 - The Key Role Vectors and Vector Stores Play in RAG]] — capítulo siguiente; el rol de los vectores y los vector stores en RAG.
- [[03 - Practical Applications of RAG]] — de ahí viene el metadata `source` de los documentos, que la UI por fin muestra.
- [[Gradio]] — la librería de UI usada en todo el capítulo (candidata a nota propia).
- [[Hugging Face]] · [[Hugging Face Spaces]] — integración y hosting permanente gratis (candidatas a nota propia).
- [[Proof of Concept]] — el nivel para el que Gradio es ideal (candidata a nota propia).
- [[UI]] · [[RAG]] · [[LangChain]] — conceptos núcleo que la interfaz conecta.
- [[TensorFlow]] · [[PyTorch]] · [[Keras]] — frameworks de ML con los que Gradio integra (candidatos a nota propia).
