---
title: Cold Start
source: https://designgurus.substack.com/p/the-fundamentals-of-serverless-system
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/serverless
  - type/concept
  - status/permanent
aliases:
  - Cold Start
  - Cold Starts
  - Arranque en Frío
---

# Cold Start

> [!note] Definición
> La **latencia notable** que ocurre cuando una función serverless **despierta de un estado dormido**. Es el trade-off principal de [[Serverless]]: el proceso de alocación automatizada suena instantáneo pero requiere tiempo de procesamiento físico real.

## Por qué ocurre

- Un cold start sucede solo cuando el sistema debe **crear un entorno de ejecución nuevo desde cero**. Las tres operaciones que toman tiempo:
  1. Descargar el código de la app desde almacenamiento seguro.
  2. Crear el contenedor aislado (ver el proceso de alocación en [[AWS Lambda]]).
  3. Bootear el **runtime** del lenguaje de programación.
- Recién después de esta secuencia de inicialización la función puede procesar el request entrante.

## La conexión con scaling to zero

- Si una app escala completamente a cero (ver [[Scaling to Zero]]), el **próximo** request **siempre** experimentará un cold start: no hay ningún entorno activo para reusar.
- Para apps web interactivas, una demora de **varios segundos** crea una experiencia de usuario muy pobre.

## Cómo mitigarlo

- Los ingenieros deben **diseñar los sistemas con cuidado para minimizar la frecuencia** de estos cold starts. Las dos estrategias del provider: el [[Warm Start]] (reusar contenedores congelados) y **provisioned concurrency** (mantener contenedores siempre activos — detalle en [[Warm Start]]).

## References

- Fuente: [The Fundamentals of Serverless System Design](https://designgurus.substack.com/p/the-fundamentals-of-serverless-system) — Arslan Ahmad (Design Gurus), 2026-02-23

## Related

- [[Serverless]]
- [[Warm Start]]
- [[Scaling to Zero]]
- [[AWS Lambda]]
