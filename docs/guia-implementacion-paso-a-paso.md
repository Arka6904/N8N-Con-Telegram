# GuÃ­a de implementaciÃ³n â€” Paso a paso (CCD UNAB)

Documento complementario al [diseÃ±o del flujo n8n](./diseno-flujo-n8n-chatbot-ccd-unab.md).

Esta guÃ­a describe cÃ³mo dejar funcionando el workflow final:

```text
n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json
```

---

## 1. Orden recomendado

| Orden | Tarea | Archivo / recurso |
|-------|-------|-------------------|
| 1 | Crear servicios en Coolify | n8n + PostgreSQL |
| 2 | Configurar variables de entorno de n8n | Coolify |
| 3 | Crear bot de Telegram | BotFather |
| 4 | Configurar credenciales en n8n | Telegram, PostgreSQL, Google Calendar |
| 5 | Ejecutar scripts SQL base | `database/001_schema_ccd.sql`, `database/002_seed_ccd.sql` |
| 6 | Crear tabla de chunks RAG | `database/003_rag_schema_chunks.sql` |
| 7 | Generar chunks documentales | `database/004_rag_seed_chunks.sql` |
| 8 | Importar workflow de embeddings | `n8n/ccd-rag-generar-embeddings.json` |
| 9 | Ejecutar embeddings hasta completar chunks | n8n Manual Trigger |
| 10 | Importar workflow final del chatbot | `n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json` |
| 11 | Asignar credenciales y placeholders | n8n UI |
| 12 | Probar desde Telegram | Bot activo |

---

## 2. PostgreSQL y pgvector

Crear o usar dos conexiones PostgreSQL separadas en n8n:

| Credencial n8n | Uso |
|----------------|-----|
| `PostgreSQL CCD` | Calendario, oferta, noticias |
| `PostgreSQL CCD Real` | Cursos por estudiante y RAG |

Activar pgvector:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

La tabla `document_chunks` debe usar:

```sql
embedding VECTOR(1536)
```

Esta dimensiÃ³n corresponde al modelo `openai/text-embedding-3-small` consumido vÃ­a OpenRouter.

---

## 3. Ejecutar SQL del proyecto

Ejecutar los scripts segÃºn el entorno PostgreSQL correspondiente:

```bash
psql -h HOST -U USER -d BASE_DATOS -f database/001_schema_ccd.sql
psql -h HOST -U USER -d BASE_DATOS -f database/002_seed_ccd.sql
psql -h HOST -U USER -d BASE_DATOS -f database/003_rag_schema_chunks.sql
psql -h HOST -U USER -d BASE_DATOS -f database/004_rag_seed_chunks.sql
```

Validar chunks:

```sql
SELECT
  COUNT(*) AS total_chunks,
  COUNT(embedding) AS chunks_con_embedding
FROM document_chunks;
```

Al inicio `chunks_con_embedding` puede estar en `0`. Se completa con el workflow de embeddings.

---

## 4. Configurar Telegram

1. Crear bot en BotFather.
2. Copiar token del bot.
3. Crear credencial en n8n llamada:

```text
Telegram CCD Bot
```

4. Asignarla a:

- `Telegram Trigger`
- `Telegram - Respuesta dinamica`

5. Activar el workflow despuÃ©s de importar y configurar.

---

## 5. Variables de entorno sugeridas para n8n

```env
N8N_PROTOCOL=https
N8N_HOST=TU_DOMINIO_N8N
WEBHOOK_URL=https://TU_DOMINIO_N8N/
N8N_EDITOR_BASE_URL=https://TU_DOMINIO_N8N/
N8N_SECURE_COOKIE=true
GENERIC_TIMEZONE=America/Bogota
TZ=America/Bogota
N8N_PORT=5678
N8N_RUNNERS_ENABLED=true
N8N_RUNNERS_MODE=external
N8N_PROXY_HOPS=1
```

---

## 6. Importar workflow de embeddings

Archivo:

```text
n8n/ccd-rag-generar-embeddings.json
```

Flujo:

```text
Manual Trigger
  â†’ PostgreSQL - Leer chunks sin embedding
  â†’ HTTP Request - OpenRouter Crear embedding chunk
  â†’ Code - Preparar update embedding
  â†’ PostgreSQL - Guardar embedding
```

Configurar:

- Credencial `PostgreSQL CCD Real`.
- Header OpenRouter:

```text
Authorization: Bearer REEMPLAZA_AQUI_TU_OPENROUTER_API_KEY
```

Ejecutar manualmente hasta que:

```sql
SELECT COUNT(*), COUNT(embedding)
FROM document_chunks;
```

