# Laboratorio 4: Construir un pipeline para ingestión masiva de logs

- **Capítulo:** 4. Ingest pipelines y procesamiento de logs

- **Duración:** 54 min

- **Entorno:** Elastic Cloud Hosted (cuenta propia), Elasticsearch 9.x, Kibana Dev Tools y una terminal con `curl`

- **Dependencias:** Laboratorio 3 (`weblogs-v1` con el alias `weblogs`, los templates `weblogs-*` y `lab02-weblogs`). Archivo del curso: `weblogs_bulk.ndjson`.

## Objetivo

Procesar logs de acceso en bruto antes de indexarlos: practicar la Bulk API y sus errores, construir un ingest pipeline probado con `_simulate`, enriquecer los documentos con una enrich policy, cargar 5000 líneas con `curl` y reprocesar documentos existentes con `_reindex`. Al terminar, el participante podrá:

- Leer la respuesta de una Bulk API con errores y reenviar solo los documentos fallidos.

- Transformar líneas CSV y líneas de log con los procesadores `csv`, `grok`, `dissect`, `convert`, `date`, `set`, `rename` y `remove`.

- Probar un pipeline con `_simulate` antes de usarlo y manejar fallos con `on_failure`.

- Crear una enrich policy y usarla desde un pipeline.

- Configurar un pipeline por defecto en un índice.

- Dejar cargado `weblogs` con 5006 documentos, que usan los laboratorios 5 y 6.

## Requisitos

- `weblogs-v1` con el alias `weblogs` y 6 documentos. Verificar con `GET weblogs/_count` (debe dar 6).

- Los templates `weblogs-settings`, `weblogs-mappings` y `weblogs-template` del laboratorio 3.

- Archivo `weblogs_bulk.ndjson` descargado en una carpeta local; los comandos `curl` se ejecutan desde esa carpeta.

- Endpoint de Elasticsearch (`<ES_URL>`) de la ficha del deployment.

## Distribución del tiempo

| **Parte** | **Contenido**                                       | **Tiempo** |
|-----------|-----------------------------------------------------|------------|
| Parte A   | Bulk API con errores y corrección                   | 8 min      |
| Parte B   | Pipelines para CSV y logs, probados con `_simulate` | 14 min     |
| Parte C   | Enriquecimiento y pipeline por defecto              | 10 min     |
| Parte D   | Carga masiva con `curl`                             | 10 min     |
| Parte E   | Reprocesar datos con `_reindex` y verificar         | 12 min     |

## Convenciones

- Los bloques con `GET`, `PUT`, `POST` o `DELETE` se ejecutan en Dev Tools; los que empiezan con `curl` se ejecutan en la terminal.

- En Windows se usa `curl.exe`. Los valores entre ángulos (`<ES_URL>`, `<ENCODED_KEY>`) se sustituyen por los propios.

- Cada comando va seguido de lo que hace y del significado de sus parámetros.

## Parte A: Bulk API con errores y corrección (8 min)

Se practica en un índice aparte, `lab04-bulk-test`, para no alterar los conteos de `weblogs`.

### Paso A1. Crear el índice de prueba

``` console
PUT lab04-bulk-test
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "service":    { "type": "keyword" },
      "client_ip":  { "type": "ip" },
      "status":     { "type": "short" }
    }
  }
}
```

**Qué hace:** crea un índice con mapping estricto en cuatro campos. Con un solo primary, las operaciones de una misma petición se procesan en orden.

### Paso A2. Enviar un bulk con errores

