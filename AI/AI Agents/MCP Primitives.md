---
title: MCP Primitives
source: Architecture overview - Model Context Protocol
author: Model Context Protocol (modelcontextprotocol.io)
url: https://modelcontextprotocol.io/docs/learn/architecture
created: 2026-06-11
tags:
  - ai/agents/protocol
  - type/concept
  - status/permanent
aliases:
  - MCP Primitives
  - Primitivos de MCP
  - MCP Tools
  - MCP Resources
  - MCP Prompts
---

# MCP Primitives

> [!note] Definición
> Los **primitivos** son las unidades que definen **qué se ofrecen client y server entre sí**: tipos de información contextual a compartir y rango de acciones a ejecutar. Son el concepto central del data layer de [[MCP]].

Cada tipo de primitivo expone métodos estándar:
- **`*/list`** — discovery (descubrir los disponibles).
- **`*/get`** / **`*/read`** — retrieval (obtener uno).
- y a veces **ejecución** (ej. `tools/call`).

Los clients usan `*/list` para descubrir primitivos; los **listados son dinámicos** (pueden cambiar en runtime → ver [[MCP Lifecycle]] notifications).

## 3 primitivos de SERVER

### MCP Tools (Tools — primitivo de server)
- Funciones **ejecutables** que la app de IA invoca para realizar acciones (operaciones de archivo, llamadas a API, queries a DB).
- Métodos: **`tools/list`** (discovery) + **`tools/call`** (ejecución).
- Campos de cada tool: `name`, `title`, `description`, `inputSchema`.

> [!tip] Desambiguación
> MCP Tools es **cómo el server expone** funciones; [[Function Calling]] / [[Tool Calling]] es **cómo el modelo propone** una llamada. Distintas capas: el server publica el catálogo (con `inputSchema`), el modelo decide invocar.

### Resources
- **Fuentes de datos** que proveen contexto a la app de IA: contenido de archivos, registros de DB, respuestas de API.
- Métodos: **`resources/list`** + **`resources/read`**.

### Prompts
- **Plantillas reutilizables** que estructuran interacciones con los modelos: system prompts, ejemplos few-shot.
- (Distinto del concepto genérico de "prompt": acá es un primitivo MCP versionado y descubrible.)

## 3 primitivos de CLIENT

Permiten interacciones más ricas a los autores de servers (el server pide algo al client):

- **Sampling** — el server pide **completions del LLM del host** a través del client. Útil cuando el autor del server quiere acceso a un LLM pero **manteniéndose model-independent** y **sin incluir un SDK de LLM** en su server. Método: **`sampling/createMessage`**.
- **Elicitation** — el server pide **información adicional al usuario** (más datos, o confirmación de una acción). Método: **`elicitation/create`**.
- **Logging** — el server envía **mensajes de log al client** para debugging y monitoreo.

## Utility cross-cutting

- **Tasks (Experimental)** — wrappers de **ejecución durable** que habilitan **recuperación diferida de resultados** y **status tracking** para requests MCP. Casos: cómputos costosos, automatización de workflows, batch processing, operaciones multi-step. ⚠️ Marcado **Experimental** en la spec.

## Ejemplo — server de base de datos

Un server que da contexto sobre una DB puede exponer a la vez: **tools** para consultar la DB, un **resource** con el schema de la DB, y un **prompt** con ejemplos few-shot para interactuar con esas tools.

### Discovery — `tools/list` (response, resumida)

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "calculator_arithmetic",
        "title": "Calculator",
        "description": "Perform basic arithmetic operations",
        "inputSchema": {
          "type": "object",
          "properties": {
            "expression": { "type": "string" }
          },
          "required": ["expression"]
        }
      },
      {
        "name": "weather_current",
        "title": "Current Weather",
        "description": "Get current weather for a location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": { "type": "string" },
            "units": {
              "type": "string",
              "enum": ["metric", "imperial", "kelvin"],
              "default": "metric"
            }
          },
          "required": ["location"]
        }
      }
    ]
  }
}
```

El client combina las tools de **todos** los servers en un [[Unified tool registry|registro unificado]] accesible al LLM:

```python
available_tools = []
for session in app.mcp_server_sessions():
    tools_response = await session.list_tools()
    available_tools.extend(tools_response.tools)
conversation.register_available_tools(available_tools)
```

### Ejecución — `tools/call`

Request (`name` + `arguments` conformes al `inputSchema`):

```json
{
  "jsonrpc": "2.0", "id": 3, "method": "tools/call",
  "params": { "name": "weather_current", "arguments": { "location": "San Francisco", "units": "imperial" } }
}
```

Response — usa el **sistema flexible de content** (array `content` con Content Types `type:"text"`, además soporta Structured Output):

```json
{
  "jsonrpc": "2.0", "id": 3,
  "result": { "content": [ { "type": "text", "text": "Current weather in San Francisco: 68°F, partly cloudy..." } ] }
}
```

```python
async def handle_tool_call(conversation, tool_name, arguments):
    session = app.find_mcp_session_for_tool(tool_name)
    result = await session.call_tool(tool_name, arguments)
    conversation.add_tool_result(result.content)
```

## References

- Fuente: [Architecture overview - Model Context Protocol](https://modelcontextprotocol.io/docs/learn/architecture) — modelcontextprotocol.io, 2026.

## Related

- [[MCP]] — el hub del protocolo.
- [[MCP Lifecycle]] — discovery dinámico vía notifications.
- [[Tool Calling]] · [[Function Calling]] — cómo el modelo propone (no cómo el server expone).
