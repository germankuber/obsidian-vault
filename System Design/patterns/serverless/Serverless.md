---
title: Serverless
source: https://designgurus.substack.com/p/the-fundamentals-of-serverless-system
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/serverless
  - type/concept
  - status/permanent
aliases:
  - Serverless
  - Serverless Computing
  - Computación sin servidor
---

# Serverless

> [!note] Definición
> **Serverless computing**: modelo donde el desarrollador **no gestiona, aprovisiona ni mantiene servidores físicos**. El hardware **sigue existiendo** y procesando datos en data centers remotos — "serverless" NO significa "sin servidores", significa que el **cloud provider asume toda la gestión de infraestructura**. El developer solo escribe el código de la aplicación y lo sube a la plataforma; el provider se encarga de ejecutarlo de forma segura.

## El problema que resuelve

- Predecir el tráfico entrante con precisión es un problema técnico muy difícil. Un pico súbito de requests satura el hardware limitado: los servidores se quedan sin memoria y la app crashea entera.
- La solución tradicional es **sobre-aprovisionar**: comprar mucho cómputo por adelantado y mantener decenas de servidores prendidos todo el tiempo para absorber picos. Pero cuando el tráfico baja en horas tranquilas, esas máquinas caras quedan **completamente ociosas** → las empresas sobre-gastan solo para mantener el sistema estable.
- Serverless elimina la gestión manual de hardware de la ecuación, y con ella el dilema de sobre-aprovisionar vs. crashear.

## Limitaciones del modelo tradicional (lo que serverless reemplaza)

- Antes, desplegar una app requería configurar una máquina física dedicada o una **virtual machine** (VM): un programa de software que se comporta exactamente como una computadora física. El equipo tenía que instalar el SO, configurar firewalls de seguridad y gestionar el ruteo de red.
- La VM quedaba prendida **cada hora del día** sin importar el uso real, y el SO requería actualizaciones de seguridad y mantenimiento constante.
- Ante un pico, el sistema dependía de auto-scaling para bootear VMs nuevas — pero **bootear un SO fresco tarda varios minutos**. Para cuando la máquina nueva estaba operativa, la app ya podía haber crasheado. La arquitectura tradicional **reaccionaba demasiado lento** a los picos, lo que forzaba a mantener capacidad ociosa excesiva de forma permanente.

## La división de labores

- **Cloud provider**: instala SOs, aplica parches de seguridad, monitorea almacenamiento, y escala la infraestructura **hacia arriba y hacia abajo automáticamente** según la demanda, sin intervención manual.
- **Developer**: solo escribe la lógica de negocio y sube el código.
- Resultado: el ciclo de desarrollo se acelera drásticamente; el equipo gasta su tiempo en business logic en vez de configurar redes.

## El paradigma: event-driven

- A diferencia de las apps tradicionales que corren continuamente en segundo plano esperando tráfico, serverless opera sobre [[Event-Driven Architecture]]: el código **solo se ejecuta en respuesta a triggers** específicos del sistema.
- Un **trigger** es una acción digital predefinida que despierta al código dormido. Ejemplos: un browser pidiendo una página, o una base de datos recibiendo información nueva.
- Cuando la plataforma detecta un trigger válido, activa al instante el código necesario para manejar el request; tras devolver el resultado, la app vuelve a su estado **dormido**.

## Conceptos centrales (notas atómicas)

- [[Scaling to Zero]] — remover **todo** el cómputo cuando no hay tráfico → costo cero en horas tranquilas. La innovación central.
- [[Cold Start]] — la latencia de arrancar un entorno de ejecución nuevo desde cero. El trade-off principal.
- [[Warm Start]] — reusar un contenedor congelado para ejecutar en milisegundos (+ provisioned concurrency).
- [[AWS Lambda]] — el servicio serverless de AWS; cómo orquesta y aloca los contenedores.
- [[Stateless]] — la restricción de no retener estado local entre ejecuciones.
- [[Infrastructure as Code]] — gestionar la infra con archivos de texto versionables.

## Cuándo conviene / trade-offs

> [!tip] Ideal para tráfico impredecible
> El billing granular por milisegundos (ver [[AWS Lambda]]) hace a serverless muy costo-efectivo para sistemas con patrones de tráfico **impredecibles** o spiky: pagás solo cuando el código corre.

> [!warning] No es gratis de adoptar
> Impone restricciones reales: las funciones deben ser [[Stateless]] (obliga a una DB externa), sufren [[Cold Start]] tras escalar a cero, y el modelo de costo premia código optimizado (código lento = facturas enormes). No es un reemplazo universal de las VMs.

## References

- Fuente: [The Fundamentals of Serverless System Design for Aspiring Developers](https://designgurus.substack.com/p/the-fundamentals-of-serverless-system) — Arslan Ahmad (Design Gurus), 2026-02-23

## Related

- [[Scaling to Zero]]
- [[Cold Start]]
- [[Warm Start]]
- [[AWS Lambda]]
- [[Stateless]]
- [[Infrastructure as Code]]
- [[Event-Driven Architecture]]
- [[Auto-Scaling]]
- [[API Gateway]]