``` console
POST lab04-bulk-test/_bulk?refresh=wait_for&filter_path=errors,items.*.status,items.*.error.type
{ "index": { "_id": "1" } }
{ "@timestamp": "2026-10-14T10:00:00Z", "service": "search", "client_ip": "201.10.5.7", "status": 200 }
{ "index": {} }
{ "@timestamp": "2026-10-14T10:00:05Z", "service": "search", "client_ip": "no-es-una-ip", "status": 200 }
{ "index": {} }
{ "@timestamp": "ayer por la tarde", "service": "search", "client_ip": "201.10.5.8", "status": 200 }
{ "index": {} }
{ "@timestamp": "2026-10-14T10:00:09Z", "service": "search", "client_ip": "201.10.5.9", "status": "ok" }
{ "create": { "_id": "1" } }
{ "@timestamp": "2026-10-14T10:00:12Z", "service": "search", "client_ip": "201.10.5.7", "status": 200 }
```

**Qué hace:** envía cinco operaciones en una sola petición. Cada operación son dos líneas (acción y documento) en formato NDJSON.

- `index`: indexa; con `_id` lo fija, sin `_id` lo genera Elasticsearch. Reemplaza si ya existe.

- `create`: indexa solo si el `_id` no existe; si existe, falla.

- `refresh=wait_for`: la respuesta espera al siguiente refresh.

- `filter_path`: limita la respuesta a si hubo errores, el código de cada operación y el tipo de error.

**Resultado esperado:** la petición responde `200 OK` aunque haya fallos: `errors: true`. Cada operación tiene su propio código: `201`, `400`, `400`, `400` y `409`. Los errores son `document_parsing_exception` (en versiones anteriores, `mapper_parsing_exception`) para los tres primeros fallos y `version_conflict_engine_exception` para el último.

Regla: un bulk nunca falla completo por un documento malo. Siempre hay que revisar `errors` y los `items`.

### Paso A3. Corregir y reenviar solo lo fallido

``` console
POST lab04-bulk-test/_bulk?refresh=wait_for&filter_path=errors,items.*.status
{ "index": {} }
{ "@timestamp": "2026-10-14T10:00:05Z", "service": "search", "client_ip": "201.10.5.6", "status": 200 }
{ "index": {} }
{ "@timestamp": "2026-10-14T10:00:07Z", "service": "search", "client_ip": "201.10.5.8", "status": 200 }
{ "index": {} }
{ "@timestamp": "2026-10-14T10:00:09Z", "service": "search", "client_ip": "201.10.5.9", "status": 200 }

GET lab04-bulk-test/_count
```

**Qué hace:** reenvía las tres operaciones corregidas (IP válida, fecha ISO 8601 y `status` numérico). La operación `create` en conflicto se descarta.

**Resultado esperado:** `errors: false` y `count: 4`. Si se reenviara el bulk original completo, la primera operación se duplicaría al no tener `_id`.

## Parte B: pipelines probados con \_simulate (14 min)

### Paso B1. Transformar líneas CSV

``` console
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      { "csv": {
          "field": "csv_line",
          "target_fields": ["incident_id", "opened_at", "service", "severity", "summary", "duration_min"],
          "trim": true } },
      { "convert": { "field": "duration_min", "type": "integer" } },
      { "uppercase": { "field": "severity" } },
      { "date": { "field": "opened_at", "formats": ["ISO8601"], "target_field": "@timestamp" } },
      { "remove": { "field": ["csv_line", "opened_at"] } }
    ]
  },
  "docs": [
    { "_source": { "csv_line": "INC-1001,2026-10-13T14:05:00Z,payments,high,\"Timeout en pasarela, reintentos activados\",45" } },
    { "_source": { "csv_line": "INC-1002,2026-10-13T14:40:00Z,checkout,medium,Latencia elevada,12" } }
  ]
}
```

**Qué hace:** prueba un pipeline definido en la propia petición sobre dos documentos de ejemplo. No indexa nada.

- `POST _ingest/pipeline/_simulate`: simula un pipeline que se envía en el cuerpo, en `pipeline`. Para uno ya guardado se usa `_ingest/pipeline/<id>/_simulate`.

- `docs[]._source`: documentos de prueba.

- `csv`: separa una línea CSV. `field` es el origen, `target_fields` los nombres de salida y `trim` quita espacios. Respeta las comillas dobles: la coma dentro del resumen no divide el campo.

