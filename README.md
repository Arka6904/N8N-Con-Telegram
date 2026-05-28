# CCD UNAB - Chatbot Inteligente con n8n, PostgreSQL, Telegram y RAG

## 1. Título del proyecto

**CCD UNAB - Chatbot Inteligente con n8n, PostgreSQL, Telegram y RAG**

Este proyecto implementa un chatbot institucional para el Centro de Competencias Digitales CCD UNAB. El asistente funciona desde Telegram, se orquesta con n8n, consulta información estructurada en PostgreSQL, utiliza búsqueda vectorial con pgvector y genera respuestas documentales mediante OpenRouter.

## 2. Descripción general

El chatbot permite atender preguntas frecuentes, saludos, despedidas, consultas de calendario, oferta de cursos, historial de cursos por estudiante, noticias y preguntas documentales mediante RAG.

El flujo está diseñado para evitar llamadas innecesarias a modelos de lenguaje. Primero normaliza el mensaje, analiza el contexto conversacional, detecta la intención y enruta la pregunta a la rama correspondiente. Las respuestas pasan por un nodo final de pulido antes de enviarse por Telegram.

## 3. Objetivo del proyecto

El objetivo es construir un asistente conversacional que:

- Atienda consultas comunes del CCD de forma rápida.
- Consulte datos estructurados desde PostgreSQL.
- Permita buscar información documental usando RAG.
- Responda en tono amable, claro e institucional.
- Funcione como flujo importable y mantenible en n8n.
- Use Telegram como canal principal de interacción.

## 4. Arquitectura general

Arquitectura lógica del flujo final:

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

El diseño separa claramente tres tipos de respuesta:

- Respuestas directas sin base de datos.
- Consultas estructuradas sobre PostgreSQL.
- Consultas documentales con RAG y OpenRouter.

## 5. Tecnologías utilizadas

- Azure VM
- Ubuntu Server
- Docker
- Coolify
- Traefik / Coolify Proxy
- n8n
- n8n Task Runners
- PostgreSQL
- pgvector
- Telegram Bot API
- OpenRouter API
- Google Calendar opcional
- JSON workflows
- SQL

## 6. Requisitos previos

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

## 7. Infraestructura usada

La infraestructura se compone de:

- **Azure VM:** servidor principal.
- **Ubuntu Server:** sistema operativo base.
- **Docker:** ejecución de contenedores.
- **Coolify:** administración de servicios.
- **Traefik / Coolify Proxy:** exposición HTTPS.
- **n8n:** motor de automatización y orquestación.
- **PostgreSQL:** almacenamiento de datos estructurados y documentos.
- **pgvector:** búsqueda vectorial para RAG.

## 8. Configuración de la VM y Coolify

La VM se usa como servidor central para desplegar servicios con Docker y Coolify.

Configuración general:

1. Crear la VM en Azure.
2. Instalar o habilitar Docker.
3. Instalar Coolify.
4. Acceder a Coolify desde el puerto público configurado.
5. Crear los servicios necesarios:
   - n8n.
   - PostgreSQL.
   - Servicios auxiliares si se requieren.

Coolify permite administrar variables de entorno, dominios, certificados HTTPS y reinicios de contenedores desde una interfaz web.

## 9. Configuración de puertos

Puertos utilizados:

```text
80 TCP
443 TCP
8000 TCP
```

Uso de cada puerto:

- **80 TCP:** tráfico HTTP gestionado por el proxy.
- **443 TCP:** tráfico HTTPS gestionado por el proxy.
- **8000 TCP:** acceso a la interfaz de Coolify.

El puerto **5678** corresponde al puerto interno de n8n dentro del contenedor. No se usa como acceso público directo cuando n8n se expone mediante el proxy de Coolify.

## 10. Despliegue de n8n

El servicio n8n se despliega como aplicación Docker administrada por Coolify.

Configuración general:

1. Crear un nuevo servicio en Coolify.
2. Seleccionar imagen de n8n.
3. Configurar dominio público:

```text
https://TU_DOMINIO_N8N
```

