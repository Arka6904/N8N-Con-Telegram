# Diseño del flujo principal n8n — Chatbot Inteligente CCD UNAB

**Proyecto:** Chatbot Inteligente para el Centro de Competencias Digitales (CCD) de la Universidad Autónoma de Bucaramanga (UNAB)  
**Versión del documento:** 2.0  
**Workflow de referencia:** `CCD - Chatbot flujo principal (MVP mejorado)`  
**Archivo:** `n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json`  
**Orquestador:** n8n | **Canal principal:** Telegram  
**Infraestructura:** Azure VM + Coolify | **Datos y RAG:** PostgreSQL + pgvector | **IA:** OpenRouter

---

## 1. Introducción

Este documento describe el diseño actualizado del flujo principal en n8n para el chatbot institucional del CCD UNAB.

El chatbot atiende preguntas de estudiantes y usuarios interesados en información sobre cursos, calendario, eventos, noticias, ruta CCD, certificaciones, insignias e historial académico. n8n actúa como orquestador central: recibe mensajes desde Telegram, normaliza el texto, analiza el contexto conversacional, detecta intención, consulta fuentes estructuradas cuando corresponde y usa RAG documental solo cuando la pregunta requiere recuperación semántica.

La versión actual mejora el MVP inicial con:

- Normalización avanzada del mensaje.
- Análisis conversacional previo a la intención.
- Detección ampliada de saludos, agradecimientos, despedidas, códigos estudiantiles y agenda.
- Rama independiente para Google Calendar.
- RAG documental con OpenRouter y pgvector.
- Nodo único de pulido final antes de responder por Telegram.

---

## 2. Objetivo del flujo

El objetivo del flujo es responder de forma clara, amable e institucional, usando el camino más eficiente según el tipo de pregunta.

| Paso | Acción | Uso de IA |
|------|--------|-----------|
| 1 | Recibir mensaje de Telegram | No |
| 2 | Normalizar texto y extraer metadatos | No |
| 3 | Analizar contexto conversacional | No |
| 4 | Detectar intención por reglas | No |
| 5 | Consultar PostgreSQL o Google Calendar si aplica | No |
| 6 | Ejecutar RAG documental si aplica | Sí |
| 7 | Pulir respuesta final | No |
| 8 | Enviar respuesta por Telegram | No |

**Principio rector:** no usar generación con IA cuando la pregunta puede resolverse con FAQ, reglas, SQL o calendario.

---

## 3. Arquitectura general del flujo

