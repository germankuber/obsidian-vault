---
title: 09 - Agent-Level Patterns
libro: Agentic Architectural Patterns for Building Multi-Agent Systems
autor: Ali Arsanjani
capitulo: 9
created: 2026-06-11
tags:
  - libros/agentic-architectural-patterns
  - type/case-study
  - status/permanent
aliases:
  - Agent-Level Patterns
updated: 2026-07-15
---

# 09 - Agent-Level Patterns

> [!info] Capítulo 9 · *Agentic Architectural Patterns for Building Multi-Agent Systems* — Ali Arsanjani
> Los cinco patrones internos de un agente individual —Single Agent Baseline, Agent-Specific Memory, RAG, Structured Reasoning & Self-Correction y Multimodal Sensory Input— ordenados como un Agent Maturity Model que se adopta capa por capa. Navegá: [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] · anterior [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]].

## Resumen

Tras los capítulos sobre cómo múltiples agentes colaboran dentro de una arquitectura, este hace zoom en la **unidad fundamental: el agente individual autónomo**. Define los patrones *agent-level* que dan forma a la arquitectura interna del agente y traen a la vida sus componentes: cómo percibe su entorno, cómo gestiona su memoria, cómo estructura su razonamiento y cómo actúa sobre el mundo. El enfoque es "map before the journey": primero un **Agent Maturity Model** estratégico, para entender dónde encaja cada patrón antes de verlo en detalle. La idea central de adopción es *progresiva y por capas*: para pasar de un nivel de madurez al siguiente, hay que implementar los patrones del nivel siguiente, lo que da un pathway claro de dónde estás a dónde querés llegar en sofisticación, capacidad, adopción y madurez de IA agéntica empresarial.

Los patrones mapean los componentes anatómicos del agente (Sense, Reason, Plan, Act, Recall/Memory — los "órganos" definidos en el cap. 1 y el cap. 4) a soluciones concretas de ingeniería, pasando del *qué es* un agente al *cómo* se ingenia cada facultad para confiabilidad y escala. No son features aisladas sino componentes integrados de la arquitectura interna, que potencian el loop cognitivo **Perceive → Reason → Act** organizado en cuatro capas: capa de percepción (Multimodal Sensory Input), capa de memoria y conocimiento (Agent-Specific Memory + RAG retriever — recurso crítico para la cognición, fuera del loop core), capa de cognición/razonamiento (Structured Reasoning & Self-Correction, el "cerebro" con LLM) y capa de acción (Single Agent Baseline, uso de tools/APIs).

El capítulo recorre los cinco patrones en orden de madurez, cada uno motivando al siguiente (baseline stateless → necesita memoria; memoria general → necesita conocimiento factual externo con RAG; RAG da info pero el agente necesita un proceso de pensamiento robusto → Structured Reasoning; el razonamiento es sobre texto pero el mundo real es visual → Multimodal). Cierra con una guía táctica de rollout enterprise en 3 fases (responde al *cuándo*) y una tabla de métricas de evaluación por patrón para cuantificar el valor de cada decisión arquitectónica.

## Mapeo anatomía → patrón (Tabla 9.1)

![[09-fig-9.1.png|525]]
*Figura 9.1 – The agent anatomy*

Patrones que describen las interacciones internas entre los componentes de un solo agente:

| Componente | Capacidad | Patrón que la implementa | Cómo |
|---|---|---|---|
| Act and tools | Ejecuta comandos directos y usa tools simples | **Single Agent Baseline** | El agente funciona como automatizador básico: usa el bloque Act para ejecutar workflows y el bloque Tools para interfacear con APIs externas. |
| Memory | Maneja conversaciones multi-turn y recuerda hechos clave | **Agent-Specific Memory** | El agente se vuelve *stateful*: el componente Memory persiste historial de sesión y preferencias del usuario para mantener contexto. |
| Memory | Accede y razona sobre datos externos de dominio | **Context-Aware Retrieval (RAG)** | El agente queda *grounded* en conocimiento factual: el componente Memory se aumenta para recuperar datos externos, reduciendo alucinaciones. |
| Reason and plan | Razona de forma transparente y corrige sus propios errores | **Structured Reasoning and Self-Correction** | El agente se vuelve confiable: usa Reason para pensar los problemas y Plan para estructurar tareas complejas. |
| Sense | Entiende y procesa info visual de documentos y UIs | **Multimodal Sensory Input** | El agente funciona como gateway al mundo: el componente Sense ingiere inputs multimodales (imágenes, audio) antes de procesar. |

