---
title: Implementing the ReAct Pattern Over RabbitMQ
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 9
created: 2026-06-22
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/case-study
  - status/permanent
aliases:
  - Implementing the ReAct Pattern Over RabbitMQ
  - Cap 9 - Implementing the ReAct Pattern Over RabbitMQ
updated: 2026-06-22
---
# Implementing the ReAct Pattern Over RabbitMQ

> [!info] Capítulo 9 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> El capítulo **baja a código** el método del [[08 - Pattern-Guided Coding|cap. 8]]: implementa el **[[ReAct]]** (Reasoning + Acting) como un sistema real corriendo sobre un broker **[[RabbitMQ]]** vivo. El agente razona y actúa en un loop **Thought → Action → Observation**, despacha tools a workers desacoplados por colas, hila cada paso con un **[[Correlation Identifier|correlation_id]]**, usa **manual ACK/NACK** y la **[[Dead-Letter Queue|DLQ]] de 3 tiers** (retry1 30s → retry2 5min → quarantine) diseñada en el cap. 8. Tratá al LLM como el **endpoint poco fiable** del [[02 - Embeddings The Language of AI|cap. 2]]: cada respuesta que no parsea es un *poison message*. Cuatro componentes (`react_agent.py`, `tool_worker.py`, `publish_command.py`, `topology.json`), un ejemplo end-to-end (Torre Eiffel → 1.083 pies) y 8 mejoras para producción. Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[08 - Pattern-Guided Coding]] · siguiente [[10 - The Future and Limitations of LLMs]].

## Resumen

El capítulo es la **contraparte práctica del [[08 - Pattern-Guided Coding|cap. 8]]**: si el cap. 8 definió **[[Topologos]]** y el método para traducir cualquier GenAI workflow pattern a una topología [[RabbitMQ]] production-ready, el cap. 9 **toma uno de esos patrones — [[ReAct]] — y lo construye de punta a punta** sobre un broker real, con código completo y corriendo. ReAct (Reasoning + Acting, del paper de Yao et al. 2022) entrelaza pensamiento y acción en un **loop iterativo Thought → Action → Observation**: el agente piensa qué hacer, ejecuta una **tool**, observa el resultado y repite hasta tener la respuesta final. El reto de ingeniería no es el prompt sino **hacer ese loop fiable** cuando el LLM es un endpoint impredecible y las tools fallan.

La solución reparte el sistema en **cuatro componentes desacoplados por colas**: un **`react_agent.py`** (el cerebro: corre el loop, llama al LLM, despacha tools, colecta resultados), tres **tool workers** (`tool_worker.py` con `search`, `calculator`, `weather`), un **`publish_command.py`** (CLI que inyecta preguntas) y un **`topology.json`** (la infraestructura como código: vhost, exchanges, queues, bindings y DLQ). Entre ellos no hay dependencia de red directa: **el broker es el sistema**. Cada mensaje viaja con un **[[Correlation Identifier|correlation_id]]** que hila todo el episodio ReAct — aparece en cada log line, encarnando el **Correlation Identifier** de los EIP.

El capítulo desarrolla cómo entender ReAct, **montar la topología** (Table 9.1: vhost `/react`, 6 exchanges, 6+ queues, DLQ de 3 tiers con `x-message-ttl` + `x-dead-letter-exchange`), **construir el agente** (message schemas como dataclasses, `_think` que fuerza JSON estricto del LLM, `_dispatch_tool` que publica y pollea con `basic_get` filtrando por correlation_id, y la **clasificación de fallas** de la Table 9.2 en poison/transient/permanent/unknown — todas con `basic_nack(requeue=False)` hacia la cadena de DLQs), **implementar los tool workers** (la distinción crítica entre un *tool error* — `ValueError`, que el agente razona — y un *system failure* — excepción inesperada, que va a DLQ), **enviar comandos**, **correr end-to-end** (el ejemplo de la Torre Eiffel: search → 330 metres → calculator → 1082.6772 → "approximately 1,083 feet tall"), **observar la DLQ** (parar un worker y ver el mensaje fluir retry1 → retry2 → quarantine en la Management UI, donde quarantine es un *holding area* que requiere replay manual del operador) y **preparar para producción** (8 mejoras: reemplazar stubs, per-correlation reply queues, persistir respuestas, publisher confirms, headers explícitos, TLS/mTLS, monitorear quarantine, parametrizar el modelo). La **Table 9.3** resume las 7 decisiones de diseño con su *why*. El **capítulo siguiente** extiende ReAct a **Plan-and-Execute** (un orchestrator que descompone en subtasks paralelas con **[[Scatter-Gather]]**).

## Understanding the ReAct pattern

El capítulo abre explicando **qué es [[ReAct]]** antes de implementarlo. ReAct — **Reasoning + Acting** — proviene del paper de **Yao et al. (2022)** y es uno de los GenAI workflow patterns mapeados en la **Table 8.1** del [[08 - Pattern-Guided Coding|cap. 8]]. La idea es **entrelazar razonamiento y acción** en un único loop, en vez de que el LLM razone todo de una sola vez (chain-of-thought puro) o actúe sin pensar.

![[B34134_9_1.png]]
*Figure 9.1 – The ReAct loop showing the Thought → Action → Observation cycle*

El loop tiene tres fases que se repiten hasta llegar a la respuesta:

> [!note] **El ciclo Thought → Action → Observation**
> - **Thought (Reasoning)** — el LLM piensa: dado lo que sabe hasta ahora, decide cuál es el próximo paso. Puede ser usar una tool o dar la respuesta final.
> - **Action (Acting)** — si decide actuar, el agente ejecuta una **tool** (search, calculator, weather…) con los argumentos que el LLM eligió.
> - **Observation** — el resultado de la tool se devuelve al LLM como una **observación**, que se incorpora al contexto y alimenta el siguiente Thought.

El loop continúa — pensar, actuar, observar, pensar de nuevo — hasta que el LLM concluye que tiene suficiente información y emite una **respuesta final** (`final_answer`) en vez de otra acción. Lo que hace a ReAct potente es que el LLM **no necesita saberlo todo de antemano**: descubre lo que necesita sobre la marcha, usando tools para obtener datos frescos o cómputos que no puede hacer mentalmente.

> [!warning] El desafío de ingeniería no es lograr que el LLM "piense en voz alta", sino hacer ese loop **production-ready**: el LLM puede devolver JSON malformado, una tool puede timeoutear o crashear, el loop puede no converger nunca. Por eso ReAct sobre [[RabbitMQ]] usa correlation_id threading, manual ACK/NACK, un **step limit** y la DLQ de 3 tiers — cada mecanismo existe porque un modo de falla concreto lo exige.

## Building the RabbitMQ topology

Antes de escribir un solo agente, se declara la **infraestructura como código** en un único `topology.json` que se carga al broker vía la Management API. La topología vive en un **vhost dedicado `/react`** con su usuario y permisos, **seis exchanges** y **seis o más queues**, incluyendo la **cadena de DLQ de 3 tiers** que diseñó el [[08 - Pattern-Guided Coding|cap. 8]]. La filosofía de diseño: ningún elemento es boilerplate; cada uno existe porque un modo de falla específico lo requiere.

> [!note] **El broker es el sistema.** Agente y workers **no se conocen entre sí** ni se hablan por red directa: publican y consumen mensajes de colas. Esto es **loose coupling** llevado al extremo — podés escalar, reiniciar o reemplazar cualquier componente sin tocar los demás.

La DLQ de 3 tiers funciona con **TTL encadenado**: una cola con `x-message-ttl` y `x-dead-letter-exchange` re-rutea el mensaje al siguiente tier cuando vence su tiempo. Así las colas `*.retry1.queue` retienen 30 segundos y luego dead-letterean a `react.dlq.retry2.exchange`, las `*.retry2.queue` retienen 5 minutos y luego caen a `react.dlq.quarantine.exchange` (un **fanout** cuya `react.quarantine.queue` es el holding area final). El **`alternate-exchange`** captura mensajes no ruteables.

