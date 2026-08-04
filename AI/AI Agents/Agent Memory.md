---
title: Agent Memory
source: https://jamwithai.substack.com/p/agent-memory-the-7-types-you-should
author: Shantanu Ladhwe, Shirin Khosravi Jam
created: 2026-07-04
tags:
  - ai/agents
  - type/concept
  - status/permanent
aliases:
  - Agent Memory
  - Memoria de agentes
  - Agent Memory Types
  - 7 Memory Types
  - Tipos de memoria de agentes
reading:
  total_words: 2691
  read_words: 0
  pct: 0
  last_read: ""
updated: 2026-07-04
---

# Agent Memory

> [!note] Tesis operativa
> **Más memoria no es un mejor agente. Un mejor agente olvida a propósito.** La pregunta de diseño no es *"¿cómo le agrego memoria?"* sino **QUÉ** debe recordar el agente, por **CUÁNTO TIEMPO**, y **BAJO QUÉ CONDICIONES** eso debe volver. Amontonar todo en un único store lo hace más lento, más caro e inconsistente: preferencias viejas siguen activas, conversaciones irrelevantes resurgen, y una instrucción de una sola vez se vuelve una regla permanente.

## Marco mental (leé esto primero)

Un LLM **solo**, apenas termina la sesión, **no recuerda nada**. Toda "memoria" de un agente es infraestructura que vos construís alrededor del modelo. Antes de los 7 tipos, necesitás tres andamios sin los cuales lo demás no se entiende:

- **Las categorías vienen de la ciencia cognitiva** (semántica vs. episódica, declarativa vs. procedimental) y fueron formalizadas para agentes LLM en **CoALA: Cognitive Architectures for Language Agents** (Sumers, Yao, Narasimhan & Griffiths, 2023). El modelo mental de CoALA: **una** working memory de corto plazo + **varias** memorias de largo plazo **opcionales**. *Por qué importa nombrarlas*: nombrar cada slot te dice **cuál te falta** cuando el agente falla.
- **Distinción 1 — Stored vs Active.** Guardar algo es *potencial*, no uso. El modelo **no puede razonar sobre un byte hasta que ese byte vuelve al prompt actual**. La working memory es *lo que es usable ahora*; todo lo demás es almacenamiento esperando ser traído.
- **Distinción 2 — What vs Where.** Semántica / episódica / procedimental describen **QUÉ** se recuerda. Retrieval y parametric describen **DÓNDE** vive y **cómo vuelve**. Mezclar estos dos ejes es exactamente por qué *"che, agregale un vector DB"* fracasa como respuesta a todo.

Este es el mapa completo antes de bajar al detalle:

![[agent-memory-overview.jpg]]
*The seven memory types of AI agents*

> [!tip] Dos ejes, un sistema
> Leé la tabla de los 7 tipos con las dos preguntas en la cabeza: *¿esto es QUÉ o es DÓNDE?* y *¿está STORED o ACTIVE?* Casi todos los errores de diseño de memoria son confundir uno de estos dos ejes.

## La cadena causal — los 7 tipos

Cada tipo resuelve un problema que el anterior no cubre, y a la vez introduce su propio modo de fallar. Ese hilo es la nota: working se desborda → persistís hechos (semantic) → un hecho no basta, guardás el evento (episodic) → los eventos repetidos destilan un método (procedural) → todo crece más que el window, lo guardás afuera y traés lo relevante (retrieval) → pero el modelo ya trae conocimiento del training (parametric) → y todo eso mira el pasado; falta recordar el futuro (prospective).

## 1 · Working / In-Context Memory

> [!note] Qué es
> El LLM olvida **todo** cuando termina una respuesta. Para simular continuidad, la aplicación **re-envía en cada turno** todo el paquete que el modelo necesita ver: las instrucciones + los mensajes previos (o los últimos N) + los resultados de tools + los lookups. **Ese paquete ES la working memory** — literalmente lo único que el modelo ve. Todo lo demás (los otros 6 tipos) existe solo para *decidir qué entra en este paquete*.

![[agent-memory-working.jpg]]
*Working / in-context memory*

- *Por qué importa*: el modelo **solo usa lo que está en el paquete ahora**. No hay memoria "de fondo" a la que acceda mágicamente.
- **El ejemplo que lo aclara todo**: el usuario dice al principio que corre en **AWS** y **no usa Kubernetes**. Diez mensajes después el agente sigue sugiriendo **ECS** (no-K8s) en vez de **EKS** (K8s). No es que "se acordó" de la preferencia: es que el primer mensaje **sigue estando en el paquete re-enviado**. Sacá ese mensaje del paquete y la "memoria" desaparece. Eso te muestra que la working memory *no recuerda* — solo *contiene lo que está ahí ahora mismo*.

