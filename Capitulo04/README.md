# Crear, probar y depurar un pipeline para logs JSON y no estructurados

## Metadatos

| Propiedad | Valor |
|---|---|
| Duración | 110 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica se creará el pipeline de ingestión `logs-acme-api-v1` en Elasticsearch 9.3.0 para procesar eventos de una API de ACME. El pipeline distinguirá entre eventos JSON serializados en `message` y líneas no estructuradas, preservará el contenido original en `event.original`, y normalizará los datos hacia campos ECS.

También se usarán `dissect`, `grok`, `date`, `set`, `rename`, `remove` y `convert`, junto con rutas `on_failure`, etiquetas de error y campos de diagnóstico. Finalmente, se validará el pipeline con la API `_simulate` e indexarán eventos en el data stream `logs-acme.api-default`.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Crear un pipeline de ingestión versionado y reutilizable para logs JSON y texto no estructurado.
- [ ] Aplicar `dissect` a formatos estables y `grok` a excepciones o formatos variables.
- [ ] Normalizar fechas, tipos numéricos y nombres de campos hacia ECS.
- [ ] Usar `_simulate` con ejecución detallada para inspeccionar resultados y depurar errores.
- [ ] Indexar eventos procesados y eventos con fallos controlados en `logs-acme.api-default`.

## Prerrequisitos

### Conocimientos

- Prácticas 1, 2 y 3 completadas.
- Uso básico de `curl`, JSON y Elasticsearch Query DSL.
- Comprensión de campos ECS, data streams e ingest pipelines.
- Comprensión básica de expresiones regulares y patrones `grok`.
- Conocimiento de la diferencia entre el timestamp de ingestión y el timestamp original del evento.

### Acceso y estado requerido

Debe disponer de:

- Elasticsearch 9.3.0 disponible en `http://localhost:9201`.
- Kibana 9.3.0 disponible en `http://localhost:5602`.
- Data stream existente: `logs-acme.api-default`.
- Template existente: `logs-acme-api-template`.
- Directorio de trabajo obligatorio: `/opt/elastic-labs`.
- Archivo `/opt/elastic-labs/.env` con permisos `0600`.
- Credenciales de laboratorio definidas mediante las variables `ELASTIC_USER` y `ELASTIC_PASSWORD`.

> **Importante:** esta práctica amplía los mappings y el data stream creados anteriormente. No elimine índices, templates, data streams ni volúmenes persistentes.

## Entorno de laboratorio

### Componentes utilizados

| Componente | Valor |
|---|---|
| Elasticsearch objetivo | 9.3.0 |
| Contenedor Elasticsearch | `es930-lab` |
| Kibana objetivo | 9.3.0 |
| Contenedor Kibana | `kibana930-lab` |
| Endpoint Elasticsearch | `http://localhost:9201` |
| Endpoint Kibana | `http://localhost:5602` |
| Cluster | `es930-lab-cluster` |
| Data stream | `logs-acme.api-default` |
| Pipeline | `logs-acme-api-v1` |
| Directorio de evidencias | `/opt/elastic-labs/work` |

### Preparación inicial

**Objetivo:** comprobar que Elasticsearch 9.3.0, el data stream y el entorno de trabajo están disponibles antes de crear el pipeline.

**Instrucciones:**

1. Abra una terminal y acceda al directorio obligatorio del laboratorio.

   ```bash
   cd /opt/elastic-labs
   ```

2. Cargue las credenciales del archivo `.env` sin mostrar la contraseña en pantalla.

   ```bash
   set -a
   source /opt/elastic-labs/.env
   set +a
   ```

3. Compruebe que el archivo de credenciales tiene permisos restrictivos.

   ```bash
   stat -c '%a %n' /opt/elastic-labs/.env
   ```

4. Inicie los servicios de Elasticsearch y Kibana 9.3.0 si no están activos.

   ```bash
   docker compose up -d es930-lab kibana930-lab
   ```

5. Espere hasta que Elasticsearch responda.

   ```bash
   until curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     http://localhost:9201/_cluster/health \
     | jq -e '.status == "yellow" or .status == "green"' >/dev/null; do
     echo "Esperando Elasticsearch 9.3.0..."
     sleep 3
   done
   ```

6. Cree el directorio de trabajo para los archivos de la práctica.

   ```bash
   mkdir -p /opt/elastic-labs/work
   ```

7. Verifique el cluster, el data stream y el template creados en la práctica anterior.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     http://localhost:9201/ \
     | jq '{cluster_name, version: .version.number, tagline}'

   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     http://localhost:9201/_data_stream/logs-acme.api-default \
     | jq '.data_streams[] | {name, generation, status, template}'

   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     http://localhost:9201/_index_template/logs-acme-api-template \
     | jq '.index_templates[].name'
   ```

**Salida esperada:**

- El archivo `.env` muestra permisos `600`.
- El cluster se identifica como `es930-lab-cluster`.
- La versión de Elasticsearch es `9.3.0`.
- El data stream `logs-acme.api-default` existe.
- El template `logs-acme-api-template` existe.

**Verificación:**

Ejecute:

```bash
curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
  http://localhost:9201/_cluster/health?pretty