```text
Telegram Trigger
        ↓
Code - Normalizar pregunta avanzada
        ↓
Code - Analizar contexto conversacional
        ↓
Code - Detectar intención avanzada
        ↓
Switch - Intencion
        ├── faq_directa        → Code - Preparar respuesta directa
        ├── saludo             → Code - Preparar respuesta directa
        ├── agradecimiento     → Code - Preparar respuesta directa
        ├── despedida          → Code - Preparar respuesta directa
        ├── calendario         → PostgreSQL - Calendario
        ├── calendario_google  → Google Calendar - Consultar próximos eventos CCD
        ├── oferta_cursos      → PostgreSQL - Oferta cursos
        ├── cursos_estudiante  → Code - Verificar codigo estudiante
        ├── noticias           → PostgreSQL - Noticias
        ├── rag_documentos     → OpenRouter + pgvector
        └── desconocido        → Code - Respuesta desconocido
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

### Componentes del ecosistema

| Componente | Rol |
|------------|-----|
| Telegram Bot API | Canal de entrada y salida del usuario |
| n8n | Orquestación del flujo conversacional |
| PostgreSQL | Datos estructurados del chatbot |
| pgvector | Búsqueda semántica para RAG |
| OpenRouter | Embeddings y generación de respuesta documental |
| Google Calendar | Consulta opcional de eventos próximos |
| Coolify | Despliegue y administración de servicios |

---

## 4. Principio de ahorro de consumo del modelo

El modelo de lenguaje solo se usa en la rama `rag_documentos`.

No se usa IA para:

- Saludos.
- Agradecimientos.
- Despedidas.
- FAQ directa.
- Calendario en PostgreSQL.
- Eventos de Google Calendar.
- Oferta de cursos.
- Cursos por estudiante.
- Noticias.
- Respuesta desconocida.

Esto reduce latencia, costo y riesgo de respuestas inventadas.

---

## 5. Nodos principales del workflow

Nombre del workflow:

```text
CCD - Chatbot flujo principal (MVP mejorado)
```

### 5.1 Telegram Trigger

Recibe mensajes enviados al bot de Telegram.

Salida relevante:

- `message.text`
- `message.chat.id`
- `message.from.id`
- `message.from.username`
- `message.from.first_name`
- `message.from.last_name`

### 5.2 Code - Normalizar pregunta avanzada

Este nodo extrae el texto y metadatos de Telegram, normaliza el mensaje y detecta códigos.

Campos principales generados:

| Campo | Descripción |
|-------|-------------|
| `pregunta_original` | Texto original del usuario |
| `pregunta_normalizada` | Texto en minúsculas, sin tildes ni puntuación |
| `palabras` | Lista de palabras normalizadas |
| `longitud_texto` | Cantidad de caracteres |
| `cantidad_palabras` | Cantidad de palabras |
| `chat_id` | ID del chat de Telegram |
| `telegram_user_id` | ID del usuario |
| `username` | Nombre de usuario de Telegram |
| `first_name` | Nombre del usuario |
| `last_name` | Apellido del usuario |
| `canal` | Canal de entrada, en este caso `telegram` |
| `codigo_unab_detectado` | Código tipo `U00175325` si existe |
| `id_estudiante_detectado` | ID numérico si existe |
| `tiene_codigo_detectado` | Booleano para saber si hay código o ID |
| `fecha_mensaje` | Fecha ISO del procesamiento |

### 5.3 Code - Analizar contexto conversacional

Este nodo enriquece el mensaje con señales conversacionales antes de clasificar intención.

Detecta:

- Mensaje vacío.
- Mensaje corto.
- Saludo.
- Agradecimiento.
- Despedida.
- Mensaje con prioridad o urgencia.
- Mención a Google Calendar o agenda.
- Mención a calendario institucional.
- Mención a historial de cursos.
- Presencia de código estudiantil.

Salida principal:

```text
contexto_conversacional
respuesta_contextual
```

### 5.4 Code - Detectar intención avanzada

Clasifica el mensaje en una intención controlada.

Intenciones válidas:

```text
faq_directa
saludo
agradecimiento
despedida
calendario
calendario_google
oferta_cursos
cursos_estudiante
noticias
rag_documentos
desconocido
```

También genera `respuesta_directa` cuando la respuesta puede resolverse sin consultar bases ni IA.

### 5.5 Switch - Intencion

El `Switch - Intencion` enruta por valor exacto de `intencion`.

Orden de salidas:

| Salida | Intención |
|--------|-----------|
| 1 | `faq_directa` |
| 2 | `saludo` |
| 3 | `agradecimiento` |
| 4 | `despedida` |
| 5 | `calendario` |
| 6 | `calendario_google` |
| 7 | `oferta_cursos` |
| 8 | `cursos_estudiante` |
| 9 | `noticias` |
| 10 | `rag_documentos` |
| 11 | `desconocido` |

---

## 6. Ramas del flujo

### 6.1 Respuestas directas

Incluye:

- `faq_directa`
- `saludo`
- `agradecimiento`
- `despedida`

Flujo:

```text
Switch - Intencion
        ↓
Code - Preparar respuesta directa
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

Estas respuestas no consultan base de datos ni modelo.

### 6.2 Rama calendario PostgreSQL

Intención:

```text
calendario
```

Flujo:

```text
PostgreSQL - Calendario
        ↓
Code - Formatear calendario
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

Consulta base:

```sql
SELECT evento, fecha_inicio, fecha_fin, descripcion
FROM calendario_ccd
WHERE activo = TRUE
ORDER BY fecha_inicio
LIMIT 5;
```

Uso: fechas institucionales, inscripciones, cronograma e hitos del CCD.

### 6.3 Rama Google Calendar

Intención:

```text
calendario_google
```

Flujo:

```text
Google Calendar - Consultar próximos eventos CCD
        ↓
Code - Formatear eventos Google Calendar
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

Esta rama es independiente de PostgreSQL y no usa `Merge`.

Busca próximos eventos de los siguientes 90 días y devuelve hasta 5 resultados.

Términos que pueden activar esta rama:

- agenda CCD
- agenda del CCD
- actividades del CCD
- actividades próximas
- eventos próximos
- próximas actividades
- qué actividades hay
- qué eventos hay
- Google Calendar

