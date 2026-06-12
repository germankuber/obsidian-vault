---
title: 07 - Deploying Haystack-Based Applications
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 7
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Deploying Haystack-Based Applications
  - Cap 7 - Deploying Haystack Applications
updated: 2026-06-12
---

# 07 - Deploying Haystack-Based Applications

> [!info] Capítulo 7 · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> El capítulo que cierra el viaje dev→producción: llevar el pipeline [[Hybrid Search|hybrid]] [[RAG]] reproducible del Cap 6 a un servicio production-grade sobre [[Haystack 2.0]]. Contrasta dos estrategias de deploy — **control** (**FastAPI** custom, "from-scratch") vs **velocity** ([[Hayhooks]], "batteries-included") — ambas Dockerizables y escalables con **Kubernetes**, y termina exponiendo el pipeline como **[[Model Context Protocol (MCP)|MCP]] server**. Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · anterior [[06 - Building Reproducible and Production-Ready RAG Systems]] · siguiente [[08 - Hands-On Projects]].

## Resumen

Este capítulo cierra el arco que empezó como un script local y termina como una **API containerizada y viva**: tomar el sistema [[RAG]] reproducible y evaluado del [[06 - Building Reproducible and Production-Ready RAG Systems|Cap 6]] y desplegarlo en producción. El capítulo arranca enumerando las **seis necesidades clave** de cualquier deploy de pipeline NLP — scalability (con elasticity), accessibility, resource management, reliability/resilience, modularity/maintainability y security/compliance — aclarando que implementarlas todas está fuera de scope: el foco es la **infraestructura inicial** de dos estrategias contrastadas. El **tema central es el trade-off entre control y velocity**, materializado en dos métodos que NO son mutuamente excluyentes (porque [[Hayhooks]] está construido **sobre FastAPI**, son capas de abstracción distintas, no rivales).

El **Método 1 — FastAPI custom** es la vía de máximo control: se construye a mano una REST API que importa y corre el [[SuperComponent]] hybrid RAG, exponiendo `/query`, `/health`, `/info` y `/`; se apoya en **Starlette** (async, vía **Uvicorn**) para inference I/O-bound y en **Pydantic** para validar requests y autogenerar la doc OpenAPI; se asegura con **API key authentication** (`APIKeyHeader` + dependency injection) y un `config.py` con `BaseSettings` + `Field`; se empaqueta con un **multi-stage Dockerfile** (Python 3.11 slim, **`uv`**, usuario non-root, `HEALTHCHECK`, `start.sh` que primero indexa y luego arranca Uvicorn en el puerto 8000); y se automatiza con un workflow **CI/CD** de GitHub Actions (trigger por path, build, test, health-check, cleanup), extensible a un job CD de login/tag/push/deploy a la nube (Figura 7.1).

El **Método 2 — Hayhooks** es la vía de máxima velocity: se **serializa** el pipeline Python a un `pipeline.yml` con `dump()` (un artefacto MLOps limpio, el "YAML-as-contract" que desacopla el dev lifecycle del deployment lifecycle), y un `PipelineWrapper` (`BasePipelineWrapper` + `setup()` con `Pipeline.loads()` + `run_api()`) le dice a Hayhooks cómo servirlo; corriendo `uv run hayhooks run`, Hayhooks autogenera todo el boilerplate FastAPI (app, endpoints, modelos Pydantic, OpenAPI) sin escribir `app = FastAPI()` ni `import uvicorn`, sirviendo indexing y RAG **online** en el puerto 1416. Se asegura con un reverse proxy **Nginx** (HTTP basic auth, rate limiting 10 req/s, uploads 50 MB, network isolation vía docker-compose). Finalmente, Hayhooks puede exponer los pipelines como **MCP tools** (`hayhooks mcp run`, transport SSE, `skip_mcp = True` para los que no deben listarse), integrable tanto en la app FastAPI custom (`hayhooks.create_app()`) como en el deploy Hayhooks. El cierre sintetiza el patrón MLOps maduro: YAML serialization + Hayhooks + Docker = flexibility + maintainability + scalability, con Kubernetes/Docker Compose, **Ray** para elasticidad y **Prometheus** para monitoring.

## Deployment strategies for Haystack pipelines

Antes de elegir cómo desplegar, el capítulo define **qué necesita** todo deploy de un pipeline NLP/RAG. Son seis necesidades, cada una con su matiz para el caso LLM:

