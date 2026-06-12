---
title: MCP Transports
source: Architecture overview - Model Context Protocol
author: Model Context Protocol (modelcontextprotocol.io)
url: https://modelcontextprotocol.io/docs/learn/architecture
created: 2026-06-11
tags:
  - ai/agents/protocol
  - type/concept
  - status/permanent
aliases:
  - MCP Transports
  - Transportes de MCP
  - Stdio transport
  - Streamable HTTP transport
---

# MCP Transports

> [!note] Definición
> El **transport layer** de [[MCP]] gestiona los canales de comunicación y la autenticación entre client y server (establecimiento de conexión, message framing, comunicación segura). MCP soporta **2 transportes**, y el **mismo [[JSON-RPC 2.0]] corre sobre ambos** — el transporte queda abstraído del protocolo.

## Stdio transport

- Usa **streams estándar (stdin/stdout)** para comunicación directa entre **procesos locales en la misma máquina**.
- **Rendimiento óptimo, sin overhead de red.**
- Típicamente sirve a **un solo client** (1:1, proceso local lanzado por el host).
- Caso típico: el host lanza el server localmente (ej. filesystem server de Claude Desktop).

## Streamable HTTP transport

- Usa **HTTP POST** para mensajes **client→server**, con **[[Server-Sent Events]] (SSE) opcional** para streaming server→client.
- Habilita comunicación con **servers remotos**.
- Soporta métodos de **autenticación HTTP estándar**: **bearer tokens, API keys, custom headers**. **MCP recomienda usar [[OAuth]]** para obtener los tokens de autenticación.
- Típicamente sirve a **muchos clients**.

## STDIO vs Streamable HTTP

| Dimensión        | Stdio                          | Streamable HTTP                          |
|------------------|--------------------------------|------------------------------------------|
| Ubicación server | **Local** (misma máquina)      | **Remoto**                               |
| Canal            | stdin/stdout                   | HTTP POST + **SSE opcional**             |
| Performance      | **Óptimo**, sin overhead de red| Overhead de red                          |
| Auth             | (proceso local, sin auth de red)| bearer tokens / API keys / headers; **OAuth recomendado** |
| Nº de clients    | típicamente **1**              | típicamente **muchos**                   |

> [!tip] Cómo elegir
> Server local empaquetado con el host → **Stdio** (simple, rápido). Server compartido/remoto multi-tenant → **Streamable HTTP** con [[OAuth]].

## References

- Fuente: [Architecture overview - Model Context Protocol](https://modelcontextprotocol.io/docs/learn/architecture) — modelcontextprotocol.io, 2026.

## Related

- [[MCP]] — el hub del protocolo.
- [[Server-Sent Events]] — el stream server→cliente que usa el transporte HTTP.
- [[OAuth]] — auth recomendada para el transporte remoto.
- [[MCP Lifecycle]] — el lifecycle stateful que corre sobre cualquier transporte.