4. Configurar variables de entorno.
5. Habilitar HTTPS mediante el proxy de Coolify.
6. Iniciar el servicio.
7. Acceder al editor web de n8n.

## 11. Variables de entorno de n8n

Variables usadas:

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

Estas variables permiten que n8n genere webhooks correctos, funcione detrás del proxy HTTPS y use zona horaria de Colombia.

## 12. Configuración HTTPS y webhook de Telegram

Telegram requiere que el webhook del bot esté disponible por HTTPS.

El dominio público debe apuntar a n8n:

```text
https://TU_DOMINIO_N8N/
```

En n8n:

1. Crear credencial de Telegram.
2. Usar el token del bot:

```text
TU_TOKEN_TELEGRAM
```

3. Configurar el nodo **Telegram Trigger**.
4. Activar el workflow.
5. Verificar que Telegram pueda entregar mensajes al webhook.

## 13. Configuración de PostgreSQL y pgvector

PostgreSQL se usa para:

- Datos del calendario.
- Oferta de cursos.
- Datos de estudiantes.
- Noticias.
- Documentos RAG.
- Chunks documentales.
- Embeddings vectoriales.

pgvector se habilita con:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

La tabla de chunks usa una columna vectorial:

```sql
embedding VECTOR(1536)
```

La dimensión 1536 corresponde al modelo de embeddings `openai/text-embedding-3-small` usado mediante OpenRouter.

## 14. Bases de datos usadas

El proyecto usa dos bases:

### Base de datos 1

Usada para datos estructurados generales del chatbot, por ejemplo:

- Calendario.
- Oferta de cursos.
- Noticias.
- Información operativa del MVP.

### Base de datos 2

Usada para datos reales o enriquecidos, por ejemplo:

- Estudiantes.
- Cursos por estudiante.
- Documentos RAG.
- Chunks documentales.
- Embeddings.

En n8n se configuran como credenciales separadas para evitar mezclar responsabilidades.

## 15. Modelo de datos general

Tablas principales:

```text
calendario_ccd
cursos_ccd
noticias_ccd
estudiante
cursos_estudiantes
documentos_rag
document_chunks
preguntas_sin_resolver
```

Uso general:

- `calendario_ccd`: eventos o fechas institucionales.
- `cursos_ccd`: oferta de cursos.
- `noticias_ccd`: noticias disponibles para el chatbot.
- `estudiante`: información general del estudiante.
- `cursos_estudiantes`: cursos asociados al estudiante.
- `documentos_rag`: documentos fuente.
- `document_chunks`: fragmentos para búsqueda vectorial.
- `preguntas_sin_resolver`: registro opcional de preguntas no atendidas.

## 16. Enriquecimiento de datos

El enriquecimiento de datos consiste en preparar la información para que el chatbot pueda responder mejor:

- Normalizar textos.
- Cargar cursos y categorías.
- Registrar eventos de calendario.
- Cargar noticias.
- Cargar documentos institucionales.
- Dividir documentos en chunks.
- Generar embeddings para cada chunk.

Para RAG, cada documento se transforma en uno o más fragmentos dentro de `document_chunks`.

## 17. Configuración de credenciales en n8n

Credenciales necesarias:

### Telegram CCD Bot

Usada por:

- `Telegram Trigger`
- `Telegram - Respuesta dinamica`

Valor requerido:

```text
TU_TOKEN_TELEGRAM
```

### PostgreSQL Base de datos 1

Usada por:

- `PostgreSQL - Calendario`
- `PostgreSQL - Oferta cursos`
- `PostgreSQL - Noticias`

### PostgreSQL Base de datos 2

Usada por:

- `PostgreSQL - Cursos estudiante`
- `PostgreSQL - Buscar chunks similares`

### Google Calendar CCD

Usada por:

- `Google Calendar - Consultar próximos eventos CCD`

### OpenRouter

Usada por nodos HTTP:

- `HTTP Request - OpenRouter Embedding pregunta`
- `HTTP Request - OpenRouter Generar respuesta RAG`