### 6.4 Rama oferta de cursos

Intención:

```text
oferta_cursos
```

Flujo:

```text
PostgreSQL - Oferta cursos
        ↓
Code - Formatear oferta cursos
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

Consulta:

```sql
SELECT nombre, categoria
FROM cursos_ccd
WHERE activo = TRUE
ORDER BY categoria, nombre;
```

El formateo agrupa los cursos por categoría.

### 6.5 Rama cursos por estudiante

Intención:

```text
cursos_estudiante
```

Flujo:

```text
Code - Verificar codigo estudiante
        ↓
IF - Tiene codigo estudiante
        ├── Sí
        │     ↓
        │   PostgreSQL - Cursos estudiante
        │     ↓
        │   Code - Formatear cursos estudiante
        │
        └── No
              ↓
            Code - Pedir codigo estudiante

Ambas rutas:
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

La consulta usa parámetros:

```sql
SELECT 
    e.codigo_unab,
    e.id_estudiante,
    e.nombre_completo,
    e.programa_academico,
    e.semestre_actual,
    ce.codigo_materia,
    ce.nombre_materia,
    ce.semestre,
    ce.anio,
    ce.fecha_matricula
FROM cursos_estudiantes ce
JOIN estudiante e ON e.id_estudiante = ce.id_estudiante
WHERE 
    UPPER(e.codigo_unab) = UPPER($1)
    OR e.id_estudiante::TEXT = $2
ORDER BY ce.anio, ce.codigo_curso;
```

`queryReplacement`:

```text
={{ $json.codigo_unab || '' }},{{ $json.id_estudiante || '' }}
```

Si no hay código, el bot solicita el código UNAB o ID de estudiante con ejemplos.

### 6.6 Rama noticias

Intención:

```text
noticias
```

Flujo:

```text
PostgreSQL - Noticias
        ↓
Code - Formatear noticias
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

Consulta:

```sql
SELECT titulo, fecha_publicacion, enlace
FROM noticias_ccd
WHERE activo = TRUE AND disponible_chatbot = TRUE
ORDER BY fecha_publicacion DESC
LIMIT 3;
```

### 6.7 Rama RAG documental

Intención:

```text
rag_documentos
```

Flujo:

```text
Code - Preparar pregunta RAG
        ↓
HTTP Request - OpenRouter Embedding pregunta
        ↓
PostgreSQL - Buscar chunks similares
        ↓
Code - Armar contexto RAG
        ↓
HTTP Request - OpenRouter Generar respuesta RAG
        ↓
Code - Extraer respuesta OpenRouter
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

Se usa para preguntas sobre:

- Certificación CCD.
- Insignias.
- Requisitos.
- Ruta CCD.
- Documentos institucionales.
- Información no estructurada.

### 6.8 Rama desconocido

Intención:

```text
desconocido
```

Flujo:

```text
Code - Respuesta desconocido
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

Esta rama responde con orientación amable y puede registrar preguntas sin resolver con `PostgreSQL - Log desconocido` si el nodo se activa.

---

## 7. RAG con OpenRouter y pgvector

La rama RAG usa embeddings y generación de respuesta mediante OpenRouter.

### 7.1 Embedding de la pregunta

Nodo:

```text
HTTP Request - OpenRouter Embedding pregunta
```

Endpoint:

```text
https://openrouter.ai/api/v1/embeddings
```

Modelo:

```text
openai/text-embedding-3-small
```

### 7.2 Búsqueda de chunks similares

Nodo:

```text
PostgreSQL - Buscar chunks similares
```

Consulta:

```sql
SELECT 
    dc.chunk_text,
    dc.metadata,
    dr.nombre_documento,
    dr.categoria
FROM document_chunks dc
LEFT JOIN documentos_rag dr ON dr.id = dc.documento_id
WHERE dc.embedding IS NOT NULL
ORDER BY dc.embedding <-> '[embedding_pregunta]'::vector
LIMIT 5;
```

El operador `<->` permite ordenar por distancia vectorial.

### 7.3 Construcción de contexto

Nodo:

```text
Code - Armar contexto RAG
```

Combina los chunks recuperados en un bloque de contexto documental con:

- Categoría.
- Título.
- Contenido.

### 7.4 Generación de respuesta

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

Instrucciones principales del prompt:

- Responder en español colombiano.
- Usar tono claro, amable e institucional.
- Usar únicamente el contexto proporcionado.
- No inventar fechas, requisitos, nombres de cursos ni enlaces.
- Indicar amablemente cuando no hay información suficiente.
- Organizar con viñetas si es útil.

---

## 8. Pulido final y respuesta Telegram

Todas las ramas terminan en:

```text
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

