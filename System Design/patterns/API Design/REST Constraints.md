---
title: REST Constraints
source: https://designgurus.substack.com/p/what-is-a-rest-api-a-system-design
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/api
  - type/pattern
  - status/permanent
aliases:
  - REST Constraints
  - REST Principles
  - Principios REST
---

# REST Constraints

> [!note] Definición
> Los **cinco constraints** que un sistema debe cumplir para ser verdaderamente RESTful. Garantizan que el sistema sea escalable, confiable y fácil de modificar con el tiempo. Son las reglas de [[REST API]].

## 1. Client-Server Separation

- Separación estricta de responsabilidades. El **client** es la interfaz que hace el request; el **server** es el backend que tiene los datos y procesa la lógica de negocio.
- Deben ser **completamente independientes**: el client nunca se ocupa de algoritmos de storage ni queries de DB; el server nunca se ocupa de UI ni de cómo se muestran los datos.
- Por esa separación, **ambos evolucionan independientemente**: se puede reescribir todo el frontend sin tocar una línea del backend. Modularidad esencial para equipos grandes trabajando en paralelo.

## 2. Statelessness (el más crítico)

> [!tip] El concepto más crítico del diseño de sistemas distribuidos
> El server **no guarda memoria de requests pasados**. Cada request del client debe contener **toda la información necesaria** para procesar esa acción. El server no mantiene sesiones de login en memoria activa; si el client necesita datos restringidos, manda sus credenciales de autenticación **en cada llamada**.

- Es lo que permite **escalar horizontalmente**: para escalar, se agregan más servers al cluster. Como no guardan estado, un router puede mandar un request a **cualquier server disponible** al azar — cualquiera puede procesarlo porque todo el contexto viene en el payload del request.
- Ver [[Stateful vs Stateless]].

## 3. Cacheability

- Las llamadas de red son lentas y caras en procesamiento. REST enfatiza la **cacheabilidad**: las respuestas del server deben **definir claramente** si el client puede cachear los datos localmente.
- Si una respuesta es cacheable, el client guarda una copia local y la reusa la próxima vez que necesite el mismo dato → **reduce carga de DB y mejora la velocidad**.

## 4. Uniform Interface

- Una **interfaz uniforme** simplifica la arquitectura. Todos los recursos se acceden con un método estandarizado, consistente y predecible.
- Se interactúa con **protocolos de red universales** (no comandos propietarios). Los recursos se identifican con **URLs estándar**; el server devuelve datos en formatos estándar como **JSON**.
- Esta uniformidad permite que programas en lenguajes distintos se comuniquen sin fricción.

## 5. Layered System

- Un sistema distribuido rara vez es un solo client hablando con un solo DB server. **Layered system** = el client no puede saber si está conectado directo al server final o a un **intermediario**.
- Intermediarios: **firewalls de seguridad, [[Load Balancing|load balancers]], proxy servers**. Un load balancer distribuye el tráfico entre varios backends para prevenir sobrecarga.
- Como la arquitectura es lógicamente en capas, se pueden agregar/quitar estos componentes **sin romper** la comunicación client-server.

## References

- Fuente: [The Anatomy of a REST API](https://designgurus.substack.com/p/what-is-a-rest-api-a-system-design) — Arslan Ahmad (Design Gurus), 2026-02-19

## Related

- [[REST API]]
- [[HTTP Methods]]
- [[Stateful vs Stateless]]
- [[Load Balancing]]
