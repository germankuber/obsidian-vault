---
title: 03 - Practical Applications of RAG
libro: Unlocking Data with Generative AI and RAG
autor: Keith Bourne
capitulo: 3
created: 2026-06-12
tags:
  - libros/unlocking-data-with-generative-ai-and-rag
  - type/case-study
  - status/permanent
aliases:
  - Practical Applications of RAG
  - Aplicaciones prácticas de RAG
updated: 2026-06-12
---

# 03 - Practical Applications of RAG

> [!info] Capítulo 3 · *Unlocking Data with Generative AI and RAG* — Packt (ISBN 9781835887905) · Keith Bourne
> Un recorrido por las aplicaciones reales de [[RAG]] en la empresa —[[Chatbots]] de soporte, reportes automatizados, e-commerce, [[Knowledge Base|knowledge bases]] internas y externas, innovation scouting, marketing y formación—, todas pensadas para **despertar la creatividad** más que para ser una lista exhaustiva, cerrando con un code lab que extiende el pipeline del cap. 2 para devolver la **fuente** de los documentos recuperados. Navegá: [[_Unlocking Data with Generative AI and RAG|Libro]] · anterior [[02 - Code Lab - An Entire RAG Pipeline]] · siguiente [[04 - Components of a RAG System]].

## Resumen

Este capítulo expande las aplicaciones que el cap. [[01 - What Is Retrieval-Augmented Generation (RAG)]] solo había enumerado, mostrándolas en detalle con ejemplos concretos. El autor advierte que los ejemplos son **ilustrativos, no exhaustivos**: están para inspirar y despertar la creatividad sobre dónde aplicar [[RAG]] y la [[GenAI|GenAI (IA generativa)]], no para cubrir todos los casos posibles. El arco recorre siete grandes áreas: **soporte al cliente y [[Chatbots]]** (técnico, financiero, salud), **reportes automatizados**, **e-commerce** (descripciones de producto y recomendaciones), **knowledge bases** (internas y externas, con el tema de las [[Hallucinations|alucinaciones]]), **innovation scouting y análisis de tendencias**, **recomendaciones en marketing**, y **formación y educación**. El hilo común: RAG conecta los LLMs con los datos —a menudo no estructurados— de la empresa para dar respuestas fundamentadas, personalizadas y contextuales que antes exigían trabajo manual o intervención humana. El capítulo cierra con el **code lab 3.1**, que extiende el código del cap. [[02 - Code Lab - An Entire RAG Pipeline]] para devolver, junto a la respuesta, la **fuente (metadata source)** de los documentos recuperados, usando [[RunnableParallel]] de [[LangChain]]. El código del capítulo vive en github.com/PacktPublishing/Unlocking-Data-with-Generative-AI-and-RAG/tree/main/Chapter_03.

## Soporte al cliente y chatbots con RAG

Los [[Chatbots]] evolucionaron de guiones rígidos (scripted) a **agentes conversacionales impulsados por RAG**. La idea: recuperar información de los datos de la empresa y del cliente, y generar respuestas coherentes y contextuales, superando ampliamente a los chatbots pre-programados o de NLP básico. Un chatbot con RAG puede leer datos profundos del cliente —por ejemplo, **todos tus extractos bancarios en PDF**— habilitando un servicio verdaderamente 1-a-1. El autor subraya que estos sistemas están todavía **"in their infancy"**: cabe esperar chatbots capaces de resolver consultas únicas sin intervención humana, en lenguaje natural en casi cualquier idioma, con memoria y cómputo masivos.

> [!note] RAG transforma al chatbot de un árbol de respuestas pre-programadas en un agente que **recupera de los datos reales del cliente y de la empresa** y **genera** respuestas coherentes y contextuales, habilitando atención personalizada 1-a-1.

### Soporte técnico

Muchas consultas técnicas son **recurrentes**: el mismo problema una y otra vez (ejemplo: proveedores de cable, donde gran parte de los reclamos son el mismo issue). Un chatbot con RAG **reconoce el problema a partir de interacciones previas** y entrega pasos de troubleshooting personalizados que **reconocen los intentos pasados** del cliente, lo que construye confianza en lugar de hacerlo repetir lo que ya probó.

### Servicios financieros

