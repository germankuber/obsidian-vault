---
title: MCP Lifecycle
source: Architecture overview - Model Context Protocol
author: Model Context Protocol (modelcontextprotocol.io)
url: https://modelcontextprotocol.io/docs/learn/architecture
created: 2026-06-11
tags:
  - ai/agents/protocol
  - type/concept
  - status/permanent
aliases:
  - MCP Lifecycle
  - Ciclo de vida de MCP
  - MCP Notifications
  - Capability negotiation
---

# MCP Lifecycle

> [!note] Definición
> [[MCP]] es un protocolo **stateful**. El **lifecycle** gestiona el ciclo de vida de la conexión y su propósito central es **negociar las capabilities** que client y server soportan **antes** de operar.

## Initialize handshake

Secuencia de 3 mensajes [[JSON-RPC 2.0]]:

1. El **client** envía `initialize` (con `protocolVersion`, `capabilities`, `clientInfo`):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": { "elicitation": {} },
    "clientInfo": { "name": "example-client", "version": "1.0.0" }
  }
}
```

2. El **server** responde (con su `protocolVersion`, sus `capabilities` y `serverInfo`):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": {}
    },
    "serverInfo": { "name": "example-server", "version": "1.0.0" }
  }
}
```

3. El **client** confirma con una notification (sin `id`, sin respuesta):

```json
{ "jsonrpc": "2.0", "method": "notifications/initialized" }
```

> [!example] Qué cumple la inicialización
> - **Protocol Version Negotiation** — `protocolVersion` `2025-06-18`.
> - **Capability Discovery** — bloques `capabilities`. El client declaró `"elicitation": {}` (habilita `elicitation/create`); el server declaró `"tools": {"listChanged": true}` (habilita la notificación `tools/list_changed`) y `"resources": {}` (habilita `resources/list`, `resources/read`).
> - **Identity Exchange** — `clientInfo` / `serverInfo`.

```python
# Pseudo Code
async with stdio_client(server_config) as (read, write):
    async with ClientSession(read, write) as session:
        init_response = await session.initialize()
        if init_response.capabilities.tools:
            app.register_mcp_server(session, supports_tools=True)
        app.set_server_ready(session)
```

## Capability negotiation

La **mecánica de negociación de capabilities vive acá**: en `initialize`, cada lado declara qué primitivos y notifications soporta, y eso **condiciona qué mensajes son válidos** durante la sesión. (El hub [[MCP]] solo la menciona.)

## MCP Notifications

Mensajes [[JSON-RPC 2.0]] con tres propiedades:

- **No Response Required** — son notifications, **sin `id`**, no esperan respuesta.
- **Capability-Based** — solo se envían si la capability fue declarada (ej. el server solo emite `tools/list_changed` si declaró `"listChanged": true`).
- **Event-Driven** — disparadas por cambios reales de estado.

Ejemplo — el server avisa que cambió su lista de tools:

```json
{ "jsonrpc": "2.0", "method": "notifications/tools/list_changed" }
```

El client reacciona pidiendo `tools/list` **de nuevo** (ciclo de refresh), manteniendo su [[Unified tool registry|registro]] al día:

```python
async def handle_tools_changed_notification(session):
    tools_response = await session.list_tools()
    app.update_available_tools(session, tools_response.tools)
    if app.conversation.is_active():
        app.conversation.notify_llm_of_new_capabilities()
```

> [!tip] Por qué importan las notifications
> **Dynamic Environments** (las capabilities cambian en runtime), **Efficiency** (refresh dirigido en vez de polling), **Consistency** (client y server alineados) y **Real-time Collaboration**.

## References

- Fuente: [Architecture overview - Model Context Protocol](https://modelcontextprotocol.io/docs/learn/architecture) — modelcontextprotocol.io, 2026.

## Related

- [[MCP]] — el hub del protocolo.
- [[MCP Primitives]] — lo que las notifications de `*/list_changed` mantienen al día.
- [[MCP Transports]] — el lifecycle corre sobre cualquier transporte.