Las cuatro capas de la arquitectura interna (loop Perceive → Reason → Act):
- **Capa de percepción** — gateway al mundo; procesa inputs crudos y los prepara para el motor de razonamiento. Patrón: *Multimodal Sensory Input* (texto + imágenes + sonido + otros formatos).
- **Capa de memoria y conocimiento** — provee el contexto para razonar; NO es parte del loop core, sino recurso crítico para la cognición. Patrones: *Agent-Specific Memory* (historial de sesión + preferencias) y *RAG retriever* (conocimiento externo on-demand).
- **Capa de cognición y razonamiento** — el "cerebro", powered by LLM; planifica, decide, formula respuestas. Patrones: *Structured Reasoning and Self-Correction* (proceso de pensamiento robusto, transparente, alineado con goals).
- **Capa de acción** — ejecuta las decisiones de la cognición; donde el agente interactúa con el entorno. Patrón: *Single Agent Baseline* (uso de tools/APIs/funciones).

## Single Agent Baseline

- **Qué es** — la forma más simple de sistema agéntico: un único agente maneja un workflow completo accediendo a una variedad de tools. Punto de partida de la mayoría de las implementaciones y benchmark contra el cual medir arquitecturas más avanzadas.
- **Contexto** — ideal para tareas complejas que requieren uso de tools pero NO justifican el overhead de múltiples agentes colaborando. El starting point más común para sistemas agénticos task-oriented.
- **Problema** — ¿cómo estructurar un agente básico pero funcional que ejecute una serie de acciones semi-autónomas para lograr un goal?
- **Solución** — un único agente con LLM, un set de tools (funciones/APIs) y un goal en el instruction prompt. Usando un framework de razonamiento como **ReAct (Reason-Act)** o el más sofisticado **Fractal Chain of Thought (FCoT)**, el núcleo LLM decide independientemente qué tools llamar, en qué secuencia y con qué parámetros. Todo el razonamiento (thought process) y la ejecución (planning) viven dentro de este único agente.
- **Ejemplo (loan)** — `SingleLoanAgent`; goal: aprobar/denegar según umbral de credit score de **680**; tool `get_credit_score`. Flujo: (1) recibe task `applicant_id: 'john_doe_123', loan_amount: 50000` → (2) el LLM razona que necesita el credit score → (3) selecciona y ejecuta `get_credit_score(applicant_id='john_doe_123')` → (4) observación: devuelve **720** → (5) razonamiento final: 720 ≥ 680 → (6) response: **Approved**. (En el código, la decisión "real" sería una segunda llamada LLM; el ejemplo la simula.)
- **Consecuencias** — *Pro*: **simplicidad** de implementación y debug; más fácil gestionar y observar el estado de un solo agente que de un multi-agent. *Con*: **no escala** cuando crecen las tools o la complejidad del dominio; el agente se sobrecarga, el prompt se vuelve inmanejable, degrada performance y sube la probabilidad de error.
- **Guía** — provee un loop de ejecución completo y self-contained, pero es **stateless**: no tiene memoria de interacciones pasadas y trata cada task como la primera. Para experiencias inteligentes y personalizadas hace falta memoria → siguiente patrón.

![[09-fig-9.2.png|455]]
*Figura 9.2 – The Single Agent Baseline pattern*

![[09-fig-9.3.png|250]]
*Figura 9.3 – Single Agent loan agent workflow*

## Agent-Specific Context and Memory

- **Qué es** — el agente mantiene y utiliza su propia información contextual dedicada para guiar la decisión. Incluye sus instrucciones/goals iniciales y su **chain of orchestration**: qué instrucciones le pasó su orquestador padre (si tiene); si no, se basa en las instrucciones dadas directamente. Eleva al agente de ejecutor de comandos a entidad *stateful* que puede aprender y adaptarse.
- **Contexto** — crucial para cualquier agente que vaya más allá de comandos one-shot: agentes conversacionales, automatización de tareas long-running, y cualquier sistema donde eventos pasados influyen en acciones futuras.
- **Problema** — ¿cómo mantiene el agente una comprensión coherente de su tarea y entorno a lo largo del tiempo, especialmente en interacciones multi-turn?
- **Solución** — equipar a cada agente con su propio sistema de memoria/estado, separado de las instrucciones inmediatas. Dos tipos:
  - **Memoria de corto plazo (state management)** — gestionada dentro de la ventana de contexto del LLM; guarda el historial inmediato de la tarea/conversación. Técnicas: **summarization** o **sliding window** de los últimos N mensajes, para mantener el contexto relevante y conciso.
  - **Memoria de largo plazo** — almacenamiento persistente (vector database o key-value store); guarda y recupera hechos clave, preferencias del usuario y conclusiones pasadas para sesiones futuras. **RAG** es el patrón común para acceder a esta memoria.
