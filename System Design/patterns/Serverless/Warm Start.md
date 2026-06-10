---
title: Warm Start
source: https://designgurus.substack.com/p/the-fundamentals-of-serverless-system
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/serverless
  - type/concept
  - status/permanent
aliases:
  - Warm Start
  - Warm Starts
  - Provisioned Concurrency
  - Arranque en Caliente
---

# Warm Start

> [!note] Definición
> La **reutilización de un contenedor ya inicializado** para ejecutar un request en **milisegundos**, sin la latencia de un [[Cold Start]]. Es la estrategia técnica que usan los cloud providers para reducir el impacto de la inicialización.

## Mecánica del warm start

- Tras procesar un request, la plataforma **NO destruye el contenedor de inmediato**: lo **congela** y lo mantiene disponible por un período de espera corto.
- Si llega otro request durante esa ventana de espera, la plataforma **reusa el contenedor congelado**.
- Como el código y el runtime **ya están inicializados**, la app ejecuta en milisegundos sin demora notable.

## Provisioned Concurrency (eliminar cold starts por completo)

- AWS ofrece una feature premium llamada **provisioned concurrency** que resuelve la inicialización **completamente**.
- Permite mantener un **número específico de contenedores totalmente inicializados y permanentemente activos**.

> [!warning] El trade-off del provisioned concurrency
> Elimina los cold starts del todo, pero el developer **paga tarifas continuas por el tiempo de cómputo ocioso** de esos contenedores siempre prendidos. Es volver, en parte, al modelo de pagar por capacidad reservada que [[Serverless]] busca evitar — se justifica solo cuando la latencia del cold start es inaceptable.

## References

- Fuente: [The Fundamentals of Serverless System Design](https://designgurus.substack.com/p/the-fundamentals-of-serverless-system) — Arslan Ahmad (Design Gurus), 2026-02-23

## Related

- [[Cold Start]]
- [[Serverless]]
- [[AWS Lambda]]
