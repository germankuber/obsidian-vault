---
title: AWS Lambda
source: https://designgurus.substack.com/p/the-fundamentals-of-serverless-system
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/serverless
  - type/technology
  - status/permanent
aliases:
  - AWS Lambda
  - Lambda
---

# AWS Lambda

> [!note] Definición
> El servicio de cómputo **[[Serverless]]** de Amazon Web Services. Maneja la orquestación compleja necesaria para **ejecutar código automáticamente en respuesta a eventos de internet**.

> [!warning] No confundir con Lambda Architecture
> AWS Lambda (este servicio serverless) **no tiene relación** con [[Lambda Architecture]] (el patrón de procesamiento de datos batch + stream). Comparten el nombre "Lambda" y nada más.

## Cómo llega el request

- Los requests web externos suelen llegar a Lambda a través de un [[API Gateway]]: un servicio cloud dedicado que **recibe y rutea de forma segura** el tráfico entrante.
- El gateway **valida** los datos del request entrante y los **reenvía a la función serverless correcta**.

## El proceso de alocación de contenedor

Cuando Lambda recibe un trigger nuevo, prepara rápidamente un espacio para correr el código:

1. **Crea un execution environment** — un contenedor de software aislado, equipado con memoria y poder de procesamiento dedicados. AWS aloca **exactamente la cantidad de memoria** que el developer pidió en su configuración.
2. **Descarga el código** de la app desde almacenamiento digital seguro e **instala** el código en el contenedor recién creado.
3. **Inicia el runtime** del lenguaje — el **runtime** es el software subyacente necesario para ejecutar un lenguaje de programación específico.
4. El código **ejecuta**, procesa el payload de datos y genera un output final.

Este proceso de creación desde cero es exactamente lo que causa un [[Cold Start]]; reusar un contenedor ya armado es un [[Warm Start]].

## El modelo de costo: pay-per-execution

- A diferencia del cómputo cloud tradicional (tarifa horaria plana por VM sin importar la utilización), Lambda usa un billing **granular pay-per-execution**.
- El costo se calcula por el **número exacto de milisegundos** que el código corre activamente, más la **memoria asignada** a la función.
- Ejemplo: si el código tarda **exactamente 40 milisegundos**, el sistema factura solo 40 ms de cómputo.
- Esto hace a serverless muy costo-efectivo para tráfico impredecible — pero **código ineficiente que tarda varios segundos genera facturas enormes** con el tiempo.

## Balancear memoria y CPU (puzzle de optimización)

- En Lambda, **asignar más memoria otorga automáticamente más poder de procesamiento**. Una función con el doble de memoria procesa cálculos complejos significativamente más rápido.
- **Contraintuitivo**: a veces **subir** la memoria **baja** el costo total de la ejecución. El procesamiento extra permite que el código termine en la mitad del tiempo; como la duración total cae, el costo por milisegundos también cae.
- Los ingenieros deben monitorear métricas del sistema para encontrar el balance perfecto entre velocidad de ejecución y costo de memoria.

## References

- Fuente: [The Fundamentals of Serverless System Design](https://designgurus.substack.com/p/the-fundamentals-of-serverless-system) — Arslan Ahmad (Design Gurus), 2026-02-23

## Related

- [[Serverless]]
- [[Cold Start]]
- [[Warm Start]]
- [[API Gateway]]
- [[Stateless]]
- [[Infrastructure as Code]]