Cubre consultas de cuenta, problemas de transacciones y **asesoramiento personalizado** a partir del historial de transacciones y los detalles de la cuenta. El ejemplo central es la **tasa de interés de la tarjeta de crédito**: el dato suele estar enterrado en bases de datos, PDFs o detrás de barreras de seguridad; una vez identificado online, el chatbot responde **21.9%** y añade contexto del estilo:

> "You have only paid interest on your balance twice during your tenure as a customer."

Además, puede **cambiar de la tarjeta a tu cuenta hipotecaria sin derivar a un nuevo agente**, y maneja preguntas comparativas que requieren entender todo el contexto de la conversación para hacer seguimiento, como:

> "How much more is the interest rate on my credit card than my mortgage?"

### Salud

Con los permisos adecuados, el chatbot accede a **historiales médicos** para dar consejos de salud personalizados o gestionar turnos. El ejemplo es un paciente **recién diagnosticado de diabetes**: el sistema analiza su historia clínica, medicaciones y resultados de laboratorio recientes para dar consejo a medida sobre estilo de vida, dieta y medicación, y programa seguimientos con recordatorios. El resultado: mejores outcomes de salud y menor carga sobre los profesionales.

## RAG para reportes automatizados

RAG actúa de puente entre los **data lakes no estructurados** y los **insights accionables**: agiliza la elaboración de reportes, mejora la precisión y saca a la luz insights ocultos.

### Cómo se usa RAG en los reportes automatizados

El target son los reportes **estándar o codificables**. Se parte de un reporte automatizado ya existente y se le suma una capa de GenAI/RAG: se le alimenta un conjunto de **preguntas de análisis iniciales sin necesidad de input del usuario**, añadiendo ese contenido (gráficos y diagramas + **comentario del LLM**) como un componente inicial del pipeline. El reporte automatizado suele ser solo el primer paso —los tomadores de decisión siempre piden más—, así que **hacer del reporte parte de un sistema RAG más amplio** reemplaza o acelera el ida-y-vuelta con el analista.

> [!tip] Exponé **tanto los datos del reporte como los datos subyacentes** para que el personal no técnico pueda hacer preguntas amplias sin esperar a un analista de datos. Sumá más fuentes de datos para ganar profundidad.

### Transformar datos no estructurados en insights accionables

La mayor parte de los datos de una organización son [[Unstructured Data|no estructurados]] (artículos, papers de investigación, feeds sociales, contenido web) y son difíciles para el análisis tradicional. RAG los parsea, crea borradores y resúmenes **personalizables por individuo**, y resalta la información crítica según el rol o el interés —más sofisticado que usar un LLM directamente, porque **reemplaza pasos manuales**.

> [!note] Los datos no estructurados se transforman en la **etapa de [[Indexing|indexing]]** a un formato más usable: un PDF se extrae en **elementos con distintos niveles de importancia** (títulos y headers pesan más que los párrafos), las imágenes detectadas como tablas se extraen como un **table summary**, se vectorizan y se usan después.

> [!tip] Empezá por las áreas donde la información oportuna importa más —por ejemplo, **market analysis**: resumir noticias, reportes financieros e info de la competencia para condensar tendencias de mercado y poder reaccionar rápido.

### Mejorar la toma de decisiones y la planificación estratégica

Reportes resumidos y concisos + insights sobre tendencias de la industria, tecnología y panorama competitivo conducen a decisiones más informadas. Poder **hacer preguntas de seguimiento sobre los datos fuente** suma valor, y permite a las empresas ser **proactivas** (anticipar cambios de mercado o tecnológicos) en vez de reactivas.

## Soporte en e-commerce

### Descripciones dinámicas de producto

RAG genera descripciones **personalizadas** a partir del historial de navegación, compras pasadas y redes sociales. Los ejemplos giran en torno a **Rylee**: si es un comprador eco-friendly, la descripción enfatiza la **sostenibilidad**; si busca zapatillas de running o entrena para una maratón, enfatiza **amortiguación, estabilidad y durabilidad**, más el hecho de estar **probadas por maratonistas** y la reducción de lesiones. RAG también analiza las **reseñas** para volcar los pros y contras más mencionados en la descripción (más balanceada → menos devoluciones y reseñas negativas), y genera descripciones **multi-idioma** culturalmente apropiadas para mercados internacionales.

### Recomendaciones de producto para sitios de e-commerce

