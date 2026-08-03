---
created: 2026-08-03
updated: 2026-08-03
---
# 🧠 Obsidian Vault

Base de conocimiento personal sobre **IA**, **System Design** y lectura técnica, construida como un grafo Zettelkasten en [Obsidian](https://obsidian.md).

No es una carpeta de apuntes sueltos: es un grafo navegable donde cada nota es atómica, está enlazada con las demás por `[[wikilinks]]`, y cuelga de un **MOC** (Map of Content) que le da contexto. La idea es que el conocimiento se descubra navegando, no buscando.

**255 notas** · **5 dominios** · **24 MOCs** · **5 libros técnicos resumidos capítulo por capítulo**

---

## 🗂️ Dominios

| Dominio | Notas | Qué hay adentro |
|---|---:|---|
| **[AI](AI/)** | 102 | Agents · RAG · MLOps · GNN · Evals · Inference · Fundamentals |
| **[System Design](System%20Design/)** | 82 | Patrones de arquitectura, tecnologías, preparación de entrevistas |
| **[Libros](Libros/)** | 65 | Notas de lectura por capítulo, un MOC por libro |
| **[Semantic Web](Semantic%20Web/)** | 2 | RDF · OWL · SKOS · SPARQL · SHACL — el stack de representación de conocimiento |
| **wiki** | — | Espacio de trabajo y notas en curso |

El punto de entrada real del vault es **[`_Home.md`](_Home.md)** — desde ahí se llega a todos los dominios y a los dashboards vivos.

### AI

Siete sub-dominios, cada uno con su propio MOC:

- **AI Agents** — harnesses, arquitecturas de agentes, memoria, orquestación
- **RAG** — retrieval-augmented generation, con sub-MOCs de *Chunking* y *Reranking*
- **MLOps** — ciclo de vida de modelos, automatización, despliegue
- **GNN** — graph neural networks e interpretabilidad
- **Evals** — evaluación de sistemas LLM
- **Inference** — inference engineering, KV cache, optimización
- **AI Fundamentals** — los conceptos transversales que el resto asume

### System Design

Patrones de diseño de sistemas organizados por familia: almacenamiento, caching, comunicación, confiabilidad, escalado, consistencia y observabilidad. Incluye sub-MOCs para *API Design*, *Idempotency*, *Serverless*, *Service Mesh* y *Pagination*, más una sección de tecnologías concretas y material de entrevistas.

### Libros

Un resumen denso por capítulo, escrito para **estudiar sin volver al original**. Cada libro vive en su carpeta con notas `NN - Título.md` y un MOC que acumula la tesis, el índice y las ideas que cruzan toda la obra.

| Libro | Capítulos |
|---|---:|
| Agentic Architectural Patterns for Building Multi-Agent Systems | 12 |
| Unlocking Data with Generative AI and RAG | 14 |
| RAG-Driven Generative AI (2nd Edition) | 12 |
| Building Natural Language and LLM Pipelines | 10 |
| Ontology Engineering | 8 |

---

## 🧭 Cómo está organizado

### MOCs — el esqueleto de navegación

Cada carpeta tiene un **MOC** con prefijo `_` (`_AI.md`, `_RAG.md`, `_System Design.md`). No es un índice automático: es una nota escrita que explica **cómo se relacionan** las notas de esa carpeta, cuál es la tesis del conjunto y por dónde empezar. Los MOCs se anidan — `_Home` → `_AI` → `_RAG` → `_Chunking`.

### Notas atómicas y wikilinks

Una nota, un concepto. Los enlaces van **inline en el cuerpo**, donde el concepto se explica — no solo en una sección de referencias al final. Los enlaces sin resolver son deliberados: marcan crecimiento futuro del grafo.

### Frontmatter y tags

Todas las notas llevan frontmatter con `title`, `created`, `updated`, `tags` y `aliases`. El vocabulario de tags es **cerrado y de dos facetas**, para que el grafo no se degrade:

**`type/`** — qué clase de nota es:

| Tag | Uso | Notas |
|---|---|---:|
| `type/pattern` | Un patrón de diseño reusable | 75 |
| `type/concept` | Un concepto atómico | 72 |
| `type/case-study` | Nota de lectura sobre una fuente concreta | 63 |
| `type/moc` | Map of Content | 23 |
| `type/technology` | Una tecnología o herramienta específica | 19 |

**`status/`** — qué tan terminada está:

| Tag | Significado | Notas |
|---|---|---:|
| `status/permanent` | Nota completa y validada | 238 |
| `status/stub` | Parcial o pendiente de revisión | 14 |

**`libros/<slug>`** — la única familia que crece: un tag por libro, compartido por todos sus capítulos.

> No se inventan tags fuera de estas tres familias. Un vocabulario abierto convierte el grafo en ruido.

### Dashboards vivos

Los MOCs incluyen bloques **Dataview** que se recalculan solos: notas por dominio, por tipo, capítulos de un libro ordenados. El índice manual da el contexto; Dataview da la vista siempre actualizada.

### Tracking de lectura

- **[`_Reading Tracker.md`](_Reading%20Tracker.md)** — porcentaje leído por nota, con las que tienen contenido nuevo sin consumir.
- **[`_Imported Sources.md`](_Imported%20Sources.md)** — registro de cada artículo importado, para no duplicar fuentes.

---

## 🚀 Cómo usarlo

```bash
git clone https://github.com/germankuber/obsidian-vault.git
```

Abrí la carpeta como vault desde Obsidian (*Open folder as vault*) y arrancá por `_Home.md`.

**Plugins que el vault asume:**

- **Dataview** — sin él, los dashboards de los MOCs quedan como bloques de código sin renderizar
- **Meta Bind** — botones interactivos del Reading Tracker

Para navegar el grafo: `Cmd/Ctrl + G` abre la vista de grafo, donde los clusters por dominio se ven a simple vista.

---

## ✍️ Convenciones al escribir

- **Español** en el cuerpo de las notas; **términos técnicos, código y citas en su idioma original**, sin traducir.
- **Soft wrap**: cada párrafo, bullet o callout en **una sola línea**. Nunca cortar a 80 columnas.
- **Callouts** para estructurar: `> [!note]` definiciones · `> [!tip]` accionables · `> [!warning]` trade-offs y errores comunes · `> [!info]` cabecera de nota.
- **Tablas markdown** para cualquier tabla estructural — nunca parafrasearla en prosa.
- **Código en bloques cercados** con lenguaje, reproducido tal cual aparece en la fuente.
- **Densidad sin pérdida**: comprimir la redacción, nunca los hechos. Si un capítulo tiene 14 ideas, la nota tiene 14.

---

## 📄 Licencia

Notas personales de estudio. Los resúmenes de libros son trabajo derivado con fines de aprendizaje: los derechos de las obras originales pertenecen a sus autores y editoriales.