- `convert`: cambia el tipo. Todo lo que sale de `csv` es cadena.

- `uppercase`: pasa un campo a mayúsculas.

- `date`: interpreta `opened_at` y escribe el resultado en `@timestamp`. `ISO8601` es un formato predefinido.

- `remove`: elimina los campos temporales.

**Resultado esperado:** dos documentos con `incident_id`, `service`, `severity` en mayúsculas (`HIGH`, `MEDIUM`), `summary`, `duration_min` numérico (45 y 12) y `@timestamp`. El primer `summary` conserva la coma.

### Paso B2. Crear el pipeline de logs

El archivo de logs del curso tiene líneas como esta:

``` console
2026-10-14T09:15:02.113Z 187.141.22.10 checkout "GET /api/cart HTTP/1.1" 200 1532 42.5 "Mozilla/5.0 (Windows NT 10.0; Win64; x64)" cache hit
```

Campos, en orden: fecha, IP del cliente, servicio, petición (método, ruta y versión HTTP), estado, bytes, tiempo de respuesta en ms, agente de usuario y detalle libre.

``` console
PUT _ingest/pipeline/weblogs-pipeline
{
  "description": "Parsea líneas de log de acceso web",
  "processors": [
    { "grok": {
        "field": "raw_log",
        "if": "ctx.raw_log != null",
        "patterns": [
          "%{TIMESTAMP_ISO8601:ts} %{IP:client_ip} %{WORD:service} \"%{WORD:method} %{NOTSPACE:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:status:int} %{NUMBER:bytes:int} %{NUMBER:response_time_ms:float} \"%{DATA:user_agent}\" %{GREEDYDATA:detail}"
        ] } },
    { "date": { "field": "ts", "if": "ctx.ts != null", "formats": ["ISO8601"], "target_field": "@timestamp" } },
    { "set": {
        "field": "message",
        "if": "ctx.raw_log != null",
        "value": "{{method}} {{url}} {{status}} {{response_time_ms}}ms {{detail}}" } },
    { "remove": { "field": ["ts", "raw_log"], "ignore_missing": true } }
  ],
  "on_failure": [
    { "set": { "field": "tags", "value": ["parse_failure"] } },
    { "set": { "field": "error.message", "value": "{{ _ingest.on_failure_message }}" } }
  ]
}
```

**Qué hace:** guarda un pipeline que convierte la línea cruda (`raw_log`) en campos estructurados.

- `grok`: extrae campos con patrones. La sintaxis es `%{PATRÓN:campo}` o `%{PATRÓN:campo:tipo}`. `TIMESTAMP_ISO8601`, `IP`, `WORD`, `NOTSPACE`, `NUMBER`, `DATA` y `GREEDYDATA` son patrones predefinidos; el sufijo `:int` o `:float` convierte el tipo.

- `if`: condición en lenguaje Painless. Los procesadores solo se ejecutan si el documento trae `raw_log`; así el mismo pipeline sirve para documentos ya estructurados.

- `date`: convierte el texto de `ts` en `@timestamp`.

- `set` con `{{campo}}`: construye `message` combinando campos (plantilla mustache).

- `remove` con `ignore_missing`: elimina los campos temporales sin fallar si no existen.

- `on_failure` (a nivel de pipeline): si cualquier procesador falla, el documento no se rechaza; se marca con `tags` y `error.message`. `_ingest.on_failure_message` contiene la causa del fallo.

### Paso B3. Probar el pipeline guardado

``` console
POST _ingest/pipeline/weblogs-pipeline/_simulate
{
  "docs": [
    { "_source": { "raw_log": "2026-10-14T09:15:02.113Z 187.141.22.10 checkout \"GET /api/cart HTTP/1.1\" 200 1532 42.5 \"Mozilla/5.0 (Windows NT 10.0; Win64; x64)\" cache hit" } },
    { "_source": { "raw_log": "2026-10-14T09:18:12.404Z 148.202.90.3 payments \"POST /api/pay HTTP/2.0\" 500 740 1503.9 \"okhttp/4.12.0\" upstream timeout" } },
    { "_source": { "raw_log": "corrupted entry: 0x5482 binary blob truncated at offset 83060" } }
  ]
}
```

