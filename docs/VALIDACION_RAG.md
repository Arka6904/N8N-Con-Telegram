# Validación RAG — Chatbot CCD UNAB

Guía de validación para la rama RAG del workflow:

```text
n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json
```

---

## 1. Validar documentos fuente

```sql
SELECT COUNT(*)
FROM documentos_rag;
```

Debe devolver una cantidad mayor que `0`.

---

## 2. Validar chunks y embeddings

```sql
SELECT
  COUNT(*) AS total_chunks,
  COUNT(embedding) AS chunks_con_embedding
FROM document_chunks;
```

Interpretación:

| Resultado | Significado |
|----------|-------------|
| `total_chunks > 0` y `chunks_con_embedding = 0` | Chunks creados, embeddings pendientes |
| `total_chunks > 0` y `chunks_con_embedding = total_chunks` | RAG listo para consulta |
| `total_chunks = 0` | Falta ejecutar seed de chunks |

---

## 3. Validar columna vectorial

```sql
SELECT column_name, data_type, udt_name
FROM information_schema.columns
WHERE table_name = 'document_chunks'
  AND column_name = 'embedding';
```

Resultado esperado:

```text
udt_name = vector
```

La dimensión debe corresponder a:

```text
VECTOR(1536)
```

---

## 4. Validar búsqueda vectorial

Ejecutar solo cuando existan embeddings:

```sql
SELECT chunk_text
FROM document_chunks
WHERE embedding IS NOT NULL
ORDER BY embedding <-> (
  SELECT embedding
  FROM document_chunks
  WHERE embedding IS NOT NULL
  LIMIT 1
)
LIMIT 3;
```

Resultado esperado:

- Devuelve hasta 3 fragmentos.
- No debe fallar por tipo de dato.
- No debe devolver vacío si hay embeddings.

---

## 5. Validar workflow de embeddings

Workflow:

```text
n8n/ccd-rag-generar-embeddings.json
```

Cadena esperada:

```text
Manual Trigger
  → PostgreSQL - Leer chunks sin embedding
  → HTTP Request - OpenRouter Crear embedding chunk
  → Code - Preparar update embedding
  → PostgreSQL - Guardar embedding
```

Validar:

- Credencial `PostgreSQL CCD Real`.
- Header OpenRouter configurado.
- Modelo `openai/text-embedding-3-small`.
- La consulta lee solo chunks con `embedding IS NULL`.
- El update guarda `embedding` como `::vector`.

---

## 6. Validar rama RAG principal

Workflow:

```text
n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json
```

Cadena esperada:

```text
Code - Preparar pregunta RAG
  → HTTP Request - OpenRouter Embedding pregunta
  → PostgreSQL - Buscar chunks similares
  → Code - Armar contexto RAG
  → HTTP Request - OpenRouter Generar respuesta RAG
  → Code - Extraer respuesta OpenRouter
  → Code - Pulir respuesta final
  → Telegram - Respuesta dinamica
```

La salida de Telegram debe usar:

```text
={{ $json.respuesta_final || $json.respuesta }}
```

---

## 7. Validar endpoints OpenRouter

Embeddings:

```text
https://openrouter.ai/api/v1/embeddings
```

Modelo:

```text
openai/text-embedding-3-small
```

Chat:

```text
https://openrouter.ai/api/v1/chat/completions
```

Modelo:

```text
deepseek/deepseek-chat-v3.1:free
```

---

## 8. Pruebas Telegram para RAG

| Pregunta | Intención esperada |
|----------|--------------------|
| ¿Qué es la certificación CCD? | `rag_documentos` |
| ¿Cómo funcionan las insignias? | `rag_documentos` |
| ¿Qué es la ruta CCD? | `rag_documentos` |
| ¿Cuáles son los requisitos para obtener la insignia? | `rag_documentos` |
| ¿Qué documentos necesito para la certificación? | `rag_documentos` |

Resultado esperado:

- El bot no responde con FAQ genérica.
- Se genera embedding de la pregunta.
- PostgreSQL devuelve chunks similares.
- OpenRouter genera respuesta usando contexto.
- `Code - Pulir respuesta final` agrega salida final.
- Telegram responde con `respuesta_final`.

---

## 9. Pruebas de no regresión

Validar que otras ramas siguen funcionando:

| Pregunta | Intención esperada |
|----------|--------------------|
| Hola | `saludo` |
| Gracias | `agradecimiento` |
| Chao | `despedida` |
| ¿Qué es el CCD? | `faq_directa` |
| ¿Cuándo son las inscripciones? | `calendario` |
| ¿Qué eventos hay en la agenda del CCD? | `calendario_google` |
| ¿Qué cursos hay disponibles? | `oferta_cursos` |
| Mis cursos U00175325 | `cursos_estudiante` |
| ¿Hay noticias nuevas? | `noticias` |

---

## 10. Troubleshooting

| Error | Causa probable | Solución |
|------|----------------|----------|
| `chunks_con_embedding = 0` | No se ejecutó workflow de embeddings | Ejecutar `ccd-rag-generar-embeddings.json` |
| Error 401 en OpenRouter | Header Authorization ausente o incorrecto | Revisar API Key configurada |
| Búsqueda vectorial devuelve 0 filas | No hay embeddings o chunks válidos | Validar `document_chunks` |
| Error de dimensión vectorial | Embedding incompatible con `VECTOR(1536)` | Usar `openai/text-embedding-3-small` |
| OpenRouter no devuelve `choices` | Modelo no disponible o error de API | Revisar salida cruda del HTTP Request |
| Telegram responde vacío | Falta `respuesta` o `respuesta_final` | Revisar `Code - Pulir respuesta final` |

---

## 11. Checklist de validación final

- [ ] `documentos_rag` tiene registros.
- [ ] `document_chunks` tiene chunks.
- [ ] `embedding` es vector.
- [ ] `chunks_con_embedding` coincide con `total_chunks`.
- [ ] El workflow de embeddings corre sin errores.
- [ ] La rama RAG principal llega hasta OpenRouter.
- [ ] `Code - Pulir respuesta final` devuelve `respuesta_final`.
- [ ] Telegram envía `respuesta_final || respuesta`.
- [ ] Las ramas no RAG siguen funcionando.
