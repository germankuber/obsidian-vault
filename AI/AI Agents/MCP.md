---
title: MCP
source: Architecture overview - Model Context Protocol
author: Model Context Protocol (modelcontextprotocol.io)
url: https://modelcontextprotocol.io/docs/learn/architecture
created: 2026-06-11
tags:
  - ai/agents/protocol
  - type/concept
  - status/permanent
aliases:
  - MCP
  - Model Context Protocol
---

# MCP

> [!note] Definición
> El **Model Context Protocol (MCP)** es un protocolo **cliente-servidor basado en [[JSON-RPC 2.0]]** que estandariza cómo las aplicaciones de IA obtienen **contexto** desde servidores externos (tools, datos, plantillas). Los SDK abstraen muchos detalles, así que el *data layer protocol* es la parte más útil para el developer.

> [!important] Scope acotado
> *"MCP se enfoca SOLO en el protocolo de intercambio de contexto — NO dicta cómo las apps usan los LLM ni cómo gestionan el contexto"*. MCP define el canal y los mensajes; la estrategia de uso del modelo es del lado de la aplicación.

## Los 3 participantes

Arquitectura **cliente-servidor**: un **MCP Host** abre conexiones con uno o más **MCP Servers**, y crea **un MCP Client por cada server** (relación dedicada **1:1** client↔server).

- **MCP Host** — la aplicación de IA (ej. Claude Code, Claude Desktop, VS Code) que coordina y gestiona uno o varios MCP clients.
- **MCP Client** — componente que mantiene una conexión **dedicada 1:1** con un MCP server y obtiene contexto de él para que lo use el host.
- **MCP Server** — programa que provee contexto a los clients. Puede correr **local (transporte STDIO)** o **remoto (transporte HTTP)**. "MCP server" refiere al **programa**, independientemente de dónde corra: ej. el *filesystem server* que Claude Desktop lanza localmente vía STDIO es un server *local*; el *Sentry MCP server* oficial que corre en la plataforma Sentry vía Streamable HTTP es un server *remoto*.

> [!example] Un client por server
> VS Code actúa como MCP host. Al conectarse al **Sentry MCP server** instancia un MCP client que mantiene esa conexión; al conectarse además al **filesystem server** local, instancia un **segundo** MCP client. Cada server tiene su propio client dedicado.

## Las 2 capas

MCP consiste en dos capas; conceptualmente el **data layer es la capa interna** y el **transport layer la externa**.

```
┌─────────────────────────────────────────────┐
│  MCP Host (app de IA)                         │
│   └─ MCP Client ──┐                           │
│                   │  Data layer (interna)     │
│                   │  JSON-RPC 2.0:            │
│                   │  lifecycle + primitivos   │
│                   ▼                           │
│            Transport layer (externa)          │
│            STDIO  /  Streamable HTTP          │
└───────────────────┬───────────────────────────┘
                    ▼
              MCP Server (contexto)
```

- **Data layer** (interna) — protocolo de intercambio basado en **[[JSON-RPC 2.0]]** que define estructura y semántica de los mensajes. Incluye: gestión de **lifecycle** (inicialización, negociación de capabilities, terminación), **server features** ([[MCP Primitives|tools, resources, prompts]]), **client features** (sampling, elicitation, logging) y **utility features** (notifications, progress tracking). Detalle de los primitivos en [[MCP Primitives]]; del lifecycle y las notifications en [[MCP Lifecycle]].
- **Transport layer** (externa) — gestiona los canales de comunicación y la autenticación: establecimiento de conexión específico del transporte, **message framing** y autorización. Soporta dos transportes (STDIO y Streamable HTTP) y **abstrae el transporte del protocolo**, de modo que el **mismo JSON-RPC 2.0 corre sobre todos**. Detalle en [[MCP Transports]].

## Capability negotiation

En el handshake `initialize`, client y server **declaran sus capabilities** mutuas (qué primitivos y notifications soportan) antes de operar. Es el motor del lifecycle stateful; la mecánica completa vive en [[MCP Lifecycle]].

## Posición en el stack agéntico

- MCP es la **"Layer 2" del stack de interoperabilidad agéntico**: integración **vertical / intra-org**, agente↔herramientas ("AI agent connects to tools").
- Su contraparte **horizontal / inter-org** es [[A2A]] (agente↔agente, "AI agents connect to each other"), una capa por encima.
- Se apoya en y complementa: [[Function Calling]] y [[Tool Calling]] (cómo el modelo propone llamadas), [[Orchestrator]] (secuenciación), [[AgentOps]] (operación/observabilidad).

> [!tip] MCP vs Function/Tool Calling
> [[Tool Calling]] y [[Function Calling]] son cómo el **modelo propone** una llamada; MCP es cómo un **server expone** tools/datos/prompts y cómo el host los **descubre y ejecuta** sobre un protocolo estándar. Son capas distintas que se componen.

## El proyecto MCP

- **MCP Specification** — requisitos de implementación para clients y servers.
- **MCP SDKs** — SDKs por lenguaje que implementan el protocolo.
- **MCP Development Tools** — incluye el [[MCP Inspector]] (github.com/modelcontextprotocol/inspector) para desarrollar y depurar servers/clients.
- **MCP Reference Server Implementations** — servers de referencia (github.com/modelcontextprotocol/servers).

## Unified tool registry

Patrón del lado de la app: el host **combina las tools de todos los servers conectados en un único registro** ([[Unified tool registry]]) accesible al LLM, de modo que el modelo ve un catálogo unificado sin importar de qué server viene cada tool.

## References

- Fuente: [Architecture overview - Model Context Protocol](https://modelcontextprotocol.io/docs/learn/architecture) — modelcontextprotocol.io, 2026.

## Related

- [[MCP Primitives]] — los 7 primitivos (tools/resources/prompts + sampling/elicitation/logging + tasks).
- [[MCP Transports]] — STDIO vs Streamable HTTP.
- [[MCP Lifecycle]] — handshake `initialize` + notifications.
- [[A2A]] — la capa horizontal agente↔agente (contraparte de MCP).
- [[Tool Calling]] · [[Function Calling]] — cómo el modelo propone llamadas.
- [[Orchestrator]] · [[AgentOps]]