**Qué hace:** ejecuta el pipeline guardado sobre tres líneas: dos válidas y una corrupta.

**Resultado esperado:**

- Documentos 1 y 2: campos `client_ip`, `service`, `method`, `url`, `http_version`, `status` y `bytes` (numéricos), `response_time_ms`, `user_agent`, `detail`, `message` (por ejemplo, `GET /api/cart 200 42.5ms cache hit`) y `@timestamp`. `raw_log` y `ts` ya no aparecen.

- Documento 3: conserva `raw_log`, con `tags: ["parse_failure"]` y `error.message` que indica que el patrón de grok no coincide.

Añadiendo `?verbose=true` a la ruta, la respuesta muestra el resultado de cada procesador. Es la forma más rápida de localizar en cuál se rompe un pipeline.

### Paso B4. Alternativa con dissect

``` console
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      { "dissect": {
          "field": "raw_log",
          "pattern": "%{ts} %{client_ip} %{service} \"%{method} %{url} HTTP/%{http_version}\" %{status} %{bytes} %{response_time_ms} \"%{user_agent}\" %{detail}" } },
      { "convert": { "field": "status", "type": "integer" } }
    ]
  },
  "docs": [
    { "_source": { "raw_log": "2026-10-14T09:15:02.113Z 187.141.22.10 checkout \"GET /api/cart HTTP/1.1\" 200 1532 42.5 \"Mozilla/5.0 (Windows NT 10.0; Win64; x64)\" cache hit" } }
  ]
}
```

**Qué hace:** extrae los mismos campos con `dissect`.

- `dissect` divide por delimitadores literales (espacios, comillas), sin expresiones regulares. Es más rápido y más simple que `grok`, pero exige una estructura fija.

- Todo lo que extrae es texto; por eso hace falta `convert` para cada campo numérico.

- `grok` es preferible cuando el formato varía o cuando se necesita validar el tipo de cada parte (por ejemplo, que un campo sea una IP).

## Parte C: enriquecimiento y pipeline por defecto (10 min)

### Paso C1. Crear el índice de referencia

``` console
PUT services-catalog
{
  "mappings": {
    "properties": {
      "service": { "type": "keyword" },
      "team":    { "type": "keyword" },
      "tier":    { "type": "keyword" }
    }
  }
}

POST services-catalog/_bulk?refresh=wait_for
{ "index": { "_id": "checkout" } }
{ "service": "checkout", "team": "ventas",   "tier": "critical" }
{ "index": { "_id": "catalog" } }
{ "service": "catalog",  "team": "catalogo", "tier": "standard" }
{ "index": { "_id": "payments" } }
{ "service": "payments", "team": "pagos",    "tier": "critical" }
{ "index": { "_id": "search" } }
{ "service": "search",   "team": "busqueda", "tier": "standard" }
```

**Qué hace:** crea una tabla de consulta con el equipo responsable y la criticidad de cada servicio.

### Paso C2. Crear y ejecutar la enrich policy

``` console
PUT _enrich/policy/services-policy
{
  "match": {
    "indices": "services-catalog",
    "match_field": "service",
    "enrich_fields": ["team", "tier"]
  }
}

POST _enrich/policy/services-policy/_execute
```

**Qué hace:** define cómo se enriquecen los documentos y construye el índice interno de enriquecimiento.

- `match`: tipo de policy. Busca coincidencia exacta; otros tipos son `geo_match` y `range`.

- `indices`: índice de origen con los datos de referencia.

- `match_field`: campo del índice de origen que se compara con el campo del documento entrante.