> [!note] **Las seis necesidades clave del deploy de pipelines NLP.**
> - **Scalability** — manejar carga creciente sin degradar; incluye **elasticity** (escalar recursos up/down según demanda; **Docker + Kubernetes** habilitan elastic scaling). Es la "número uno para salir del notebook": servir desde 1 hasta 1000 usuarios concurrentes.
> - **Accessibility** — exponer el sistema vía **APIs/REST**, remote/distributed access, cloud (AWS/GCP/Azure). En este contexto, accessibility = **una API**: un building block modular consumible por una web, una mobile app u otro backend.
> - **Resource management** — asignación eficiente de memoria/CPU/GPU/storage; serverless + containerization. Los **LLM y embedding models son memory/GPU-hungry**: hay que manejarlos para no explotar los costos.
> - **Reliability and resilience** — error-handling robusto, fault tolerance, automatic restarts, monitoring/logging. Es el "lado trust": un **500 Internal Server Error** rompe la confianza; hacen falta health checks, logging y recovery.
> - **Modularity and maintainability** — componentes modulares, updates sin disrupción, CI/CD. Poder actualizar un `PromptBuilder` o swappear el embedding model **sin tirar el sistema offline**.
> - **Security and compliance** — encryption, access controls, GDPR/HIPAA, authn/authz para APIs externas. El hurdle final, no negociable.

> [!warning] Implementar las **seis** necesidades completas está **fuera de scope** del capítulo. El foco es la **infraestructura inicial** de las dos estrategias de deploy — no un manual exhaustivo de cada pilar.

Las dos estrategias que el capítulo desarrolla son: **API deployment con FastAPI** (custom, high-control) y **pipeline serialization + Hayhooks** (streamlined, high-velocity).

### Comparing the two strategies

> [!note] **El tema central del capítulo: el trade-off entre control (Método 1) y velocity (Método 2).** No son soluciones competidoras: **Hayhooks está construido sobre FastAPI**, así que operan en **capas de abstracción distintas**.

- **Método 1 (control)** — build manual de una REST API con **FastAPI**: máximo control y customización; definís cada endpoint, cada validation model y cada pieza de lógica. Es el enfoque "from-scratch".
- **Método 2 (velocity)** — serializar el pipeline a **YAML** y dejar que **Hayhooks** lo exponga como REST API automáticamente: máxima velocity e integración nativa con Haystack. Es el enfoque "batteries-included".

El libro enseña **primero el modo manual** (FastAPI) para entender Pydantic y la creación de endpoints, y **luego Hayhooks** como la abstracción que automatiza ese boilerplate. El ejemplo FastAPI expone el hybrid RAG del Cap 6 como endpoint asumiendo el indexing corrido por separado (offline); después se ve un sistema dinámico que sube un PDF/URL al endpoint, lo indexa y lo consulta.

### Tabla 7.1 — Comparison of Haystack pipeline deployment

| Feature | Method 1: Custom FastAPI | Method 2: Hayhooks |
|---|---|---|
| **Primary goal** | Total control, custom logic, complex endpoints | Speed, simplicity, native Haystack integration |
| **Boilerplate code** | High (manual setup de app/endpoints/models) | Minimal to zero |
| **How it works** | Importás y corrés el objeto `Pipeline` de Haystack dentro de tu código API Python | Hayhooks lee un `pipeline.yml` serializado y auto-genera los endpoints |
| **Flexibility** | Maximum (cualquier endpoint/middleware/lógica) | High (sobre FastAPI, extensible pero optimizado para servir pipelines) |
| **Best for...** | Apps complejas multi-facéticas donde el pipeline Haystack es **una parte** de un sistema mayor | Desplegar RAG APIs rápido, MLOps workflows, servicios donde el pipeline **ES** el sistema |
| **Learning curve** | Moderate (FastAPI, Pydantic, ASGI) | Low (Haystack y YAML) |

*Table 7.1 – Comparison of Haystack pipeline deployment*

> [!tip] La tabla deja clara la regla de decisión: si el pipeline es **un componente** de un sistema mayor que necesita custom middleware/auth/lógica, elegí **FastAPI**. Si el pipeline **ES** el sistema y querés desplegarlo rápido, elegí **Hayhooks**.

## Implementing the deployment strategies → Method 1: FastAPI

**FastAPI** es un web framework high-performance para construir REST APIs en Python: rápido, fácil de usar y compatible con type annotations. Sus features clave:

