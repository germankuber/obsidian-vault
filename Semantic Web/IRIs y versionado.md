---
title: IRIs y versionado
source: (Cool URIs for the Semantic Web, W3C; OWL 2 Structural Specification; práctica de OBO Foundry y w3id.org)
author: —
created: 2026-08-04
tags:
  - semantic-web
  - type/reference
  - status/done
aliases:
  - IRIs y versionado
  - política de IRIs
  - IRI
  - versionado de ontologías
  - versionIRI
updated: 2026-08-04
---

# IRIs y versionado

> [!warning] **Es la decisión más cara de revertir de todo un proyecto de ontología.** Una vez que alguien publicó datos usando tu IRI, cambiarlo rompe esos datos — y lo hace **en silencio**. No hay error de compilación: las tripletas que referencian el IRI viejo simplemente dejan de conectar con el modelo, y las consultas devuelven menos resultados sin que nada falle visiblemente.

Diseñá la política **antes del primer release**. Retrofittearla equivale a cambiar el esquema de una base de datos en producción, con la diferencia de que no sabés quiénes son todos tus consumidores.

## Anatomía de un IRI de ontología

```
https://ejemplo.org/onto/productos/1.2.0/Medicamento
└──┬──┘ └────┬────┘ └──┬─┘ └───┬────┘ └─┬──┘ └────┬────┘
 esquema   dominio    base   ontología versión  entidad
```

Tres identificadores distintos que conviene no confundir:

| Identificador | Qué identifica | Cambia |
|---|---|---|
| **IRI de la ontología** | La ontología como tal | **Nunca** |
| **`owl:versionIRI`** | Un release concreto | En cada release |
| **IRI de entidad** | Una clase o propiedad | Nunca (se depreca, no se renombra) |

```turtle
<https://ejemplo.org/onto/productos>  a  owl:Ontology ;
    owl:versionIRI   <https://ejemplo.org/onto/productos/1.2.0> ;
    owl:versionInfo  "1.2.0" ;
    dct:modified     "2026-08-04"^^xsd:date .
```

> [!note] **Un consumidor que importa el IRI genérico obtiene la última versión** y se expone a los cambios; **uno que importa el versionIRI obtiene estabilidad** y se queda atrás. Ambas necesidades son legítimas y hay que soportar las dos — igual que un `^1.2.0` versus un `1.2.0` exacto en un `package.json`.

## Hash vs slash

Decisión con consecuencias reales de resolución HTTP:

```
Hash:   https://ejemplo.org/onto/productos#Medicamento
Slash:  https://ejemplo.org/onto/productos/Medicamento
```

| | Hash (`#`) | Slash (`/`) |
|---|---|---|
| **Resolución** | El fragmento **no se envía al servidor**: se descarga la ontología entera y el cliente busca el fragmento | Cada entidad puede resolverse por separado |
| **Requiere** | Nada especial | Content negotiation (303 o 200 con `Content-Type`) en el servidor |
| **Escala** | Mala para ontologías grandes: bajás todo para una clase | Buena: bajás solo lo que pedís |
| **Complejidad de infra** | Mínima | Requiere configurar el servidor |
| **Conviene si** | Ontología chica o mediana, sin infra dedicada | Vocabulario grande o público |

> [!tip] Para la mayoría de los proyectos internos, **hash** es la elección pragmática: funciona sirviendo un archivo estático. Slash vale la pena cuando el vocabulario es grande, público, y querés que cada IRI sea desreferenciable individualmente.

## Opacos vs descriptivos

```
Descriptivo:  :Medicamento
Opaco:        :C_0000042
```

| | Descriptivo | Opaco |
|---|---|---|
| **Legibilidad** | Alta: el Turtle se lee solo | Nula sin herramienta |
| **Ante renombre del concepto** | El IRI queda mintiendo, o rompés consumidores | **Inmune**: solo cambia el `rdfs:label` |
| **Multilingüe** | Sesgado al idioma elegido | Neutral |
| **Debugging** | Fácil | Necesitás siempre el label a mano |
| **Usado por** | schema.org, FOAF, la mayoría de los vocabularios web | OBO Foundry (GO, CHEBI), SNOMED CT |

> [!note] La comunidad biomédica adoptó opacos de forma casi universal, y por una razón concreta: en dominios científicos los conceptos **se renombran** cuando cambia la nomenclatura, y el IRI no debería depender de eso. En cambio los vocabularios de la web general usan descriptivos, porque el valor está en que un humano lea el dato y lo entienda.