Los [[Recommendation Engine|motores de recomendación]] son uno de los usos mayores de la IA. RAG analiza historial de navegación, compras pasadas, search queries y redes sociales para encontrar **patrones más profundos** que los métodos previos. El ejemplo es **Aubri** (clienta VIP): navegando botas de senderismo, el sistema recomienda botas adecuadas **+ productos complementarios** (medias de trekking, mochilas, bastones), impulsando una compra de varios ítems. También analiza reseñas y ratings de productos para medir satisfacción y refinar futuras recomendaciones (que coincidan con los intereses **y** con las expectativas de calidad). Esto se extiende más allá del sitio web, a medios, contenido y marketing digital.

## Aprovechar las knowledge bases con RAG

### Searchability y utilidad de las KB internas

Combinando retrieval avanzado + LLMs, RAG convierte las [[Knowledge Base|knowledge bases]] internas en **buscadores internos**. Las KB internas (PDFs, Word, Google Docs, hojas de cálculo, slide decks) suelen estar **subutilizadas** por su volumen y la dificultad de extracción. RAG genera resúmenes concisos, da respuestas directas a partir de contenido enterrado en millones de páginas y mejora la categorización y el retrieval, optimizando el flujo de trabajo.

### KB externas + alucinaciones

Las KB externas son cruciales para leyes, regulaciones y estándares de la industria, compliance, I+D, medicina, academia y patentes: dominios **vastos y en cambio constante**. RAG recupera y resume:
- **Legal / compliance** — peinar miles de documentos para encontrar jurisprudencia (case laws), regulaciones y guías de compliance.
- **I+D** — acceso rápido a estudios, patentes y documentación técnica para **evitar duplicación** y disparar ideas.
- **Medicina** — case studies, investigación y guías de tratamiento para decisiones informadas.

> [!warning] Los LLMs como ChatGPT pueden dar información **ficticia pero convincente** ([[Hallucinations|alucinaciones]]). Usar un LLM "pelado" para investigación es riesgoso por esto.

> [!note] RAG **reduce las alucinaciones** manteniendo al LLM **fundamentado (grounded) en datos reales**. Se pueden añadir **múltiples llamadas al LLM** para verificar la relevancia de la respuesta antes de devolverla, e incluir documentación de soporte y **citas**.

## Innovation scouting y análisis de tendencias

RAG escanea y resume fuentes de calidad para **detectar tendencias emergentes** y áreas de innovación alineadas a una especialización. En el sector tech: analizar patentes, noticias tecnológicas e investigación para identificar tecnologías emergentes y patrones de innovación que **dirijan la I+D**. El propio ejemplo del autor es la **industria farmacéutica (pharma)**: analizar revistas médicas, reportes de ensayos clínicos (clinical trial reports) y bases de patentes para identificar hallazgos de investigación y oportunidades de desarrollo de fármacos, acelerando la innovación y mejor asignación de los presupuestos de investigación.

## Recomendaciones personalizadas en comunicaciones de marketing

RAG arma **bundles y colecciones de producto personalizados** en campañas digitales (analiza los datos del cliente + productos complementarios para entregar paquetes pre-ensamblados = conveniencia + valor). Potencia el **email marketing** con recomendaciones personalizadas que elevan los **CTRs (click-through rates)** y las conversiones, y crea experiencias a medida en todos los touchpoints digitales para fomentar la lealtad.

## Formación y educación

Aplica a universidades, educación secundaria **y** formación corporativa interna. RAG puede:
- **Generar y personalizar materiales** según necesidades, nivel de conocimiento y función.
- Crear **rutas de aprendizaje personalizadas** a partir del rol, la experiencia y el historial de aprendizaje.
- Producir materiales **interactivos** (quizzes, case studies, simulaciones) adaptados al estilo de aprendizaje con feedback inmediato.
- Habilitar **just-in-time (JIT) learning** / performance support: acceso rápido a la info para resolver el problema en el momento, reduciendo la formación formal.
- Identificar **subject matter experts (SMEs)** y conectar empleados con ellos → cultura de aprendizaje continuo y retención del conocimiento colectivo.

> [!tip] Usá **JIT learning** (performance support) para reducir la formación formal: la persona accede a la info justo cuando la necesita para resolver el problema.

El cierre del capítulo: un RAG exitoso requiere un **enfoque estratégico alineado con los objetivos de la empresa**. De ahí se pasa al code lab sobre cómo citar las fuentes.

