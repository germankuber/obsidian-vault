---
title: Paragraph-Based Chunking
source: https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking
author: togoAI Labs
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Paragraph-Based Chunking
  - paragraph-based-chunking
---

# Paragraph-Based Chunking

> [!note] Definición
> Partir en límites de **párrafo** (doble salto de línea o marcadores) y acumular
> párrafos hasta el límite de tamaño. Si un solo párrafo ya excede el máximo, cae
> a [[Sentence-Based Chunking|corte por oraciones]]. La **mejor coherencia** del set.

## Cómo funciona

- Separa por `"\n\n"` → acumula párrafos hasta el límite de tamaño.
- **Fallback graceful**: párrafo solo > `max_chunk_size` → se parte por oraciones.

```python
paragraphs = text.split("\n\n")
chunks, current_chunk, current_length = [], [], 0

for para in paragraphs:
    para_length = len(tokenizer.encode(para))
    if para_length > max_chunk_size:
        # párrafo solo demasiado grande → fallback a oraciones
        if current_chunk:
            chunks.append("\n\n".join(current_chunk))
            current_chunk, current_length = [], 0
        chunks.extend(sentence_chunk(para, max_chunk_size))
    elif current_length + para_length > max_chunk_size:
        chunks.append("\n\n".join(current_chunk))
        current_chunk = [para]
        current_length = para_length
    else:
        current_chunk.append(para)
        current_length += para_length
```

**Por qué funciona**
- El párrafo es una unidad semántica **diseñada deliberadamente por el autor**
  para expresar una idea/argumento completo. Respetarlo mantiene su estructura e
  intención → los chunks más coherentes.

## Números

- Preservación de contexto: excelente (mantiene la estructura del autor).
- Coherencia: **0.84/1.0** (la más alta del set).
- Retrieval en documentación técnica: **76-80% MRR@10**.
- Especialmente fuerte en reportes, manuales, documentación narrativa.

## Cuándo usarlo

> [!tip]
> - Contenido narrativo: blog posts, noticias, libros, ensayos.
> - Reportes y documentación con estructura de párrafos clara.
> - Cuando la coherencia temática es crítica.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - Formatos sin párrafos claros (código, listas, tablas, chat logs).
> - Documentos con saltos de párrafo inconsistentes o ausentes (PDFs mal
>   extraídos, transcripciones).
> - Documentos muy cortos.
> - Párrafos muy desiguales → chunks de tamaño irregular. Para formatos mixtos o
>   sin estructura confiable, [[Recursive Character Splitting]].

## References

- Fuente: [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs, 2026-02-09

## Related

- [[Chunking Strategies]]
- [[Sentence-Based Chunking]]
- [[Recursive Character Splitting]]
