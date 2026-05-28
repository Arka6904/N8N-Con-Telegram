# RAG — Chatbot CCD UNAB

## 1. Qué es RAG

RAG (Retrieval-Augmented Generation) permite responder preguntas documentales usando información recuperada desde PostgreSQL + pgvector antes de llamar al modelo de chat.

En este proyecto, RAG se usa para consultas sobre certificación, insignias, requisitos, ruta CCD y documentos institucionales. Las preguntas simples, saludos, calendario, oferta, noticias y cursos por estudiante no usan RAG.

## 2. Flujo de datos RAG

```text
documentos_rag
  ↓
document_chunks
  ↓
embedding VECTOR(1536)
  ↓
OpenRouter embeddings
  ↓
pgvector búsqueda por similitud
  ↓
Top 5 chunks relevantes
  ↓
OpenRouter chat completions
  ↓
Code - Pulir respuesta final
  ↓
Telegram - Respuesta dinamica
```

## 3. Tablas principales

| Tabla | Uso |
|------|-----|
| `documentos_rag` | Documentos fuente |
| `document_chunks` | Fragmentos consultables |

`document_chunks.embedding` debe mantenerse como:

```sql
VECTOR(1536)
```

La dimensión corresponde a:

```text
openai/text-embedding-3-small
```

## 4. Crear tabla de chunks

Archivo:

```text
database/003_rag_schema_chunks.sql
```

Ejecutar:

```bash
psql -h HOST -U USER -d BASE_DATOS -f database/003_rag_schema_chunks.sql
```

## 5. Generar chunks desde documentos

Archivo:

```text
database/004_rag_seed_chunks.sql
```

Este script toma el texto de `documentos_rag`, genera fragmentos y los inserta en `document_chunks`.

Ejecutar:

```bash
psql -h HOST -U USER -d BASE_DATOS -f database/004_rag_seed_chunks.sql
```

Validar:

```sql
SELECT
  COUNT(*) AS total_chunks,
  COUNT(embedding) AS chunks_con_embedding
FROM document_chunks;
```

Antes de generar embeddings, `chunks_con_embedding` puede estar en `0`.

## 6. Generar embeddings con OpenRouter

Workflow:

```text
n8n/ccd-rag-generar-embeddings.json
```

Flujo:

```text
Manual Trigger
  → PostgreSQL - Leer chunks sin embedding
  → HTTP Request - OpenRouter Crear embedding chunk
  → Code - Preparar update embedding
  → PostgreSQL - Guardar embedding
```

Endpoint:

```text
https://openrouter.ai/api/v1/embeddings
```

Modelo:

```text
openai/text-embedding-3-small
```

Header requerido:

```text
Authorization: Bearer REEMPLAZA_AQUI_TU_OPENROUTER_API_KEY
```

El workflow procesa un lote de chunks por ejecución. Se ejecuta manualmente hasta que todos los chunks tengan embedding.

Validar avance:

```sql
SELECT
  COUNT(*) AS total_chunks,
  COUNT(embedding) AS chunks_con_embedding,
  COUNT(*) - COUNT(embedding) AS chunks_pendientes
FROM document_chunks;
```

## 7. Rama RAG en el workflow final

Workflow final:

```text
n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json
```

Cadena de nodos:

```text
Switch - Intencion
  → Code - Preparar pregunta RAG
  → HTTP Request - OpenRouter Embedding pregunta
  → PostgreSQL - Buscar chunks similares
  → Code - Armar contexto RAG
  → HTTP Request - OpenRouter Generar respuesta RAG
  → Code - Extraer respuesta OpenRouter
  → Code - Pulir respuesta final
  → Telegram - Respuesta dinamica
```

## 8. Búsqueda vectorial

Nodo:

```text
PostgreSQL - Buscar chunks similares
```

Consulta base:

```sql
SELECT 
    dc.chunk_text,
    dc.metadata,
    dr.nombre_documento,
    dr.categoria
FROM document_chunks dc
LEFT JOIN documentos_rag dr ON dr.id = dc.documento_id
WHERE dc.embedding IS NOT NULL
ORDER BY dc.embedding <-> '[EMBEDDING_PREGUNTA]'::vector
LIMIT 5;
```

El operador `<->` ordena por distancia vectorial.

## 9. Contexto documental

Nodo:

```text
Code - Armar contexto RAG
```

Construye un contexto con:

- Categoría.
- Título o nombre del documento.
- Contenido del chunk.

Si no encuentra fragmentos, genera un mensaje de contexto vacío controlado.

## 10. Generación de respuesta con OpenRouter

Nodo:

```text
HTTP Request - OpenRouter Generar respuesta RAG
```

Endpoint:

```text
https://openrouter.ai/api/v1/chat/completions
```

Modelo configurado:

```text
deepseek/deepseek-chat-v3.1:free
```

El prompt exige:

- Español colombiano.
- Tono claro, amable e institucional.
- Uso exclusivo del contexto documental.
- No inventar fechas, cursos, requisitos ni enlaces.
- Explicar cuando no hay información suficiente.

## 11. Intenciones que activan RAG

La rama RAG se activa con preguntas relacionadas con:

- insignia
- certificación
- certificado
- ruta CCD
- documento
- requisito
- requisitos

Las preguntas sobre `competencias digitales` se mantienen como FAQ directa para evitar consumo innecesario de RAG.

## 12. Errores comunes

| Error | Causa probable | Solución |
|------|----------------|----------|
| `chunks_con_embedding = 0` | No se ejecutó workflow de embeddings | Ejecutar `ccd-rag-generar-embeddings.json` |
| Búsqueda devuelve 0 filas | No hay embeddings o no hay chunks | Validar `document_chunks` |
| Error 401 en OpenRouter | API Key ausente o incorrecta | Revisar header Authorization |
| Respuesta sin información | Contexto insuficiente | Revisar documentos y chunks |
| Error de dimensión vectorial | Embedding no es de 1536 dimensiones | Usar `openai/text-embedding-3-small` |

## 13. Pruebas RAG

Preguntas recomendadas:

```text
¿Qué es la certificación CCD?
¿Cómo funcionan las insignias?
¿Qué es la ruta CCD?
¿Cuáles son los requisitos para obtener la insignia?
¿Qué documentos necesito para la certificación?
```

Resultado esperado:

- Intención: `rag_documentos`
- Recuperación de chunks con pgvector.
- Respuesta generada con OpenRouter.
- Pulido final antes de Telegram.