```

El estado debe ser `yellow` o `green`. En un cluster de un solo nodo, `yellow` puede ser normal si existen réplicas no asignadas.

## Procedimiento paso a paso

### Paso 1. Diseñar la estrategia de normalización

**Objetivo:** definir el comportamiento esperado del pipeline antes de implementarlo.

**Instrucciones:**

1. Revise los dos formatos principales que procesará el pipeline.

   Evento JSON serializado en el campo `message`:

   ```json
   {
     "message": "{\"timestamp\":\"2026-08-24T14:35:12Z\",\"level\":\"INFO\",\"service\":\"api-gateway\",\"environment\":\"production\",\"request_id\":\"ab12-json\",\"method\":\"GET\",\"path\":\"/v1/orders\",\"status\":\"200\",\"duration_ms\":\"37\"}"
   }
   ```

   Línea de acceso HTTP estable:

   ```text
   2026-08-24 14:35:12 INFO api-gateway request_id=ab12-access GET /v1/orders 200 37
   ```

2. Identifique las decisiones de diseño aplicadas:

   | Necesidad | Decisión |
   |---|---|
   | Preservar la evidencia original | Copiar `message` a `event.original` antes de procesar. |
   | Detectar JSON | Ejecutar el procesador `json` únicamente cuando `message` comienza por `{`. |
   | Procesar accesos HTTP estables | Usar `dissect`, más eficiente para delimitadores fijos. |
   | Procesar errores de aplicación | Usar `grok`, adecuado para contenido variable como excepciones y mensajes largos. |
   | Normalizar hora del evento | Usar `date` para escribir la fecha en `@timestamp`. |
   | Evitar fallos de indexación por timestamps inválidos | Establecer inicialmente `@timestamp` con `_ingest.timestamp`. |
   | Normalizar tipos | Convertir códigos HTTP y duración a valores numéricos con `convert`. |
   | Evitar pérdida silenciosa | Usar `on_failure`, `tags`, `error.type` y `error.message`. |
   | Evitar conflictos de mapping | Eliminar un código HTTP inválido después de registrar su error de conversión. |

3. Tome nota de los campos normalizados que se utilizarán posteriormente en investigaciones:

   ```text
   @timestamp
   event.original
   event.dataset
   service.name
   service.environment
   log.level
   trace.id
   http.request.method
   http.response.status_code
   url.original
   event.duration_ms
   error.type
   error.message
   tags
   ```

**Salida esperada:**

Debe poder distinguir claramente cuándo aplicar `dissect` y cuándo aplicar `grok`:

- `dissect`: formato de acceso HTTP estable.
- `grok`: errores de aplicación con contenido variable.
- `json`: mensajes JSON serializados.
- `on_failure`: valores malformados o patrones que no coinciden.

**Verificación:**

Confirme que el diseño conserva el evento original antes de eliminar `message`. Esto es esencial para la trazabilidad y la depuración posterior.

---

### Paso 2. Crear el archivo de definición del pipeline

**Objetivo:** construir un pipeline con normalización, rutas de error y limpieza de campos temporales.

**Instrucciones:**

1. Cree el archivo `/opt/elastic-labs/work/logs-acme-api-v1.json`.

   ```bash
   cat > /opt/elastic-labs/work/logs-acme-api-v1.json <<'EOF'
   {
     "description": "Pipeline v1 para normalizar logs JSON y no estructurados de ACME API",
     "processors": [
       {
         "set": {
           "field": "@timestamp",
           "value": "{{{_ingest.timestamp}}}",
           "override": false,
           "tag": "set_fallback_ingest_timestamp"
         }
       },
       {
         "set": {
           "field": "event.original",
           "copy_from": "message",
           "override": false,
           "if": "ctx.message != null",
           "tag": "preserve_original_message"
         }
       },
       {
         "set": {
           "field": "event.kind",
           "value": "event",
           "override": false,
           "tag": "set_event_kind"
         }
       },
       {
         "set": {
           "field": "event.dataset",
           "value": "acme.api",
           "override": false,
           "tag": "set_event_dataset"
         }
       },
       {
         "set": {
           "field": "data_stream.type",
           "value": "logs",
           "override": false,
           "tag": "set_data_stream_type"
         }
       },
       {
         "set": {
           "field": "data_stream.dataset",
           "value": "acme.api",
           "override": false,
           "tag": "set_data_stream_dataset"
         }
       },
       {
         "set": {
           "field": "data_stream.namespace",
           "value": "default",
           "override": false,
           "tag": "set_data_stream_namespace"
         }
       },
       {
         "json": {
           "field": "message",
           "target_field": "_json",
           "add_to_root": false,
           "if": "ctx.message != null && ctx.message.trim().startsWith('{')",
           "tag": "parse_json_message",
           "on_failure": [
             {
               "set": {
                 "field": "tags",
                 "value": [
                   "json_parse_failure"
                 ],
                 "tag": "tag_json_parse_failure"
               }
             },
             {
               "set": {
                 "field": "error.type",
                 "value": "json_parse_failure",
                 "tag": "set_json_error_type"
               }
             },
             {
               "set": {
                 "field": "error.message",
                 "value": "{{{_ingest.on_failure_message}}}",
                 "tag": "set_json_error_message"
               }
             }
           ]
         }
       },
       {
         "rename": {
           "field": "_json.timestamp",
           "target_field": "log.timestamp",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_timestamp"
         }
       },
       {
         "rename": {
           "field": "_json.level",
           "target_field": "log.level",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_level"
         }
       },
       {
         "rename": {
           "field": "_json.service",
           "target_field": "service.name",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_service"
         }
       },
       {
         "rename": {
           "field": "_json.environment",
           "target_field": "service.environment",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_environment"
         }
       },
       {
         "rename": {
           "field": "_json.request_id",
           "target_field": "trace.id",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_request_id"
         }
       },
       {
         "rename": {
           "field": "_json.method",
           "target_field": "http.request.method",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_method"
         }
       },
       {
         "rename": {
           "field": "_json.path",
           "target_field": "url.original",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_path"
         }
       },
       {
         "rename": {
           "field": "_json.status",
           "target_field": "http.response.status_code",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_status"
         }
       },
       {
         "rename": {
           "field": "_json.duration_ms",
           "target_field": "event.duration_ms",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_duration"
         }
       },
       {
         "rename": {
           "field": "_json.error",
           "target_field": "error.message",
           "ignore_missing": true,
           "override": false,
           "tag": "rename_json_error"
         }
       },
       {
         "dissect": {
           "field": "message",
           "pattern": "%{log.timestamp} %{log.level} %{service.name} request_id=%{trace.id} %{http.request.method} %{url.original} %{http.response.status_code} %{event.duration_ms}",
           "if": "ctx.message != null && !ctx.message.trim().startsWith('{') && ctx.message.contains('request_id=') && !ctx.message.contains(' exception=')",
           "tag": "dissect_http_access",
           "on_failure": [
             {
               "set": {
                 "field": "tags",
                 "value": [
                   "dissect_parse_failure"
                 ],
                 "tag": "tag_dissect_failure"
               }
             },
             {
               "set": {
                 "field": "error.type",
                 "value": "dissect_parse_failure",
                 "tag": "set_dissect_error_type"
               }
             },
             {
               "set": {
                 "field": "error.message",
                 "value": "{{{_ingest.on_failure_message}}}",
                 "tag": "set_dissect_error_message"
               }
             }
           ]
         }
       },
       {
         "grok": {
           "field": "message",
           "patterns": [
             "%{TIMESTAMP_ISO8601:log.timestamp} %{LOGLEVEL:log.level} %{DATA:service.name} request_id=%{NOTSPACE:trace.id} exception=%{DATA:error.type} message=%{GREEDYDATA:error.message}"
           ],
           "trace_match": true,
           "if": "ctx.message != null && !ctx.message.trim().startsWith('{') && ctx.message.contains(' exception=')",
           "tag": "grok_application_error",
           "on_failure": [
             {
               "set": {
                 "field": "tags",
                 "value": [
                   "grok_parse_failure"
                 ],
                 "tag": "tag_grok_failure"
               }
             },
             {
               "set": {
                 "field": "error.type",
                 "value": "grok_parse_failure",
                 "tag": "set_grok_error_type"
               }
             },
             {
               "set": {
                 "field": "error.message",
                 "value": "{{{_ingest.on_failure_message}}}",
                 "tag": "set_grok_error_message"
               }
             }
           ]
         }
       },
       {
         "set": {
           "field": "service.name",
           "value": "acme-api",
           "override": false,
           "tag": "set_default_service"
         }
       },
       {
         "set": {
           "field": "service.environment",
           "value": "production",
           "override": false,
           "tag": "set_default_environment"
         }
       },
       {
         "date": {
           "field": "log.timestamp",
           "target_field": "@timestamp",
           "formats": [
             "yyyy-MM-dd HH:mm:ss",
             "ISO8601"
           ],
           "timezone": "UTC",
           "if": "ctx.log != null && ctx.log.timestamp != null",
           "tag": "normalize_event_timestamp",
           "on_failure": [
             {
               "set": {
                 "field": "tags",
                 "value": [
                   "date_parse_failure"
                 ],
                 "tag": "tag_date_failure"
               }
             },
             {
               "set": {
                 "field": "error.type",
                 "value": "date_parse_failure",
                 "tag": "set_date_error_type"
               }
             },
             {
               "set": {
                 "field": "error.message",
                 "value": "{{{_ingest.on_failure_message}}}",
                 "tag": "set_date_error_message"
               }
             }
           ]
         }
       },
       {
         "convert": {
           "field": "http.response.status_code",
           "type": "long",
           "ignore_missing": true,
           "tag": "convert_http_status",
           "on_failure": [
             {
               "set": {
                 "field": "tags",
                 "value": [
                   "status_code_conversion_failure"
                 ],
                 "tag": "tag_status_conversion_failure"
               }
             },
             {
               "set": {
                 "field": "error.type",
                 "value": "status_code_conversion_failure",
                 "tag": "set_status_conversion_error_type"
               }
             },
             {
               "set": {
                 "field": "error.message",
                 "value": "{{{_ingest.on_failure_message}}}",
                 "tag": "set_status_conversion_error_message"
               }
             },
             {
               "remove": {
                 "field": "http.response.status_code",
                 "ignore_missing": true,
                 "tag": "remove_invalid_status_code"
               }
             }
           ]
         }
       },
       {
         "convert": {
           "field": "event.duration_ms",
           "type": "long",
           "ignore_missing": true,
           "tag": "convert_duration_ms",
           "on_failure": [
             {
               "set": {
                 "field": "tags",
                 "value": [
                   "duration_conversion_failure"
                 ],
                 "tag": "tag_duration_conversion_failure"
               }
             },
             {
               "set": {
                 "field": "error.type",
                 "value": "duration_conversion_failure",
                 "tag": "set_duration_conversion_error_type"
               }
             },
             {
               "set": {
                 "field": "error.message",
                 "value": "{{{_ingest.on_failure_message}}}",
                 "tag": "set_duration_conversion_error_message"
               }
             },
             {
               "remove": {
                 "field": "event.duration_ms",
                 "ignore_missing": true,
                 "tag": "remove_invalid_duration"
               }
             }
           ]
         }
       },
       {
         "set": {
           "field": "tags",
           "value": [
             "unparsed_log"
           ],
           "if": "ctx.message != null && (ctx.log == null || ctx.log.timestamp == null) && (ctx.error == null || ctx.error.type == null)",
           "tag": "tag_unparsed_log"
         }
       },
       {
         "set": {
           "field": "error.type",
           "value": "unparsed_log",
           "if": "ctx.message != null && (ctx.log == null || ctx.log.timestamp == null) && (ctx.error == null || ctx.error.type == null)",
           "tag": "set_unparsed_error_type"
         }
       },
       {
         "set": {
           "field": "error.message",
           "value": "La línea no coincide con los formatos JSON, acceso HTTP o error de aplicación esperados.",
           "if": "ctx.message != null && (ctx.log == null || ctx.log.timestamp == null) && (ctx.error == null || ctx.error.type == null)",
           "tag": "set_unparsed_error_message"
         }
       },
       {
         "remove": {
           "field": "_json",
           "ignore_missing": true,
           "tag": "remove_json_temporary_object"
         }
       },
       {
         "remove": {
           "field": "log.timestamp",
           "ignore_missing": true,
           "if": "ctx.log != null && ctx.log.timestamp != null && (ctx.error == null || ctx.error.type != 'date_parse_failure')",
           "tag": "remove_normalized_timestamp_source"
         }
       },
       {
         "remove": {
           "field": "message",
           "ignore_missing": true,
           "tag": "remove_processed_message"
         }
       }
     ],
     "on_failure": [
       {
         "set": {
           "field": "tags",
           "value": [
             "pipeline_unhandled_failure"
           ],
           "tag": "tag_pipeline_failure"
         }
       },
       {
         "set": {
           "field": "error.type",
           "value": "pipeline_unhandled_failure",
           "tag": "set_pipeline_error_type"
         }
       },
       {
         "set": {
           "field": "error.message",
           "value": "{{{_ingest.on_failure_message}}}",
           "tag": "set_pipeline_error_message"
         }
       }
     ]
   }
   EOF
   ```

2. Valide localmente que el archivo contiene JSON correcto.

   ```bash
   jq empty /opt/elastic-labs/work/logs-acme-api-v1.json
   ```

3. Revise los nombres de procesadores y sus etiquetas de depuración.

   ```bash
   jq -r '.processors[] | keys[0] as $p | .[$p].tag // "sin-tag" | "\($p): \(.)"' \
     /opt/elastic-labs/work/logs-acme-api-v1.json
   ```

**Salida esperada:**

El comando `jq empty` no debe mostrar errores. El segundo comando debe listar procesadores como `set`, `json`, `dissect`, `grok`, `date`, `convert`, `rename` y `remove`.

**Verificación:**

Compruebe que el pipeline:

- Copia `message` en `event.original`.
- Usa `dissect` para accesos HTTP.
- Usa `grok` para líneas con `exception=`.
- Conserva un `@timestamp` válido ante una fecha inválida.
- Elimina `http.response.status_code` si no puede convertirse a `long`.
- Elimina `message` solamente después de preservar su contenido original.

---

### Paso 3. Crear el pipeline en Elasticsearch

**Objetivo:** cargar la definición del pipeline en Elasticsearch 9.3.0.

**Instrucciones:**

1. Cree o actualice el pipeline mediante la Ingest API.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X PUT \
     http://localhost:9201/_ingest/pipeline/logs-acme-api-v1 \
     --data-binary @/opt/elastic-labs/work/logs-acme-api-v1.json \
     | jq .
   ```

2. Recupere el pipeline desde Elasticsearch y guarde una evidencia.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     http://localhost:9201/_ingest/pipeline/logs-acme-api-v1 \
     | tee /opt/elastic-labs/work/evidencia-pipeline-creado.json \
     | jq '{pipeline_id: keys[0], description: .["logs-acme-api-v1"].description}'
   ```

3. Consulte las estadísticas de ingestión iniciales.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     http://localhost:9201/_nodes/stats/ingest?filter_path=nodes.*.ingest.pipelines.logs-acme-api-v1 \
     | jq .
   ```

**Salida esperada:**

La creación devuelve:

```json
{
  "acknowledged": true
}
```

La consulta posterior debe devolver el pipeline `logs-acme-api-v1` con su descripción.

**Verificación:**

Ejecute:

```bash
curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
  http://localhost:9201/_ingest/pipeline/logs-acme-api-v1 \
  | jq 'has("logs-acme-api-v1")'
```

El resultado debe ser:

```json
true
```

---

### Paso 4. Simular seis escenarios de procesamiento

**Objetivo:** validar el pipeline sin indexar documentos y observar el comportamiento de éxito y fallo controlado.

**Instrucciones:**

1. Cree un archivo de simulación con seis casos.

   ```bash
   cat > /opt/elastic-labs/work/simulate-logs-acme-api-v1.json <<'EOF'
   {
     "docs": [
       {
         "_id": "sim-json-valido",
         "_source": {
           "message": "{\"timestamp\":\"2026-08-24T14:35:12Z\",\"level\":\"INFO\",\"service\":\"api-gateway\",\"environment\":\"production\",\"request_id\":\"ab12-json\",\"method\":\"GET\",\"path\":\"/v1/orders\",\"status\":\"200\",\"duration_ms\":\"37\"}"
         }
       },
       {
         "_id": "sim-acceso-http",
         "_source": {
           "message": "2026-08-24 14:36:02 INFO api-gateway request_id=ab12-access POST /v1/orders 201 52"
         }
       },
       {
         "_id": "sim-error-aplicacion",
         "_source": {
           "message": "2026-08-24 14:37:18 ERROR orders-api request_id=ab12-error exception=IllegalStateException message=Order cannot be confirmed because payment is pending"
         }
       },
       {
         "_id": "sim-timestamp-invalido",
         "_source": {
           "message": "2026-99-99 25:99:99 INFO api-gateway request_id=ab12-date GET /v1/health 200 3"
         }
       },
       {
         "_id": "sim-status-invalido",
         "_source": {
           "message": "2026-08-24 14:39:44 WARN api-gateway request_id=ab12-status GET /v1/orders ABC 44"
         }
       },
       {
         "_id": "sim-linea-no-coincide",
         "_source": {
           "message": "socket peer closed unexpectedly after receiving incomplete payload"
         }
       }
     ]
   }
   EOF
   ```

2. Ejecute `_simulate` con `verbose=true` para obtener resultados intermedios de todos los procesadores.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/_ingest/pipeline/logs-acme-api-v1/_simulate?verbose=true' \
     --data-binary @/opt/elastic-labs/work/simulate-logs-acme-api-v1.json \
     | tee /opt/elastic-labs/work/evidencia-simulate-verbose.json \
     | jq '.docs[] | {id: .doc._id, final_source: .doc._source}'
   ```

3. Obtenga un resumen compacto de los campos relevantes de los seis documentos simulados.

   ```bash
   jq '
     .docs[] |
     {
       id: .doc._id,
       timestamp: .doc._source["@timestamp"],
       original: .doc._source.event.original,
       service: .doc._source.service.name,
       level: .doc._source.log.level,
       method: .doc._source.http.request.method,
       status: .doc._source.http.response.status_code,
       duration_ms: .doc._source.event.duration_ms,
       error_type: .doc._source.error.type,
       error_message: .doc._source.error.message,
       tags: .doc._source.tags
     }
   ' /opt/elastic-labs/work/evidencia-simulate-verbose.json
   ```

4. Inspeccione, específicamente, los pasos ejecutados para el documento de error de aplicación.

   ```bash
   jq '
     .docs[2].processor_results[]
     | {
         processor: .processor_type,
         tag: .tag,
         status: (if .error then "error" else "ok" end)
       }
   ' /opt/elastic-labs/work/evidencia-simulate-verbose.json
   ```

**Salida esperada:**

| Caso | Resultado esperado |
|---|---|
| JSON válido | Campos JSON normalizados, `@timestamp` igual al timestamp del evento, código HTTP `200` como número. |
| Acceso HTTP | `dissect` extrae método, URL, código `201`, duración `52` y `trace.id`. |
| Error de aplicación | `grok` extrae `IllegalStateException` en `error.type` y el detalle en `error.message`. |
| Timestamp inválido | Se conserva el timestamp de ingestión en `@timestamp`; aparecen `date_parse_failure` y detalles en `error`. |
| Código HTTP inválido | Aparece `status_code_conversion_failure`; el campo HTTP inválido se elimina para evitar conflicto de mapping. |
| Línea no coincidente | Aparecen `unparsed_log`, `error.type: unparsed_log` y `event.original`. |

**Verificación:**

Compruebe que todos los documentos incluyen `event.original`:

```bash
jq -r '.docs[] | [.doc._id, (.doc._source.event.original != null)] | @tsv' \
  /opt/elastic-labs/work/evidencia-simulate-verbose.json
```

Todos los valores de la segunda columna deben ser `true`.

---

### Paso 5. Analizar los resultados de depuración

**Objetivo:** interpretar la simulación detallada y confirmar que los fallos son controlados.

**Instrucciones:**

1. Revise el documento de timestamp inválido.

   ```bash
   jq '
     .docs[]
     | select(.doc._id == "sim-timestamp-invalido")
     | .doc._source
     | {
         "@timestamp": .["@timestamp"],
         log,
         error,
         tags,
         event
       }
   ' /opt/elastic-labs/work/evidencia-simulate-verbose.json
   ```

2. Revise el documento con código HTTP inválido.

   ```bash
   jq '
     .docs[]
     | select(.doc._id == "sim-status-invalido")
     | .doc._source
     | {
         http,
         error,
         tags,
         event
       }
   ' /opt/elastic-labs/work/evidencia-simulate-verbose.json
   ```

3. Revise el documento que no coincide con ningún patrón.

   ```bash
   jq '
     .docs[]
     | select(.doc._id == "sim-linea-no-coincide")
     | .doc._source
     | {
         "@timestamp": .["@timestamp"],
         event,
         error,
         tags,
         service
       }
   ' /opt/elastic-labs/work/evidencia-simulate-verbose.json
   ```

4. Explique en sus notas técnicas por qué el pipeline no descarta estos documentos:

   - Los errores de fecha no bloquean la indexación porque `@timestamp` tiene un valor inicial de `_ingest.timestamp`.
   - Los códigos HTTP inválidos se eliminan después de registrar la falla; de este modo, un texto como `ABC` no intenta indexarse en un campo `long`.
   - Las líneas no reconocidas conservan `event.original` y reciben etiquetas de diagnóstico.
   - El operador puede buscar posteriormente documentos con `tags` o `error.type` para investigar problemas de formato.

**Salida esperada:**

Los documentos problemáticos contienen evidencia suficiente para investigación y no se silencian mediante `ignore_failure` sin contexto.

**Verificación:**

El caso de código HTTP inválido no debe contener `http.response.status_code`:

```bash
jq '
  .docs[]
  | select(.doc._id == "sim-status-invalido")
  | .doc._source.http.response.status_code
' /opt/elastic-labs/work/evidencia-simulate-verbose.json
```

La salida esperada es `null`.

---

### Paso 6. Indexar los eventos procesados en el data stream

**Objetivo:** enviar eventos reales al data stream usando explícitamente el pipeline creado.

**Instrucciones:**

1. Indexe los seis eventos usando el parámetro `pipeline=logs-acme-api-v1`.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_doc/lab04-json-valido?pipeline=logs-acme-api-v1' \
     -d '{
       "message": "{\"timestamp\":\"2026-08-24T14:35:12Z\",\"level\":\"INFO\",\"service\":\"api-gateway\",\"environment\":\"production\",\"request_id\":\"lab04-json\",\"method\":\"GET\",\"path\":\"/v1/orders\",\"status\":\"200\",\"duration_ms\":\"37\"}"
     }' | jq .

   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_doc/lab04-acceso-http?pipeline=logs-acme-api-v1' \
     -d '{
       "message": "2026-08-24 14:36:02 INFO api-gateway request_id=lab04-access POST /v1/orders 201 52"
     }' | jq .

   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_doc/lab04-error-aplicacion?pipeline=logs-acme-api-v1' \
     -d '{
       "message": "2026-08-24 14:37:18 ERROR orders-api request_id=lab04-error exception=IllegalStateException message=Order cannot be confirmed because payment is pending"
     }' | jq .

   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_doc/lab04-timestamp-invalido?pipeline=logs-acme-api-v1' \
     -d '{
       "message": "2026-99-99 25:99:99 INFO api-gateway request_id=lab04-date GET /v1/health 200 3"
     }' | jq .

   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_doc/lab04-status-invalido?pipeline=logs-acme-api-v1' \
     -d '{
       "message": "2026-08-24 14:39:44 WARN api-gateway request_id=lab04-status GET /v1/orders ABC 44"
     }' | jq .

   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_doc/lab04-linea-no-coincide?pipeline=logs-acme-api-v1' \
     -d '{
       "message": "socket peer closed unexpectedly after receiving incomplete payload"
     }' | jq .
   ```

2. Fuerce una actualización de visibilidad para las búsquedas.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_refresh' \
     | jq .
   ```

3. Consulte los seis documentos indexados.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_search' \
     -d '{
       "size": 10,
       "sort": [
         {
           "_id": "asc"
         }
       ],
       "query": {
         "prefix": {
           "_id": "lab04-"
         }
       },
       "_source": [
         "@timestamp",
         "event.original",
         "event.dataset",
         "service.name",
         "service.environment",
         "log.level",
         "trace.id",
         "http.request.method",
         "http.response.status_code",
         "url.original",
         "event.duration_ms",
         "error.type",
         "error.message",
         "tags"
       ]
     }' \
     | tee /opt/elastic-labs/work/evidencia-documentos-indexados.json \
     | jq '.hits.hits[] | {_id, _source}'
   ```

**Salida esperada:**

Cada operación de indexación debe devolver una respuesta con:

```json
{
  "result": "created"
}
```

La búsqueda debe devolver seis documentos con identificadores que empiezan por `lab04-`.

**Verificación:**

Ejecute:

```bash
jq '.hits.total.value' /opt/elastic-labs/work/evidencia-documentos-indexados.json
```

El valor esperado es:

```text
6
```

---

## Validación y pruebas

### Validación mediante Elasticsearch Query DSL

**Objetivo:** comprobar que los eventos normalizados y los fallos controlados pueden investigarse desde Elasticsearch.

1. Consulte eventos HTTP exitosos con código 2xx.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_search' \
     -d '{
       "query": {
         "range": {
           "http.response.status_code": {
             "gte": 200,
             "lt": 300
           }
         }
       },
       "aggs": {
         "por_metodo": {
           "terms": {
             "field": "http.request.method"
           }
         }
       }
     }' \
     | jq '{total: .hits.total.value, por_metodo: .aggregations.por_metodo.buckets}'
   ```

2. Consulte todos los documentos con errores de parsing o conversión.

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X POST \
     'http://localhost:9201/logs-acme.api-default/_search' \
     -d '{
       "query": {
         "exists": {
           "field": "error.type"
         }
       },
       "_source": [
         "@timestamp",
         "event.original",
         "error.type",
         "error.message",
         "tags"
       ]
     }' \
     | jq '.hits.hits[] | ._source'
   ```

### Validación mediante KQL en Kibana

**Objetivo:** verificar visualmente los resultados en Discover.

1. Abra Kibana en:

   ```text
   http://localhost:5602
   ```

2. Acceda a **Discover**.

3. Seleccione o cree un data view para:

   ```text
   logs-acme.api-default
   ```

4. Use `@timestamp` como campo temporal.

5. Amplíe el rango temporal para incluir la fecha de los eventos de prueba, por ejemplo desde el 24 de agosto de 2026 hasta el momento actual.

6. Ejecute las siguientes consultas KQL:

   Eventos con errores controlados:

   ```text
   error.type: *
   ```

   Eventos no procesados:

   ```text
   tags: "unparsed_log"
   ```

   Errores de conversión HTTP:

   ```text
   error.type: "status_code_conversion_failure"
   ```

   Eventos del servicio `api-gateway`:

   ```text
   service.name: "api-gateway"
   ```

### Validación mediante ES|QL

**Objetivo:** resumir los resultados operativos del pipeline.

Ejecute desde Kibana Dev Tools o mediante la API de consultas:

```bash
curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
  -H 'Content-Type: application/json' \
  -X POST \
  'http://localhost:9201/_query' \
  -d '{
    "query": "FROM logs-acme.api-default | WHERE event.dataset == \"acme.api\" | STATS eventos = COUNT(*) BY error.type | SORT eventos DESC"
  }' | jq .
```

La consulta debe mostrar una agrupación para eventos sin error y agrupaciones como:

- `date_parse_failure`
- `status_code_conversion_failure`
- `unparsed_log`
- `IllegalStateException`

### Comprobación de estadísticas del pipeline

Ejecute:

```bash
curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
  'http://localhost:9201/_nodes/stats/ingest?filter_path=nodes.*.ingest.pipelines.logs-acme-api-v1' \
  | jq .
```

Verifique que el contador de ingestión del pipeline aumentó después de la simulación y de la indexación. Documente en su informe técnico:

- Número de documentos procesados.
- Número de fallos de procesador.
- Tipo de eventos que generan cada etiqueta de error.
- Evidencia almacenada en `/opt/elastic-labs/work`.

## Solución de problemas

### Problema 1: la indexación falla con un error de mapping para `http.response.status_code`

**Síntoma:**

La respuesta de indexación contiene un error similar a:

```text
mapper_parsing_exception
failed to parse field [http.response.status_code]
```

**Causa:**

Un valor no numérico, por ejemplo `ABC`, intenta indexarse en un campo definido como `long`. Esto ocurre si el procesador `convert` no tiene una ruta `on_failure` que elimine el valor inválido o si el pipeline no se aplicó realmente durante la indexación.

**Corrección:**

1. Verifique que la petición de indexación incluye:

   ```text
   ?pipeline=logs-acme-api-v1
   ```

2. Revise que el bloque `on_failure` del procesador `convert` para `http.response.status_code` contiene un procesador `remove`.

3. Actualice el pipeline si fue modificado:

   ```bash
   curl -sS -u "$ELASTIC_USER:$ELASTIC_PASSWORD" \
     -H 'Content-Type: application/json' \
     -X PUT \
     http://localhost:9201/_ingest/pipeline/logs-acme-api-v1 \
     --data-binary @/opt/elastic-labs/work/logs-acme-api-v1.json \
     | jq .
   ```

4. Pruebe nuevamente el caso inválido con `_simulate` antes de volver a indexarlo.

### Problema 2: todos los documentos aparecen como `unparsed_log`

**Síntoma:**

Las búsquedas muestran documentos con:

```json
{
  "tags": ["unparsed_log"],
  "error": {
    "type": "unparsed_log"
  }
}
```

incluso para líneas que deberían ser accesos HTTP válidos.

**Causa:**

La línea de entrada no coincide exactamente con el patrón `dissect`. `dissect` depende de delimitadores literales y orden fijo. Espacios adicionales, ausencia de `request_id=`, orden distinto de los campos o una URL con espacios provocan que no se extraiga `log.timestamp`.

**Corrección:**

1. Compare la línea recibida con el patrón configurado:

   ```text
   %{log.timestamp} %{log.level} %{service.name} request_id=%{trace.id} %{http.request.method} %{url.original} %{http.response.status_code} %{event.duration_ms}
   ```

2. Ejecute `_simulate?verbose=true` y revise el procesador con etiqueta `dissect_http_access`.

3. Si el formato es estable pero cambió, ajuste el patrón `dissect`.

4. Si el formato tiene variantes reales, cree un patrón alternativo `grok` o una nueva versión del pipeline, por ejemplo `logs-acme-api-v2`, en lugar de modificar sin validación el pipeline usado por producción.

## Limpieza

**Objetivo:** conservar el estado requerido para las prácticas posteriores sin eliminar volúmenes, data streams ni evidencias.

1. Mantenga los siguientes archivos como evidencia:

   ```text
   /opt/elastic-labs/work/logs-acme-api-v1.json
   /opt/elastic-labs/work/simulate-logs-acme-api-v1.json
   /opt/elastic-labs/work/evidencia-pipeline-creado.json
   /opt/elastic-labs/work/evidencia-simulate-verbose.json
   /opt/elastic-labs/work/evidencia-documentos-indexados.json
   ```

2. No ejecute el siguiente comando:

   ```bash
   docker compose down -v
   ```

   Este comando eliminaría volúmenes persistentes necesarios para la continuidad del laboratorio.

3. Si necesita liberar recursos temporalmente, detenga los contenedores sin borrar volúmenes:

   ```bash
   docker compose stop es930-lab kibana930-lab
   ```

4. Para continuar en la siguiente práctica, vuelva a iniciarlos:

   ```bash
   docker compose start es930-lab kibana930-lab
   ```

> No elimine el pipeline `logs-acme-api-v1` ni los documentos `lab04-*`. Estos eventos serán la fuente principal de consultas e investigaciones en la práctica siguiente.

## Resumen

En esta práctica se creó el pipeline `logs-acme-api-v1` para procesar logs de ACME API en Elasticsearch 9.3.0. El pipeline preserva el contenido original, detecta JSON serializado, procesa accesos HTTP estables con `dissect`, analiza excepciones con `grok`, normaliza fechas con `date` y convierte valores numéricos con `convert`.

También se implementaron controles de fallo mediante `on_failure`, etiquetas y campos `error.*`, evitando la pérdida silenciosa de eventos y reduciendo errores de mapping. Los documentos procesados y los documentos con fallos controlados quedaron indexados en `logs-acme.api-default`, listos para investigaciones con Query DSL, KQL y ES|QL.