Placeholder:

```text
TU_API_KEY_OPENROUTER
```

## 18. Creación del flujo final en n8n

El flujo final se crea importando el JSON del workflow en n8n.

Pasos generales:

1. Abrir n8n.
2. Ir a **Workflows**.
3. Seleccionar **Import from File**.
4. Importar el workflow JSON.
5. Revisar cada nodo.
6. Asignar credenciales.
7. Activar el workflow.

Workflow final sugerido:

```text
n8n/ccd-chatbot-flujo-principal-mvp-mejorado.json
```

## 19. Configuración del flujo final en n8n

Después de importar el workflow:

1. Configurar `Telegram Trigger`.
2. Configurar `Telegram - Respuesta dinamica`.
3. Configurar credenciales PostgreSQL.
4. Configurar OpenRouter en nodos HTTP.
5. Configurar Google Calendar si se usará esa rama.
6. Revisar placeholders.
7. Ejecutar pruebas manuales.
8. Activar el workflow.

El flujo debe terminar siempre en:

```text
Code - Pulir respuesta final
↓
Telegram - Respuesta dinamica
```

## 20. Descripción de ramas del flujo

El nodo `Switch - Intencion` enruta hacia:

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

Cada rama genera una respuesta parcial en el campo:

```text
respuesta
```

Luego el nodo `Code - Pulir respuesta final` produce:

```text
respuesta_final
respuesta
```

## 21. Rama de FAQ, saludos y respuestas simples

Las intenciones simples son:

- `faq_directa`
- `saludo`
- `agradecimiento`
- `despedida`

Todas pasan por:

```text
Code - Preparar respuesta directa
```

Este nodo toma `respuesta_directa` y la convierte en `respuesta`.

Luego continúa hacia:

```text
Code - Pulir respuesta final
↓
Telegram - Respuesta dinamica
```

## 22. Rama de calendario

La rama `calendario` consulta PostgreSQL:

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

Esta rama responde con fechas o eventos almacenados en Base de datos 1.

## 23. Rama de oferta de cursos

La rama `oferta_cursos` consulta la oferta activa:

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

La respuesta agrupa cursos por categoría.

## 24. Rama de consulta de estudiantes

La rama `cursos_estudiante` revisa si el usuario envió:

- Código UNAB.
- ID de estudiante.

Flujo:

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
    OR e.id_estudiante::TEXT = $2