- **Asynchronous support** — basado en `asyncio`, soporta alta concurrencia.
- **Automatic documentation** — genera doc interactiva OpenAPI/Swagger UI.
- **Validation con Pydantic** — define modelos para validar requests/responses contra schemas (**Pydantic** = la librería que usa type hints para validar data).

¿Por qué FastAPI es el **de-facto standard para ML**? Por dos fundaciones:

> [!note] **Built on Starlette + built on Pydantic.**
> - **Starlette** implementa la spec **ASGI**, async desde el inicio, corrido por el servidor ASGI **Uvicorn**. Esto es crítico para inference NLP, que es **I/O-bound**: mientras la API espera al LLM, en vez de bloquear puede manejar cientos de otros requests, mejorando el throughput.
> - **Pydantic** valida con type hints en runtime: definís un base model para el request de `/query`, valida el JSON entrante automáticamente y autogenera la doc OpenAPI/Swagger.

Para empaquetar el servicio se usa **Docker** (containerización): features de **Isolation**, **Portability** y **Efficient resource usage**; combinado con **Kubernetes** para load balancing y automatic failover.

### Building the FastAPI application

La app custom expone el [[SuperComponent]] hybrid RAG del Cap 6 vía endpoints. Usa **dos pipelines** abstraídos como SuperComponents: el indexing pipeline y el hybrid RAG. El **indexing corre offline / por separado** para poblar un vector store local **Qdrant**; el hybrid pipeline usa retrievers sparse + dense compatibles con Qdrant. El document store se inicializa así:

```python
document_store = QdrantDocumentStore(
            path=qdrant_path,
            index=qdrant_index,
            embedding_dim=1536,
            recreate_index=False,
            use_sparse_embeddings=True
        )
```

`app.py` es el entry point. Los endpoints:

- **`/query (POST)`** — acepta la pregunta, corre el RAG y devuelve la answer + los source documents.
- **`/health (GET)`** — status de Qdrant y del RAG component.
- **`/info (GET)`** — config: modelos en uso y Qdrant settings.
- **`/ (GET)`** — root: status + rutas disponibles.

Todo usa **Pydantic models** para validar los requests y estructurar las responses.

### Dockerizing

Un **Dockerfile** es el blueprint de la "shipping container". Sus key functionalities:

- **Defines the environment** — base image: OS + runtime.
- **Packages the application** — código + dependencias + config.
- **Automates build and setup** — instalar deps, copiar files, configurar el runtime.
- **Configures the container** — ports, env vars, comandos de arranque.

> [!tip] **Multi-stage build (best practice).** Un stage **build** instala las dependencias y un stage **runtime** copia solo lo necesario → la imagen final es lean, secure y small.

Pasos del Dockerfile del ejemplo:

1. **Foundation** — base image Python 3.11 slim, instalar `curl`, copiar el dependency manager **`uv`**.
2. **Code and dependencies** — workdir `/app`; copiar `pyproject.toml` + `uv.lock` primero (para aprovechar el caching), luego `src/`, los scripts y la data de indexing.
3. **Install** — `uv sync` + hacer ejecutables los startup scripts.
4. **Security** — crear y cambiar al usuario **non-root** `app`.
5. **Runtime setup** — expone el puerto **8000**; incluye un `HEALTHCHECK`; crea `/app/start.sh`, que primero corre el RAG indexing pipeline (`src.rag.indexing`) y luego arranca el Uvicorn API server.
6. **Execution** — el comando default corre `/app/start.sh`.

La portabilidad de Docker habilita correr en la nube y escalar con Kubernetes.

### Securing and validating our endpoints

El endpoint `/query` se asegura con **API key authentication** de FastAPI: solo los clientes con la secret key válida en el header acceden. La lógica vive en `app.py` + `config.py`, usando **dependency injection**:

```python
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=True)
async def get_api_key(api_key: str = Security(api_key_header)):
    """Validate API key from request header."""
    if api_key != settings.rag_api_key:
        raise HTTPException(
            status_code=401,
            detail="Invalid API Key"
        )
    return api_key
```

El endpoint protegido inyecta la validación como dependencia:

```python
@app.post("/query", response_model=QueryResponse)
async def query_documents(
    request: QueryRequest,
    api_key: str = Depends(get_api_key)
):
```