muestre embeddings completos.

---

## 7. Importar workflow final del chatbot

Archivo final:

```text
n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json
```

Workflow:

```text
Telegram Trigger
  â†’ Code - Normalizar pregunta avanzada
  â†’ Code - Analizar contexto conversacional
  â†’ Code - Detectar intenciÃ³n avanzada
  â†’ Switch - Intencion
```

Todas las ramas deben terminar en:

```text
Code - Pulir respuesta final
  â†’ Telegram - Respuesta dinamica
```

---

## 8. Configurar credenciales del workflow final

| Nodo | Credencial |
|------|------------|
| `Telegram Trigger` | `Telegram CCD Bot` |
| `Telegram - Respuesta dinamica` | `Telegram CCD Bot` |
| `PostgreSQL - Calendario` | `PostgreSQL CCD` |
| `PostgreSQL - Oferta cursos` | `PostgreSQL CCD` |
| `PostgreSQL - Noticias` | `PostgreSQL CCD` |
| `PostgreSQL - Cursos estudiante` | `PostgreSQL CCD Real` |
| `PostgreSQL - Buscar chunks similares` | `PostgreSQL CCD Real` |
| `Google Calendar - Consultar prÃ³ximos eventos CCD` | `Google Calendar CCD` |

Configurar en nodos OpenRouter:

```text
Authorization: Bearer REEMPLAZA_AQUI_TU_OPENROUTER_API_KEY
```

Configurar Google Calendar:

```text
REEMPLAZA_CALENDAR_ID_CCD
```

---

## 9. Ramas del workflow final

| IntenciÃ³n | Rama |
|-----------|------|
| `faq_directa` | `Code - Preparar respuesta directa` |
| `saludo` | `Code - Preparar respuesta directa` |
| `agradecimiento` | `Code - Preparar respuesta directa` |
| `despedida` | `Code - Preparar respuesta directa` |
| `calendario` | `PostgreSQL - Calendario` |
| `calendario_google` | `Google Calendar - Consultar prÃ³ximos eventos CCD` |
| `oferta_cursos` | `PostgreSQL - Oferta cursos` |
| `cursos_estudiante` | `Code - Verificar codigo estudiante` |
| `noticias` | `PostgreSQL - Noticias` |
| `rag_documentos` | OpenRouter + pgvector |
| `desconocido` | `Code - Respuesta desconocido` |

---

## 10. Pruebas recomendadas

| Mensaje | Resultado esperado |
|---------|--------------------|
| `Hola` | saludo |
| `Gracias` | agradecimiento |
| `Chao` | despedida |
| `Â¿QuÃ© es el CCD?` | FAQ directa |
| `Â¿CuÃ¡ndo son las inscripciones?` | calendario PostgreSQL |
| `Â¿QuÃ© eventos hay en la agenda del CCD?` | Google Calendar |
| `Â¿QuÃ© cursos hay disponibles?` | oferta de cursos |
| `Mis cursos` | solicita cÃ³digo |
| `Mis cursos U00175325` | consulta estudiante |
| `Â¿Hay noticias recientes?` | noticias |
| `Â¿CuÃ¡les son los requisitos para la certificaciÃ³n CCD?` | RAG documental |

---

## 11. Checklist antes de activar

- [ ] Credencial `Telegram CCD Bot` configurada.
- [ ] Credencial `PostgreSQL CCD` configurada.
- [ ] Credencial `PostgreSQL CCD Real` configurada.
- [ ] Credencial `Google Calendar CCD` configurada si se usarÃ¡ agenda.
- [ ] API Key de OpenRouter configurada en nodos HTTP.
- [ ] `REEMPLAZA_CALENDAR_ID_CCD` reemplazado.
- [ ] Chunks RAG creados.
- [ ] Embeddings generados.
- [ ] Workflow probado manualmente.
- [ ] Workflow activado.

---

## 12. DiagnÃ³stico rÃ¡pido

Validar vector:

```sql
SELECT extname
FROM pg_extension
WHERE extname = 'vector';
```

Validar embeddings:

```sql
SELECT
  COUNT(*) AS total_chunks,
  COUNT(embedding) AS chunks_con_embedding
FROM document_chunks;
```

Validar eventos de calendario:

```sql
SELECT evento, fecha_inicio
FROM calendario_ccd
WHERE activo = TRUE
ORDER BY fecha_inicio
LIMIT 5;
```

Validar noticias:

```sql
SELECT titulo
FROM noticias_ccd
WHERE activo = TRUE
  AND disponible_chatbot = TRUE
ORDER BY fecha_publicacion DESC
LIMIT 3;
```