## Code lab 3.1 – Añadir fuentes a tu RAG

Este lab **continúa el código del cap. [[02 - Code Lab - An Entire RAG Pipeline]]** para que la respuesta devuelva también la **fuente** de los documentos recuperados. Se introduce un import nuevo:

```python
from langchain_core.runnables import RunnableParallel
```

[[RunnableParallel]] ejecuta el retriever y la pregunta **en paralelo** (ganancia de performance: trae el contexto mientras procesa la pregunta). La cadena original del cap. 2 era:

```python
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
```

Ahora se parte en dos cadenas:

```python
rag_chain_from_docs = (
    RunnablePassthrough.assign(context=(
        lambda x: format_docs(x["context"])))
    | prompt
    | llm
    | StrOutputParser()
)
rag_chain_with_source = RunnableParallel(
    {"context": retriever,
     "question": RunnablePassthrough()}
).assign(answer=rag_chain_from_docs)
```

`rag_chain_from_docs` usa `RunnablePassthrough.assign()` para **formatear el contexto recuperado** y luego lo pasa por `prompt | llm | StrOutputParser()`. `rag_chain_with_source` usa `RunnableParallel()` para correr el `retriever` y `RunnablePassthrough()` en paralelo (asignando "context" y "question"), y luego `.assign(answer=rag_chain_from_docs)` calcula la respuesta. La diferencia clave: **separa el RETRIEVAL del contexto de su formateo/procesamiento**, dando más flexibilidad para manipular el contexto recuperado antes de llegar al prompt.

La invocación cambia a:

```python
rag_chain_with_source.invoke("What are the Advantages of using RAG?")
```

Y la salida es un **diccionario** con las claves `'context'` (lista de objetos `Document`) y `'answer'`:

```python
{'context': [Document(page_content='Can you imagine what you could do with all of the benefits mentioned above, but combined with all of your internal company data?...', metadata={'source': 'https://kbourne.github.io/chapter1.html'})],
 'answer': 'The advantages of using RAG (Retrieval Augmented Generation)...'}
```

> [!note] Fijate en el **`metadata={'source': ...}`** que sigue a cada `page_content`: ese es el **link de la fuente** que devolverías. Con varios documentos, cada uno podría tener una fuente distinta; en este ejemplo se usa un solo documento, así que hay una sola fuente.

> [!tip] Para RAG legal o científico, **citá las fuentes**: incluí la documentación de soporte y las citas para que la respuesta sea verificable y se reduzcan las alucinaciones.

## Citas

> "You have only paid interest on your balance twice during your tenure as a customer."

> "How much more is the interest rate on my credit card than my mortgage?"

## Para aplicar

- **Empezá los reportes automatizados donde la info oportuna importa más** — p.ej. market analysis (noticias, reportes financieros, competencia) para reaccionar rápido.
- **Exponé los datos subyacentes al personal no técnico** — que puedan hacer preguntas amplias sin esperar a un analista de datos.
- **Citá las fuentes en RAG legal/científico** — sumá documentación de soporte y citas; añadí llamadas extra al LLM para verificar relevancia antes de responder.
- **Usá JIT learning** — performance support en lugar de formación formal; identificá SMEs y conectá empleados con ellos.
- **Devolvé la fuente con `RunnableParallel`** — separá retrieval de formateo (`rag_chain_from_docs` + `rag_chain_with_source`) para exponer el `metadata['source']` de cada documento recuperado.

## Conexiones

- [[_Unlocking Data with Generative AI and RAG|Libro]] — el MOC del libro.
- [[01 - What Is Retrieval-Augmented Generation (RAG)]] — este capítulo expande las aplicaciones que el cap. 1 solo enumeró.
- [[02 - Code Lab - An Entire RAG Pipeline]] — el code lab 3.1 extiende este pipeline para devolver la fuente.
- [[04 - Components of a RAG System]] — capítulo siguiente: los componentes técnicos del sistema RAG (indexing, retrieval, generation) y cómo se integran.
- [[RAG]] · [[Chatbots]] · [[GenAI]] · [[Hallucinations]] · [[Recommendation Engine]] · [[Unstructured Data]] · [[Indexing]] · [[Knowledge Base]] — conceptos núcleo de las aplicaciones.
- [[RunnableParallel]] · [[LCEL]] · [[LangChain]] — la orquestación del code lab 3.1.