- `enrich_fields`: campos que se copian al documento.

- `_execute`: crea una copia optimizada de los datos. Si cambia `services-catalog`, hay que volver a ejecutar la policy.

**Resultado esperado:** `status.phase: "COMPLETE"`.

### Paso C3. Ampliar el mapping con los campos nuevos

``` console
PUT weblogs-v1/_mapping
{
  "properties": {
    "http_version": { "type": "keyword" },
    "user_agent":   { "type": "keyword", "ignore_above": 256 },
    "detail":       { "type": "keyword" },
    "team":         { "type": "keyword" },
    "tier":         { "type": "keyword" },
    "tags":         { "type": "keyword" },
    "raw_log":      { "type": "text", "index": false },
    "error": { "properties": { "message": { "type": "text" } } }
  }
}
```

**Qué hace:** agrega campos al mapping del índice existente. Se pueden añadir campos nuevos; no se puede cambiar el tipo de uno existente (para eso se reindexa).

Para que los índices futuros hereden los campos, repetir el mismo bloque `properties` dentro del component template (se reemplaza completo):

``` console
PUT _component_template/weblogs-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp":       { "type": "date" },
        "service":          { "type": "keyword" },
        "client_ip":        { "type": "ip" },
        "method":           { "type": "keyword" },
        "url": { "type": "keyword", "ignore_above": 512,
                 "fields": { "path": { "type": "text", "analyzer": "url_path_analyzer" } } },
        "status":           { "type": "short" },
        "bytes":            { "type": "integer", "index": false },
        "response_time_ms": { "type": "float" },
        "message": { "type": "text",
                     "fields": { "keyword": { "type": "keyword", "ignore_above": 512 } } },
        "http_version":     { "type": "keyword" },
        "user_agent":       { "type": "keyword", "ignore_above": 256 },
        "detail":           { "type": "keyword" },
        "team":             { "type": "keyword" },
        "tier":             { "type": "keyword" },
        "tags":             { "type": "keyword" },
        "raw_log":          { "type": "text", "index": false },
        "error": { "properties": { "message": { "type": "text" } } }
      }
    }
  }
}
```

### Paso C4. Añadir el enriquecimiento al pipeline

``` console
PUT _ingest/pipeline/weblogs-pipeline
{
  "description": "Parsea y enriquece logs de acceso web",
  "processors": [
    { "grok": {
        "field": "raw_log",
        "if": "ctx.raw_log != null",
        "patterns": [
          "%{TIMESTAMP_ISO8601:ts} %{IP:client_ip} %{WORD:service} \"%{WORD:method} %{NOTSPACE:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:status:int} %{NUMBER:bytes:int} %{NUMBER:response_time_ms:float} \"%{DATA:user_agent}\" %{GREEDYDATA:detail}"
        ] } },
    { "date": { "field": "ts", "if": "ctx.ts != null", "formats": ["ISO8601"], "target_field": "@timestamp" } },
    { "set": {
        "field": "message",
        "if": "ctx.raw_log != null",
        "value": "{{method}} {{url}} {{status}} {{response_time_ms}}ms {{detail}}" } },
    { "enrich": {
        "policy_name": "services-policy",
        "field": "service",
        "target_field": "svc",
        "ignore_missing": true } },
    { "rename": { "field": "svc.team", "target_field": "team", "ignore_missing": true } },
    { "rename": { "field": "svc.tier", "target_field": "tier", "ignore_missing": true } },
    { "remove": { "field": ["ts", "raw_log", "svc"], "ignore_missing": true } }
  ],
  "on_failure": [
    { "set": { "field": "tags", "value": ["parse_failure"] } },
    { "set": { "field": "error.message", "value": "{{ _ingest.on_failure_message }}" } }
  ]
}
```

**Qué hace:** reemplaza el pipeline anterior (el mismo `PUT` con el mismo nombre lo actualiza) agregando el enriquecimiento.

