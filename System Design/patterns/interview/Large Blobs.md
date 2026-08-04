---
title: Large Blobs
reading:
  total_words: 1760
  read_words: 1760
  pct: 100
  last_read: 2026-07-02
source: HelloInterview — pattern "Handling Large Blobs" (system design), complementado con AWS docs, breakdowns de Dropbox/YouTube, Medium, DEV Community
author: Compilado siguiendo la estructura de HelloInterview
created: 2026-07-02
tags:
  - system-design/storage
  - type/pattern
  - status/permanent
aliases:
  - Large Blobs
  - Manejo de Archivos Grandes
updated: 2026-07-02
---

> [!note] Definición
> Pattern de system design para manejar archivos grandes (videos, imágenes, backups) sin que pasen por tus servidores de aplicación. La regla central: el cliente sube/baja **directo** a blob storage (S3, GCS, R2) o CDN usando **presigned URLs** / **signed URLs**, mientras el backend solo se ocupa de autenticación, metadata y post-procesamiento. Para archivos muy grandes se suma **multipart upload** (chunks paralelos, resumibles) y sincronización de estado entre la metadata en base y el objeto en storage.

## The Problem

Archivos grandes — videos, imágenes, documentos, backups — no se pueden manejar como datos normales. Un video de varios GB que pasa por tu app server la convierte en cuello de botella: consume ancho de banda, memoria y conexiones, y no escala durante uploads pesados.

En la arquitectura ingenua, cada upload pasa por el backend: cliente → API server → blob storage. Eso se cae rápido: alto uso de bandwidth, latencia, problemas de escala. Lo mismo para descargas.

**La idea central del patrón:** **no rutees los bytes por tus servidores.** El cliente sube/baja **directo** al blob storage (S3, GCS, R2) o CDN; tu backend solo maneja autenticación, metadata y post-procesamiento.

**Señal para reconocer el patrón:** cualquier sistema que maneje archivos grandes — YouTube (videos), Dropbox/Google Drive (archivos), Instagram (fotos), Slack (adjuntos), servicios de backup.

---

## The Solution

### Simple Direct Upload

**Cómo funciona — presigned URLs:** en vez de `POST /upload` con el archivo, cambiás a `POST /presigned_url`. El cliente manda solo la **metadata** (nombre, tamaño, tipo). El backend genera una **presigned URL** — un link temporal y firmado que autoriza subir un objeto específico a S3 sin credenciales de AWS. El cliente usa esa URL para subir el archivo **directo** a S3. Después, notifica al backend para actualizar la metadata (estado "completo").

**Por qué es seguro y bueno:** la presigned URL es tan segura como cualquier otro método (es temporal, scoped a un objeto, expira). Descarga la transferencia a S3/CDN, reduce la carga de tus servidores, y el backend nunca toca los bytes. Así funcionan YouTube, Drive, Dropbox y Slack.