El nodo `Code - Pulir respuesta final`:

- Garantiza que exista una respuesta.
- Limita longitud a 3500 caracteres.
- Agrega cierre amable si corresponde.
- Devuelve `respuesta_final` y `respuesta`.

El nodo Telegram envía:

```text
={{ $json.respuesta_final || $json.respuesta }}
```

al chat:

```text
={{ $json.chat_id }}
```

---

## 9. Base de datos relacionada

Áreas principales:

| Área | Tablas / objetos | Uso |
|------|------------------|-----|
| Calendario | `calendario_ccd` | Fechas institucionales |
| Oferta | `cursos_ccd` | Cursos disponibles |
| Estudiantes | `estudiante`, `cursos_estudiantes` | Historial del estudiante |
| Noticias | `noticias_ccd` | Noticias visibles al chatbot |
| RAG | `documentos_rag`, `document_chunks` | Documentos y búsqueda vectorial |
| Mejora | `preguntas_sin_resolver` | Registro opcional de desconocidos |

La columna `document_chunks.embedding` usa `vector(1536)`.

---

## 10. Credenciales necesarias

| Credencial en n8n | Uso |
|-------------------|-----|
| `Telegram CCD Bot` | Entrada y salida por Telegram |
| `PostgreSQL CCD` | Calendario, oferta y noticias |
| `PostgreSQL CCD Real` | Cursos por estudiante y RAG |
| `Google Calendar CCD` | Eventos próximos |
| OpenRouter API Key | Embeddings y generación RAG |

OpenRouter se configura en los headers de los nodos HTTP:

```text
Authorization: Bearer REEMPLAZA_AQUI_TU_OPENROUTER_API_KEY
```

Google Calendar usa:

```text
REEMPLAZA_CALENDAR_ID_CCD
```

---

## 11. Buenas prácticas del flujo actualizado

| # | Práctica |
|---|----------|
| 1 | Mantener respuestas simples fuera del modelo. |
| 2 | Priorizar PostgreSQL para datos estructurados. |
| 3 | Usar Google Calendar solo para agenda y eventos próximos. |
| 4 | Usar RAG solo para preguntas documentales. |
| 5 | No inventar fechas, requisitos, cursos ni enlaces. |
| 6 | Centralizar el tono en `Code - Pulir respuesta final`. |
| 7 | Mantener el Switch con intenciones controladas. |
| 8 | Usar consultas parametrizadas para datos de estudiante. |
| 9 | Validar embeddings antes de probar RAG. |
| 10 | Versionar el workflow JSON junto con la documentación. |

---

## 12. Pruebas recomendadas

| Pregunta | Intención esperada |
|----------|--------------------|
| `Hola` | `saludo` |
| `Gracias` | `agradecimiento` |
| `Chao` | `despedida` |
| `¿Qué es el CCD?` | `faq_directa` |
| `¿Cuándo son las inscripciones?` | `calendario` |
| `¿Qué eventos hay en la agenda del CCD?` | `calendario_google` |
| `¿Qué cursos hay disponibles?` | `oferta_cursos` |
| `Mis cursos` | `cursos_estudiante` |
| `Mis cursos U00175325` | `cursos_estudiante` |
| `¿Hay noticias recientes?` | `noticias` |
| `¿Cuáles son los requisitos para la certificación CCD?` | `rag_documentos` |
| `¿Cómo funcionan las insignias?` | `rag_documentos` |

---

## 13. Resultado esperado

Al implementar el workflow actualizado se obtiene:

1. Un chatbot modular y fácil de extender.
2. Menor consumo de modelos de lenguaje.
3. Respuestas más humanas e institucionales.
4. Consulta estructurada para cursos, calendario, noticias y estudiantes.
5. Consulta de agenda real mediante Google Calendar.
6. RAG documental con pgvector y OpenRouter.
7. Un único punto de salida por Telegram con respuesta pulida.

---

*Fin del documento — Diseño flujo n8n Chatbot CCD UNAB v2.0*