### Tabla 9.1 — Elementos de la topología RabbitMQ (vhost `/react`)

Los exchanges y queues que componen el sistema ReAct, con su tipo y configuración:

| Element | Name | Type / Config |
|---|---|---|
| vhost | `/react` | Aísla el ejemplo de otros workloads del broker |
| exchange | `react.commands.exchange` | topic — recibe preguntas de los publishers |
| exchange | `react.tool.dispatch.exchange` | topic — el agente publica aquí los tool requests |
| exchange | `react.tool.results.exchange` | topic — los tool workers publican aquí los resultados |
| exchange | `react.dlq.retry1.exchange` | topic — Tier 1 retry (colas con TTL de 30 s) |
| exchange | `react.dlq.retry2.exchange` | topic — Tier 2 retry (colas con TTL de 5 min) |
| exchange | `react.dlq.quarantine.exchange` | fanout — DLQ terminal, replay manual solamente |
| queue | `react.commands.queue` | durable, DLX → retry1 |
| queue | `react.tool.search.queue` | durable, DLX → retry1 |
| queue | `react.tool.calculator.queue` | durable, DLX → retry1 |
| queue | `react.tool.weather.queue` | durable, DLX → retry1 |
| queue | `react.tool.results.queue` | durable, DLX → retry1, TTL 5 min |
| queue | `react.quarantine.queue` | durable, sin TTL, fanout-bound |

> [!note] El `topology.json` declara además las **retry queues intermedias** por cada cola de trabajo — `react.commands.retry1.queue`/`retry2.queue`, `react.tool.search.retry1.queue`/`retry2.queue`, y sus equivalentes para `calculator` y `weather`. Las `retry1` tienen `x-message-ttl` 30 000 ms y dead-letterean a `react.dlq.retry2.exchange`; las `retry2` tienen `x-message-ttl` 300 000 ms y dead-letterean a `react.dlq.quarantine.exchange`. Los exchanges `react.commands.exchange` y `react.tool.dispatch.exchange` llevan además un `alternate-exchange` → `react.dlq.quarantine.exchange` para capturar mensajes no ruteables.

> [!tip] La cadena `commands/tool.*/results` → (al fallar) `dlq.retry1` (30s) → `dlq.retry2` (5min) → `dlq.quarantine` da **dos reintentos espaciados automáticos** antes de aislar el mensaje. El espaciado creciente (30s, luego 5min) le da tiempo a un worker caído a recuperarse sin saturarlo con reintentos inmediatos.

El `topology.json` completo es el núcleo del capítulo (infrastructure-as-code, cargable de un solo curl a la Management API):

```json
{
  "rabbit_version": "3.12.0",
  "vhosts": [
    { "name": "/react" }
  ],
  "users": [
    {
      "name":     "react.agent",
      "password": "CHANGE_ME_agent",
      "tags":     ""
    },
    {
      "name":     "react.operator",
      "password": "CHANGE_ME_operator",
      "tags":     "management"
    }
  ],
  "permissions": [
    {
      "user":      "react.agent",
      "vhost":     "/react",
      "configure": "react\\..*",
      "write":     "react\\..*",
      "read":      "react\\..*"
    },
    {
      "user":      "react.operator",
      "vhost":     "/react",
      "configure": "react\\..*",
      "write":     "react\\..*",
      "read":      "react\\..*"
    }
  ],
  "exchanges": [
    {
      "name":        "react.commands.exchange",
      "vhost":       "/react",
      "type":        "topic",
      "durable":     true,
      "auto_delete": false,
      "arguments":   {
        "alternate-exchange": "react.dlq.quarantine.exchange"
      }
    },
    {
      "name":        "react.tool.dispatch.exchange",
      "vhost":       "/react",
      "type":        "topic",
      "durable":     true,
      "auto_delete": false,
      "arguments":   {
        "alternate-exchange": "react.dlq.quarantine.exchange"
      }
    },
    {
      "name":        "react.tool.results.exchange",
      "vhost":       "/react",
      "type":        "topic",
      "durable":     true,
      "auto_delete": false
    },
    {
      "name":        "react.dlq.retry1.exchange",
      "vhost":       "/react",
      "type":        "topic",
      "durable":     true,
      "auto_delete": false
    },
    {
      "name":        "react.dlq.retry2.exchange",
      "vhost":       "/react",
      "type":        "topic",
      "durable":     true,
      "auto_delete": false
    },
    {
      "name":        "react.dlq.quarantine.exchange",
      "vhost":       "/react",
      "type":        "fanout",
      "durable":     true,
      "auto_delete": false
    }
  ],
  "queues": [
    {
      "name":        "react.commands.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry1.exchange",
        "x-dead-letter-routing-key":  "dlq.react.commands.queue"
      }
    },
    {
      "name":        "react.tool.search.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry1.exchange",
        "x-dead-letter-routing-key":  "dlq.react.tool.search.queue"
      }
    },
    {
      "name":        "react.tool.calculator.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry1.exchange",
        "x-dead-letter-routing-key":  "dlq.react.tool.calculator.queue"
      }
    },
    {
      "name":        "react.tool.weather.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry1.exchange",
        "x-dead-letter-routing-key":  "dlq.react.tool.weather.queue"
      }
    },
    {
      "name":        "react.tool.results.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry1.exchange",
        "x-dead-letter-routing-key":  "dlq.react.tool.results.queue",
        "x-message-ttl":              300000
      }
    },
    {
      "name":        "react.commands.retry1.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry2.exchange",
        "x-dead-letter-routing-key":  "dlq.react.commands.queue",
        "x-message-ttl":              30000
      }
    },
    {
      "name":        "react.commands.retry2.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.quarantine.exchange",
        "x-message-ttl":              300000
      }
    },
    {
      "name":        "react.tool.search.retry1.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry2.exchange",
        "x-dead-letter-routing-key":  "dlq.react.tool.search.queue",
        "x-message-ttl":              30000
      }
    },
    {
      "name":        "react.tool.search.retry2.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.quarantine.exchange",
        "x-message-ttl":              300000
      }
    },
    {
      "name":        "react.tool.calculator.retry1.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry2.exchange",
        "x-dead-letter-routing-key":  "dlq.react.tool.calculator.queue",
        "x-message-ttl":              30000
      }
    },
    {
      "name":        "react.tool.calculator.retry2.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.quarantine.exchange",
        "x-message-ttl":              300000
      }
    },
    {
      "name":        "react.tool.weather.retry1.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.retry2.exchange",
        "x-dead-letter-routing-key":  "dlq.react.tool.weather.queue",
        "x-message-ttl":              30000
      }
    },
    {
      "name":        "react.tool.weather.retry2.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments": {
        "x-dead-letter-exchange":     "react.dlq.quarantine.exchange",
        "x-message-ttl":              300000
      }
    },
    {
      "name":        "react.quarantine.queue",
      "vhost":       "/react",
      "durable":     true,
      "auto_delete": false,
      "arguments":   {}
    }
  ],
  "bindings": [
    {
      "source":           "react.commands.exchange",
      "vhost":            "/react",
      "destination":      "react.commands.queue",
      "destination_type": "queue",
      "routing_key":      "react.command.#",
      "arguments":        {}
    },
    {
      "source":           "react.tool.dispatch.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.search.queue",
      "destination_type": "queue",
      "routing_key":      "react.tool.search.#",
      "arguments":        {}
    },
    {
      "source":           "react.tool.dispatch.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.calculator.queue",
      "destination_type": "queue",
      "routing_key":      "react.tool.calculator.#",
      "arguments":        {}
    },
    {
      "source":           "react.tool.dispatch.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.weather.queue",
      "destination_type": "queue",
      "routing_key":      "react.tool.weather.#",
      "arguments":        {}
    },
    {
      "source":           "react.tool.results.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.results.queue",
      "destination_type": "queue",
      "routing_key":      "react.result.#",
      "arguments":        {}
    },
    {
      "source":           "react.dlq.retry1.exchange",
      "vhost":            "/react",
      "destination":      "react.commands.retry1.queue",
      "destination_type": "queue",
      "routing_key":      "dlq.react.commands.queue",
      "arguments":        {}
    },
    {
      "source":           "react.dlq.retry1.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.search.retry1.queue",
      "destination_type": "queue",
      "routing_key":      "dlq.react.tool.search.queue",
      "arguments":        {}
    },
    {
      "source":           "react.dlq.retry1.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.calculator.retry1.queue",
      "destination_type": "queue",
      "routing_key":      "dlq.react.tool.calculator.queue",
      "arguments":        {}
    },
    {
      "source":           "react.dlq.retry1.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.weather.retry1.queue",
      "destination_type": "queue",
      "routing_key":      "dlq.react.tool.weather.queue",
      "arguments":        {}
    },
    {
      "source":           "react.dlq.retry2.exchange",
      "vhost":            "/react",
      "destination":      "react.commands.retry2.queue",
      "destination_type": "queue",
      "routing_key":      "dlq.react.commands.queue",
      "arguments":        {}
    },
    {
      "source":           "react.dlq.retry2.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.search.retry2.queue",
      "destination_type": "queue",
      "routing_key":      "dlq.react.tool.search.queue",
      "arguments":        {}
    },
    {
      "source":           "react.dlq.retry2.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.calculator.retry2.queue",
      "destination_type": "queue",
      "routing_key":      "dlq.react.tool.calculator.queue",
      "arguments":        {}
    },
    {
      "source":           "react.dlq.retry2.exchange",
      "vhost":            "/react",
      "destination":      "react.tool.weather.retry2.queue",
      "destination_type": "queue",
      "routing_key":      "dlq.react.tool.weather.queue",
      "arguments":        {}
    }
  ]
}
```