- `enrich.policy_name`: policy que se usa.

- `enrich.field`: campo del documento que se compara con el `match_field` de la policy.

- `enrich.target_field`: campo donde se escribe el resultado (`svc.team`, `svc.tier`, y el campo de coincidencia).

- `rename`: mueve `team` y `tier` a la raíz del documento. `remove` elimina el campo temporal `svc`.

Repetir la simulación del paso B3. **Resultado esperado:** los dos documentos válidos incluyen ahora `team` y `tier` (`ventas`/`critical` para checkout y `pagos`/`critical` para payments); el documento corrupto mantiene `tags: ["parse_failure"]`.

### Paso C5. Fijar el pipeline por defecto del índice

``` console
PUT weblogs-v1/_settings
{
  "index.default_pipeline": "weblogs-pipeline"
}
```

**Qué hace:** todo documento que se indexe en `weblogs-v1` (directamente o por el alias) pasa por `weblogs-pipeline`, sin que el cliente tenga que indicarlo. Para un pipeline puntual se usa el parámetro `?pipeline=<nombre>`, que tiene prioridad sobre el pipeline por defecto.

## Parte D: carga masiva con curl (10 min)

### Paso D1. Crear una API Key de ingesta

``` console
POST _security/api_key
{
  "name": "lab04-ingesta",
  "expiration": "7d",
  "role_descriptors": {
    "weblogs_writer": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["weblogs", "weblogs-v*"],
          "privileges": ["create_index", "write", "view_index_metadata"]
        }
      ]
    }
  }
}
```

**Qué hace:** crea una credencial que solo puede escribir en `weblogs` y sus índices. Copiar el valor `encoded` de la respuesta (como en el laboratorio 2). El privilegio `write` cubre las operaciones `index`, `create`, `update`, `delete` y `bulk`.

### Paso D2. Cargar el archivo

``` console
curl.exe -s -X POST "<ES_URL>/weblogs/_bulk?filter_path=took,errors" -H "Authorization: ApiKey <ENCODED_KEY>" -H "Content-Type: application/x-ndjson" --data-binary "@weblogs_bulk.ndjson"
```

**Qué hace:** envía las 5000 líneas del archivo a través del alias `weblogs`. Cada documento tiene solo el campo `raw_log` y el pipeline por defecto lo procesa.

- `-s`: modo silencioso (sin barra de progreso).

- `-X POST`: método HTTP.

- `Content-Type: application/x-ndjson`: formato requerido por la Bulk API.

- `--data-binary "@archivo"`: envía el archivo tal cual. Con `-d` se eliminarían los saltos de línea y el bulk fallaría.

- `filter_path=took,errors`: solo se muestra el tiempo y si hubo errores.

**Resultado esperado:** `{"took":...,"errors":false}`. Las 30 líneas corruptas no producen error: el `on_failure` del pipeline las acepta marcadas con `parse_failure`.

Si la respuesta es `401` o `403`: revisar que `<ENCODED_KEY>` esté completo y que la clave no haya expirado. Como comprobación, repetir la carga con `-u elastic:<contraseña>` en lugar del encabezado; si así funciona, el problema son los privilegios de la clave.

### Paso D3. Verificar la carga

``` console
GET weblogs/_count

GET weblogs/_count
{
  "query": { "term": { "tags": "parse_failure" } }
}

GET weblogs/_search
{
  "size": 2,
  "query": { "term": { "tags": "parse_failure" } },
  "_source": ["raw_log", "error.message"]
}
```

**Resultado esperado:** 5006 documentos en total (6 + 5000) y 30 con `parse_failure`. Los 30 conservan `raw_log` para poder analizarlos y no tienen `@timestamp`, por lo que no aparecen en filtros por fecha ni en histogramas.

## Parte E: reprocesar con \_reindex y verificar (12 min)

### Paso E1. Ver los documentos sin enriquecer

``` console
GET weblogs/_count
{
  "query": { "exists": { "field": "team" } }
}
```

