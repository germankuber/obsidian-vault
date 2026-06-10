---
title: Scaling to Zero
source: https://designgurus.substack.com/p/the-fundamentals-of-serverless-system
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/serverless
  - type/concept
  - status/permanent
aliases:
  - Scaling to Zero
  - Scale to Zero
  - Escalar a Cero
---

# Scaling to Zero

> [!note] Definición
> El proceso automatizado de **remover todos los recursos de cómputo** cuando una aplicación no recibe tráfico. Es la innovación más potente del modelo [[Serverless]]: su elasticidad hacia abajo extrema.

## Mecánica

- Si la app recibe **cero** requests entrantes, la infraestructura se apaga por completo: el cloud provider remueve **toda instancia activa** de la app de los servidores del data center.
- El código de la app **descansa en almacenamiento digital** sin consumir ciclos de procesador ni memoria. En este estado dormido, el footprint de recursos activos es de **cero absoluto**.

## Beneficio financiero

- Los cloud providers **solo cobran cuando los recursos están procesando datos activamente**.
- Como la app escala a cero en períodos tranquilos, el costo de hosting también **baja a exactamente cero**. No hay capacidad ociosa que pagar.

## El otro lado del escalado: hacia arriba (concurrency)

- La misma arquitectura maneja picos súbitos con igual eficiencia escalando **horizontalmente** (ver [[Horizontal Scaling]] / [[Auto-Scaling]]).
- Ante una ola masiva de requests, la plataforma aprovisiona al instante un **entorno de ejecución dedicado y separado por cada request entrante**. Si llegan 5.000 requests simultáneos, el sistema crea **5.000 entornos independientes en paralelo**.
- Esta ejecución simultánea se llama **concurrency**: garantiza que cada request reciba procesamiento inmediato sin esperar en cola.
- Cuando el pico baja, el provider **destruye automáticamente** los entornos innecesarios y la arquitectura se encoge para igualar el tráfico actual. Esto elimina la necesidad de capacity planning manual.

> [!warning] El costo de escalar a cero
> Si la app escala completamente a cero, el **próximo** request siempre sufrirá un [[Cold Start]] (hay que reconstruir el entorno desde cero). Escalar a cero ahorra dinero pero introduce latencia en el primer request tras la inactividad.

## References

- Fuente: [The Fundamentals of Serverless System Design](https://designgurus.substack.com/p/the-fundamentals-of-serverless-system) — Arslan Ahmad (Design Gurus), 2026-02-23

## Related

- [[Serverless]]
- [[Cold Start]]
- [[Auto-Scaling]]
- [[Horizontal Scaling]]