El `config.py` define una clase Pydantic **`Settings`** que hereda de **`BaseSettings`**: cada config lleva type hints Python (str/int) y Pydantic **enforcea los types en runtime** con type coercion (convierte `"5"` → `5` o falla). La función **`Field`** gobierna constraints y sourcing: mapea un field a su env var (ej. `env="OPENAI_API_KEY"`) y el **ellipsis `...`** marca un field como **requerido** — si falta, la app se detiene al startup. Se usa para enforcar `RAG_API_KEY`, y la validación de config se dispara al instanciar (`settings = Settings()`).

> [!warning] El ellipsis `...` en `Field` convierte un field en **mandatory**: si la env var no está presente, la app **no arranca**. Es fail-fast de configuración — preferible a un 500 silencioso en runtime.

Build y run del container:

```bash
docker build -t hybrid-rag-api .
```

```bash
docker run -d \
--name hybrid-rag \
-p 8000:8000 \
-e OPENAI_API_KEY=your_actual_openai_key \
-e RAG_API_KEY=your_secret_api_key \
hybrid-rag-api
```

Y un query autenticado vía curl (también posible con `requests.post(...)` en Python usando los mismos headers):

```bash
curl -X POST http://localhost:8000/query \
-H "Content-Type: application/json" \
-H "X-API-Key: your-secret-key" \
-d '{"query": "What is retrieval-augmented generation?"}'
```

### Automating deployment with CI/CD

**CI (Continuous Integration)** integra los cambios de código en un repo compartido, testeados y validados, minimizando conflictos. **CD (Continuous Deployment)** extiende CI automatizando el deploy de los cambios testeados a producción. Beneficios:

- **Automated testing** — preprocessing, inference y post-processing.
- **Consistent builds** — vía Docker.
- **Faster iterations**.
- **Rollback capability**.

![[07-fig-7.1-cicd-workflow.png]]
*Figure 7.1 – A sample CI/CD workflow that builds, cleans, tags, and pushes a Docker image to the registry for deployment*

El workflow (YAML en `.github/workflows/`) dispara en cada push, buildea la imagen y corre tests. Sus pasos:

1. **Trigger** — push o PR a `main`, solo si cambian files en `ch7/` o el propio workflow.
2. **Setup** — checkout + limpiar el disco agresivamente.
3. **Configuration** — crea un `.env` en `ch7/` inyectando la config de Qdrant + los secrets `OPENAI_API_KEY` y `RAG_API_KEY` desde GitHub Secrets.
4. **Build** — build de la imagen `hybrid-rag-api`.
5. **Test and run** — corre el container `test-ch7-api` detached, puerto 8000, pasando las API keys como env vars.
6. **Health check** — polling loop hasta 120s, curl al `/health` en `localhost:8000`, verificando arranque + indexing terminado.
7. **Cleanup** — stop/remove del container + prune del Docker system, pase o falle.

Para extender a un **cloud deploy** se agrega un job **CD** final, tras `docker-test-ch7`:

1. **Docker login** — autenticar con el registry (Docker Hub / Amazon ECR / Google Container Registry, con secrets).
2. **Tagging** — tag production-ready, ej. `registry-url/repo-name/hybrid-rag-api:main-${{ github.sha }}`.
3. **Pushing** — push de la imagen al registry remoto.
4. **Deployment trigger** — conectar al orquestador cloud (Kubernetes / Amazon ECS-EKS / Google Cloud Run) y actualizar la deployment definition.

## Method 2: Pipeline serialization and Hayhooks integration

> [!note] **[[Hayhooks]] no es una alternativa a FastAPI.** Es una aplicación especializada y optimizada **construida sobre FastAPI** — el "bridge" que lleva un pipeline Haystack a una API production-ready. Entiende el formato YAML serializado y autogenera **todo** el boilerplate FastAPI (app, endpoints, modelos Pydantic, doc OpenAPI) con cero código.

Simplificaciones que aporta Hayhooks:

- **Streamlined setup** — mínimo a cero boilerplate.
- **Customizable pipelines** — definidos en YAML, desplegables directo sin code adjustments.
- **Scalability** — Docker/Kubernetes, alta disponibilidad + load balancing.
- **Integration-ready** — DBs, document stores e inference services vía prebuilt connectors.

El proceso son **dos pasos**: serialization + wrapping.

### Pipeline serialization

**Serializar** es convertir un pipeline NLP a un formato guardable, transferible y recargable. Haystack soporta **YAML out-of-the-box** (human-readable y editable), útil en entornos colaborativos y en la transición dev→producción. Serializar = definir los componentes, init del pipeline, conectar los componentes y **dumpear a YAML con el método `dump()`**.