> [!note] **Ninguno de estos elementos es boilerplate de RabbitMQ — cada uno existe porque un modo de falla concreto lo requiere.** El `x-message-ttl` da el back-off temporal; el `x-dead-letter-exchange` arma la cadena de tiers; el `commands`/`results` DLX permite que el agente mismo falle sin perder el episodio.

## Building the ReAct agent

El `react_agent.py` es el cerebro: consume comandos, corre el loop ReAct, llama al LLM, despacha tools y colecta resultados. Se construye sobre **message schemas** tipados, un método `_think` que fuerza al LLM a responder JSON estricto, un `_dispatch_tool` que publica y pollea, y una **estrategia de ACK** rigurosa.

### Message schemas (dataclasses)

La cabecera del módulo `react_agent.py` declara el docstring, los imports, el logging y las **constantes** (incluido `MAX_STEPS` por env var y `PREFETCH_COUNT = 1`, un comando a la vez por consumer):

```python
"""
react_agent.py — ReAct (Reasoning + Acting) agent over RabbitMQ.

The agent runs a Thought → Action → Observation loop.  Each iteration:
  1. Receives a Command message from react.commands.queue.
  2. Calls the LLM to produce a Thought and an Action (tool name + input).
  3. Publishes a ToolRequest to react.tool.dispatch.exchange.
  4. Consumes the ToolResult from react.tool.results.queue.
  5. Feeds the Observation back into the LLM context.
  6. Repeats until the LLM emits a Final Answer.

Every message carries:
  - correlation_id   threaded from the original Command
  - x-step           current loop iteration number
  - x-failure-type   set on nack to guide DLQ routing (transient / permanent / poison)

Dependencies:
    pip install pika anthropic python-dotenv
"""

from __future__ import annotations

import json
import logging
import os
import uuid
from dataclasses import dataclass, field
from typing import Any

import anthropic
import pika
import pika.exceptions
from dotenv import load_dotenv

load_dotenv()
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s — %(message)s")
log = logging.getLogger("react.agent")

# ── Constants ─────────────────────────────────────────────────────────────────

RABBITMQ_URL        = os.getenv("RABBITMQ_URL", "amqp://react.agent:CHANGE_ME_agent@localhost:5672/%2Freact")
COMMAND_QUEUE       = "react.commands.queue"
TOOL_DISPATCH_EX    = "react.tool.dispatch.exchange"
TOOL_RESULTS_QUEUE  = "react.tool.results.queue"
MAX_STEPS           = int(os.getenv("REACT_MAX_STEPS", "10"))
PREFETCH_COUNT      = 1  # one command at a time per consumer instance
```

Los tres mensajes del sistema son **dataclasses** con el `correlation_id` como clave que hila el episodio entero:

> [!note] **`correlation_id` como hilo conductor.** Es el **[[Correlation Identifier]]** de los EIP: el mismo id viaja en el Command, en cada ToolRequest y en cada ToolResult, permitiendo matchear las respuestas con su request original aunque lleguen desordenadas o entremezcladas con las de otros episodios.

```python
@dataclass
class Command:
    """Inbound message from react.commands.queue."""
    correlation_id: str
    question: str
    step: int = 0
    history: list[dict[str, str]] = field(default_factory=list)

    @staticmethod
    def from_delivery(body: bytes, properties: pika.BasicProperties) -> "Command":
        data = json.loads(body)
        return Command(
            correlation_id=properties.correlation_id or str(uuid.uuid4()),
            question=data["question"],
            step=int((properties.headers or {}).get("x-step", 0)),
            history=data.get("history", []),
        )

    def to_amqp_properties(self) -> pika.BasicProperties:
        return pika.BasicProperties(
            correlation_id=self.correlation_id,
            content_type="application/json",
            delivery_mode=2,           # persistent
            headers={"x-step": self.step},
        )


@dataclass
class ToolRequest:
    """Published to react.tool.dispatch.exchange."""
    correlation_id: str
    tool: str
    input: str
    step: int
    reply_to: str = TOOL_RESULTS_QUEUE

    def routing_key(self) -> str:
        return f"react.tool.{self.tool}.dispatch"

    def to_body(self) -> bytes:
        return json.dumps({
            "tool":  self.tool,
            "input": self.input,
            "step":  self.step,
        }).encode()

    def to_amqp_properties(self) -> pika.BasicProperties:
        return pika.BasicProperties(
            correlation_id=self.correlation_id,
            reply_to=self.reply_to,
            content_type="application/json",
            delivery_mode=2,
            headers={"x-step": self.step},
        )


@dataclass
class ToolResult:
    """Consumed from react.tool.results.queue."""
    correlation_id: str
    tool: str
    output: str
    step: int
    error: str | None = None

    @staticmethod
    def from_delivery(body: bytes, properties: pika.BasicProperties) -> "ToolResult":
        data = json.loads(body)
        return ToolResult(
            correlation_id=properties.correlation_id or "",
            tool=data["tool"],
            output=data.get("output", ""),
            step=int((properties.headers or {}).get("x-step", 0)),
            error=data.get("error"),
        )
```

### El método `_think` (LLM call + JSON estricto)

`_think` arma el contexto (la pregunta + las observaciones acumuladas) y llama al LLM con un **SYSTEM_PROMPT** que exige una respuesta JSON de **exactamente dos formas posibles**: una **action** (usar una tool) o una **final_answer**. Cualquier respuesta que no parsee como JSON es tratada como un **poison message** (`ValueError`), porque un LLM que no respeta el contrato no puede manejarse con retries — va directo a la DLQ.

```python
SYSTEM_PROMPT = """You are a ReAct agent. For each user question you must reason step by step.

On each step you MUST respond with valid JSON in exactly one of these two forms:

  Action step:
  {
    "thought": "<your reasoning>",
    "action": {
      "tool": "<search | calculator | weather>",
      "input": "<tool input string>"
    }
  }

  Final answer (only when you have enough information):
  {
    "thought": "<your final reasoning>",
    "final_answer": "<your complete answer to the user>"
  }

Available tools:
  search      — web search, returns a short summary
  calculator  — evaluates a mathematical expression, returns a number
  weather     — returns current weather for a city name

Rules:
  - Never guess tool results. Always call a tool if you need external information.
  - Use the minimum number of steps necessary.
  - Your JSON must be parseable. No trailing commas, no comments.
"""
```

