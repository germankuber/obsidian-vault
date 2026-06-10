---
title: Harness Responsibilities
source: https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture
author: Hamza Farooq, Aishwarya Ashok
created: 2026-06-10
tags:
  - ai/agents/architecture
aliases:
  - Harness Responsibilities
  - harness-responsibilities
  - Responsabilidades del Harness
  - Cinco Responsabilidades del Harness
---

# Harness Responsibilities

> [!note] Definición
> Las **cinco responsabilidades centrales** que **todo** [[Agent Harness|harness]] de producción debe cumplir, *"sin importar el lenguaje de implementación o el modelo de deployment"*. Son las **invariantes arquitectónicas** del harness.

## Las cinco responsabilidades

1. **Tool Execution** — El harness intercepta las tool calls generadas por el modelo y las traduce en operaciones reales del sistema: escrituras de archivos, comandos de shell, invocaciones de API; luego devuelve resultados **estructurados** al contexto del modelo. Ver [[Tool Calling]] / [[Function Calling]] para el formato de la llamada.

2. **Memory & Context Management** — El harness decide qué recuerda el agente entre turnos, tareas y sesiones, incluyendo **cómo resume, evita (evict) y recupera** contexto previo. (Los agentes **no** recuerdan entre sesiones por defecto: la memoria persistente requiere diseño explícito.)

3. **[[Sandboxing]]** — Aísla la ejecución del agente para que una tool call malformada, una **prompt injection adversaria** o un simple error de programación **no escalen en cascada** a daño irreversible sobre los sistemas que el agente opera.

4. **State Persistence** — Mantiene el entorno de trabajo del agente —archivos abiertos, estado de la branch, progreso de la tarea— a través de tareas **interrumpidas o reanudadas**, para que un agente pueda pausarse y reiniciarse sin perder contexto coherente.

5. **[[Permission Enforcement]]** — Determina a qué tools, archivos, repos y APIs externas está autorizado a acceder el agente, y **hace cumplir esos límites en tiempo de ejecución** en vez de confiar en que el modelo se autocontrole.

## Por qué importa

> [!warning] Invariantes arquitectónicas
> *"Cualquier sistema de agentes en producción al que le falte una está operando con una superficie de ataque abierta o con un radio de daño (blast radius) no controlado."* No son features opcionales: son lo que separa un agente real de un modelo que "sugiere" acciones.

> [!tip] Dónde vive la seguridad
> *"Safety lives in the harness, not the model."* El modelo solo **propone**; el harness **decide y ejecuta**. El sandboxing y el permission enforcement son los dos controles estructurales que hacen cumplir esto.

## References

- Fuente: [AI Agent Harnesses Explained](https://boringbot.substack.com/p/ai-agent-harnesses-explained-architecture) — Hamza Farooq, Aishwarya Ashok, 2026-05-08

## Related

- [[Agent Harness]]
- [[Sandboxing]]
- [[Permission Enforcement]]
- [[Harness Maturity Spectrum]]
