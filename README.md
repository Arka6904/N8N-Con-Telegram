# 🤖 CCD UNAB — Chatbot Inteligente con n8n, PostgreSQL, Telegram y RAG

### Centro de Competencias Digitales · Universidad Autónoma de Bucaramanga · Colombia

> Asistente conversacional institucional para el **Centro de Competencias Digitales CCD UNAB**.  
> Funciona desde **Telegram**, se orquesta con **n8n**, consulta datos estructurados en **PostgreSQL**, realiza búsqueda vectorial con **pgvector** y genera respuestas documentales mediante **RAG** usando **OpenRouter**.

![n8n](https://img.shields.io/badge/n8n-Automation-orange)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Vector%20DB-blue?logo=postgresql)
![pgvector](https://img.shields.io/badge/pgvector-RAG-green)
![Telegram](https://img.shields.io/badge/Telegram-Bot-blue?logo=telegram)
![OpenRouter](https://img.shields.io/badge/OpenRouter-LLM-purple)
![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)
![License](https://img.shields.io/badge/Licencia-MIT-green)

---

## 📋 Tabla de contenidos

1. [Descripción general](#-descripción-general)
2. [Objetivo del proyecto](#-objetivo-del-proyecto)
3. [Características principales](#-características-principales)
4. [Arquitectura general](#-arquitectura-general)
5. [Flujo conversacional](#-flujo-conversacional)
6. [Intenciones soportadas](#-intenciones-soportadas)
7. [Tecnologías utilizadas](#-tecnologías-utilizadas)
8. [Estructura del repositorio](#-estructura-del-repositorio)
9. [Requisitos previos](#-requisitos-previos)
10. [Variables de entorno](#-variables-de-entorno)
11. [Infraestructura](#-infraestructura)
12. [Configuración de puertos](#-configuración-de-puertos)
13. [Despliegue de n8n](#-despliegue-de-n8n)
14. [Configuración de Telegram](#-configuración-de-telegram)
15. [Configuración de PostgreSQL y pgvector](#-configuración-de-postgresql-y-pgvector)
16. [Modelo de datos](#-modelo-de-datos)
17. [Configuración de credenciales en n8n](#-configuración-de-credenciales-en-n8n)
18. [Importar el workflow en n8n](#-importar-el-workflow-en-n8n)
19. [Descripción de ramas del flujo](#-descripción-de-ramas-del-flujo)
20. [Configuración RAG documental](#-configuración-rag-documental)
21. [Pruebas desde Telegram](#-pruebas-desde-telegram)
22. [Checklist de despliegue](#-checklist-de-despliegue)
23. [Comandos útiles de diagnóstico](#-comandos-útiles-de-diagnóstico)
24. [Solución de problemas frecuentes](#-solución-de-problemas-frecuentes)
25. [Seguridad y privacidad](#-seguridad-y-privacidad)
26. [Costos y límites](#-costos-y-límites)
27. [Estado final del proyecto](#-estado-final-del-proyecto)
28. [Próximas mejoras](#-próximas-mejoras)
29. [Créditos](#-créditos)
30. [Licencia](#-licencia)

---

## 📌 Descripción general

Este proyecto implementa un chatbot institucional para el **Centro de Competencias Digitales CCD UNAB**.

El asistente permite responder preguntas frecuentes, saludos, despedidas, consultas de calendario, oferta de cursos, historial de cursos por estudiante, noticias institucionales y preguntas documentales mediante un flujo RAG.

El flujo está diseñado para evitar llamadas innecesarias a modelos de lenguaje. Primero normaliza el mensaje, analiza el contexto conversacional, detecta la intención y enruta la pregunta hacia la rama correspondiente. Finalmente, todas las respuestas pasan por un nodo de pulido antes de enviarse al usuario por Telegram.

---

## 🎯 Objetivo del proyecto

Construir un asistente conversacional que:

- Atienda consultas comunes del CCD de forma rápida.
- Responda en un tono amable, claro e institucional.
- Consulte datos estructurados desde PostgreSQL.
- Permita buscar información documental mediante RAG.
- Use Telegram como canal principal de interacción.
- Sea importable, mantenible y extensible en n8n.
- Reduzca llamadas innecesarias a modelos de lenguaje.
- Permita diagnosticar errores mediante consultas SQL y logs de n8n.

---

## ✨ Características principales

- Atención automática desde Telegram.
- Orquestación visual con n8n.
- Clasificación de intención mediante reglas.
- Respuestas directas para saludos, agradecimientos, despedidas y preguntas frecuentes.
- Consulta de calendario institucional desde PostgreSQL.
- Consulta opcional de eventos desde Google Calendar.
- Consulta de oferta activa de cursos CCD.
- Consulta de cursos asociados a estudiantes por código UNAB o ID.
- Consulta de noticias institucionales.
- Búsqueda documental con PostgreSQL y pgvector.
- Generación de embeddings mediante OpenRouter.
- Generación de respuestas RAG usando contexto documental.
- Nodo final de pulido para mantener tono institucional.
- Registro opcional de preguntas sin resolver.
- Arquitectura modular y mantenible.

---

## 🏗️ Arquitectura general

```mermaid
flowchart TD
    A[Usuario en Telegram] --> B[Telegram Trigger]
    B --> C[Code - Normalizar pregunta avanzada]
    C --> D[Code - Analizar contexto conversacional]
    D --> E[Code - Detectar intención avanzada]
    E --> F{Switch - Intención}

    F --> G[FAQ directa]
    F --> H[Saludo]
    F --> I[Agradecimiento]
    F --> J[Despedida]
    F --> K[Calendario PostgreSQL]
    F --> L[Google Calendar]
    F --> M[Oferta de cursos]
    F --> N[Cursos por estudiante]
    F --> O[Noticias]
    F --> P[RAG documental]
    F --> Q[Desconocido]

    P --> R[OpenRouter - Embedding pregunta]
    R --> S[PostgreSQL + pgvector - Buscar chunks similares]
    S --> T[Code - Armar contexto RAG]
    T --> U[OpenRouter - Generar respuesta RAG]

    G --> V[Code - Pulir respuesta final]
    H --> V
    I --> V
    J --> V
    K --> V
    L --> V
    M --> V
    N --> V
    O --> V
    U --> V
    Q --> V

    V --> W[Telegram - Respuesta dinámica]
    W --> X[Usuario recibe respuesta]
```

Arquitectura lógica resumida:

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
  ├─ faq_directa
  ├─ saludo
  ├─ agradecimiento
  ├─ despedida
  ├─ calendario
  ├─ calendario_google
  ├─ oferta_cursos
  ├─ cursos_estudiante
  ├─ noticias
  ├─ rag_documentos
  └─ desconocido
        ↓
Code - Pulir respuesta final
        ↓
Telegram - Respuesta dinamica
```

El diseño separa tres tipos de respuesta:

- Respuestas directas sin base de datos.
- Consultas estructuradas sobre PostgreSQL.
- Consultas documentales con RAG y OpenRouter.

---

## 🔁 Flujo conversacional

Cada mensaje recibido pasa por estas etapas:

1. **Normalización:** limpieza del texto, detección de palabras clave, extracción de códigos.
2. **Análisis de contexto:** detección de saludos, despedidas, agradecimientos, menciones de fechas o historial.
3. **Detección de intención:** clasificación por reglas hacia una de las ramas del Switch.
4. **Enrutamiento:** el Switch dirige el flujo a la rama correspondiente.
5. **Respuesta:** cada rama genera un campo `respuesta`.
6. **Pulido:** el nodo final ajusta tono, longitud y agrega cierres si faltan.
7. **Envío:** Telegram recibe y entrega la respuesta al usuario.

---

## 🧭 Intenciones soportadas

| Intención | Descripción |
|-----------|-------------|
| `faq_directa` | Preguntas frecuentes con respuesta predefinida |
| `saludo` | Saludos del usuario |
| `agradecimiento` | Expresiones de agradecimiento |
| `despedida` | Despedidas del usuario |
| `calendario` | Consulta de eventos en PostgreSQL |
| `calendario_google` | Consulta de eventos en Google Calendar |
| `oferta_cursos` | Consulta de cursos activos |
| `cursos_estudiante` | Historial académico por código o ID |
| `noticias` | Noticias institucionales recientes |
| `rag_documentos` | Preguntas documentales con RAG |
| `desconocido` | Preguntas no clasificadas |

---

## 🛠️ Tecnologías utilizadas

| Tecnología | Uso |
|------------|-----|
| Azure VM | Servidor principal |
| Ubuntu Server | Sistema operativo base |
| Docker | Ejecución de contenedores |
| Coolify | Administración de servicios |
| Traefik / Coolify Proxy | Exposición HTTPS |
| n8n | Motor de automatización y orquestación |
| n8n Task Runners | Ejecución de código JavaScript |
| PostgreSQL | Almacenamiento estructurado |
| pgvector | Búsqueda vectorial para RAG |
| Telegram Bot API | Canal de comunicación |
| OpenRouter API | Embeddings y generación de respuestas |
| Google Calendar | Eventos opcionales |

---

## 📁 Estructura del repositorio

```text
ccd-chatbot/
├── n8n/
│   └── ccd-chatbot-flujo-principal-mvp-mejorado.json
├── sql/
│   ├── schema.sql
│   └── datos_ejemplo.sql
├── docs/
│   └── arquitectura.png
└── README.md
```

---

## ✅ Requisitos previos

Antes de configurar el proyecto se requiere:

- Una VM Linux disponible.
- Docker instalado o gestionado mediante Coolify.
- Coolify funcionando con proxy HTTPS.
- Una instancia de n8n desplegada.
- Una instancia PostgreSQL con soporte para pgvector.
- Un bot de Telegram creado desde BotFather.
- Una API Key de OpenRouter.
- Opcionalmente, una cuenta de Google con acceso al calendario CCD.

Placeholders usados en esta documentación:

```text
TU_TOKEN_TELEGRAM
TU_API_KEY_OPENROUTER
TU_PASSWORD_POSTGRES
TU_DOMINIO_N8N
```

---

## 🌐 Variables de entorno

Variables de n8n:

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

## 🏢 Infraestructura

La infraestructura se compone de:

- **Azure VM:** servidor principal.
- **Ubuntu Server:** sistema operativo base.
- **Docker:** ejecución de contenedores.
- **Coolify:** administración de servicios.
- **Traefik / Coolify Proxy:** exposición HTTPS.
- **n8n:** motor de automatización y orquestación.
- **PostgreSQL:** almacenamiento de datos estructurados y documentos.
- **pgvector:** búsqueda vectorial para RAG.

---

## 🔌 Configuración de puertos

| Puerto | Uso |
|--------|-----|
| 80 TCP | Tráfico HTTP gestionado por el proxy |
| 443 TCP | Tráfico HTTPS gestionado por el proxy |
| 8000 TCP | Acceso a la interfaz de Coolify |

El puerto **5678** corresponde al puerto interno de n8n. No se expone directamente cuando n8n se publica mediante el proxy de Coolify.

---

## 🚀 Despliegue de n8n

1. Crear un nuevo servicio en Coolify.
2. Seleccionar imagen de n8n.
3. Configurar dominio público: `https://TU_DOMINIO_N8N`
4. Configurar variables de entorno.
5. Habilitar HTTPS mediante el proxy de Coolify.
6. Iniciar el servicio.
7. Acceder al editor web de n8n.

---

## 💬 Configuración de Telegram

Telegram requiere que el webhook esté disponible por HTTPS.

1. Crear credencial de Telegram en n8n.
2. Usar el token del bot: `TU_TOKEN_TELEGRAM`
3. Configurar el nodo **Telegram Trigger**.
4. Activar el workflow.
5. Verificar que Telegram pueda entregar mensajes al webhook.

---

## 🗄️ Configuración de PostgreSQL y pgvector

Habilitar la extensión pgvector:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

La tabla de chunks usa una columna vectorial con dimensión 1536, correspondiente al modelo `openai/text-embedding-3-small`:

```sql
embedding VECTOR(1536)
```

El proyecto usa dos bases de datos:

**Base de datos 1** — datos estructurados generales:
- Calendario, oferta de cursos, noticias.

**Base de datos 2** — datos reales y RAG:
- Estudiantes, cursos por estudiante, documentos RAG, chunks, embeddings.

---

## 📊 Modelo de datos

| Tabla | Descripción |
|-------|-------------|
| `calendario_ccd` | Eventos o fechas institucionales |
| `cursos_ccd` | Oferta de cursos activos |
| `noticias_ccd` | Noticias disponibles para el chatbot |
| `estudiante` | Información general del estudiante |
| `cursos_estudiantes` | Cursos asociados al estudiante |
| `documentos_rag` | Documentos fuente para RAG |
| `document_chunks` | Fragmentos para búsqueda vectorial |
| `preguntas_sin_resolver` | Registro opcional de preguntas no atendidas |

---

## 🔑 Configuración de credenciales en n8n

| Credencial | Nodos que la usan |
|------------|-------------------|
| Telegram CCD Bot | Telegram Trigger, Telegram - Respuesta dinamica |
| PostgreSQL Base de datos 1 | Calendario, Oferta cursos, Noticias |
| PostgreSQL Base de datos 2 | Cursos estudiante, Buscar chunks similares |
| Google Calendar CCD | Google Calendar - Consultar próximos eventos CCD |
| OpenRouter | HTTP Request - Embedding, HTTP Request - Generar respuesta RAG |

---

## 📥 Importar el workflow en n8n

1. Abrir n8n.
2. Ir a **Workflows**.
3. Seleccionar **Import from File**.
4. Importar el archivo:

```text
n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json
```

5. Asignar credenciales a cada nodo.
6. Revisar placeholders.
7. Activar el workflow.

---

## 🌿 Descripción de ramas del flujo

### Rama FAQ, saludos y respuestas simples

Intenciones: `faq_directa`, `saludo`, `agradecimiento`, `despedida`

```text
Code - Preparar respuesta directa
↓
Code - Pulir respuesta final
↓
Telegram - Respuesta dinamica
```

### Rama de calendario

```text
PostgreSQL - Calendario
↓
Code - Formatear calendario
↓
Code - Pulir respuesta final
↓
Telegram - Respuesta dinamica
```

Consulta típica:

```sql
SELECT evento, fecha_inicio, fecha_fin, descripcion
FROM calendario_ccd
WHERE activo = TRUE
ORDER BY fecha_inicio
LIMIT 5;
```

### Rama de oferta de cursos

```text
PostgreSQL - Oferta cursos
↓
Code - Formatear oferta cursos
↓
Code - Pulir respuesta final
↓
Telegram - Respuesta dinamica
```

Consulta típica:

```sql
SELECT nombre, categoria
FROM cursos_ccd
WHERE activo = TRUE
ORDER BY categoria, nombre;
```

### Rama de consulta de estudiantes

```text
Code - Verificar codigo estudiante
↓
IF - Tiene codigo estudiante
  ├─ Sí → PostgreSQL - Cursos estudiante
  │       ↓
  │     Code - Formatear cursos estudiante
  │       ↓
  │     Code - Pulir respuesta final
  │       ↓
  │     Telegram - Respuesta dinamica
  │
  └─ No → Code - Pedir codigo estudiante
          ↓
        Code - Pulir respuesta final
          ↓
        Telegram - Respuesta dinamica
```

Consulta parametrizada:

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
    OR e.id_estudiante::TEXT = $1
ORDER BY ce.anio, ce.codigo_curso;
```

### Rama de noticias

```text
PostgreSQL - Noticias
↓
Code - Formatear noticias
↓
Code - Pulir respuesta final
↓
Telegram - Respuesta dinamica
```

Consulta típica:

```sql
SELECT titulo, fecha_publicacion, enlace
FROM noticias_ccd
WHERE activo = TRUE AND disponible_chatbot = TRUE
ORDER BY fecha_publicacion DESC
LIMIT 3;
```

### Rama RAG documental

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

---

## 🧠 Configuración RAG documental

### Generación de embeddings

Modelo usado:

```text
openai/text-embedding-3-small
```

Endpoint:

```text
https://openrouter.ai/api/v1/embeddings
```

Dimensión esperada: `1536`

### Búsqueda vectorial con pgvector

```sql
SELECT 
    dc.chunk_text,
    dc.metadata,
    dr.nombre_documento,
    dr.categoria
FROM document_chunks dc
LEFT JOIN documentos_rag dr ON dr.id = dc.documento_id
WHERE dc.embedding IS NOT NULL
ORDER BY dc.embedding <-> '[VECTOR_DE_LA_PREGUNTA]'::vector
LIMIT 5;
```

### Generación de respuesta con OpenRouter

Endpoint:

```text
https://openrouter.ai/api/v1/chat/completions
```

Modelo configurado:

```text
openai/gpt-4o-mini
```

El prompt indica al modelo responder en español colombiano, usar tono claro e institucional, usar únicamente el contexto proporcionado y no inventar fechas, requisitos, cursos ni enlaces.

---

## 🧪 Pruebas desde Telegram

Pruebas sugeridas:

```text
Hola
Gracias
Chao
¿Qué es el CCD?
¿Qué cursos hay disponibles?
¿Cuándo son las inscripciones?
¿Qué eventos hay en la agenda del CCD?
Mis cursos
Mis cursos U00175325
Mis cursos 20240005
¿Hay noticias recientes?
¿Cuáles son los requisitos para la certificación CCD?
¿Cómo funcionan las insignias?
```

Resultados esperados:

- Saludos y agradecimientos responden sin base de datos.
- Calendario consulta Base de datos 1.
- Eventos de agenda consultan Google Calendar si está configurado.
- Cursos por estudiante consultan Base de datos 2.
- RAG consulta documentos y responde con contexto.

---

## ✅ Checklist de despliegue

- [ ] VM disponible con Docker y Coolify.
- [ ] n8n desplegado con HTTPS.
- [ ] PostgreSQL con pgvector habilitado.
- [ ] Tablas creadas en ambas bases de datos.
- [ ] Chunks cargados con embeddings.
- [ ] Bot de Telegram creado en BotFather.
- [ ] Credenciales configuradas en n8n.
- [ ] Workflow importado y activado.
- [ ] Pruebas realizadas desde Telegram.

---

## 🔧 Comandos útiles de diagnóstico

Verificar extensión pgvector:

```sql
SELECT extname FROM pg_extension WHERE extname = 'vector';
```

Verificar chunks con embeddings:

```sql
SELECT
  COUNT(*) AS total_chunks,
  COUNT(embedding) AS chunks_con_embedding
FROM document_chunks;
```

Verificar cursos activos:

```sql
SELECT COUNT(*) FROM cursos_ccd;
```

Verificar noticias disponibles:

```sql
SELECT COUNT(*)
FROM noticias_ccd
WHERE activo = TRUE AND disponible_chatbot = TRUE;
```

Verificar próximos eventos:

```sql
SELECT evento, fecha_inicio
FROM calendario_ccd
WHERE activo = TRUE
ORDER BY fecha_inicio
LIMIT 5;
```

Ver logs del contenedor n8n desde Coolify: abrir servicio n8n → Logs.

---

## 🛠️ Solución de problemas frecuentes

### Telegram no responde

- Verificar que el workflow esté activo.
- Confirmar `WEBHOOK_URL` en variables de entorno.
- Verificar credencial `Telegram CCD Bot` en n8n.
- Confirmar que el dominio tenga HTTPS válido.

### Error en PostgreSQL

- Revisar credenciales de Base de datos 1 y Base de datos 2.
- Verificar existencia de tablas.
- Ejecutar la consulta manualmente desde PostgreSQL.

### RAG no devuelve contexto

- Verificar cantidad de chunks con embeddings.
- Confirmar dimensión `VECTOR(1536)`.
- Revisar respuesta del nodo OpenRouter embeddings.
- Verificar que la tabla `document_chunks` no esté vacía.

### Google Calendar no devuelve eventos

- Revisar el Calendar ID configurado.
- Confirmar credencial `Google Calendar CCD`.
- Verificar que haya eventos próximos en el calendario.
- Confirmar permisos de la cuenta Google sobre el calendario.

### OpenRouter devuelve error

- Verificar `TU_API_KEY_OPENROUTER`.
- Confirmar que el modelo esté disponible.
- Revisar el formato del body en el nodo HTTP Request.

---

## 🔒 Seguridad y privacidad

- Las API Keys se almacenan como credenciales en n8n, no en el código.
- El webhook de Telegram solo acepta peticiones HTTPS.
- Los datos de estudiantes se consultan pero no se almacenan en logs.
- Se recomienda restringir el acceso al editor de n8n por IP o contraseña fuerte.

---

## 💰 Costos y límites

- **OpenRouter:** costo por tokens usados en embeddings y generación de respuestas. El modelo `openai/gpt-4o-mini` tiene un costo muy bajo por consulta.
- **Azure VM:** costo mensual según el tamaño de la VM seleccionada.
- **Telegram:** gratuito sin límites de mensajes para bots.
- **n8n self-hosted:** sin costo de licencia.
- **PostgreSQL self-hosted:** sin costo de licencia.

---

## 📦 Estado final del proyecto

- Chatbot funcionando sobre Telegram.
- Orquestación completa en n8n.
- Clasificación de intención por reglas.
- Respuestas directas para FAQ, saludos, agradecimientos y despedidas.
- Consulta de calendario desde PostgreSQL.
- Consulta opcional de eventos desde Google Calendar.
- Consulta de oferta de cursos.
- Consulta de cursos por estudiante con código UNAB o ID.
- Consulta de noticias institucionales.
- RAG documental con pgvector y OpenRouter.
- Respuesta final pulida antes de enviarse al usuario.

---

## 🔮 Próximas mejoras

- Agregar memoria conversacional por usuario.
- Registrar métricas de preguntas frecuentes.
- Crear panel de administración para preguntas sin resolver.
- Mejorar extracción de fechas desde lenguaje natural.
- Agregar más fuentes documentales al RAG.
- Implementar evaluación automática de calidad de respuestas.
- Crear pruebas automatizadas del workflow.

---

## 👥 Créditos

Proyecto desarrollado para el **Centro de Competencias Digitales** de la **Universidad Autónoma de Bucaramanga — UNAB**, Colombia.

---

## 📄 Licencia

Este proyecto está bajo la licencia **MIT**. Puedes usarlo, modificarlo y distribuirlo libremente con atribución.
