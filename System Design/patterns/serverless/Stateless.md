---
title: Stateless
source: https://designgurus.substack.com/p/the-fundamentals-of-serverless-system
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/serverless
  - type/concept
  - status/permanent
aliases:
  - Stateless
  - Stateless Application
  - Statelessness
  - Sin Estado
---

# Stateless

> [!note] Definición
> Una aplicación **stateless** **no retiene ninguna memoria ni dato local entre ejecuciones separadas**. Cada ejecución opera en **aislamiento total** de eventos o interacciones previas. Es una restricción arquitectónica obligatoria en [[Serverless]].

## Por qué serverless obliga a ser stateless

- Cuando el cloud provider **destruye un execution environment** (ver [[Scaling to Zero]]), **cualquier dato guardado en el contenedor local desaparece para siempre**.
- Por lo tanto, los developers **no pueden confiar en la memoria local** del contenedor para almacenar datos temporales de la app.
- Esto exige un cambio fundamental en cómo se diseñan los sistemas: el estado **no vive en el proceso**.

## La consecuencia: base de datos externa obligatoria

Para construir apps confiables sobre funciones stateless, el patrón es:

1. Cada vez que la función serverless ejecuta, **lee** la información necesaria desde una **base de datos externa**.
2. Antes de terminar la ejecución, **guarda** cualquier dato nuevo de vuelta en la base de datos externa de forma segura.

El estado persistente vive **siempre afuera** de la función, en un servicio de datos externo.

## References

- Fuente: [The Fundamentals of Serverless System Design](https://designgurus.substack.com/p/the-fundamentals-of-serverless-system) — Arslan Ahmad (Design Gurus), 2026-02-23

## Related

- [[Serverless]]
- [[AWS Lambda]]
- [[Scaling to Zero]]
