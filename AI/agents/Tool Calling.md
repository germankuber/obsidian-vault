---
title: Tool Calling
source: (conocimiento general — sin artículo fuente; nota escrita por Claude, verificar antes de citar)
author: —
created: 2026-06-10
tags:
  - ai/agents/architecture
  - type/concept
  - status/stub
aliases:
  - Tool Calling
  - Tool Use
  - Uso de Herramientas
---

# Tool Calling

> [!warning] Nota sin fuente externa
> Escrita desde conocimiento general, no destilada de un artículo. Sin `source:` citable. Verificá los detalles antes de citarlos; enriquecela al importar un artículo real sobre el tema.

> [!note] Definición
> Mecanismo por el cual un LLM **interactúa con sistemas externos** (calculadora, base de datos, búsqueda, ejecutor de código, API, calendario, etc.) en vez de solo generar texto. El modelo **propone** una llamada a una herramienta; el sistema la ejecuta y le devuelve el resultado al contexto.

## El ciclo

1. El modelo, dado un prompt y un set de herramientas disponibles, **propone** una tool call (qué herramienta y con qué argumentos).
2. El **sistema valida** la llamada (¿está permitida? ¿argumentos válidos?).
3. La **aplicación ejecuta** la operación real.
4. El **resultado vuelve** al contexto del modelo, que sigue razonando.

> [!tip] Principio clave
> Una tool call **no es prueba de que la acción ocurrió** — es una *solicitud*. "El modelo puede pedir; el sistema debe decidir." Por eso el control de permisos, acceso a datos y efectos secundarios queda en el software, no en el modelo. Ver [[Permission Enforcement]] y [[Agent Harness]].

## Relación con conceptos del vault

- Es lo que un [[Agent Harness|harness]] **intercepta y valida** — la primera de las [[Harness Responsibilities|cinco responsabilidades]] (Tool Execution).
- [[Function Calling]] es la versión **estructurada** de tool calling (argumentos que matchean un schema).
- Un [[AI Framework]] (LangChain, etc.) ofrece abstracciones para definir tools, pero **no** valida permisos ni aísla la ejecución — eso es del harness.

## References

- (Sin fuente externa — completar al importar un artículo sobre tool use / function calling.)

## Related

- [[Function Calling]]
- [[Agent Harness]]
- [[Harness Responsibilities]]
- [[Permission Enforcement]]