> [!note] **YAML-as-contract / decoupling.** Un data scientist itera en un notebook Python y tunea el pipeline → su artefacto final es un `pipeline.yml`; un ML engineer lo toma y lo despliega con Hayhooks **sin entender el Python interno**. El YAML **desacopla el dev lifecycle del deployment lifecycle**.

En el ejemplo se serializan el **indexing pipeline** y el **RAG pipeline** (scripts `indexing_pipeline_serialization.py` y `rag_pipeline_serialization.py`; YAMLs `indexing.yml` y `rag.yml`). El YAML define los componentes (ej. el embedder) **y las conexiones** que los cablean — el grafo entero del pipeline. El serializado se carga después en un `PipelineWrapper`.

### Wrapping pipelines for Hayhooks

En el ejemplo `ch7-hayhooks` se despliegan dos pipelines: indexing + hybrid RAG query (del Cap 6). La best practice es usar un archivo **`pipeline_wrapper.py`**: un thin Python que le dice a Hayhooks cómo cargar el YAML, cuáles son los inputs/outputs del API, y sirve de hook para custom logic. El wrapper del hybrid RAG:

- Hereda de **`BasePipelineWrapper`** (el contrato que Hayhooks espera).
- **`setup()`** — carga el pipeline al startup; en vez de buildearlo en Python, llama **`Pipeline.loads()`** sobre el `pipeline.yml`.
- **`run_api()`** — la lógica del API; Hayhooks maneja toda la maquinaria FastAPI alrededor. Auto-crea un endpoint `/hybrid_rag/run`, deserializa el JSON request al objeto `HybridRagInput` (query), corre el RAG, y serializa el `HybridRagOutput` de vuelta a JSON.

> [!tip] **Lo que desaparece vs el Método 1:** no hay `app = FastAPI()`, no hay `import uvicorn`, no hay `@app.post`. Todo el boilerplate de FastAPI se evapora — eso es la abstracción de Hayhooks.

No se escribe un `main.py`: se instalan las deps y se corre directamente.

```bash
# Set environment variables
export OPENAI_API_KEY="sk-..."
export HAYHOOKS_PIPELINES_DIR="./pipelines"
# Run Hayhooks!
uv run hayhooks run
```

Hayhooks encuentra `hybrid_rag/pipeline_wrapper.py` e `indexing/pipeline_wrapper.py`, los carga y sirve **ambos como endpoints separados**. La Swagger UI queda en `http://localhost:1416/docs`. Un query:

```bash
curl -X POST "http://localhost:1416/hybrid_rag/run" \
  -H "Content-Type: application/json" \
  -d '{"query": "What are the benefits of AI?"}'
```

> [!note] **Diferencia clave de Method 2:** ambos pipelines (indexing y retrieval) corren **online** — podés ejecutar el indexing dinámicamente como REST endpoint. En el Method 1, en cambio, el indexing era **offline** (había que correrlo antes de poder usar `/query`).

El deploy de Hayhooks también es Dockerizable (su propio Dockerfile) + un workflow de GitHub Actions (`docker-test-hayhooks.yml`).

> [!warning] **Important: Qdrant vector storage.** Al migrar a Hayhooks, el **local vector storage** necesita un upgrade a la versión **cloud-based** para que indexing y RAG usen el mismo backend (los scripts del ejemplo reflejan local storage).

### Securing and validating endpoints (Hayhooks)

Para asegurar Hayhooks se usa un reverse proxy **Nginx** (web serving, reverse proxying, caching, load balancing, media streaming) con **HTTP basic authentication**. La arquitectura: Nginx en el puerto **8080** delante del Hayhooks service (interno en el puerto **1416**). Protege todos los endpoints de pipeline mientras deja públicos los health endpoints.

Features production-grade del proxy:

- **Rate limiting** a **10 requests/segundo por IP**.
- Soporte de **uploads hasta 50 MB**.
- **Timeouts extendidos** para operaciones largas.
- Mejor seguridad por **network isolation** (Hayhooks no es directamente accesible).

> [!tip] Hayhooks **adapta** el concepto de seguridad del FastAPI (API key) usando NGINX auth estándar, y es extensible a **HTTPS/SSL**. La network isolation (1416 solo accesible por Nginx) es seguridad por arquitectura, no solo por credencial.