> [!warning] Dónde se rompe
> El límite es el **[[Context Window]]** (el máximo de texto que el modelo procesa de una vez, medido en **[[Tokens]]**). Una charla larga no entra → hay que elegir **qué conservar, qué acortar y qué tirar**. Y aunque el window fuera infinito: **más datos = más latencia y más costo** por turno, letal para tiempo real. Por eso la working memory no debe ser "todo lo que tenemos".

> [!tip] En la práctica — el checkpointer
> El **checkpointer** es un *save-file*: persiste la conversación + el estado después de cada paso, y lo recarga y re-envía en el siguiente → deja que el agente **pause y retome**. [[LangGraph]] ya trae uno listo para usar. Su segundo trabajo es **trimming/summarizing** a medida que se acerca al límite del window. El paper **MemGPT** (Packer et al., 2023) trata al window como la memoria de una computadora, **paginando** información dentro y fuera.

> [!example] La regla de oro de la working memory
> *"Working memory should hold the smallest amount the agent needs for its next good decision. Nothing more."*

## 2 · Semantic Memory

> [!note] Qué es
> **Hechos durables**, guardados por sí mismos, que **sobreviven a la conversación**: *"prefiere Python"*, *"corre en AWS"*, *"tiene plan enterprise"*. El hecho queda **separado del evento** que lo originó.

![[agent-memory-semantic.jpg]]
*Semantic memory*

- *Por qué importa*: sin ella el agente **arranca de cero en cada chat**; con ella simplemente **lee el hecho**. Sirve para personalización, perfiles de usuario, conocimiento de la organización y hechos estables de dominio.

> [!warning] Dónde se rompe
> El problema difícil es **decidir QUÉ guardar**: *"hoy sé breve"* (one-off) vs. *"siempre sé breve"* (preferencia durable) — no son lo mismo y confundirlos arruina el perfil. Guardar todo produce hechos **desactualizados y contradictorios**. Y **un hecho guardado que es incorrecto es peor que no tener ninguno**, porque el agente **confía** en él. Necesitás reglas explícitas de **add / update / delete**.

> [!tip] En la práctica
> Un store de hechos con un **bucket por usuario** + un paso (a menudo el propio modelo) que decide qué guardar. El **long-term memory store de [[LangGraph]]** es una versión lista para usar.

## 3 · Episodic Memory

> [!note] Qué es
> El registro de **eventos específicos**, guardados **enteros**. Si la semántica guarda el *hecho* (*"prefiere ECS"*), la episódica guarda la **historia**: el objetivo, los pasos, las tool calls, dónde salió mal y el resultado. Es el **diario** del agente.

![[agent-memory-episodic.jpg]]
*Episodic memory*

Un episodio guarda: el **objetivo** original; las **acciones**; las **tool calls + observaciones**; los **errores / enfoques fallidos**; el **feedback** del usuario; y el **resultado final**.

- *Por qué importa*: para **aprender de la experiencia** en vez de repetir errores. Un coding agent revisa *cómo* arregló un bug parecido; un support agent trae *cómo* se resolvió el mismo problema el mes pasado.

> [!warning] Dónde se rompe
> **Guardar cada run no es aprender.** Una pila de 10.000 runs es solo un **archivo muerto**: sirve únicamente si podés **encontrar** el correcto y **extraer la lección**. Por eso se combina con **reflection**: después de una tarea, el agente escribe un **resumen corto de qué funcionó y qué no**, guardado al lado del run. Lo que se reutiliza después es **ese resumen**, no el transcript entero.

> [!tip] En la práctica
> Un store de runs pasados (lo que tu logging/tracing **ya** recolecta) + reflection. El paper **Generative Agents** (Park et al., 2023) hace exactamente esto: los agentes registran eventos y **periódicamente los destilan** en conclusiones de más alto nivel. El flujo natural es **experiencia → comportamiento → procedimiento** (lo que nos lleva al tipo 4).

## 4 · Procedural Memory

> [!note] Qué es
> El **know-how**: el método, los pasos, las reglas, los hábitos. Ejemplos: *buscar en docs internos antes que en la web*; *validar el SQL generado antes de correrlo*; *que un humano apruebe antes de un reembolso*; *escalar temas de seguridad a una persona*.

![[agent-memory-procedural.jpg]]
*Procedural memory*