`_think` reconstruye el historial completo de mensajes y lo manda al LLM. El system prompt instruye al modelo a devolver JSON válido en una de las dos formas (un *action step* o una *final answer*) y le ordena **explícitamente no inventar resultados de tools**. Si el modelo devuelve algo que no parsea como JSON, `_think` lanza un `ValueError` — tratado como falla **poison**, NACK sin requeue, ruteado directo a quarantine vía el dead-letter exchange:

```python
class ReactAgent:
    """
    Stateless per-message ReAct loop.

    Connection and channel are created once on startup and reused across
    messages.  If the broker drops the connection, the outer retry loop
    in main() reconnects.
    """

    def __init__(self, channel: pika.adapters.blocking_connection.BlockingChannel) -> None:
        self.channel = channel
        self.llm = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

    # ── LLM call ──────────────────────────────────────────────────────────────

    def _think(self, question: str, history: list[dict[str, str]]) -> dict[str, Any]:
        """
        Call the LLM with the current question + observation history.
        Returns a parsed JSON dict — either an action step or a final answer.
        Raises ValueError on unparseable output (poison-message path).
        """
        messages: list[dict[str, str]] = [
            {"role": "user", "content": question},
        ]
        for entry in history:
            messages.append({"role": "assistant", "content": entry["assistant"]})
            if "observation" in entry:
                messages.append({"role": "user",      "content": f"Observation: {entry['observation']}"})

        response = self.llm.messages.create(
            model="claude-opus-4-5",
            max_tokens=512,
            system=SYSTEM_PROMPT,
            messages=messages,
        )

        raw = response.content[0].text.strip()
        log.debug("LLM raw output: %s", raw)

        try:
            return json.loads(raw)
        except json.JSONDecodeError as exc:
            raise ValueError(f"LLM returned non-JSON: {raw!r}") from exc
```

> [!warning] Si el LLM devuelve texto fuera del JSON, `json.loads` lanza `JSONDecodeError`, que `_think` re-empaqueta como `ValueError`. Esto **no se reintenta**: reintentar la misma llamada probablemente produzca el mismo desastre. Es la clasificación **poison** de la Table 9.2.

### El método `_dispatch_tool` (publish + poll por correlation_id)

`_dispatch_tool` publica un `ToolRequest` a `tool.dispatch.exchange` con la routing key `react.tool.<type>.dispatch`, y luego **pollea la cola `results`** durante hasta 30 segundos con `basic_get`, descartando (con `nack` + `requeue=True`) los resultados cuyo `correlation_id` **no matchea** el episodio actual — son respuestas de otros episodios que volverán a la cola para su dueño. Si pasan los 30s sin la respuesta correcta, lanza `TimeoutError` (clasificación **transient** → retry1).

```python
    def _dispatch_tool(self, request: ToolRequest) -> ToolResult:
        """
        Publish a ToolRequest and block until the matching ToolResult arrives.
        Uses the correlation_id to filter results; discards unrelated messages.
        Raises TimeoutError if no result arrives within 30 seconds.
        """
        self.channel.basic_publish(
            exchange=TOOL_DISPATCH_EX,
            routing_key=request.routing_key(),
            body=request.to_body(),
            properties=request.to_amqp_properties(),
        )
        log.info("[%s] step=%d  → tool=%s  input=%r",
                 request.correlation_id, request.step, request.tool, request.input)

        # Poll the results queue until we see our correlation_id.
        # In production, use a per-correlation reply queue instead.
        deadline = 30  # seconds
        elapsed = 0
        poll_interval = 0.2

        while elapsed < deadline:
            method, properties, body = self.channel.basic_get(
                queue=TOOL_RESULTS_QUEUE, auto_ack=False
            )
            if method is None:
                import time
                time.sleep(poll_interval)
                elapsed += poll_interval
                continue

            result = ToolResult.from_delivery(body, properties)

            if result.correlation_id != request.correlation_id:
                # Not ours — put it back (nack + requeue).
                self.channel.basic_nack(method.delivery_tag, requeue=True)
                import time
                time.sleep(poll_interval)
                elapsed += poll_interval
                continue

            self.channel.basic_ack(method.delivery_tag)
            return result

        raise TimeoutError(
            f"No tool result for correlation_id={request.correlation_id} "
            f"tool={request.tool} after {deadline}s"
        )
```

> [!tip] El **poll con filtro de `correlation_id`** sobre una cola `results` compartida es simple pero subóptimo: con muchos episodios concurrentes, cada agente nack-ea muchos mensajes ajenos. La mejora de producción (más abajo) es una **per-correlation reply queue** (una cola de respuesta por episodio), que elimina el filtrado.

### El loop ReAct y la estrategia de ACK

`run_loop` orquesta el ciclo: piensa, y si el LLM pidió una action despacha la tool y agrega la observación; si pidió `final_answer`, termina. Un **step limit** corta loops que no convergen (clasificación **permanent** → no reintentar, va a parking/quarantine vía retry chain). `on_command` envuelve todo con la clasificación de fallas y el ACK/NACK.

```python
    def run_loop(self, cmd: Command) -> str:
        """
        Execute the ReAct loop for one Command.
        Returns the final answer string.
        Raises on unrecoverable failure (caller handles ack/nack).
        """
        history: list[dict[str, str]] = list(cmd.history)
        step = cmd.step

        for _ in range(MAX_STEPS - step):
            step += 1
            log.info("[%s] step=%d  THINK", cmd.correlation_id, step)

            decision = self._think(cmd.question, history)

            if "final_answer" in decision:
                log.info("[%s] step=%d  FINAL ANSWER", cmd.correlation_id, step)
                return decision["final_answer"]

            if "action" not in decision:
                raise ValueError(f"LLM response missing 'action' and 'final_answer': {decision}")

            action    = decision["action"]
            tool_name = action.get("tool", "").strip().lower()
            tool_input = action.get("input", "")

            if tool_name not in {"search", "calculator", "weather"}:
                raise ValueError(f"Unknown tool: {tool_name!r}")

            request = ToolRequest(
                correlation_id=cmd.correlation_id,
                tool=tool_name,
                input=tool_input,
                step=step,
            )

            result = self._dispatch_tool(request)

            if result.error:
                observation = f"ERROR: {result.error}"
                log.warning("[%s] step=%d  tool=%s  error=%s",
                            cmd.correlation_id, step, tool_name, result.error)
            else:
                observation = result.output
                log.info("[%s] step=%d  ← tool=%s  output=%r",
                         cmd.correlation_id, step, tool_name, result.output[:120])

            history.append({
                "assistant":   json.dumps(decision),
                "observation": observation,
            })

        raise RuntimeError(
            f"Exceeded MAX_STEPS={MAX_STEPS} without final answer "
            f"for correlation_id={cmd.correlation_id}"
        )

    def on_command(
        self,
        channel: pika.adapters.blocking_connection.BlockingChannel,
        method: pika.spec.Basic.Deliver,
        properties: pika.BasicProperties,
        body: bytes,
    ) -> None:
        """
        Called by pika for each message delivered from react.commands.queue.
        Handles ack/nack and DLQ header tagging.
        """
        correlation_id = (properties.correlation_id or "unknown")
        log.info("[%s] received command", correlation_id)

        try:
            cmd    = Command.from_delivery(body, properties)
            answer = self.run_loop(cmd)
            log.info("[%s] DONE  answer=%r", correlation_id, answer[:200])
            channel.basic_ack(method.delivery_tag)

        except (json.JSONDecodeError, ValueError, KeyError) as exc:
            # Poison message — malformed body or bad LLM output.
            # Skip retries, go straight to quarantine.
            log.error("[%s] POISON  %s", correlation_id, exc)
            channel.basic_nack(
                method.delivery_tag,
                requeue=False,
            )
            # Ideally set x-failure-type: permanent via a per-message policy;
            # shown here as a log marker for illustration.

        except TimeoutError as exc:
            # Transient — tool worker may have been restarting.
            log.warning("[%s] TRANSIENT  %s", correlation_id, exc)
            channel.basic_nack(method.delivery_tag, requeue=False)
            # Dead-letters to retry1 (30s) via x-dead-letter-exchange.

        except RuntimeError as exc:
            # Step limit exceeded — treat as permanent.
            log.error("[%s] PERMANENT  %s", correlation_id, exc)
            channel.basic_nack(method.delivery_tag, requeue=False)

        except Exception as exc:  # noqa: BLE001
            log.exception("[%s] UNEXPECTED  %s", correlation_id, exc)
            channel.basic_nack(method.delivery_tag, requeue=False)
```