**Qué hace:** cuenta los documentos que ya tienen el campo `team`.

**Resultado esperado:** 4970. Faltan los 6 documentos del laboratorio 2, que se indexaron antes de que existiera el pipeline.

### Paso E2. Reprocesarlos con \_reindex y un pipeline

``` console
POST _reindex?refresh=true
{
  "source": { "index": "lab02-weblogs" },
  "dest": {
    "index": "weblogs-v1",
    "pipeline": "weblogs-pipeline"
  }
}
```

**Qué hace:** vuelve a copiar los 6 documentos originales al índice destino, pasándolos por el pipeline.

- `dest.pipeline`: pipeline que se aplica a cada documento durante la copia.

- Los `_id` se conservan; por eso los 6 documentos reemplazan a los ya existentes en lugar de duplicarse.

- Otras opciones útiles: `source.query` para copiar solo un subconjunto, y `source._source` para elegir campos.

**Resultado esperado:** `total: 6`, `created: 0`, `updated: 6`.

Alternativa en el mismo índice: `POST weblogs-v1/_update_by_query?pipeline=weblogs-pipeline&conflicts=proceed`. Reprocesa todos los documentos de un índice sin moverlos; se usa cuando basta con corregir los datos en su lugar.

### Paso E3. Verificar el resultado final

``` console
GET weblogs/_doc/1

GET weblogs/_search
{
  "size": 0,
  "track_total_hits": true,
  "aggs": {
    "por_servicio": { "terms": { "field": "service" } },
    "por_equipo":   { "terms": { "field": "team" } }
  }
}
```

**Resultado esperado:**

- `weblogs/_doc/1` incluye `team: "ventas"` y `tier: "critical"`.

- `hits.total.value`: 5006.

- Por servicio: catalog 1782, checkout 1499, search 930 y payments 765 (4976 en total; los 30 documentos corruptos no tienen `service`).

- Por equipo: los mismos conteos, con los nombres `catalogo`, `ventas`, `busqueda` y `pagos`.

Verificación adicional con `GET weblogs/_count` y `exists` sobre `team`: debe dar 4976.

## Limpieza y estado final

``` console
DELETE lab04-bulk-test

DELETE _security/api_key
{
  "name": "lab04-ingesta"
}
```

**Qué hace:** elimina el índice de prueba e invalida la API Key de ingesta.

| **Elemento**                                           | **Se conserva** | **Lo reutiliza**                        |
|--------------------------------------------------------|-----------------|-----------------------------------------|
| `weblogs-v1` (alias `weblogs`) con 5006 documentos     | Sí              | Laboratorios 5, 6 y 9, y proyecto final |
| Pipeline `weblogs-pipeline` y `index.default_pipeline` | Sí              | Laboratorios 6 y 9                      |
| `services-catalog` y `services-policy`                 | Sí              | Laboratorio 5 (`team`, `tier`)          |
| Component template `weblogs-mappings` ampliado         | Sí              | Laboratorios 6 y 9                      |
| `lab04-bulk-test` y API Key `lab04-ingesta`            | No              | Solo para practicar                     |

## Preguntas de discusión

- ¿Por qué un bulk con errores devuelve `200 OK`? ¿Qué debe revisar siempre el cliente?

- ¿Qué pasa con los documentos malos si el pipeline no tiene `on_failure`? ¿Qué decisión de negocio implica aceptarlos con una marca?

- ¿Cuándo elegirías `dissect` y cuándo `grok`?

- ¿Qué ocurre con los documentos ya indexados si se modifica el pipeline? ¿Cómo se reprocesan?

- ¿Qué pasaría con la enrich policy si cambia `services-catalog`?

- ¿Qué ventajas tiene hacer el parseo en un ingest pipeline en lugar de hacerlo en el cliente o en Logstash?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 4: Pipeline para ingestión masiva de logs](imagenes/lab04_resultado_final.png)