- *Por qué importa*: da **comportamiento consistente** run tras run, y es **cómo un agente mejora SIN reentrenar**.
- **El ejemplo del ciclo completo**: el agente nota a lo largo de varios runs que los deploys fallan por una **environment variable** olvidada → se fija una regla: *"siempre chequear las env vars antes de deployar"*. Un patrón detectado en la **episódica** se convirtió en un **procedimiento estable**. Ese es el eslabón que cierra la cadena experiencia → procedimiento.

> [!warning] Dónde se rompe
> Dos errores opuestos. **(a)** Reglas amontonadas en un único prompt gigante se **contradicen** entre sí y son **imposibles de actualizar con seguridad**. **(b)** Un run desafortunado se convierte en una regla permanente → un **mal hábito** grabado a fuego.

> [!tip] En la práctica
> Vive en las **instrucciones/políticas del prompt**, en las **descripciones de las tools**, en el **workflow** (secuencia fija de pasos) y en **"skills"** reutilizables. El trabajo real es **convertir las buenas lecciones que se acumulan en la episódica en procedimientos repetibles**.

## 5 · External / Retrieval Memory

> [!note] Qué es
> Acá cambia el eje: los 3 tipos anteriores eran **QUÉ** se recuerda; este es el **DÓNDE**. El **store** puede ser un **vector database** (encuentra texto por **significado**, no por palabras exactas), una DB común, documentos o archivos. El **retrieval** es el acto de **meter la mano en el store y traer lo relevante**.

![[agent-memory-retrieval.jpg]]
*External / retrieval memory*

- *Por qué importa*: los hechos, runs y procedimientos **crecen más allá del context window**. La solución es guardar todo en **almacenamiento externo** y traer **solo las pocas piezas que importan** → a la working memory. Ese *"traé solo lo que necesitás"* es el **corazón de [[_RAG|RAG]]**.

> [!warning] Dónde se rompe
> *"Agregale un vector DB"* es **el error clásico**. La búsqueda por significado es genial para **ideas relacionadas** pero **incorrecta para muchos lookups**. Hay que **casar el método con la necesidad**:
> - **significado** (vector search) → documentos relacionados por tema.
> - **lookup exacto en DB** → el plan de un usuario, un número de orden.
> - **filtros** → categorías/permisos, "solo la data de este equipo".
> - **time-based** → deadlines, schedules.
> - **seguir links** entre ítems relacionados.
>
> Para las estrategias combinadas ver [[Hybrid Search]].

> [!tip] En la práctica
> Una **mezcla de stores** + algo que decide **a cuál preguntarle**. Pase lo que pase, lo que traiga **entra primero a la working memory** — retrieval no es un atajo alrededor del window, es cómo lo llenás bien. (La semilla [[Vector Database]] es el store más citado, todavía sin nota propia.)

## 6 · Parametric Memory

> [!note] Qué es
> El conocimiento **horneado en el modelo durante el entrenamiento**. Es **la única memoria que NO construís ni gestionás** — viene con el modelo. Es *por qué* un modelo general escribe Python sin buscar nada. El nombre viene de los **parámetros**: los miles de millones de números (también llamados **weights**) fijados durante el training.

![[agent-memory-parametric.jpg]]
*Parametric memory*

- *Por qué importa*: es la **base de todo** — lenguaje, conocimiento general del mundo, razonamiento básico. **Todos los otros seis tipos se apoyan encima de esta capa.**

> [!warning] Dónde se rompe
> Está **CONGELADA en el momento del training** → puede estar **desactualizada**; **no puede** saber nada privado de tu empresa; puede sonar **segura y estar equivocada**; y **no podés meter la mano y corregir un hecho puntual**. El **[[Fine-tuning]]** (entrenar más sobre tus ejemplos) cambia **estilo y comportamiento**, pero es una **mala forma de guardar hechos que cambian**.

> [!tip] En la práctica
> Tratala como la **capa de habilidad general** y recurrí a **memoria externa** para todo lo específico, actual o privado. El modelo conoce las políticas de gastos *en general*; **el límite de reembolso real de tu empresa viene de una fuente confiable que vos controlás**, no de los weights.

## 7 · Prospective Memory

> [!note] Qué es
> Recordar **hacer algo MÁS TARDE** — la **lista de pendientes futura** del agente: seguir con el cliente el viernes; reintentar el job cuando se resetee el rate limit; re-chequear el deployment cuando lleguen los datos de monitoreo; recordarle al usuario **antes** de la renovación.

![[agent-memory-prospective.jpg]]
*Prospective memory*

