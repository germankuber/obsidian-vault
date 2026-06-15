---
title: AI Fundamentals — Mapa del tema
created: 2026-06-10
tags:
  - ai/fundamentals
  - type/moc
  - status/permanent
aliases:
  - AI Fundamentals
  - AI Fundamentals MOC
  - Fundamentos de IA
updated: 2026-06-15
---

# AI Fundamentals — Mapa del tema

> [!note] Cómo usar esta nota
> Índice (MOC) de `AI/fundamentals`: conceptos **transversales** de IA/LLM que no pertenecen a un subdominio específico (no son solo de RAG, agents o MLOps) sino que los cruzan a todos. Abrí esta nota, no la carpeta.

## 🧠 Entrenamiento y alineación

- [[RLHF]] — alinear un LLM con preferencias humanas vía RL (SFT → reward model → PPO). Incluye RLAIF, DPO, Constitutional AI.

## ✅ Calidad, evaluación y confiabilidad

- [[Hallucinations]] — salida plausible pero incorrecta/no respaldada; el problema es la confianza sin incertidumbre.
- [[Grounding]] — atar la respuesta a evidencia verificable; la principal defensa contra las alucinaciones.
- **[[_Evals|Evals]]** 📁 — la evaluación de sistemas IA/LLM creció a sub-dominio propio (`AI/Evals/`): error analysis bottom-up, LLM-as-Judge, ground truth. Abrí su MOC.

## 🧱 Conceptos base de inferencia

Fundamentos transversales que sostienen el sub-dominio **[[_Inference|Inference]]** (`AI/Inference/`):

- [[Tokens]] — la unidad atómica que procesa un LLM (≈ palabra o fragmento de palabra).
- [[KV Cache]] — valores intermedios de atención (prefill→decode) y **hub de la evolución del manejo de memoria del serving de LLMs** (6 eras: naive → PagedAttention → heterogéneo → distribuido → unificado).
- [[Quantization]] — comprimir la precisión de los pesos (16→8/4-bit) para acelerar ambas fases de inferencia.

## 🌱 Por escribir (semillas del grafo)

Conceptos base que aún no tienen nota propia — candidatos a promover cuando un artículo aporte material (varios vienen del pendiente "25 AI Concepts"):

- [[Attention]] · [[Transformers]] · [[Embeddings]] · [[Context Window]]
- [[Fine-tuning]] · [[LoRA]] · [[Distillation]] · [[Mixture-of-Experts]]
- [[Prompting]] · [[Context Engineering]]

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "AI/AI Fundamentals"
WHERE file.name != this.file.name
SORT file.name ASC
```