El script helper `generate_password.sh` (`./scripts/generate_password.sh`) prompea user/password y los guarda en `.env`. El `docker-compose.yml` levanta **dos servicios**:

- **Hayhooks (application backend)** — corre los pipelines; env vars incluyendo `OPENAI_API_KEY`; device a `cpu`; **volume mount** para persistir los archivos Qdrant en restart; restringe el puerto **1416 a la red interna** (accesible solo por Nginx, aislado del internet público); health check.
- **Nginx (security and routing proxy)** — imagen `nginx:alpine`; **único componente expuesto**; rutea el puerto host **8080 → interno 80**; monta `nginx.conf` (proxy, rate limiting, timeouts) y `.htpasswd` (credenciales HTTP basic auth); depende de Hayhooks y usa la red compartida.

Arrancar y probar:

```bash
docker-compose up -d
./scripts/test_secured_api.sh
curl -u $username:$password http://localhost:8080/status
```

El acceso queda en `http://localhost:8080/docs` con password. Hayhooks ofrece además soporte **MCP**.

## Exposing a Haystack pipeline as an MCP server

El **[[Model Context Protocol (MCP)|Model Context Protocol]]** es una forma **estandarizada** para que un agente externo reconozca tus pipelines Haystack desplegados como **tools especializadas**. Hayhooks lo soporta nativamente y actúa como el **MCP server**.

> [!note] **Cómo Hayhooks hace de MCP server.**
> - **Function** — Hayhooks lista **todos** tus pipelines desplegados como MCP tools.
> - **Execution** — el MCP server se arranca con `hayhooks mcp run`.
> - **Protocol** — usa **Server-Sent Events (SSE)** como transport.

Para exponer un pipeline como MCP tool, su wrapper debe estar bien estructurado: Hayhooks usa el **nombre del `PipelineWrapper`** como nombre del MCP tool y el **docstring del método `run_api`** como la descripción del tool.

> [!tip] Para desplegar un pipeline **sin** listarlo como MCP tool (ej. un indexing pipeline que no debería ser invocable por un agente), poné **`skip_mcp = True`** en la clase `PipelineWrapper`.

La integración del MCP server cambia según la estrategia elegida:

- **Integración en custom FastAPI (control)** — embebés Hayhooks programáticamente llamando **`hayhooks.create_app()`** dentro de tu `app.py`. Así tenés el MCP server reteniendo **control total** sobre custom routes, authentication middleware y error handling existentes. Mejor para apps grandes/complejas donde el pipeline es solo un componente.
- **Integración en Hayhooks (velocity)** — como ya usás la estructura `PipelineWrapper` requerida, solo reemplazás `hayhooks run` por **`hayhooks mcp run`** en el Dockerfile / startup script. Como Hayhooks está sobre FastAPI, tu **Nginx reverse proxy + HTTP basic auth siguen funcionando**, protegiendo los nuevos MCP tool endpoints.

> [!note] El deployment journey (FastAPI + Hayhooks) cubre **scalability, reliability y security**. **Docker** da portabilidad + prep para Kubernetes. **FastAPI** = control granular sobre auth middleware y data validation (sistemas bespoke); **Hayhooks** = máxima velocity abstrayendo el boilerplate + YAML serialization como clean MLOps artifact.

## Transitioning from serialized development to production

El capítulo cierra con el workflow MLOps que articula todo lo anterior:

- **Development to artifact** — los pipelines se prototipan en Python y luego se **serializan a YAML**: un artefacto reproducible y validable. El YAML simplifica el mantenimiento — refinar componentes o parámetros sin reprogramar.
- **Consistent deployment** — los YAML se despliegan con Hayhooks + entornos Dockerizados; imágenes Haystack en Docker = **comportamiento consistente** dev/test/prod.
- **Scaling and monitoring** — scaling con **Kubernetes** o **Docker Compose**; para elasticidad avanzada, **serverless** o distributed processing tipo **Ray**; reliability con **Prometheus** para monitoring/troubleshooting.

> [!tip] **El patrón MLOps maduro del capítulo:** combinar **YAML serialization + Hayhooks + Docker** = **flexibility** (YAML editable) + **maintainability** (pipelines como artefactos version-controlled) + **scalability** (Docker/Kubernetes). Es la receta para graduar un pipeline de notebook a servicio.

