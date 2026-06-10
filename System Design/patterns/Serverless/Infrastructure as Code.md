---
title: Infrastructure as Code
source: https://designgurus.substack.com/p/the-fundamentals-of-serverless-system
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/serverless
  - type/concept
  - status/permanent
aliases:
  - Infrastructure as Code
  - IaC
  - Infraestructura como Código
---

# Infrastructure as Code

> [!note] Definición
> La práctica técnica de **gestionar los data centers a través de archivos de definición legibles por máquina** (texto), en vez de configurar recursos cloud manualmente desde un dashboard. Resuelve la complejidad de operar sistemas serverless con cientos de piezas.

## El problema que resuelve

- Una app enterprise moderna suele requerir **cientos** de funciones serverless, bases de datos y network gateways interconectados.
- Gestionar todos esos recursos **manualmente desde un dashboard del browser es peligrosísimo**: clickear botones para configurar settings complejos lleva con frecuencia a **errores humanos severos y caídas del sistema**.

## Cómo funciona

- Los developers escriben **archivos de texto simples** que **declaran exactamente** qué recursos cloud necesita la app. Un solo archivo puede definir un [[API Gateway]], una base de datos primaria y 50 funciones serverless separadas.
- Esos archivos de configuración se guardan **junto al código fuente** de la app, en control de versiones.

## Deployments automatizados

- Cuando el equipo está listo para publicar, una herramienta de deployment automatizada **lee los archivos de configuración** y aprovisiona los recursos pedidos automáticamente. AWS provee una herramienta nativa para esto: **AWS CloudFormation**.
- Para cambiar algo (p. ej. un setting de DB), el ingeniero **modifica el archivo de texto**; la herramienta lee el cambio y lo aplica de forma segura al entorno cloud en vivo.

## Por qué importa

> [!tip] Documentado y repetible
> El enfoque basado en texto garantiza que la arquitectura completa quede **perfectamente documentada y altamente repetible**. Combinando serverless con deployments automatizados, **equipos chicos pueden mantener sistemas distribuidos masivos de forma segura**: el provider maneja el escalado del hardware, las herramientas manejan la configuración de recursos, y los developers se enfocan solo en escribir código.

## References

- Fuente: [The Fundamentals of Serverless System Design](https://designgurus.substack.com/p/the-fundamentals-of-serverless-system) — Arslan Ahmad (Design Gurus), 2026-02-23

## Related

- [[Serverless]]
- [[AWS Lambda]]
- [[API Gateway]]
