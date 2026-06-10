---
title: Function Calling
source: (conocimiento general — sin artículo fuente; nota escrita por Claude, verificar antes de citar)
author: —
created: 2026-06-10
tags:
  - ai/agents/architecture
  - type/concept
  - status/stub
aliases:
  - Function Calling
  - Structured Tool Calling
  - Llamada a Funciones
---

# Function Calling

> [!warning] Nota sin fuente externa
> Escrita desde conocimiento general, no destilada de un artículo. Sin `source:` citable. Verificá los detalles antes de citarlos; enriquecela al importar un artículo real sobre el tema.

> [!note] Definición
> La versión **estructurada** de [[Tool Calling]]: en vez de texto libre, el modelo devuelve **argumentos que matchean un schema** definido (típicamente JSON Schema). Ej: `function get_weather, location: "Mumbai", unit: "celsius"`.

## Por qué estructurado

- **Texto libre es flexible; salida estructurada es controlable.** Un schema permite parsear, validar, rutear, testear y rechazar la llamada de forma determinística.
- Mentalidad: **"schema primero, ejecución después"** — el sistema sabe exactamente qué forma tiene la llamada antes de ejecutar nada.
- Crítico en producción: sin estructura, extraer la intención del modelo de un blob de texto es frágil.

## Beneficios

- **Parsing** sin heurísticas ni regex frágiles.
- **Validación** de tipos/campos contra el schema.
- **Routing** a la función correcta.
- **Testing** y **rechazo** de llamadas malformadas antes de ejecutarlas.

## Relación con el vault

- Es la forma concreta en que un [[Agent Harness|harness]] recibe las [[Tool Calling|tool calls]] para poder validarlas en [[Permission Enforcement|tiempo de ejecución]].
- El patrón de **structured output** (modelo forzado a devolver un objeto que matchea un schema) es el mismo que usa el [[Grounded Eval Harness]] con su `EvaluatorOutput` (un BaseModel de Pydantic).

## References

- (Sin fuente externa — completar al importar un artículo sobre function calling / structured output.)

## Related

- [[Tool Calling]]
- [[Agent Harness]]
- [[Grounded Eval Harness]]
- [[Permission Enforcement]]