- **Nota — "lost in the middle"** — meter todo el historial de conversación al contexto del LLM suele ser **contraproducente**: la investigación muestra que los modelos olvidan información perdida "en el medio" de un contexto largo y se distraen con ruido irrelevante. La gestión efectiva de memoria es esencial.
- **Ejemplo (conversational loan agent)** — Turno 1: usuario "I need to apply for a home loan" → agente "What is the applicant's ID?" y actualiza summary a *User is asking about a home loan*. Turno 2: usuario "My ID is 'jane_doe_456'" → el agente combina el summary con el mensaje nuevo, entiende que es el applicant ID del home loan, responde "What is the loan amount for Jane Doe?" y actualiza memoria. Gracias a la memoria evita preguntas redundantes y mantiene flujo lógico. (El código de ejemplo usa el enfoque de **summarization** del último intercambio.)
- **Consecuencias** — *Pro*: comportamiento **stateful** y personalización; aprende de la experiencia, recuerda preferencias, adapta acciones. *Con*: complejidad y **contextual drift** — memoria mal gestionada hace recordar info desactualizada/irrelevante y degrada performance.
- **Guía** — corto plazo: empezar simple con sliding window de las últimas N vueltas o summarization. Largo plazo: RAG sobre vector database (enfoque estándar). Ser **deliberado** sobre qué se guarda en largo plazo: enfocarse en data de alto valor (perfiles de usuario, decisiones clave, hechos finalizados) para no contaminar la memoria con ruido.

![[09-fig-9.4.png|444]]
*Figura 9.4 – Agent-specific memory*

## Sensing with RAG

- **Qué es** — mientras la memoria general da contexto conversacional, RAG provee el mecanismo específico para *grounding* del razonamiento en conocimiento externo y factual. Uno de los patrones más efectivos y adoptados para mejorar performance, reducir alucinaciones y conectar el agente a info propietaria o en tiempo real.
- **Contexto** — esencial cuando las tareas requieren precisión factual más allá del conocimiento estático y pre-entrenado del LLM: customer support, research, análisis legal, y cualquier dominio donde las decisiones deben basarse en un cuerpo específico de documentos.
- **Problema** — los LLMs se pre-entrenan en datasets vastos pero estáticos; su conocimiento puede estar desactualizado o carecer de contexto enterprise. ¿Cómo accede un agente a info actualizada/propietaria/de dominio sin reentrenamiento costoso?
- **Solución — pipeline RAG**:
  1. **Indexing** — documentos enterprise se limpian, se dividen en chunks manejables y se convierten en vector embeddings, almacenados en una vector database.
  2. **Retrieval** — ante una query, el sistema recupera los chunks más relevantes de la vector DB por similitud semántica.
  3. **Augmentation** — los chunks recuperados se insertan en el prompt del agente como contexto adicional.
  4. **Generation** — el LLM usa ese contexto aumentado para generar una respuesta *grounded* en la info provista.
- **Tip — Simple vs Agentic RAG** — *simple RAG* es un workflow retrieve-and-generate directo; *agentic RAG* introduce agentes especializados que hacen tareas como reformulación de query o retrieval iterativo para mejorar resultados.
- **Ejemplo (RAG loan)** — préstamo de **$750.000**; el agente, antes de decidir, consulta su tool `RAGRetriever` con la query *policy for high-value loan*; el retriever hace búsqueda semántica y encuentra **Policy #23B: Loans over $500,000 require a credit score of 740 and a manual review**. Augmenta el prompt con esa política; razona que el score de **720 < 740**; responde grounded: *Denied due to insufficient credit score as per Policy #23B for high-value loans*. (El código simula un knowledge base con `high_value_loan` = Policy #23B y `standard_loan` = Policy #17A, umbral 680.)
- **Consecuencias** — *Pros*: **accuracy y trust** (reduce drásticamente alucinaciones, genera confianza), **knowledge freshness** (actualizar la fuente, sin reentrenar el LLM). *Cons*: **dependencia de la calidad del retrieval** (si trae contexto irrelevante, confunde al LLM y degrada la respuesta), y **contextual drift / RAG drift** (el conocimiento indexado se vuelve stale/desactualizado y decae la performance).
- **Guía** — la calidad del retrieval es lo primordial: invertir en un pipeline robusto de procesamiento de documentos que limpie y chunkee bien; elegir embedding model y vector DB según el dominio. Para queries complejas, **agentic RAG**: un agente dedicado refina la query inicial, hace búsquedas iterativas y sintetiza resultados de múltiples documentos para dar al LLM el mejor contexto posible.