> [!warning] **Todas las clases de falla hacen `basic_nack(requeue=False)`**, no `requeue=True`. Con `requeue=False` el mensaje **NO vuelve a la cola activa** (lo que causaría un retry-storm inmediato): va al **`x-dead-letter-exchange`** de la cola — es decir, entra en la cadena retry1 (30s) → retry2 (5min) → quarantine. El espaciado temporal de la DLQ es el que da el back-off; reencolar directo lo arruinaría.

### Tabla 9.2 — Clasificación de fallas en el agente

El agente clasifica toda excepción en cuatro clases y **todas terminan en `basic_nack(requeue=False)`** (entran a la cadena retry1 → retry2 → quarantine):

| Excepción | Clase | Causa | Acción |
|---|---|---|---|
| `JSONDecodeError` / `ValueError` / `KeyError` | **Poison** | El LLM rompió el contrato JSON o el schema del mensaje es inválido | `basic_nack(requeue=False)` → retry1→retry2→quarantine |
| `TimeoutError` | **Transient** | Una tool no respondió en 30s (worker lento/caído) | `basic_nack(requeue=False)` → retry1→retry2→quarantine |
| `RuntimeError` (step limit) | **Permanent** | El loop no convergió en `MAX_STEPS` | `basic_nack(requeue=False)` → retry1→retry2→quarantine |
| Cualquier otra (`Exception`) | **Unknown** | Falla inesperada no anticipada | `basic_nack(requeue=False)` → retry1→retry2→quarantine |

> [!note] Aunque las cuatro clases comparten el mismo `basic_nack(requeue=False)`, **distinguirlas tiene valor operativo**: los logs etiquetan POISON/TRANSIENT/PERMANENT/UNKNOWN, así el operador que inspecciona la quarantine sabe *por qué* llegó cada mensaje y decide si vale la pena replayarlo (un transient sí, un poison casi nunca).

### `_make_channel` y `main`

`_make_channel` abre la conexión contra el vhost `/react` y aplica `basic_qos(prefetch_count=PREFETCH_COUNT)` para limitar cuántos mensajes sin-ackear toma el agente a la vez (backpressure). `main` corre un **reconnection loop**: consume de `react.commands.queue` con `auto_ack=False`, y si el broker tira la conexión (`AMQPConnectionError`) reconecta a los 5 segundos.

```python
# ── Entry point ───────────────────────────────────────────────────────────────

def _make_channel() -> pika.adapters.blocking_connection.BlockingChannel:
    params  = pika.URLParameters(RABBITMQ_URL)
    conn    = pika.BlockingConnection(params)
    channel = conn.channel()
    channel.basic_qos(prefetch_count=PREFETCH_COUNT)
    return channel


def main() -> None:
    import time

    while True:
        try:
            log.info("Connecting to broker…")
            channel = _make_channel()
            agent   = ReactAgent(channel)

            channel.basic_consume(
                queue=COMMAND_QUEUE,
                on_message_callback=agent.on_command,
                auto_ack=False,
                consumer_tag=f"react.agent.{uuid.uuid4().hex[:8]}",
            )

            log.info("Waiting for commands on %s", COMMAND_QUEUE)
            channel.start_consuming()

        except pika.exceptions.AMQPConnectionError as exc:
            log.warning("Broker connection lost: %s — reconnecting in 5s", exc)
            time.sleep(5)

        except KeyboardInterrupt:
            log.info("Shutting down.")
            break


if __name__ == "__main__":
    main()
```

## Implementing the tool workers

Los **tool workers** son consumers simples: cada uno escucha su cola (`react.tool.search.queue`, `react.tool.calculator.queue`, `react.tool.weather.queue`), ejecuta la tool y publica un `ToolResult` de vuelta a `react.tool.results.exchange` (routing key `react.result.<tool>.<correlation_id_prefix>`). El capítulo implementa tres tools como **stubs sin credenciales** (no requieren API keys para correr el demo) con `PREFETCH_COUNT = 4` (los workers toleran más concurrencia que el agente), pero la lógica de manejo de fallas es real y production-shaped.

> [!note] **Distinción crítica: tool error vs system failure.**
> - **Tool error (`ValueError`)** — la tool corrió pero el input era inválido o no hay resultado (ej. una expresión que no se puede calcular). Se publica un **`ToolResult` con `error`**, el mensaje se **ackea**, y **el agente razona sobre el error** (lo recibe como observación y decide qué hacer). NO es una falla del sistema.
> - **System failure (excepción inesperada)** — algo se rompió en el worker mismo (un bug, una dependencia caída). Se hace **`basic_nack(requeue=False)`** sin publicar resultado → el mensaje entra a la DLQ (retry1).

El **calculator** no usa `eval` directo sobre la expresión: es un **safe evaluator** que parsea con el módulo `ast`, valida cada nodo contra una whitelist (`allowed_nodes`) y restringe los nombres permitidos a `sqrt`, `pi` y `e` (`allowed_names`), rechazando cualquier otro nombre o nodo antes de compilar y evaluar con `__builtins__` vacíos.

