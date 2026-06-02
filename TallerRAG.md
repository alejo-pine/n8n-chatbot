# Implementación RAG UniConnect con n8n

## Recursos del proyecto

| Recurso | Enlace |
|---------|--------|
| Registro de incidentes (Google Sheets) | [Abrir hoja de cálculo](https://docs.google.com/spreadsheets/d/1lytKo6dQ36jAgdxtirXSeCNIvuD62SjqjwRShivLCY8/edit?usp=sharing) |
| Manual oficial de UniConnect | [Abrir manual](https://docs.google.com/document/d/1TK-rfMfgdbfl93QBaRn15fuU4bv2lk1PNXE2B3NTR3o/edit?usp=sharing) |


## Visión general

Este documento describe la implementación completa de un sistema de soporte técnico para UniConnect que combina RAG (Retrieval-Augmented Generation) con un agente de inteligencia artificial en n8n. El sistema recibe mensajes de incidentes técnicos desde un portal web, consulta automáticamente el manual oficial de UniConnect para generar respuestas precisas, y en paralelo registra cada caso en Google Sheets y notifica al equipo técnico por Telegram.

La solución se divide en dos workflows independientes:

1. **Workflow de ingesta**: carga y vectoriza el manual en Supabase (se ejecuta solo cuando el documento cambia).
2. **Workflow de consulta**: recibe mensajes del portal web, consulta el RAG y devuelve respuestas al frontend sin modificar el código HTML.

***

## Supabase como Vector Store

### Tabla `documents`

La base del sistema RAG es una tabla en Supabase con la extensión pgvector habilitada:

```sql
create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(3072)
);
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | bigserial | Identificador autoincremental |
| `content` | text | Fragmento de texto del manual |
| `metadata` | jsonb | Información de origen (source, lines.from, lines.to) |
| `embedding` | vector(3072) | Vector generado por Gemini Embeddings |

El tipo `vector(3072)` se alineó con las dimensiones que produce el modelo de embeddings de Google Gemini. Si en algún momento se necesita limpiar la tabla y reiniciar el contador del `id`, se usa:

```sql
DELETE FROM documents;
SELECT setval(pg_get_serial_sequence('documents', 'id'), 1, false);
```

### Función `match_documents`

El Supabase Vector Store node de n8n requiere una función SQL que realice la búsqueda por similitud. Se creó con la siguiente definición:

```sql
DROP FUNCTION IF EXISTS match_documents;

CREATE OR REPLACE FUNCTION match_documents (
  query_embedding vector(3072),
  match_count int DEFAULT 5,
  filter jsonb DEFAULT '{}'
)
RETURNS TABLE (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    d.id,
    d.content,
    d.metadata,
    1 - (d.embedding <=> query_embedding) AS similarity
  FROM documents AS d
  WHERE d.metadata @> filter
  ORDER BY d.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

Puntos clave de esta función:

- Se usa el alias `d` para la tabla `documents` con el fin de evitar el error de ambigüedad `42702` ("column reference 'metadata' is ambiguous") que ocurre cuando el nombre de una columna coincide con un parámetro de la función.
- El operador `<=>` calcula la distancia coseno entre vectores; `1 - distancia` da el valor de similitud.
- `filter jsonb DEFAULT '{}'` permite filtrar por metadatos sin requerir un valor obligatorio.

***

## Workflow de ingesta RAG

Este workflow se ejecuta de forma manual o cuando el manual de UniConnect se actualiza. **No debe conectarse al webhook del chatbot**, ya que vectorizar el documento en cada mensaje sería costoso, lento e innecesario.

### Nodos y configuración

```
[Trigger manual]
      ↓
[Google Drive – Download File]
      ↓
[Supabase Vector Store – Add documents]
      ├── Sub-nodo Document: Default Data Loader
      └── Sub-nodo Embedding: Embeddings Google Gemini
```

#### Trigger manual

Se activó manualmente durante el taller. En producción se puede reemplazar por un trigger de Google Drive ("File Updated" o "File Created") para que la ingesta ocurra automáticamente cuando se sube una nueva versión del manual.

#### Google Drive – Download File

Descarga el [manual oficial de UniConnect](https://docs.google.com/document/d/1TK-rfMfgdbfl93QBaRn15fuU4bv2lk1PNXE2B3NTR3o/edit?usp=sharing) (PDF o DOCX) desde Google Drive y produce un binario que alimenta el siguiente nodo.

#### Supabase Vector Store – Add documents to vector store

- **Operation**: `Add documents to vector store`
- **Credenciales**: cuenta de Supabase con `service_role` key (no la `anon key`, ya que se necesitan permisos de escritura)
- **Tabla**: `documents`

Conectado a dos sub-nodos obligatorios:

**Default Data Loader (sub-nodo Document)**

| Parámetro | Valor |
|-----------|-------|
| Type of Data | Binary |
| Data Format | Automatically Detect by Mime Type |
| Text Splitting | Recursive Character Text Splitter |
| Chunk Size | 500 |
| Chunk Overlap | 50 |

Este sub-nodo convierte el binario del archivo a texto, lo divide en fragmentos de 500 caracteres con solapamiento de 50 para preservar el contexto entre chunks.

**Embeddings Google Gemini (sub-nodo Embedding)**

- **Credenciales**: API Key de Google AI Studio, configurada como "Google Gemini(PaLM) API account"
- **Modelo**: `models/gemini-embedding-001` o `models/gemini-embedding-2` (ambos producen vectores de 3072 dimensiones)

Genera el embedding de cada chunk y lo entrega al Supabase Vector Store para que lo inserte en la columna `embedding`.

### Resultado de la ingesta

Cada ejecución del workflow inserta múltiples filas en `documents`. Cada fila corresponde a un fragmento del manual con su texto, metadatos y vector. La tabla queda lista para recibir consultas semánticas del agente IA.

***

## Workflow de consulta (Webhook + Agente IA + RAG)

Este es el workflow conectado al portal web. Recibe mensajes del formulario de incidentes y del chat widget, consulta el RAG, y responde al frontend con la salida del agente.

### Arquitectura del flujo

```
[Webhook POST]
      ↓
[Edit Fields – normalización de input]
      ↓
[AI Agent – Groq + Supabase Vector Store Tool]
      ├── [Telegram – Send message]
      ├── [Google Sheets – Append row]
      └── [Edit Fields1 – empaqueta output para el Webhook]
```

### Nodos y configuración

#### Webhook

| Parámetro | Valor |
|-----------|-------|
| Método | POST |
| Respuesta | When last node finishes |
| Campos esperados | `mensaje_estudiante` y `rol` en el body JSON |

El modo `When last node finishes` hace que n8n espere a que todas las ramas del flujo terminen antes de responder al cliente HTTP. La respuesta que se devuelve es el JSON del **último nodo** según el orden del canvas (de arriba a abajo, de izquierda a derecha).

#### Edit Fields (normalización de input)

Toma el campo `mensaje_estudiante` del body del webhook y lo mapea a un campo interno `mensaje_estudiante`, que es el que se referencia en el prompt del AI Agent como `{{ $json.mensaje_estudiante}}`.

#### AI Agent

| Parámetro | Valor |
|-----------|-------|
| Chat Model | Groq Chat Model |
| Tool | Supabase Vector Store (As Tool for AI Agent) |
| Memory | Opcional (para contexto de conversación multi-turno) |

**System Message configurado:**

```
Eres un asistente de soporte técnico de UniConnect, una plataforma universitaria. Tu trabajo es:

1. Analizar el mensaje del usuario y extraer de forma estructurada:
   a. Nivel de prioridad (Crítica, Alta, Media, Baja).
   b. Componente afectado del sistema.
   c. Impacto estimado en la operación.

2. Buscar en el manual oficial de UniConnect usando la herramienta buscar_en_manual_uniconnect para encontrar información relevante que responda directamente al problema del usuario.

3. Redactar una respuesta completa que incluya:
   - El análisis estructurado (prioridad, componente, impacto).
   - Una respuesta clara y útil basada en el manual, con los pasos exactos para solucionar el problema.

Siempre usa la herramienta de búsqueda antes de responder. Si el manual contiene información relevante, cítala en tu respuesta. Si no encuentras información en el manual, indícalo claramente.

Mensaje del usuario: {{ $json.body.mensaje_estudiante }}
```

**Lógica de control por rol:**

El System Message también evalúa el campo `rol` recibido en el body del webhook. La lógica es:

- Si `rol` es `"estudiante"`: el agente responde normalmente, ejecutando el análisis RAG completo sobre el manual.
- Si `rol` es `"docente"` o `"administrativo"`: el agente responde con un mensaje fijo indicando que el sistema de soporte está disponible únicamente para estudiantes, sin consultar el manual.

Esto permite reutilizar el mismo workflow para cualquier tipo de usuario sin crear ramas separadas en n8n.

#### Supabase Vector Store (Tool del agente)

| Parámetro | Valor |
|-----------|-------|
| Operation Mode | Retrieve Documents (As Tool for AI Agent) |
| Tabla | documents |
| Límite de documentos | 4 |
| Credenciales | Cuenta de Supabase |
| Sub-nodo Embedding | Embeddings Google Gemini (mismo modelo que la ingesta) |

Es fundamental usar el **mismo modelo de embeddings** en la ingesta y en la consulta, para que los vectores sean comparables. Si se usan modelos distintos, la búsqueda por similitud no funcionará correctamente.

#### Telegram – Send a text message

Recibe el `output` del AI Agent y notifica al equipo técnico con el análisis estructurado + respuesta del manual.

#### Google Sheets – Append Row

Inserta una fila nueva en la hoja de seguimiento con:

- Fecha de ejecución (`{{ $now }}`)
- Mensaje original (`mensaje_estudiante`)
- Análisis y respuesta generada por el AI Agent (`output`)

#### Edit Fields1 (respuesta al Webhook)

Este es el nodo clave para que el frontend reciba la respuesta del agente sin modificar el HTML.

- **Mode**: Manual Mapping

```json
{{ $json.output }}
```

Este nodo:
1. Toma el campo `output` del AI Agent.
2. Al estar ubicado como el nodo más a la derecha del canvas, n8n lo toma como el último en finalizar y usa su JSON como cuerpo de la respuesta HTTP al Webhook.

***

## Integración con el portal web

### Selector de rol (chat widget y formulario)

Se agregó un selector de rol (`<select>`) en ambos puntos de entrada del portal para que el usuario indique su perfil antes de enviar un mensaje. El valor seleccionado se incluye en el body JSON de cada petición POST al webhook.

**Posición en el chat widget:** banda horizontal entre el header morado y el área de mensajes, con estilos coherentes (`bg-slate-900`, `border-slate-700`, `text-slate-200`).

**Posición en el formulario de incidentes:** campo independiente entre el textarea y el botón de envío, con la misma apariencia que los demás controles del formulario.

Opciones disponibles en ambos selectores:

| Valor | Etiqueta visible |
|-------|-----------------|
| `estudiante` | Estudiante (por defecto) |
| `docente` | Docente |
| `administrativo` | Administrativo |

### Chat widget

El JavaScript del chat widget lee la selección activa con:

```js
const chatRole = document.getElementById('chatRole');
```

Y la incluye en el payload de cada mensaje:

```js
body: JSON.stringify({ mensaje_estudiante: text, rol: chatRole.value })
```

La respuesta del agente se consume con el siguiente orden de prioridad (sin cambios):

```js
const reply = data.output
           || data.response_ai_agent
           || data.message
           || 'Incidente recibido y procesado correctamente.';
```

Antes de agregar el nodo `Edit Fields1`, el Webhook no devolvía el campo `output` del agente, por lo que el código siempre caía al texto de fallback hardcodeado. Con el nodo `Edit Fields1` devolviendo `{ "output": "..." }`, `data.output` se llena y el chatbot muestra la respuesta real.

### Formulario de incidentes

El formulario lee el rol seleccionado directamente en el handler de submit y lo agrega al payload:

```js
body: JSON.stringify({
    mensaje_estudiante: mensajeTexto,
    rol: document.getElementById('formRole').value
})
```

El formulario sigue verificando únicamente `response.ok` para mostrar el mensaje de éxito; no consume el contenido de `output`.

***

## Por qué funcionan las ramas en paralelo

Desde el AI Agent salen tres ramas en paralelo: Telegram, Google Sheets y Edit Fields1. Las tres se ejecutan completas. Sin embargo, cuando el Webhook está en modo `When last node finishes`, n8n espera a que todas terminen y luego usa el JSON del **último nodo según la posición en el canvas** (más a la derecha y abajo) como cuerpo de la respuesta HTTP. Al colocar `Edit Fields1` como el nodo más a la derecha, es su JSON el que regresa al frontend.

Este comportamiento es importante en producción: si se agregaran más nodos después de `Edit Fields1`, ese nodo dejaría de ser el "último" y la respuesta al frontend cambiaría. El orden visual del canvas en n8n **sí importa** cuando se usa el modo `When last node finishes`.

***

## Flujo de datos completo (resumen)

| Etapa | Entrada | Proceso | Salida |
|-------|---------|---------|--------|
| Ingesta | Archivo PDF/DOCX en Drive | Chunk + Embed | Filas en `documents` (Supabase) |
| Consulta | `mensaje_estudiante` (POST) | Clasificación + RAG | `output` estructurado |
| Notificación | `output` del agente | Formato de mensaje | Mensaje en Telegram |
| Registro | `output` + mensaje original | Append row | Fila en Google Sheets |
| Respuesta frontend | `output` del agente | Edit Fields1 | `{ "output": "..." }` vía HTTP |


## Autor

**Alejandro Pineda Gómez**  
Ingeniería de Sistemas y Computación