![[09-fig-9.5.png|328]]
*Figura 9.5 – Context-aware retrieval (RAG) agent*

## Structured Reasoning and Self-Correction

- **Qué es** — familia de patrones que estructura el proceso de pensamiento interno del agente para mejorar la calidad del razonamiento y habilitar autocorrección, más allá de decisiones de un solo paso.
- **Contexto** — se usa cuando la confiabilidad y explicabilidad de las conclusiones son críticas; especialmente valioso en tareas multi-step complejas con alto riesgo de malinterpretar instrucciones o no considerar todas las restricciones.
- **Problema** — ¿cómo asegurar que el razonamiento del agente sea lógico, transparente y alineado con sus instrucciones core, sobre todo en tareas complejas multi-step?
- **Solución — técnicas componibles** (a menudo combinadas):
  - **Persistent Instruction Anchoring** *(del Capítulo 6)* — instrucciones/goals clave embebidos en el prompt con tags distintivos (ej. `#OBJECTIVE:`), repetidos o colocados al **inicio y al final** del contexto para contrarrestar el problema "lost in the middle".
  - **Instruction Fidelity Auditing (Self-Correction)** *(del Capítulo 6)* — el agente hace un self-review de su output antes de finalizarlo: primero genera una respuesta preliminar, luego corre un paso de **"critique"** donde evalúa su propio output contra las instrucciones y el contexto originales.
  - **Chain-of-Thought (CoT) y variantes "vanilla"** — en vez de pedir respuesta inmediata, se prompt-ea al agente a "think step by step", forzándolo a externalizar su razonamiento (más preciso + audit trail claro). Variantes: **Graph of Thought**, **Tree of Thought**, etc.
  - **Fractal Chain-of-Thought (FCoT)** — usando una **Context Aperture** cada vez más detallada, se hace zoom en niveles de granularidad: **macro → meso → micro**. En cada nivel se definen y aplican **dual objective functions** (una que maximiza y otra que minimiza el factor hacia el que se hace *hill-climbing*). En cada iteración/loop se auto-reflexiona y corrige lo que se perdió en el loop de razonamiento anterior.
- **Ejemplo (self-correcting loan agent)** — contexto recuperado: *Policy #23B: Loans over $500,000 require a credit score of 740 and a manual review*. (1) **Razonamiento inicial (CoT)**: aplicación de $750.000, score 720 → "Step 1: loan high-value. Step 2: Policy #23B aplica. Step 3: 720 < 740. Preliminary Decision: Denied". (2) **Self-critique**: el auditor interno revisa contra el contexto buscando detalles omitidos. (3) **Corrección identificada**: "la decisión de denegar es correcta pero el razonamiento es incompleto: la Policy #23B también menciona elegibilidad para *manual review*, debe incluirse". (4) **Output final corregido**: *FINAL DECISION: Denied for automatic approval based on Policy #23B. However, the application is eligible for a manual review.*
- **Consecuencias** — *Pro*: **fiabilidad y transparencia**; el pensamiento externalizado facilita el debug a developers y la comprensión a auditores de *por qué* se tomó una decisión. *Con*: **latencia y costo** por los pasos extra; un self-correction loop requiere ≥2 llamadas LLM separadas en vez de una.
- **Guía** — componer los patrones para máximo efecto: Persistent Instruction Anchoring en todos los prompts complejos para mantener al agente on-task; Chain-of-Thought para el pase inicial (análisis step-by-step transparente); y para decisiones high-stakes, un **Self-Correction loop** donde un segundo prompt pide explícitamente al LLM jugar el rol de crítico/auditor revisando el CoT inicial contra el contexto y las restricciones originales.

![[09-fig-9.6.png|404]]
*Figura 9.6 – A self-correction loop, where an agent generates a preliminary output and then critiques it against its goals, leading to a more reliable final result*

## Multimodal Sensory Input