**Dato:** la expiración default de una presigned URL es corta (15 min en S3, configurable), pero el máximo real con SigV4 es de 7 días (604.800 segundos) — el default es corto pero el techo configurable es mucho más alto. Ojo con credenciales temporales (IAM role): la URL no puede durar más que la sesión del rol, aunque le pongas una expiración más larga ([Download and upload objects with presigned URLs — AWS docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html), [S3 PresignedURL Limitations — AWS re:Post](https://repost.aws/questions/QUxaEYVXbVREamltPSmKRotg/s3-presignedurl-limitations)). En cualquier caso, conviene mantenerla mínima por seguridad.

### Simple Direct Download

**Cómo funciona:** el cliente baja directo de blob storage o, mejor, de un **CDN** con **signed URLs** (para control de acceso). El [[Content Delivery Network|CDN]] cachea el archivo cerca del usuario, reduciendo latencia. Para archivos grandes, S3 y HTTP soportan **Range requests** nativamente: el cliente baja distintos rangos de bytes en paralelo o **resume** una descarga interrumpida sin empezar de cero.

**Distinción a nombrar:** "S3 presigned URL" (acceso directo a S3) vs "CDN signed URL" (acceso vía CDN con control). Para downloads a escala, preferís CDN.

### Resumable Uploads for Large Files

**El problema:** un upload de varios GB en una red móvil flaky puede fallar al 99% — y reiniciar desde cero es inaceptable.

**Cómo funciona — multipart upload:** partís el archivo en chunks (típicamente ~5-10MB cada uno). Los límites duros del protocolo en S3: parte mínima 5 MB (excepto la última, sin mínimo), parte máxima 5 GB, hasta 10.000 partes por upload, y el objeto total puede ir de 5 MB a 50 TB; AWS recomienda usar multipart a partir de objetos de 100 MB — útil tenerlo a mano para un deep dive de "cuántas partes necesito para un archivo de X GB" ([Amazon S3 multipart upload limits](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)). El flujo:
1. El cliente pide iniciar: el backend llama a `CreateMultipartUpload` de S3, obtiene un `uploadId`, genera una **presigned URL por cada parte**, guarda la metadata con estado "uploading", y devuelve el `uploadId` + las URLs.
2. El cliente sube cada chunk directo a S3 con su presigned URL (en **paralelo**, pero no demasiados a la vez). Cada parte devuelve un `ETag`.
3. El cliente manda los pares `ETag`/`PartNumber` al backend, que llama a `CompleteMultipartUpload`, y S3 ensambla las partes en un solo objeto.

**Por qué escala:** los reintentos son baratos (re-mandás solo la parte fallida, no todo), el throughput es alto (paralelismo), y el uso de memoria queda razonable.

**Resume:** mantenés un ledger server-side (`key → partes subidas / offset`, en Redis/KV). Al reanudar, el cliente consulta el progreso y sigue desde la última parte buena. Como las presigned URLs expiran, al resumir puede necesitar pedir URLs nuevas. Dropbox usa esto + deduplicación: chequea por fingerprint si el archivo ya existe (para dedup) o si hay un upload a medio hacer para resumir. En producción, Dropbox trabaja con chunks de 4 MB (no 5-10MB) para su block-level sync, con hash SHA-256 por chunk como chunk-id; si el hash ya existe en el metadata service, no se sube contenido — solo se referencia. El content-defined chunking (CDC, tamaño de chunk variable según el contenido) mejora la deduplicación entre 10-20% frente a chunking de tamaño fijo, a costa de más CPU ([Arslan Ahmad — How Dropbox is Doing Block-Level Deduplication](https://substack.com/@arslanah/note/c-205234857), [Dropbox Architecture Deep Dive](https://www.chengchangyu.com/blog/Dropbox-Architecture-Deep-Dive)).

**Downloads chunked:** no hacen falta — una vez que S3 ensambla el objeto con `CompleteMultipartUpload`, la descarga es de un archivo normal (con Range requests para paralelismo/resume). El cliente no necesita saber los límites de los chunks originales.

### State Synchronization Challenges

El desafío central del patrón: mantener sincronizados la **metadata en tu base** y el **objeto en blob storage**, que son dos sistemas separados.

- **Uploads fallidos:** si el cliente sube a S3 pero nunca notifica al backend, quedás con un objeto huérfano en S3 y metadata en estado "uploading" para siempre.
- **Solución — event notifications:** S3 emite eventos (S3 Event Notifications) cuando un objeto se crea; un handler los escucha y actualiza la metadata a "completo", en vez de depender solo de que el cliente avise.
- **Lifecycle / cleanup:** jobs de fondo que borran multipart uploads incompletos vencidos (S3 tiene lifecycle rules para abortar multipart uploads viejos) y objetos huérfanos.
- La metadata es la fuente de verdad de "qué archivos existen y su estado"; el blob storage tiene los bytes. Reconciliás los dos.
- **Caso real — arquitectura de dos planos de Dropbox:** separan un "Block Service" (stateless, horizontalmente escalable, guarda los blobs inmutables — su sistema propio se llama Magic Pocket) de un "Metadata Service" con garantías ACID sobre MySQL shardeado con Vitess para mantener la integridad del árbol de archivos. Es una instancia concreta de este mismo desafío de sincronización, con nombres reales de los sistemas ([System Design of Dropbox — Medium](https://medium.com/@lazygeek78/system-design-of-dropbox-6edb397a0f67), [How Dropbox scaled its storage infrastructure](https://medium.com/@rohitlakhotia/how-dropbox-scaled-its-storage-infrastructure-e9126970cd60)).

### Cloud Provider Terminology

- **S3 / GCS / Azure Blob / Cloudflare R2:** los object stores.
- **Presigned URL (S3) / Signed URL (GCS):** link temporal firmado para upload/download directo.
- **Multipart Upload:** el mecanismo de subir en partes.
- **Transfer Acceleration (S3):** rutea el upload por la red edge de CloudFront para acelerar subidas desde lejos. En números concretos: mejoras de 50% a 500% en transferencias de objetos grandes a larga distancia, más notorio cuanto más lejos está el cliente de la región de S3; para uploads dentro de la misma región el beneficio es mínimo o nulo ([S3 Transfer Acceleration — AWS](https://aws.amazon.com/s3/transfer-acceleration/), [Using AWS Edge to optimize object uploads to Amazon S3](https://aws.amazon.com/blogs/networking-and-content-delivery/using-aws-edge-to-optimize-object-uploads-to-amazon-s3/)).
- **CDN signed URL:** para downloads controlados vía CDN.
- **Range request:** header HTTP para bajar/resumir por rangos de bytes.

---

## When to Use in Interviews

### Common interview scenarios

YouTube (upload de videos), Dropbox/Google Drive (file storage), Instagram (fotos), servicios de backup, document sharing, cualquier sistema donde el archivo es grande. Suele combinarse con otros patterns: un video usa Large Blobs (upload) + Long Running Tasks (transcoding) + Real-time Updates (progreso).

### When NOT to use it in an interview

- Si los archivos son chicos (unos KB, avatares, thumbnails), no necesitás presigned URLs ni multipart — pasarlos por el backend está bien.
- Si no hay archivos grandes en el problema, no fuerces este patrón.
- **Frase:** "Como los archivos son de varios GB, uso presigned URLs para que el cliente suba directo a S3 y mis servidores no sean el cuello de botella; para archivos chicos no valdría la complejidad."

---

## Common Deep Dives

### "¿Qué pasa si el upload falla al 99%?"

- **Multipart upload:** solo re-mandás la **parte** fallida, no el archivo entero. Con chunks de 5-10MB, perdés como máximo un chunk.
- **Resume desde ledger:** el server trackea qué partes se subieron (`key → nextPart` en Redis); al reanudar, el cliente sigue desde ahí.
- **Presigned URLs vencidas:** al resumir tras mucho tiempo, el cliente pide URLs nuevas para las partes faltantes.
- **Idempotencia:** subir la misma parte dos veces es seguro (S3 la sobrescribe por PartNumber).

### "¿Cómo prevenís el abuso?"

- **Presigned URLs scoped y de vida corta:** cada URL autoriza **solo** un objeto/parte específico y expira rápido (minutos), así un link filtrado no da acceso amplio.
- **Límites de tamaño:** validás el tamaño máximo (distinto para imágenes vs videos) en la metadata antes de generar la URL, y S3 puede enforzar content-length.
- **Autenticación:** el backend valida al usuario antes de generar cualquier presigned URL.
- **Virus/content scanning:** post-procesamiento que escanea el objeto subido antes de marcarlo disponible.
- **Rate limiting** sobre la generación de presigned URLs.

### "¿Cómo manejás la metadata?"

- **Tabla FileMetadata** en tu base: `file_id, owner, name, size, type, status (uploading/complete), s3_key, created_at`. Es la fuente de verdad de qué existe.
- Se crea con estado "uploading" al iniciar, y pasa a "complete" vía la notificación del cliente o el evento de S3.
- Para dedup (Dropbox), guardás un **fingerprint/hash** del contenido y chequeás si ya existe antes de subir.
- Índices para queries típicas (archivos de un usuario, de una carpeta).

### "¿Cómo asegurás que las descargas sean rápidas?"

- **CDN:** cacheás el archivo en edge cerca del usuario, reduciendo la distancia y la latencia. Netflix empuja a caches a nivel ISP.
- **Range requests / descargas paralelas:** el cliente baja múltiples rangos de bytes en paralelo.
- **Transfer acceleration** para uploads desde regiones lejanas.
- **Compresión** para tipos comprimibles.
- **Signed URLs del CDN** para no perder control de acceso al cachear.

---

## Conclusion

El patrón se resume en una regla: **los bytes no pasan por tus servidores.** El cliente sube directo a blob storage con presigned URLs y baja directo del CDN con signed URLs; tu backend solo maneja auth, metadata y post-procesamiento. Para archivos grandes agregás multipart upload (subir en chunks, reintentar solo la parte fallida) y resume (ledger server-side). El desafío operativo central es sincronizar la metadata de tu base con el objeto en storage (event notifications + cleanup de huérfanos).

Los cuatro deep-dives que caen: fallo al 99% (multipart + resume), prevención de abuso (URLs scoped y de vida corta, límites de tamaño), metadata (tabla FileMetadata como fuente de verdad + fingerprint para dedup) y velocidad de descarga (CDN + range requests). La frase que resume: cambiás `POST /upload` por `POST /presigned_url` y dejás que S3 y el CDN hagan el trabajo pesado.

## References

- [Amazon S3 multipart upload limits](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)
- [Uploading and copying objects using multipart upload in Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Download and upload objects with presigned URLs — AWS docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [S3 PresignedURL Limitations — AWS re:Post](https://repost.aws/questions/QUxaEYVXbVREamltPSmKRotg/s3-presignedurl-limitations)
- [S3 Transfer Acceleration — AWS](https://aws.amazon.com/s3/transfer-acceleration/)
- [Using AWS Edge to optimize object uploads to Amazon S3](https://aws.amazon.com/blogs/networking-and-content-delivery/using-aws-edge-to-optimize-object-uploads-to-amazon-s3/)
- [Arslan Ahmad — How Dropbox is Doing Block-Level Deduplication](https://substack.com/@arslanah/note/c-205234857)
- [Dropbox Architecture Deep Dive — Chengchang Yu](https://www.chengchangyu.com/blog/Dropbox-Architecture-Deep-Dive)
- [System Design of Dropbox — Medium](https://medium.com/@lazygeek78/system-design-of-dropbox-6edb397a0f67)
- [How Dropbox scaled its storage infrastructure — Medium](https://medium.com/@rohitlakhotia/how-dropbox-scaled-its-storage-infrastructure-e9126970cd60)

## Related

- [[Content Delivery Network]]
- [[Cache-Aside]]
- [[Sharding]]
- [[Reverse Proxy]]