```python
"""
tool_worker.py — Executes tool calls dispatched by the ReAct agent.

Consumes from one of:
    react.tool.search.queue
    react.tool.calculator.queue
    react.tool.weather.queue

Publishes results to react.tool.results.exchange with routing key
react.result.<tool>.<correlation_id_prefix>.

Pass the tool name as the first CLI argument:
    python tool_worker.py search
    python tool_worker.py calculator
    python tool_worker.py weather

Each worker type is independently scalable: run more instances of
whichever worker type is the bottleneck.

Dependencies:
    pip install pika python-dotenv
"""

from __future__ import annotations

import ast
import json
import logging
import math
import operator
import os
import sys
import uuid

import pika
import pika.exceptions
from dotenv import load_dotenv

load_dotenv()
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s — %(message)s")

RABBITMQ_URL       = os.getenv("RABBITMQ_URL", "amqp://react.agent:CHANGE_ME_agent@localhost:5672/%2Freact")
TOOL_RESULTS_EX    = "react.tool.results.exchange"
PREFETCH_COUNT     = 4   # tool workers can handle more concurrency than the main agent


# ── Simulated tool implementations ───────────────────────────────────────────
# In production these would call real APIs.  The stubs here are intentionally
# simple so the example runs without external credentials.

def tool_search(query: str) -> str:
    """Simulated web search.  Returns a canned summary keyed on keywords."""
    q = query.lower()
    if "eiffel tower" in q:
        return "The Eiffel Tower is 330 metres tall.  It was completed in 1889 and is located in Paris, France."
    if "python" in q and "release" in q:
        return "Python 3.12 was released in October 2023.  Python 3.13 is the current stable release as of 2025."
    if "rabbitmq" in q:
        return "RabbitMQ is an open-source message broker implementing AMQP.  It supports topic, direct, fanout, and headers exchanges."
    return f"No results found for: {query}"


def tool_calculator(expression: str) -> str:
    """
    Safe arithmetic evaluator.
    Supports +, -, *, /, **, sqrt, and parentheses.
    Raises ValueError on unsafe or malformed input.
    """
    # Restrict to a safe subset using ast.
    allowed_nodes = (
        ast.Expression, ast.BinOp, ast.UnaryOp, ast.Num, ast.Constant,
        ast.Add, ast.Sub, ast.Mult, ast.Div, ast.Pow, ast.USub, ast.UAdd,
        ast.Call, ast.Name, ast.Load,
    )
    allowed_names = {"sqrt": math.sqrt, "pi": math.pi, "e": math.e}

    try:
        tree = ast.parse(expression.strip(), mode="eval")
    except SyntaxError as exc:
        raise ValueError(f"Invalid expression: {expression!r}") from exc

    for node in ast.walk(tree):
        if not isinstance(node, allowed_nodes):
            raise ValueError(f"Unsafe expression node: {type(node).__name__}")
        if isinstance(node, ast.Name) and node.id not in allowed_names:
            raise ValueError(f"Unknown name: {node.id!r}")

    result = eval(  # noqa: S307  (safe — restricted by ast check above)
        compile(tree, "<expr>", "eval"),
        {"__builtins__": {}},
        allowed_names,
    )
    return str(result)


def tool_weather(city: str) -> str:
    """Simulated weather lookup."""
    city_l = city.strip().lower()
    data = {
        "london":        "Overcast, 12°C, wind 15 km/h SW",
        "new york":      "Partly cloudy, 18°C, wind 10 km/h NW",
        "tokyo":         "Sunny, 24°C, humidity 65%",
        "kuala lumpur":  "Thunderstorms likely, 31°C, humidity 88%",
        "paris":         "Clear, 16°C, wind 8 km/h N",
    }
    return data.get(city_l, f"No weather data available for: {city}")


TOOLS = {
    "search":     tool_search,
    "calculator": tool_calculator,
    "weather":    tool_weather,
}

TOOL_QUEUES = {
    "search":     "react.tool.search.queue",
    "calculator": "react.tool.calculator.queue",
    "weather":    "react.tool.weather.queue",
}


# ── Worker ────────────────────────────────────────────────────────────────────

class ToolWorker:
    def __init__(
        self, tool_name: str,
        channel: pika.adapters.blocking_connection.BlockingChannel
    ) -> None:
        if tool_name not in TOOLS:
            raise ValueError(f"Unknown tool: {tool_name!r}.  Choose from: {list(TOOLS)}")
        self.tool_name = tool_name
        self.tool_fn   = TOOLS[tool_name]
        self.channel   = channel
        self.log       = logging.getLogger(f"tool.{tool_name}")

    def _publish_result(
        self,
        correlation_id: str,
        step: int,
        output: str | None,
        error: str | None,
    ) -> None:
        body = json.dumps({
            "tool":   self.tool_name,
            "output": output or "",
            "error":  error,
            "step":   step,
        }).encode()

        props = pika.BasicProperties(
            correlation_id=correlation_id,
            content_type="application/json",
            delivery_mode=2,
            headers={"x-step": step},
        )

        self.channel.basic_publish(
            exchange=TOOL_RESULTS_EX,
            routing_key=f"react.result.{self.tool_name}.{correlation_id[:8]}",
            body=body,
            properties=props,
        )

    def on_message(
        self,
        channel: pika.adapters.blocking_connection.BlockingChannel,
        method: pika.spec.Basic.Deliver,
        properties: pika.BasicProperties,
        body: bytes,
    ) -> None:
        correlation_id = properties.correlation_id or "unknown"
        step = int((properties.headers or {}).get("x-step", 0))

        try:
            data  = json.loads(body)
            inp   = data.get("input", "")
            self.log.info("[%s] step=%d  input=%r", correlation_id, step, inp[:120])

            output = self.tool_fn(inp)
            self._publish_result(
                correlation_id, step, output=output, error=None)
            self.log.info("[%s] step=%d  output=%r", correlation_id, step, output[:120])
            channel.basic_ack(method.delivery_tag)

        except (json.JSONDecodeError, KeyError) as exc:
            # Poison — malformed tool request body.
            self.log.error("[%s] POISON  %s", correlation_id, exc)
            channel.basic_nack(method.delivery_tag, requeue=False)

        except ValueError as exc:
            # Permanent tool error (bad expression, unknown city, etc.)
            # Publish an error result so the agent can reason about it
            # rather than leaving the agent waiting on a timeout.
            self.log.warning("[%s] TOOL ERROR  %s", correlation_id, exc)
            self._publish_result(
                correlation_id, step, output=None, error=str(exc))
            channel.basic_ack(method.delivery_tag)

        except Exception as exc:  # noqa: BLE001
            # Transient — unknown error, allow retry.
            self.log.exception("[%s] UNEXPECTED  %s", correlation_id, exc)
            channel.basic_nack(method.delivery_tag, requeue=False)


def main(tool_name: str) -> None:
    import time

    while True:
        try:
            logging.info("Connecting to broker (tool=%s)…", tool_name)
            params  = pika.URLParameters(RABBITMQ_URL)
            conn    = pika.BlockingConnection(params)
            channel = conn.channel()
            channel.basic_qos(prefetch_count=PREFETCH_COUNT)

            worker = ToolWorker(tool_name, channel)
            queue  = TOOL_QUEUES[tool_name]

            channel.basic_consume(
                queue=queue,
                on_message_callback=worker.on_message,
                auto_ack=False,
                consumer_tag=f"tool.{tool_name}.{uuid.uuid4().hex[:8]}",
            )

            logging.info("Waiting for tool requests on %s", queue)
            channel.start_consuming()

        except pika.exceptions.AMQPConnectionError as exc:
            logging.warning("Broker connection lost: %s — reconnecting in 5s", exc)
            time.sleep(5)

        except KeyboardInterrupt:
            logging.info("Shutting down.")
            break


if __name__ == "__main__":
    if len(sys.argv) != 2 or sys.argv[1] not in TOOLS:
        print(f"Usage: python tool_worker.py <{'|'.join(TOOLS)}>",
            file=sys.stderr)
        sys.exit(1)
    main(sys.argv[1])
```

**Escalado por [[Competing Consumers]].** Cada tipo de tool corre como un proceso worker independiente; se arrancan más instancias de la que sea el cuello de botella. El libro lo ejemplifica así:

```bash
# One search worker is often enough for low-throughput use
python tool_worker.py search &

# Calculator is CPU-bound and fast; one instance handles high load
python tool_worker.py calculator &

# Scale weather workers if the external API is slow
python tool_worker.py weather &
python tool_worker.py weather &
python tool_worker.py weather &
```

> [!tip] Esto es el patrón **[[Competing Consumers]]**: múltiples instancias del mismo worker consumen de la misma cola y RabbitMQ reparte los mensajes entre ellas. No hace falta código de coordinación; escalar throughput = agregar consumers.

## Sending commands (publish_command.py)

El `publish_command.py` es una CLI mínima (un thin wrapper) que inyecta una pregunta al sistema: genera un **`correlation_id` UUID fresco**, lo pasa en las `BasicProperties` y publica a `react.commands.exchange` con routing key `react.command.new`. Imprime el `correlation_id` para poder grepear los logs del agente y seguir esa cadena de razonamiento. Es el punto de entrada del episodio.