- **Qué es** — expande la percepción del agente más allá del lenguaje, permitiéndole interpretar imágenes, screenshots y datos visuales como parte de su razonamiento. Mueve al agente de simple chatbot a verdadero *co-pilot* que entiende su entorno visualmente.
- **Contexto** — esencial en industrias document-heavy: finanzas (procesar facturas), salud (analizar formularios médicos), logística (leer etiquetas de envío). Relevante cuando se requiere interpretar screenshots, invoices o imágenes de productos como parte del workflow.
- **Problema** — ¿cómo opera un agente eficazmente en workflows que dependen de documentos visuales y UIs?
- **Solución — integrar capacidades multimodales en el componente de percepción** (procesar imágenes, extraer texto vía **OCR**, entender layouts espaciales). Dos enfoques, trade-off entre control y simplicidad:
  - **Pipeline de tools especializados** — serie de modelos/servicios single-purpose: el agente manda la imagen a un tool dedicado (ej. servicio OCR) para extraer texto, que luego pasa a un LLM separado para razonar. Mayor control sobre cada paso.
  - **Modelo nativo multimodal** — un único foundation model inherentemente multimodal (ej. familia **Gemini** de Google), entrenado para procesar imágenes, texto, audio y video simultáneamente. El agente alimenta imagen + prompt de texto directo al modelo, que maneja el parsing visual y el razonamiento en un solo paso.
- **Protocolos para obtener datos del entorno** (no solo cómo se procesan, sino cómo se obtienen segura y confiablemente, crítico al sensar también el mundo físico):
  - **[[MCP]] (Model Context Protocol)** — forma estandarizada y segura de que un agente descubra e interactúe con tools *dentro* de su organización. Building block para desarrollar agentes individuales; permite obtener recursos (ej. de un dispositivo IoT) o info contextual como input para el razonamiento.
  - **[[A2A]] (Agent-to-Agent protocol)** — para comunicación entre agentes, a menudo a través de **fronteras organizacionales**. Building block para workflows complejos u orquestadores multi-agente; permite que agentes de distintas compañías se comuniquen seguramente para un goal compartido, manejando authorization y authentication (ej. **OAuth**).
- **Ejemplo (loan desde imagen)** — *Pipeline*: percepción (recibe imagen del formulario) → tool OCR (`ocr_service`) → observación (texto extraído: *Applicant: John Doe, Loan Amount: $600,000*) → razonamiento (pasa el texto al LLM core). *Nativo*: percepción (imagen + prompt "Analyze this loan application form image and provide a decision") → razonamiento (manda imagen + texto a un LLM nativo multimodal que hace OCR y decisión en un solo paso).

![[09-fig-9.7.png|520]]
*Figura 9.7 – Multimodal processing via multimodal pipeline*

![[09-fig-9.8.png|319]]
*Figura 9.8 – Native multimodal LLM multimodal processing*
- **Consecuencias** — *Pro*: **expande los use cases** drásticamente; de tool basada en lenguaje a asistente que interactúa con las mismas interfaces que un humano (ej. reduce tiempo de procesamiento de claims interpretando imágenes en workflows de aprobación). *Con*: **costo y latencia**; los modelos multimodales son computacionalmente más caros, requieren infra especializada (GPUs) y un enfoque distinto de testing/validación que los text-only.
- **Guía** — elegir según necesidad: el **pipeline** suele ser buen punto de partida (usar servicios best-in-class como un OCR muy preciso, más control y observabilidad por paso); el **modelo nativo multimodal** es más simple de orquestar y más potente para tareas que requieren comprensión holística profunda de imagen + texto juntos, pero ofrece menos control granular.

## Guía de rollout enterprise (Tabla 9.2)

Implementación por fases, alineada con el Maturity Model, para construir una base de confiabilidad y agregar capacidades a medida que el sistema escala:

| Fase | Patrones a implementar | Racional |
|---|---|---|
| **Fase 1: Automatización fundacional** | Single Agent Baseline, Agent-Specific Memory | Empezar con un agente simple y transaccional para una tarea high-volume / low-risk (ej. bot de IT helpdesk interno). Prueba valor inmediato y construye la infraestructura core. |
| **Fase 2: Construir expertise** | Context-Aware Retrieval (RAG) | Mejorar el agente conectándolo a una knowledge base (ej. documentación técnica). Lo vuelve una fuente confiable de información. |
| **Fase 3: Autonomía high-trust** | Structured Reasoning, Multimodal Input | Para procesos críticos: agregar self-correction (confiabilidad) y capacidades de visión (datos visuales reales). Habilita workflows high-stakes, ej. un agente de finanzas que procesa facturas escaneadas y se autoaudita antes de aprobar. |

## Métricas de evaluación por patrón (Tabla 9.3)