Con esto, el viaje del concepto NLP local a la app production-grade queda completo: dos estrategias (custom control con FastAPI from-scratch, y rapid velocity con Hayhooks), un skill set full-stack. El próximo [[08 - Hands-On Projects|Cap 8]] lo pone a prueba con tres proyectos hands-on — **sentiment analysis**, **named-entity recognition (NER)** y **text classification** — que usan estos pipelines como tools de un orquestador.

## Citas

> "The central theme of this chapter is the trade-off between control (Method 1) and velocity (Method 2)."
> "Hayhooks is not an alternative to FastAPI; it's a specialized and optimized application built on top of FastAPI."
> "This YAML-as-contract design creates a clean separation of concerns."
> "When our API calls an LLM, it's an I/O-bound operation; the API is just waiting."
> "You have mastered the full stack, from a simple script to a live, containerized API."

## Para aplicar

- **Elegir estrategia por control vs velocity** — **FastAPI** si el pipeline es parte de un sistema mayor con custom middleware/auth/lógica; **Hayhooks** si el pipeline **ES** el sistema y querés desplegar rápido.
- **Method 1 — FastAPI** — async Starlette/Uvicorn + Pydantic validation; endpoints `/query` `/health` `/info` `/`; API key auth con `APIKeyHeader` + dependency injection (`Depends(get_api_key)`); `config.py` con `BaseSettings` + `Field` con `...` para fields requeridos (fail-fast al startup).
- **Multi-stage Dockerfile** — Python 3.11 slim, `uv`, usuario non-root, `HEALTHCHECK`, `start.sh` que indexa y luego arranca Uvicorn, puerto 8000; `docker build -t hybrid-rag-api .` + `docker run -d -p 8000:8000 -e OPENAI_API_KEY=... -e RAG_API_KEY=...`.
- **CI/CD GitHub Actions** — trigger por path (`ch7/`), build/test/health-check (polling 120s al `/health`)/cleanup; extender con un job **CD**: login/tag (`...:main-${{ github.sha }}`)/push/deploy a Kubernetes/ECS/Cloud Run.
- **Method 2 — Hayhooks** — serializar con `dump()` a YAML; `PipelineWrapper(BasePipelineWrapper)` con `setup()` (`Pipeline.loads()`) + `run_api()`; correr con `uv run hayhooks run` (`HAYHOOKS_PIPELINES_DIR`); endpoints auto en `:1416`, Swagger en `/docs`. Migrar el Qdrant local a cloud-based.
- **Seguridad Hayhooks con Nginx** — reverse proxy con HTTP basic auth, rate limit 10 req/s, uploads 50 MB, network isolation (1416 interno), `docker-compose` con Hayhooks + Nginx; `generate_password.sh` para credenciales; `docker-compose up -d` + `curl -u $user:$pass http://localhost:8080/status`.
- **Exponer como MCP server** — `hayhooks mcp run` (transport SSE); nombre del tool = nombre del `PipelineWrapper`, descripción = docstring de `run_api`; `skip_mcp = True` para el indexing; integrar en FastAPI con `hayhooks.create_app()` o cambiar el comando en Hayhooks (Nginx sigue protegiendo).
- **Producción** — scaling con Kubernetes/Docker Compose; **Ray** para elasticidad avanzada; **Prometheus** para monitoring/troubleshooting.

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[06 - Building Reproducible and Production-Ready RAG Systems]] — capítulo anterior; construye el sistema RAG reproducible y evaluado que **este capítulo despliega** en producción.
- [[08 - Hands-On Projects]] — capítulo siguiente; usa estos pipelines desplegados como **tools** de un orquestador (NER, sentiment, text classification).
- [[Haystack 2.0]] · [[Hayhooks]] · [[SuperComponent]] — el framework y la abstracción desplegados.
- [[RAG]] · [[Hybrid Search]] · [[BM25]] · [[Embeddings]] — el stack de retrieval que se sirve como API.
- [[Model Context Protocol (MCP)]] — exponer los pipelines como tools estandarizadas para agentes externos (`hayhooks mcp run`, SSE).
- [[LangGraph]] · [[FinOps]] — orquestación (Cap 8) y economía de la operación.
- **FastAPI** · **Pydantic** · **Docker** · **CI/CD** · **Qdrant** · **Nginx** · **Uvicorn** · **Starlette** · **PipelineWrapper** · **pipeline serialization (YAML)** · **Kubernetes** · **Ray** · **Prometheus** — conceptos clave del capítulo; candidatos a nota propia.