```python
"""
publish_command.py — Publish a question to the ReAct agent for testing.

Usage:
    python publish_command.py "What is the height of the Eiffel Tower in feet?"
    python publish_command.py "What is sqrt(144) + the current temperature in Tokyo?"

The script publishes to react.commands.exchange and prints the
correlation_id so you can trace the message through the broker.

Dependencies:
    pip install pika python-dotenv
"""

from __future__ import annotations
import json
import sys
import uuid
import pika
from dotenv import load_dotenv
import os

load_dotenv()

RABBITMQ_URL     = os.getenv("RABBITMQ_URL", "amqp://react.agent:CHANGE_ME_agent@localhost:5672/%2Freact")
COMMANDS_EXCHANGE = "react.commands.exchange"
ROUTING_KEY       = "react.command.new"


def publish(question: str) -> str:
    correlation_id = str(uuid.uuid4())

    params  = pika.URLParameters(RABBITMQ_URL)
    conn    = pika.BlockingConnection(params)
    channel = conn.channel()

    body = json.dumps({"question": question}).encode()

    props = pika.BasicProperties(
        correlation_id=correlation_id,
        content_type="application/json",
        delivery_mode=2,
        headers={"x-step": 0},
    )

    channel.basic_publish(
        exchange=COMMANDS_EXCHANGE,
        routing_key=ROUTING_KEY,
        body=body,
        properties=props,
    )

    conn.close()
    return correlation_id


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python publish_command.py \"<question>\"", file=sys.stderr)
        sys.exit(1)

    question = " ".join(sys.argv[1:])
    cid = publish(question)
    print(f"Published question: {question!r}")
    print(f"correlation_id:     {cid}")
    print("Watch react_agent.py logs for the Thought → Action → Observation trace.")
```

## Running the system end-to-end

El capítulo muestra el setup completo y un episodio real. Requisitos: **RabbitMQ 3.12+**, **Python 3.11+**, una **Anthropic API key**.

**Setup** (preparar el entorno y cargar la topología):

```bash
# Clone or copy the four source files into a directory
cd react_example

# Install dependencies
pip install -r requirements.txt

# Configure credentials
cp .env.example .env
# Edit .env: set ANTHROPIC_API_KEY and RABBITMQ_URL

# Load the topology into RabbitMQ
curl -u guest:guest -X POST \
  http://localhost:15672/api/definitions \
  -H 'Content-Type: application/json' \
  -d @topology.json
```

**Arrancar los workers** (4 terminales: el agente + los 3 workers):

```bash
# Terminal 1 — ReAct agent
python react_agent.py

# Terminal 2 — search tool worker
python tool_worker.py search

# Terminal 3 — calculator tool worker
python tool_worker.py calculator

# Terminal 4 — weather tool worker
python tool_worker.py weather
```

**Enviar una pregunta** (en una quinta terminal):

```bash
python publish_command.py "How tall is the Eiffel Tower in feet?"

# Expected output:
# Published question: 'How tall is the Eiffel Tower in feet?'
# correlation_id:     3f8a1c2d-...
# Watch react_agent.py logs for the Thought → Action → Observation trace.
```

El sistema corre el loop ReAct: el LLM piensa que necesita la altura → llama a `search` (obtiene **330 metres**) → piensa que debe convertir a pies → llama a `calculator` con `330 * 3.28084` (obtiene **1082.6772**) → da la respuesta final. El **`correlation_id` `[3f8a1c2d]` aparece en cada log line** — es el **Correlation Identifier** en acción, hilando el episodio entero a través de agente, workers y broker:

```text
2025-10-01 14:22:01 INFO react.agent — [3f8a1c2d] received command
2025-10-01 14:22:01 INFO react.agent — [3f8a1c2d] step=1  THINK
2025-10-01 14:22:02 INFO react.agent — [3f8a1c2d] step=1  → tool=search  input='Eiffel Tower height metres'
2025-10-01 14:22:02 INFO tool.search  — [3f8a1c2d] step=1  input='Eiffel Tower height metres'
2025-10-01 14:22:02 INFO tool.search  — [3f8a1c2d] step=1  output='The Eiffel Tower is 330 metres tall...'
2025-10-01 14:22:02 INFO react.agent — [3f8a1c2d] step=1  ← tool=search  output='The Eiffel Tower is 330 metres tall...'
2025-10-01 14:22:02 INFO react.agent — [3f8a1c2d] step=2  THINK
2025-10-01 14:22:03 INFO react.agent — [3f8a1c2d] step=2  → tool=calculator  input='330 * 3.28084'
2025-10-01 14:22:03 INFO tool.calculator — [3f8a1c2d] step=2  input='330 * 3.28084'
2025-10-01 14:22:03 INFO tool.calculator — [3f8a1c2d] step=2  output='1082.6772'
2025-10-01 14:22:03 INFO react.agent — [3f8a1c2d] step=2  ← tool=calculator  output='1082.6772'
2025-10-01 14:22:03 INFO react.agent — [3f8a1c2d] step=3  THINK
2025-10-01 14:22:04 INFO react.agent — [3f8a1c2d] step=3  FINAL ANSWER
2025-10-01 14:22:04 INFO react.agent — [3f8a1c2d] DONE  answer='The Eiffel Tower is approximately 1,083 feet tall.'
```

> [!tip] El mismo `correlation_id` impreso en cada línea es lo que hace **trazable y auditable** un episodio ReAct distribuido: aunque agente y workers corran en procesos (o máquinas) separados, los logs se reconstruyen filtrando por ese id.

## Observing the DLQ in action

Para ver la resiliencia funcionando, el capítulo **provoca una falla**: se **detiene el worker de `search`** (Ctrl+C) y se manda una pregunta que lo necesita:

```bash
# Stop the search worker (Ctrl+C in its terminal)
python publish_command.py "Who invented the telephone?"
```

Pasan **dos cosas en paralelo**, ambas observables en la **Management UI (localhost:15672)** viendo crecer los conteos de mensajes:

1. **El tool request queda sin consumidor.** El agente despacha el search request, pero ningún worker lo consume. A los **30 segundos** vence el TTL de `react.tool.search.retry1.queue` y el mensaje dead-letterea a Tier 2; **5 minutos** después cae a `react.quarantine.queue`.
2. **El agente timeoutea.** Su `basic_get` sobre la results queue agota los **30 segundos** → el agente NACKea el comando, que dead-letterea a `react.commands.retry1.queue`; a los **30 s** expira a retry2, y a los **5 min** a quarantine.

Al **reiniciar el search worker**, los mensajes ya en quarantine **NO se replayan solos** — requieren intervención manual del operador.

> [!warning] **Quarantine NO se replaya solo.** Es un **holding area, no un dead-end**: cada mensaje ahí tiene preservada su cadena completa de **`x-death` headers** (cola de origen, razón, conteo y timestamp de cada tier que atravesó), suficiente para diagnosticar por qué falló y decidir si replayarlo o descartarlo. Para reprocesar, el operador usa la Management API (o `rabbitmqadmin`): saca los mensajes, inspecciona los `x-death` headers y **republica** el body a `react.commands.exchange` con los headers originales una vez resuelta la causa raíz.

```bash
# Shovel a message from quarantine back to the commands exchange
# using rabbitmqadmin or the management API.

# Via management API (republish the raw body with original headers):
curl -u guest:guest -X POST \
  'http://localhost:15672/api/queues/%2Freact/react.quarantine.queue/get' \
  -H 'Content-Type: application/json' \
  -d '{"count": 1, "ackmode": "ack_requeue_false", "encoding": "auto"}'
```

> [!note] El **`x-death` header** (que RabbitMQ agrega automáticamente en cada dead-lettering) es la fuente de verdad para el operador: registra el conteo de rechazos, la razón (`expired`, `rejected`) y la cola de origen de cada hop, permitiendo reconstruir la historia de la falla sin estado externo. Es el mismo mecanismo del [[08 - Pattern-Guided Coding|cap. 8]].

## Preparing for production

El sistema del capítulo corre y es resiliente, pero es un **demo didáctico**. El capítulo cierra con **ocho mejoras** concretas para llevarlo a producción:

