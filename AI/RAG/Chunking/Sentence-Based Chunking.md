---
title: Sentence-Based Chunking
source: https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking
author: togoAI Labs
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Sentence-Based Chunking
  - sentence-based-chunking
---

# Sentence-Based Chunking

> [!note] Definición
> Detectar límites de **oración** y acumular oraciones completas hasta llegar al
> límite de tokens; ahí se cierra el chunk y empieza otro. Nunca corta a mitad de
> oración.

## Cómo funciona

- Detecta oraciones con un tokenizer de NLP.
- Acumula oraciones llevando la cuenta de tokens; al superar `max_chunk_size`,
  cierra el chunk y resetea.
- **Tamaño adaptativo**: un chunk puede ir de 1 oración a muchas, hasta el límite.

```python
import nltk
nltk.download('punkt')  # Sentence tokenizer
sentences = nltk.sent_tokenize(text)
chunks, current_chunk, current_length = [], [], 0

for sentence in sentences:
    sentence_length = len(tokenizer.encode(sentence))
    if current_length + sentence_length > max_chunk_size:
        chunks.append(" ".join(current_chunk))
        current_chunk = [sentence]
        current_length = sentence_length
    else:
        current_chunk.append(sentence)
        current_length += sentence_length

if current_chunk:
    chunks.append(" ".join(current_chunk))
```

**Por qué funciona**
- La oración es una **unidad semántica completa** (un pensamiento). Respetar su
  límite preserva significado; cortarla lo destruye.

**Librería**: NLTK `punkt` (`nltk.sent_tokenize`).

## Números

- Velocidad: `O(n)` + overhead de detección de oraciones (rápida en la mayoría
  de idiomas).
- Retrieval: **71-75% MRR@10** (+9-13% vs token-based).
- Coherencia: **0.78/1.0** (+27 puntos vs token-based) — salto enorme.

## Cuándo usarlo

> [!tip]
> - Prosa bien estructurada: artículos, documentación, libros, reportes.
> - Cuando la coherencia semántica es crítica.
> - **Default para contenido mayormente textual.**

## Cuándo NO usarlo / trade-offs

> [!warning]
> - Contenido muy formateado (código, tablas, listas) → no hay "oraciones".
> - Documentos sin límites claros de oración.
> - Documentos muy cortos (tweets, títulos, labels).
> - Pierde el contexto de **párrafo**: oraciones relacionadas pueden caer en
>   chunks distintos → si eso importa, [[Paragraph-Based Chunking]].

## References

- Fuente: [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs, 2026-02-09

## Related

- [[Chunking Strategies]]
- [[Paragraph-Based Chunking]]
- [[Fixed-Size Chunking]]