- *Por qué importa*: le permite al agente **trabajar a lo largo de horas, días o semanas**, no solo dentro de un chat. Sin ella, cualquier cosa **inconclusa se cae por las grietas**.

> [!warning] Dónde se rompe
> Una tarea futura **no puede "esperar dentro de la conversación"**: el modelo **no está corriendo todo el tiempo**, y para cuando llegue el momento el chat puede ya no existir. El recordatorio **tiene que vivir AFUERA del modelo**, con una regla clara de **cuándo se dispara** y **cuándo se da por hecho** — así ni **molesta para siempre** ni **se evapora**.

> [!tip] En la práctica
> **Schedulers, task queues, timers (`cron`)** y sistemas confiables de **jobs demorados** como **Temporal** → ver [[Orchestrator]] (ahí ya viven Temporal/Prefect/Airflow; enlazá, no los redefinas). Un buen registro prospectivo guarda: **qué hacer**; **cuándo / qué lo dispara**; **la info para retomar**; **si ya está hecho**; y **quién aprueba antes de ejecutarlo**.

## Cómo se juntan los siete

> [!warning] Sección detrás del paywall
> El artículo cierra con un walkthrough concreto de *cómo los siete tipos operan juntos en un sistema real* (una sola request pasando por todos), pero **ese cuerpo está detrás del paywall** y no está disponible. No se reconstruye para no inventar. Lo que sí queda, como anticipo de esa integración, es la frase de cierre:

> [!example] La idea que une todo
> *"These seven are not separate modules. They are a system."*

## ¿Qué tipo necesito? (árbol de decisión)

- **El agente pierde el hilo dentro de una misma charla** → working memory (checkpointer + trimming). Ver §1.
- **Vuelve a preguntar cosas que ya sabe de vos entre sesiones** → semantic. Ver §2.
- **Repite el mismo error que ya cometió antes** → episodic + reflection. Ver §3.
- **Se comporta distinto en cada run / querés forzar un método** → procedural. Ver §4.
- **La info no entra en el window / crece sin parar** → external + retrieval (elegí el método por la necesidad, no default a vector DB). Ver §5.
- **Necesitás un hecho actual, privado o corregible** → NO parametric; usá external. Ver §6.
- **Hay algo que hacer en el futuro, fuera de esta charla** → prospective (scheduler afuera del modelo). Ver §7.

> [!question] 🎯 ¿Por qué "agregale un vector DB" no alcanza como estrategia de memoria?
> Porque confunde los dos ejes del marco mental. Un vector DB es una respuesta al eje **DÓNDE** (external/retrieval) y encima **solo** para el sub-método de *búsqueda por significado*. No dice nada sobre el eje **QUÉ** (qué hecho, qué episodio o qué procedimiento vale la pena guardar y con qué reglas de update/delete), ni resuelve lookups exactos, filtros, time-based o prospective. La memoria de un agente es un **sistema** de decisiones sobre *qué*, *por cuánto* y *bajo qué condición vuelve* — no un único store.

## Related

- [[Agent Harness]] — su responsabilidad #2 es **Memory & Context Management**: este tema es esa responsabilidad en detalle.
- [[Orchestrator]] — Temporal/Prefect/Airflow para la prospective memory (schedulers/jobs demorados).
- [[_RAG|RAG]] — el deep-dive de retrieval; acá la retrieval memory solo la enmarca como el eje DÓNDE.
- [[Hybrid Search]] — combinar estrategias de retrieval (significado + exacto + filtros).
- [[Context Window]] — el límite que restringe la working memory.
- [[Tokens]] — la unidad en que se mide el context window.
- [[Fine-tuning]] — modifica la parametric memory (estilo/comportamiento), no un buen store de hechos.
- [[LangGraph]] — trae checkpointer y long-term memory store listos.
- [[Vector Database]] — el store más citado para retrieval (semilla, aún sin nota).

## Referencias

- Fuente: [Agent Memory: the 7 types you should know](https://jamwithai.substack.com/p/agent-memory-the-7-types-you-should) — Shantanu Ladhwe y Shirin Khosravi Jam, 2026-07-01. La sección final "How the seven work together" está detrás del paywall y no se incluye.
- **CoALA: Cognitive Architectures for Language Agents** — Sumers, Yao, Narasimhan & Griffiths, 2023 · https://arxiv.org/abs/2309.02427
- **MemGPT: Towards LLMs as Operating Systems** — Packer et al., 2023 · https://arxiv.org/abs/2310.08560
- **Generative Agents: Interactive Simulacra of Human Behavior** — Park et al., 2023 · https://arxiv.org/abs/2304.03442