> [!tip] **Las 8 mejoras para producción**
> 1. **Reemplazar los tool stubs por API clients reales** — con **timeouts** y **circuit breakers** (no dejar que una API lenta cuelgue el worker).
> 2. **Reemplazar el `basic_get` poll por per-correlation reply queues** — una cola de respuesta por episodio elimina el filtrado por correlation_id y el nack de mensajes ajenos.
> 3. **Persistir las respuestas finales** — publicar a un `react.answers.exchange` y guardarlas en una **DB keyed on `correlation_id`** (hoy la respuesta solo se imprime).
> 4. **Publisher confirms en todos los `basic_publish`** — confirmar que el broker recibió cada mensaje (guaranteed delivery del lado del producer).
> 5. **Setear `x-failure-type` headers explícitos** — etiquetar la clase de falla (poison/transient/permanent/unknown) en el mensaje, no solo en el log, para que el operador filtre la quarantine.
> 6. **Agregar TLS/amqps + mTLS** — cifrar el tráfico al broker y autenticar mutuamente cliente y servidor.
> 7. **Monitorear la profundidad de la cola `quarantine`** con una **alerta** — un crecimiento de quarantine es señal de un problema sistémico.
> 8. **Parametrizar el modelo de Anthropic** — hoy está **hardcodeado `claude-opus-4-5`** en `_think`; debería venir de configuración para poder cambiarlo sin tocar código.

### Tabla 9.3 — Decisiones de diseño y trade-offs

Las siete decisiones de diseño del sistema, con el porqué de cada una:

| Decisión | Choice | Why |
|---|---|---|
| **LLM response format** | JSON estricto (action \| final_answer), parseado; non-JSON → poison | El LLM es un endpoint poco fiable; un contrato estricto convierte el desorden en una falla detectable y clasificable, no en un bug silencioso |
| **Tool dispatch transport** | Publish a `tool.dispatch.exchange` (topic) con routing key por tool | Desacopla agente y workers; permite escalar cada tool independientemente con Competing Consumers |
| **Result collection** | `basic_get` poll sobre cola `results` compartida, filtrando por correlation_id | Simple para el demo; los resultados ajenos se devuelven con nack(requeue=True). En producción → per-correlation reply queue |
| **Failure classification** | 4 clases (poison/transient/permanent/unknown), todas nack(requeue=False) | El requeue directo causaría retry-storms; la cadena DLQ con TTL da el back-off; las etiquetas guían al operador |
| **Tool error propagation** | `ValueError` en el worker → ToolResult con error, acked; el agente razona sobre él | Un input inválido NO es una falla del sistema: el agente debe poder recuperarse razonando, igual que un humano ante un error |
| **Step limit** | `MAX_STEPS` corta loops que no convergen → RuntimeError (permanent) | Sin límite, un loop que no converge consumiría tokens y recursos para siempre |
| **Consumer tag / prefetch** | `prefetch_count=1` (backpressure) | Limita los mensajes sin-ackear por consumer, evitando que un agente acapare la cola y permitiendo reparto justo entre instancias |

> [!note] La tabla deja explícito el principio del capítulo: **cada elección existe porque un modo de falla concreto la exige.** No hay decisión cosmética — JSON estricto frena los poison silenciosos, el nack(requeue=False) frena los retry-storms, el step limit frena los loops infinitos, el prefetch frena el acaparamiento.

## Citas

> "There is no direct network dependency between them — the broker is the system."

> "None of these are RabbitMQ boilerplate as each one exists because a specific failure mode requires it."

## Para aplicar

- **Implementá el loop ReAct con un contrato JSON estricto** — un SYSTEM_PROMPT que fuerce al LLM a responder solo en dos formas (`action` o `final_answer`); tratá cualquier respuesta non-JSON como **poison** (`ValueError`), no como algo reintentable.
- **Hilá cada episodio con un `correlation_id`** — el mismo id en el Command, cada ToolRequest y cada ToolResult; imprimilo en cada log line para trazabilidad (es el **[[Correlation Identifier]]** de los EIP).
- **Desacoplá agente y workers por el broker** — sin dependencia de red directa; publicá tool requests a un topic exchange con routing key por tool (`react.tool.<type>.dispatch`) y escalá con **[[Competing Consumers]]** (N instancias del worker cuello de botella).
- **Clasificá toda falla y mandala a la DLQ con `nack(requeue=False)`** — distinguí poison / transient / permanent / unknown (Table 9.2); nunca `requeue=True` directo (causa retry-storms); dejá que la cadena **retry1 (30s) → retry2 (5min) → quarantine** dé el back-off temporal.
- **Distinguí tool error de system failure en el worker** — un input inválido (`ValueError`) se publica como `ToolResult` con `error` y se **ackea** (el agente lo razona); una excepción inesperada se **nackea** a la DLQ.
- **Usá un safe evaluator, no `eval`** — para el calculator, parseá con `ast` y permití solo nodos aritméticos; rechazá llamadas, nombres y todo lo demás.
- **Poné un step limit** — `MAX_STEPS` para cortar loops que no convergen (→ falla permanent), evitando consumo infinito de tokens.
- **Tratá quarantine como holding area, no dead-end** — no se replaya solo; el operador inspecciona los `x-death` headers vía Management API y republica a `react.commands.exchange` tras resolver la causa raíz.
- **Para producción, aplicá las 8 mejoras** — API clients reales con timeouts/circuit breakers, per-correlation reply queues, persistir respuestas en DB, publisher confirms, `x-failure-type` headers, TLS/amqps + mTLS, alerta de profundidad de quarantine y parametrizar el modelo (hoy hardcodeado `claude-opus-4-5`).

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro.
- [[08 - Pattern-Guided Coding]] — **la base directa**: este capítulo implementa el patrón **[[ReAct]]** (uno de los 8 de la Table 8.1) sobre la misma **topología [[RabbitMQ]] + DLQ de 3 tiers** que diseñó Topologos; reusa los EIP **[[Correlation Identifier]]**, **[[Dead-Letter Queue|DLQ]]**, **[[Channel Adapter]]**, **[[Competing Consumers]]** y la regla de **[[Manual ACK-NACK|manual ACK/NACK]]** + el **[[x-death header]]**.
- [[10 - The Future and Limitations of LLMs]] — capítulo siguiente (y último del libro): un cierre **conceptual** que da el marco de por qué toda esta ingeniería de verificación/poison/DLQ es necesaria — el LLM no entiende ni razona de forma fiable. (El cap. 9 había anticipado un "Plan-and-Execute" como cap. 10, pero el capítulo real es *The Future and Limitations of LLMs*.)
- [[02 - Embeddings The Language of AI]] — el **LLM como endpoint poco fiable**: motiva tratar cada respuesta non-JSON como poison y la estrategia de DLQ/clasificación de fallas.
- [[04 - Building Your First RAG App]] — la app RAG production-grade sobre RabbitMQ y los EIP nativos (Request-Reply, Correlation Identifier, Content Enricher, DLQ) que este capítulo reusa para un patrón agéntico.
- [[A2 - Appendix B - Topologos User Manual|Appendix B]] — el manual de referencia de Topologos, que genera topologías como la de este capítulo (placeholder).
- **[[ReAct]]** (Reasoning + Acting, Yao et al. 2022) · **[[RabbitMQ]]** · **[[Correlation Identifier]]** · **[[Dead-Letter Queue]]** — el patrón, el broker y los EIP centrales.
- Conceptos sembrados: **[[Thought-Action-Observation loop]]** · **[[Correlation Identifier|correlation_id threading]]** · **[[Manual ACK-NACK]]** · **[[DLQ tiering]]** (retry1 30s → retry2 5min → quarantine) · **[[x-death header]]** · **[[x-message-ttl]]** · **[[Competing Consumers]]** · **[[Per-correlation reply queue]]** · **[[Safe AST evaluator]]** · **[[Publisher confirms]]** · **[[Circuit Breaker]]** · **[[mTLS]]** (candidatos a nota propia).
- EIP/GoF reusados del cap. 8: **[[Channel Adapter]]** · **[[Command]]** · **[[Strategy]]** · **[[Invalid Message Channel]]** (poison) — la implementación encarna el mapping ReAct de la Table 8.1.