ORDER BY ce.anio, ce.codigo_curso;
```

Esta rama usa Base de datos 2.

## 25. Rama de noticias

La rama `noticias` consulta noticias disponibles:

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

## 26. Rama RAG documental

La rama `rag_documentos` se usa para preguntas sobre:

- Certificación.
- Insignias.
- Ruta CCD.
- Requisitos.
- Documentos.
- Información institucional no estructurada.

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

## 27. Generación de embeddings

Los embeddings convierten texto en vectores numéricos.

Modelo usado:

```text
openai/text-embedding-3-small
```

Endpoint:

```text
https://openrouter.ai/api/v1/embeddings
```

La dimensión esperada es:

```text
1536
```

Cada chunk documental debe tener un embedding almacenado en `document_chunks.embedding`.

## 28. Búsqueda vectorial con pgvector

La búsqueda vectorial compara el embedding de la pregunta contra los embeddings almacenados.

Consulta típica:

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

El operador `<->` permite ordenar por distancia vectorial.

## 29. Generación de respuesta con OpenRouter

Después de recuperar los chunks relevantes, el flujo arma un contexto documental.

Luego llama a:

```text
https://openrouter.ai/api/v1/chat/completions
```

Modelo configurado:

```text
deepseek/deepseek-chat-v3.1:free
```

El prompt indica al modelo:

- Responder en español colombiano.
- Usar tono claro e institucional.
- Usar únicamente el contexto.
- No inventar fechas, requisitos, cursos ni enlaces.
- Indicar cuando no haya información suficiente.

## 30. Publicación del workflow

Para publicar el workflow:

1. Importar el JSON.
2. Asignar credenciales.
3. Verificar nodos con placeholders.
4. Guardar cambios.
5. Activar el workflow.
6. Probar desde Telegram.

Cuando el workflow está activo, Telegram envía mensajes al webhook de n8n.

## 31. Pruebas en Telegram

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

## 32. Errores comunes y soluciones

### Telegram no responde

Posibles causas:

- Workflow inactivo.
- Webhook HTTPS incorrecto.
- Credencial Telegram no asignada.
- Dominio mal configurado.

Solución:

- Revisar que el workflow esté activo.
- Confirmar `WEBHOOK_URL`.
- Verificar credencial `Telegram CCD Bot`.

### Error en PostgreSQL

Posibles causas:

- Credencial incorrecta.
- Tabla inexistente.
- Consulta apuntando a la base equivocada.

Solución:

- Revisar credenciales de Base de datos 1 y Base de datos 2.
- Verificar existencia de tablas.
- Ejecutar consulta manualmente desde PostgreSQL.

### RAG no devuelve contexto

Posibles causas:

- Chunks sin embeddings.
- Tabla `document_chunks` vacía.
- Dimensión del vector incorrecta.
- API de embeddings no respondió correctamente.

Solución:

- Verificar cantidad de chunks.
- Verificar `COUNT(embedding)`.
- Confirmar dimensión `VECTOR(1536)`.
- Revisar respuesta del nodo OpenRouter embeddings.

### Google Calendar no devuelve eventos

Posibles causas:

- Calendar ID incorrecto.
- Credencial no asignada.
- No hay eventos próximos.
- Permisos insuficientes sobre el calendario.

Solución:

- Revisar `REEMPLAZA_CALENDAR_ID_CCD`.
- Confirmar credencial `Google Calendar CCD`.
- Probar el nodo manualmente en n8n.

### OpenRouter devuelve error

Posibles causas:

- API key incorrecta.
- Modelo no disponible.
- Formato del body inválido.

Solución:

- Revisar placeholder `TU_API_KEY_OPENROUTER`.
- Probar otro modelo compatible.
- Revisar el nodo HTTP Request.

## 33. Comandos útiles de diagnóstico

Verificar extensión pgvector:

```sql
SELECT extname
FROM pg_extension
WHERE extname = 'vector';
```

Verificar chunks:

```sql
SELECT
  COUNT(*) AS total_chunks,
  COUNT(embedding) AS chunks_con_embedding
FROM document_chunks;
```

Verificar cursos:

```sql
SELECT COUNT(*)
FROM cursos_ccd;
```

Verificar noticias:

```sql
SELECT COUNT(*)
FROM noticias_ccd
WHERE activo = TRUE
  AND disponible_chatbot = TRUE;
```

Verificar calendario:

```sql
SELECT evento, fecha_inicio
FROM calendario_ccd
WHERE activo = TRUE
ORDER BY fecha_inicio
LIMIT 5;
```

Ver logs del contenedor n8n desde Coolify:

```text
Abrir servicio n8n → Logs
```

## 34. Estado final del proyecto

Estado logrado:

- Chatbot funcionando sobre Telegram.
- Orquestación completa en n8n.
- Clasificación de intención por reglas.
- Respuestas directas para FAQ, saludos, agradecimientos y despedidas.
- Consulta de calendario desde PostgreSQL.
- Consulta opcional de eventos desde Google Calendar.
- Consulta de oferta de cursos.
- Consulta de cursos por estudiante.
- Consulta de noticias.
- RAG documental con pgvector y OpenRouter.
- Respuesta final pulida antes de enviarse al usuario.

## 35. Próximas mejoras

Mejoras posibles:

- Agregar memoria conversacional por usuario.
- Registrar métricas de preguntas frecuentes.
- Crear panel de administración para preguntas sin resolver.
- Mejorar extracción de fechas desde lenguaje natural.
- Agregar más fuentes documentales al RAG.
- Implementar evaluación automática de calidad de respuestas.
- Crear pruebas automatizadas del workflow.
 
