---
title: RLHF
source: (conocimiento general — sin artículo fuente; nota escrita por Claude, verificar antes de citar)
author: —
created: 2026-06-10
tags:
  - ai/fundamentals
  - type/concept
  - status/stub
aliases:
  - RLHF
  - Reinforcement Learning from Human Feedback
  - Aprendizaje por Refuerzo con Feedback Humano
---

# RLHF

> [!warning] Nota sin fuente externa
> Esta nota se escribió desde conocimiento general (no destilada de un artículo). A diferencia del resto del vault, no tiene `source:` citable. Verificá los detalles antes de usarlos en algo serio, y enriquecela/reemplazala cuando importes un artículo real sobre el tema.

> [!note] Definición
> **Reinforcement Learning from Human Feedback**: técnica para alinear un modelo de lenguaje con las preferencias humanas usando aprendizaje por refuerzo, donde la señal de recompensa se deriva de comparaciones hechas por humanos en vez de una métrica fija.

## El pipeline clásico (3 fases)

1. **Supervised Fine-Tuning (SFT)** — se parte de un modelo pre-entrenado y se lo afina con ejemplos curados de "buenas" respuestas (demostraciones humanas). Le enseña el formato y el estilo de seguir instrucciones.
2. **Reward Model (RM)** — se recogen comparaciones humanas: dado un prompt y varias respuestas del modelo, un humano las ordena de mejor a peor. Con esos rankings se entrena un **modelo de recompensa** que aprende a puntuar qué tan "preferible" es una respuesta.
3. **RL (típicamente PPO)** — se optimiza el modelo de lenguaje para maximizar el score del reward model, usando un algoritmo de RL como **PPO (Proximal Policy Optimization)**. Se agrega una penalización **KL** contra el modelo SFT para que no se aleje demasiado y degenere.

## Por qué importa

- Convierte un modelo que "predice el siguiente token" en uno que **sigue instrucciones y responde de forma útil/segura** según criterio humano.
- La preferencia humana captura matices (tono, utilidad, seguridad) que una métrica automática no expresa bien.

## Variantes y evolución

- **RLAIF** (RL from AI Feedback) — reemplaza/complementa el feedback humano con feedback de otro modelo, para escalar la recolección de preferencias.
- **DPO (Direct Preference Optimization)** — alternativa más simple que evita entrenar un reward model y correr RL: optimiza directamente sobre los pares de preferencia. Muy adoptada por ser más estable y barata.
- **Constitutional AI** — usa un conjunto de principios ("constitución") para que el modelo critique y revise sus propias respuestas, reduciendo la dependencia de etiquetas humanas.

## Limitaciones

- **Reward hacking**: el modelo puede explotar fallas del reward model (optimizar el proxy, no la intención real) — el mismo patrón que en [[Offline vs Business Metrics]] de MLOps.
- Caro y complejo (recolección de preferencias + entrenamiento RL).
- La calidad depende de quién etiqueta y de cómo se definieron las preferencias.

## Conexión en el vault

- En [[Grounded Eval Harness]] / [[Generator-Evaluator Pattern]] se describe el loop de feedback evaluador→generador como *"estructuralmente análogo a RLHF"*: señales de corrección en inference time, sin re-entrenar pesos. La diferencia clave: RLHF **actualiza los pesos** del modelo; el harness corrige **en tiempo de inferencia** sin tocar el modelo.

## References

- (Sin fuente externa — completar al importar un artículo sobre RLHF/alignment.)

## Related

- [[Generator-Evaluator Pattern]]
- [[Grounded Eval Harness]]
- [[Offline vs Business Metrics]]