> [!tip] Regla práctica: **descriptivos si el vocabulario es estable y el público es amplio; opacos si el dominio renombra conceptos con frecuencia o si trabajás en varios idiomas**. Si elegís opacos, `rdfs:label` deja de ser opcional — es la única forma de leer el modelo.

## Persistencia

> [!warning] Los IRIs deben resolver **en el tiempo**, más allá de reorganizaciones internas, cambios de dominio corporativo o el fin del proyecto. Usar un dominio que la organización pueda abandonar es una bomba de tiempo.

Los servicios de identificadores persistentes desacoplan la identidad de la infraestructura:

| Servicio | Qué hace |
|---|---|
| **[w3id.org](https://w3id.org)** | Redirecciones permanentes mantenidas por la comunidad, vía pull request a GitHub. Gratuito |
| **PURL** | Persistent URLs, el mecanismo clásico |
| **DOI** | Para releases citables de vocabularios |

> [!tip] `https://w3id.org/miorg/onto/productos` te permite mover el hosting real cuantas veces quieras sin romper un solo dato publicado. Para un vocabulario destinado a durar, es la decisión correcta y cuesta un PR.

## Convenciones de nomenclatura

Elegí una y **documentala**; la consistencia importa más que cuál elijas.

| Elemento | Convención habitual | Ejemplo |
|---|---|---|
| **Clases** | `PascalCase`, sustantivo singular | `:Medicamento`, `:PolizaVida` |
| **Propiedades** | `camelCase`, empezando con verbo o preposición | `:tienePrincipioActivo`, `:contraindicadoEn` |
| **Individuos** | `camelCase` o `snake_case` | `:aspirina100mg` |
| **Object property inversa** | Par explícito | `:tieneAutor` / `:esAutorDe` |

> [!warning] **Singular, no plural**: una clase denota el **tipo**, no la colección. `:Medicamento`, no `:Medicamentos`. Es un error frecuente que delata que alguien está pensando en tablas.

> [!tip] Convención de idioma: los IRIs en **inglés** aunque trabajes en español, si hay chance de interoperar con vocabularios externos. Las etiquetas (`rdfs:label`, `skos:prefLabel`) van en todos los idiomas que necesites. Separa identidad de presentación — es el mismo principio de [[SKOS]].

## Tipos de cambio y su impacto

La clasificación que gobierna todo el versionado, y donde está el aporte menos obvio:

| Tipo | Qué hace | Riesgo | Requiere |
|---|---|---|---|
| **Aditivo** | Agrega entidades nuevas | Bajo | Release menor |
| **Restrictivo** | Agrega axiomas que **acotan** (disjunción, cardinalidad) | **Alto — puede invalidar datos existentes** | Análisis de impacto + aviso |
| **Correctivo** | Arregla un error de modelado | Medio-alto; cambia inferencias a propósito | Documentar qué inferencias cambian |
| **Refactorización** | Reorganiza sin cambiar semántica | Bajo *si* hay tests que lo prueben | Suite de [[competency questions]] |
| **Eliminación** | Quita entidades | **Alto — rompe seguro** | Deprecación previa, nunca borrado directo |

> [!warning] **Los cambios restrictivos son la categoría traicionera.** Agregar una disjunción parece aditivo —estás agregando un axioma, no quitando nada— pero puede volver **inconsistentes datos que ayer eran válidos**. Un dataset con instancias que caen en ambas clases recién declaradas disjuntas pasa de correcto a inconsistente sin que nadie haya tocado los datos. Ver [[clases insatisfacibles]].

## SemVer adaptado a ontologías

El paralelo con [[Semantic Versioning]] funciona, pero la compatibilidad **no se mide en la interfaz sino en las inferencias**:

> [!note] **Un cambio es compatible hacia atrás si todo lo que antes se deducía se sigue deduciendo, y todo lo que antes era consistente lo sigue siendo.**

| Cambio | SemVer | Por qué |
|---|---|---|
| Agregar clase o propiedad nueva | **MINOR** | Nada de lo anterior cambia |
| Agregar `rdfs:label` o documentación | **PATCH** | Sin efecto lógico |
| Agregar una **clase definida** | **MINOR** — pero avisá | Individuos existentes pueden reclasificarse: nuevas inferencias sin romper las viejas |
| Agregar disjunción o cardinalidad | **MAJOR** | Datos válidos pueden volverse inconsistentes |
| Agregar `domain`/`range` a propiedad existente | **MAJOR** | Infiere tipos nuevos sobre datos existentes; puede colisionar con disjunciones |
| Relajar una restricción | **MINOR** | Lo que era consistente lo sigue siendo |
| Corregir jerarquía mal modelada | **MAJOR** | Cambia inferencias por diseño |
| Deprecar una entidad | **MINOR** | Sigue resolviendo |
| Eliminar una entidad | **MAJOR** | Rompe consumidores |

> [!tip] La regla operativa: **si el cambio puede volver inconsistente un dataset que antes era válido, es MAJOR** — aunque el diff textual sea de una línea. Es el criterio que el diff semántico (`robot diff`) te permite aplicar de verdad. Ver [[Herramental de ontologías]].

## Deprecación, nunca borrado

```turtle
:MedicamentoGenerico  a  owl:Class ;
    owl:deprecated      true ;
    rdfs:comment        "Deprecada en 1.2.0. Usar :Medicamento con :esGenerico true."@es ;
    dct:isReplacedBy    :Medicamento ;
    :deprecatedInVersion  "1.2.0" ;
    :plannedRemoval       "2.0.0" .
```

El procedimiento:

1. **Marcar con `owl:deprecated`** — la entidad sigue existiendo y resolviendo.
2. **Documentar el reemplazo** con `dct:isReplacedBy`, para que el consumidor sepa hacia dónde migrar.
3. **Período de gracia** explícito y comunicado.
4. **Recién entonces**, y solo en un release MAJOR, eliminar.

> [!warning] **Una ontología publicada tiene consumidores que no sabés que existen.** Cualquiera puede haber importado tu vocabulario sin avisarte. Esa asimetría —a diferencia de una API interna, donde podés enumerar los clientes— es lo que hace que la deprecación no sea una cortesía sino la única política responsable.

## Migración de datos

Lo que la teoría casi nunca cubre: deprecaste la entidad, ¿y los millones de tripletas que la usan?

```turtle
# Puente temporal: durante el período de gracia, ambos IRIs coexisten
:MedicamentoGenerico  owl:equivalentClass  :Medicamento .
```

El paralelo con las migraciones de esquema es directo:

1. **Puente semántico** — `owl:equivalentClass` o `owl:sameAs` durante la transición: los datos viejos siguen funcionando.
2. **Script de reescritura** — un `SPARQL UPDATE` que reemplaza el IRI viejo por el nuevo en el grafo.
3. **Doble escritura** — si hay ingesta activa, escribir ambos durante la ventana.
4. **Retirar el puente** en el release MAJOR.

> [!warning] El puente con `owl:equivalentClass` tiene un costo inferencial: mientras esté, el razonador trata ambas clases como la misma y hereda todo cruzado. Está bien como transición; es un error dejarlo permanente.

## Release notes que sirven

> [!tip] **Centrá las notas en qué inferencias cambiaron, no en qué archivos se tocaron.** *"Se agregó disjunción entre `:PolizaVida` y `:PolizaPatrimonial`; los individuos declarados en ambas clases pasan a ser inconsistentes"* le sirve a un consumidor. *"Modificado onto.ttl"* no le sirve a nadie.

Contenido mínimo por release: versión y fecha, entidades agregadas, entidades deprecadas con su reemplazo, **cambios que alteran inferencias** con su impacto, y las eliminaciones planificadas para el próximo MAJOR.

## Conexión en el vault

- [[07 - Lifecycle, Versioning and Governance]] — el capítulo del libro que esta nota instancia.
- [[Herramental de ontologías]] — `robot diff` para el diff semántico, `robot annotate` para metadatos de versión.
- [[clases insatisfacibles]] — la consecuencia típica de un cambio restrictivo mal evaluado.
- [[competency questions]] — la suite que detecta regresiones inferenciales antes del release.
- [[OWL]] — `owl:versionIRI`, `owl:deprecated`, `owl:imports`.
- [[SKOS]] — el mismo principio de separar identidad (IRI) de presentación (labels).

## References

- [Cool URIs for the Semantic Web](https://www.w3.org/TR/cooluris/) — W3C. Hash vs slash y content negotiation.
- [OWL 2 Structural Specification](https://www.w3.org/TR/owl2-syntax/#Ontology_IRI_and_Version_IRI) — W3C. Ontology IRI y versionIRI.
- [w3id.org](https://w3id.org) — identificadores persistentes mantenidos por la comunidad.
- [OBO Foundry ID Policy](https://obofoundry.org/id-policy) — la política de IRIs opacos más documentada del campo.

## Related

- [[OWL]]
- [[Herramental de ontologías]]
- [[clases insatisfacibles]]
- [[competency questions]]
- [[Semantic Versioning]]
- [[07 - Lifecycle, Versioning and Governance]]