| Patrón | Métrica | Instrumentación |
|---|---|---|
| Single Agent Baseline | Task completion rate / tool call success rate | Loguear el outcome final de cada task (éxito/fallo). Monitorear logs por llamadas a tool API fallidas o erróneas. |
| Agent-Specific Memory | Session coherence score / reducción de preguntas repetidas | Raters humanos puntúan la calidad de conversación. Trackear cuántas veces el usuario repite info que el agente debería haber recordado. |
| Context-Aware Retrieval (RAG) | Hallucination rate / factual accuracy score | Evaluar respuestas contra un **golden dataset** para medir frecuencia de info fabricada. |
| Structured Reasoning | Self-correction trigger rate / reducción de errores finales | Trackear cuán seguido el paso de crítica identifica una falla. Comparar tasa de error de outputs preliminares vs finales. |
| Multimodal Sensory Input | Data extraction accuracy / éxito en tareas visuales | Medir accuracy de OCR o extracción de campos contra ground-truth. Trackear task completion en workflows que requieren input de imagen. |

## Citas

> "Agents are built, not summoned"
> "Start simple, scale intelligently"
> "Simply feeding an entire conversation history into an LLM's context window is often counterproductive."
> "models tend to forget information \"lost in the middle\" of a long context and can be distracted by irrelevant noise."

## Para aplicar

- **Adoptar los patrones por capas, no todos de golpe** — empezar por el Single Agent Baseline y agregar Memory → RAG → Structured Reasoning → Multimodal a medida que crece la responsabilidad del agente. Para subir un nivel de madurez, implementar los patrones del nivel siguiente.
- **Gestionar la memoria deliberadamente** — corto plazo: sliding window de las últimas N vueltas o summarization; largo plazo: RAG sobre vector DB guardando solo data de alto valor (perfiles, decisiones clave, hechos finalizados) para no contaminar con ruido. Evitar volcar todo el historial (lost in the middle).
- **Invertir en la calidad del retrieval** — pipeline robusto de limpieza y [[Chunking Strategies|chunking]]; elegir embedding model y vector DB según el dominio; para queries complejas, *agentic RAG* (reformulación de query + retrieval iterativo + síntesis multi-documento).
- **Componer las técnicas de razonamiento** — Persistent Instruction Anchoring en todos los prompts complejos + Chain-of-Thought para el pase inicial + un loop de Self-Correction (segundo prompt como auditor/crítica) para decisiones high-stakes.
- **Elegir el enfoque multimodal según necesidad** — pipeline de tools especializados como punto de partida (control, observabilidad, OCR best-in-class) vs modelo nativo multimodal para comprensión holística imagen+texto.
- **Seguir la secuencia de rollout en 3 fases** — fundacional (tarea low-risk high-volume) → expertise (RAG) → high-trust (reasoning + multimodal), construyendo confianza por capas.
- **Instrumentar métricas por patrón desde el día uno** — para justificar cada decisión arquitectónica y guiar la mejora continua (ver Tabla 9.3).

## Conexiones

- [[_Agentic Architectural Patterns for Building Multi-Agent Systems|Mapa del libro]] — el MOC del libro.
- [[01 - GenAI in the Enterprise - Landscape, Maturity, and Agent Focus]] — cap. 1: la anatomía del agente (sense→reason→plan→act) que este capítulo operacionaliza en patrones.
- [[06 - Explainability and Compliance Agentic Patterns]] — cap. 6 (Persistent Instruction Anchoring e Instruction Fidelity Auditing provienen de ahí).
- [[Generator-Evaluator Pattern]] — el patrón generador-evaluador del vault; calca el loop de Self-Correction (genera → critica → corrige).
- [[Function Calling]] · [[Tool Calling]] · [[Orchestrator]] — mecanismo de acción del Single Agent Baseline (ReAct, tool use).
- [[_RAG|RAG]] · [[Chunking Strategies]] · [[Hybrid Search]] · [[Reranking]] · [[Enterprise RAG Assistant]] — el patrón Sensing with RAG y su pipeline.
- [[Hallucinations]] · [[Grounding]] — lo que RAG y el grounding mitigan.
- [[Ground Truth]] · [[LLM as Judge]] · [[Evals]] — métricas de evaluación por patrón (golden dataset, factual accuracy).
- [[MCP]] · [[A2A]] — protocolos reintroducidos en Multimodal Sensory Input (conceptos candidatos a nota propia).
- [[LLM]] — núcleo de cognición del agente (candidato a nota propia).
