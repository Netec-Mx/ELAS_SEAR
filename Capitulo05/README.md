# Administración de Elasticsearch para gestión y análisis de logs

## Práctica 5a. Consumir logs de servidor web y de aplicación con Filebeat, ingest pipelines y data streams

### Objetivo

Al finalizar la práctica, serás capaz de:

- Diseñar ingest pipelines que conviertan líneas de log en documentos estructurados y normalizados con ECS, usando `grok`, `dissect`, `date` (con zona horaria), `geoip`, `user_agent`, `rename`, `remove`, `convert`, `script` y `on_failure`.
- Simular y depurar pipelines antes de recibir datos reales.
- Crear index templates para data streams con el esquema `logs-<dataset>-<namespace>`.
- Instalar y configurar Filebeat en Windows para enviar logs de archivos, incluidos stack traces multilínea.
- Monitorear la ejecución de los pipelines.

### Objetivo visual

![Flujo de ingesta con Filebeat e ingest pipelines](img/cap05a.png)

### Duración aproximada

- 70 minutos.

### Prerrequisitos

- Prácticas 2 a 4 completadas.
- Archivos de log en `C:\labs\logs\web\access.log` (3,421 líneas de Nginx) y `C:\labs\logs\app\app.log` (2,458 eventos de los microservicios Java; algunos incluyen stack traces de varias líneas).
- Archivo de configuración `C:\labs\config\filebeat.yml`.

---

### Instrucciones

**Escenario:** la tienda en línea tiene dos servidores web Nginx (`srv-web01` y `srv-web02`) que reparten las peticiones entre dos servidores de aplicación (`srv-app01` y `srv-app02`). Cada petición lleva un `trace_id` que aparece en el log del servidor web y en el de la aplicación. Debes consumir ambos logs del día **20 de octubre de 2026** para que puedan investigarse en la Práctica 7.

---

### Tarea 1. Conocer los formatos de log

Antes de escribir un pipeline hay que entender el formato exacto de cada línea: qué campos trae, qué separadores usa y cómo expresa la fecha y la hora.

#### Paso 1. Examina las primeras líneas de cada archivo

En PowerShell:

```powershell
Get-Content C:\labs\logs\web\access.log -TotalCount 3
Get-Content C:\labs\logs\app\app.log -TotalCount 3
Select-String -Path C:\labs\logs\app\app.log -Pattern "ERROR" -Context 0,6 | Select-Object -First 1
```

**Qué hace y por qué:** identifica la estructura de cada formato.

Log de acceso de Nginx (formato personalizado):

```text
srv-web02 177.228.46.19 - - [20/Oct/2026:13:38:51 -0600] "POST /api/checkout HTTP/1.1" 503 187 "https://tienda.ejemplo.mx/carrito" "Mozilla/5.0 (Macintosh; ...)" rt=3.041 upstream=srv-app02:8080 trace_id=664595a620d2a6692d4bc1330b677eae
```

| Fragmento | Significado | Campo ECS destino |
|---|---|---|
| `srv-web02` | servidor web que atendió | `host.name` |
| `177.228.46.19` | IP del cliente | `source.ip` |
| `-` | usuario autenticado (vacío) | `user.name` |
| `[20/Oct/2026:13:38:51 -0600]` | fecha con zona horaria | `@timestamp` |
| `"POST /api/checkout HTTP/1.1"` | método, URL y versión | `http.request.method`, `url.original`, `http.version` |
| `503 187` | código de respuesta y bytes | `http.response.status_code`, `http.response.body.bytes` |
| `"https://..."` y `"Mozilla/5.0 ..."` | referer y navegador | `http.request.referrer`, `user_agent.original` |
| `rt=3.041` | tiempo de respuesta en segundos | `event.duration` (en nanosegundos) |
| `upstream=srv-app02:8080` | servidor de aplicación | `nginx.access.upstream` |
| `trace_id=...` | identificador de traza | `trace.id` |

Log de aplicación:

```text
2026-10-20T13:38:50.133-06:00 ERROR [checkout-service] [srv-app02] [trace=664595a6...] [user=vtorres] Fallo al confirmar pedido id=PED-268044
org.springframework.jdbc.CannotGetJdbcConnectionException: Failed to obtain JDBC Connection
	at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:84)
	...
```

El formato es regular, con separadores fijos, así que conviene usar `dissect`, que es más rápido que `grok`. Las líneas que **no** empiezan con fecha pertenecen al evento anterior (stack trace).

---

### Tarea 2. Crear y simular el pipeline del log de acceso

#### Paso 2. Crea el pipeline `weblab-access`

En **Dev Tools**:

```http
PUT _ingest/pipeline/weblab-access
{
  "description": "Logs de acceso Nginx de la tienda -> ECS",
  "processors": [
    { "set":   { "field": "event.original", "copy_from": "message" } },
    { "grok": {
        "field": "message",
        "patterns": [
          """%{HOSTNAME:host.name} %{IP:source.ip} - %{NOTSPACE:user.name} \[%{HTTPDATE:nginx.access.time}\] "%{WORD:http.request.method} %{NOTSPACE:url.original} HTTP/%{NUMBER:http.version}" %{INT:http.response.status_code:long} %{INT:http.response.body.bytes:long} "%{DATA:http.request.referrer}" "%{DATA:user_agent.original}" rt=%{NUMBER:nginx.access.request_time:float} upstream=%{NOTSPACE:nginx.access.upstream} trace_id=%{NOTSPACE:trace.id}"""
        ]
    } },
    { "grok": {
        "field": "url.original",
        "patterns": [ """%{URIPATH:url.path}(?:\?%{GREEDYDATA:url.query})?""" ],
        "ignore_failure": true
    } },
    { "date": {
        "field": "nginx.access.time",
        "formats": ["dd/MMM/yyyy:HH:mm:ss Z"],
        "locale": "en",
        "target_field": "@timestamp"
    } },
    { "remove": { "field": "nginx.access.time" } },
    { "remove": { "field": "user.name", "if": "ctx.user?.name == '-'" } },
    { "remove": { "field": "trace.id",  "if": "ctx.trace?.id == '-'" } },
    { "dissect": {
        "field": "nginx.access.upstream",
        "pattern": "%{nginx.access.upstream_host}:%{nginx.access.upstream_port}",
        "if": "ctx.nginx?.access?.upstream != '-'"
    } },
    { "convert": { "field": "nginx.access.upstream_port", "type": "integer", "ignore_missing": true } },
    { "script": {
        "description": "rt (segundos) -> event.duration (nanosegundos, ECS)",
        "source": "if (ctx.nginx?.access?.request_time != null) { if (ctx.event == null) { ctx.event = [:]; } ctx.event.duration = (long) (((Number) ctx.nginx.access.request_time).doubleValue() * 1000000000.0); }"
    } },
    { "set": { "field": "event.outcome", "value": "failure",
               "if": "ctx.http?.response?.status_code != null && ctx.http.response.status_code >= 400" } },
    { "set": { "field": "event.outcome", "value": "success", "override": false } },
    { "set": { "field": "event.kind", "value": "event" } },
    { "set": { "field": "event.category", "value": ["web"] } },
    { "set": { "field": "event.dataset", "value": "weblab.access" } },
    { "user_agent": { "field": "user_agent.original", "ignore_missing": true } },
    { "geoip": { "field": "source.ip", "target_field": "source.geo", "ignore_missing": true } }
  ],
  "on_failure": [
    { "set": { "field": "event.kind", "value": "pipeline_error" } },
    { "set": { "field": "error.message", "value": "{{{ _ingest.on_failure_message }}}" } },
    { "set": { "field": "error.type", "value": "{{{ _ingest.on_failure_processor_type }}}" } }
  ]
}
```

**Qué hace y por qué:** un **ingest pipeline** es una secuencia de procesadores que transforma cada documento en el nodo de ingesta, **antes** de indexarlo:

| Procesador | Función en este pipeline |
|---|---|
| `set` (`copy_from`) | Conserva la línea original en `event.original`, como recomienda ECS, para auditoría y reprocesamiento. |
| `grok` | Extrae campos con expresiones regulares nombradas (`%{PATRÓN:campo:tipo}`). El sufijo `:long` o `:float` convierte el tipo. Las comillas triples `"""` de Console evitan escapar comillas y diagonales invertidas. |
| segundo `grok` | Separa la ruta (`url.path`) de los parámetros (`url.query`). |
| `date` | Convierte la fecha de Nginx a `@timestamp`. El formato incluye la zona (`Z` = `-0600`), así que la hora se interpreta correctamente y se almacena en UTC. `locale: en` es necesario para reconocer `Oct`. |
| `remove` con `if` | Elimina campos con el valor `-` (sin dato), para no contaminar agregaciones ni consultas `exists`. |
| `dissect` | Divide `srv-app02:8080` en host y puerto con un patrón simple, sin regex. |
| `convert` | Cambia el puerto de texto a entero. |
| `script` (Painless) | Calcula `event.duration` en **nanosegundos**, la unidad que define ECS. |
| `set` con `override: false` | Asigna `success` solo si el procesador anterior no asignó `failure`. |
| `user_agent` | Descompone el navegador en `user_agent.name`, `user_agent.os.name`, `user_agent.device.name`, etc. |
| `geoip` | Agrega país, ciudad y coordenadas (`source.geo.*`) a partir de la IP. |
| `on_failure` | Si un procesador falla, el documento **no se pierde**: se indexa con `event.kind: pipeline_error` y el motivo en `error.message`. |

> **Importante:** `geoip` usa las bases GeoLite2, que Elasticsearch descarga automáticamente de Internet. En una red sin acceso, el documento recibe la etiqueta `_geoip_database_unavailable_GeoLite2-City.mmdb`. En ambientes on premise aislados se desactiva la descarga con `ingest.geoip.downloader.enabled: false` y se copian las bases `.mmdb` a `config\ingest-geoip`.

#### Paso 3. Simula el pipeline con líneas reales

```http
POST _ingest/pipeline/weblab-access/_simulate
{
  "docs": [
    { "_source": { "message": """srv-web02 177.228.46.19 - - [20/Oct/2026:13:38:51 -0600] "POST /api/checkout HTTP/1.1" 503 187 "https://tienda.ejemplo.mx/carrito" "Mozilla/5.0 (Macintosh; Intel Mac OS X 14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.6 Safari/605.1.15" rt=3.041 upstream=srv-app02:8080 trace_id=664595a620d2a6692d4bc1330b677eae""" } },
    { "_source": { "message": """srv-web01 200.68.52.205 - - [20/Oct/2026:08:03:37 -0600] "GET /productos?categoria=hogar HTTP/1.1" 200 119666 "https://tienda.ejemplo.mx/productos" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36" rt=0.006 upstream=- trace_id=-""" } },
    { "_source": { "message": "esta linea no tiene el formato esperado" } }
  ]
}
```

**Qué hace y por qué:** `_simulate` ejecuta el pipeline sobre documentos de prueba **sin indexarlos**. Verifica lo siguiente:

- Documento 1: `@timestamp` = `2026-10-20T19:38:51.000Z` (13:38:51 en la Ciudad de México equivale a 19:38:51 UTC), `http.response.status_code` = `503` como número, `event.duration` = `3041000000`, `event.outcome` = `failure`, `nginx.access.upstream_host` = `srv-app02`, `url.path` = `/api/checkout` y `user_agent.name` = `Safari`.
- Documento 2: no tiene `trace.id` ni `user.name`, porque se eliminaron los `-`; `url.query` = `categoria=hogar`; no tiene `nginx.access.upstream_host`.
- Documento 3: `event.kind` = `pipeline_error` y `error.message` explica que grok no encontró coincidencias. Gracias a `on_failure`, la línea mal formada no se pierde.

#### Paso 4. Depura paso a paso con `verbose`

```http
POST _ingest/pipeline/weblab-access/_simulate?verbose=true
{
  "docs": [
    { "_source": { "message": """srv-web02 177.228.46.19 - - [20/Oct/2026:13:38:51 -0600] "POST /api/checkout HTTP/1.1" 503 187 "-" "curl/8.4.0" rt=3.041 upstream=srv-app02:8080 trace_id=abc123""" } }
  ]
}
```

**Qué hace y por qué:** con `verbose=true`, la respuesta muestra el documento **después de cada procesador** (`processor_results`), con su `status` (`success`, `skipped` por una condición `if`, o `error`). Es la herramienta principal para encontrar qué procesador falla o produce un valor inesperado.

---

### Tarea 3. Crear y simular el pipeline del log de aplicación

#### Paso 5. Crea el pipeline `weblab-app`

```http
PUT _ingest/pipeline/weblab-app
{
  "description": "Logs de aplicacion Java de la tienda -> ECS",
  "processors": [
    { "rename":  { "field": "message", "target_field": "event.original" } },
    { "dissect": {
        "field": "event.original",
        "pattern": "%{app.ts} %{log.level} [%{service.name}] [%{host.name}] [trace=%{trace.id}] [user=%{user.name}] %{message}"
    } },
    { "date": {
        "field": "app.ts",
        "formats": ["ISO8601"],
        "target_field": "@timestamp"
    } },
    { "remove": { "field": "app.ts" } },
    { "remove": { "field": "trace.id",  "if": "ctx.trace?.id == '-'" } },
    { "remove": { "field": "user.name", "if": "ctx.user?.name == '-'" } },
    { "grok": { "field": "message", "patterns": ["duracion_ms=%{INT:app.duration_ms:long}"], "ignore_failure": true } },
    { "grok": { "field": "message", "patterns": ["status=%{INT:http.response.status_code:long}"], "ignore_failure": true } },
    { "grok": { "field": "message", "patterns": ["ip=%{IP:source.ip}"], "ignore_failure": true } },
    { "grok": { "field": "message", "patterns": ["version=%{NOTSPACE:service.version}"], "ignore_failure": true } },
    { "set": { "field": "error.stack_trace", "copy_from": "message", "if": "ctx.message != null && ctx.message.contains('\n')" } },
    { "set": { "field": "event.kind", "value": "event" } },
    { "set": { "field": "event.dataset", "value": "weblab.app" } },
    { "geoip": { "field": "source.ip", "target_field": "source.geo", "ignore_missing": true } }
  ],
  "on_failure": [
    { "set": { "field": "event.kind", "value": "pipeline_error" } },
    { "set": { "field": "error.message", "value": "{{{ _ingest.on_failure_message }}}" } }
  ]
}
```

**Qué hace y por qué:**

- `rename` mueve la línea completa a `event.original`, y `dissect` la divide usando los separadores fijos (espacios, corchetes y los prefijos `trace=` y `user=`). La **última** llave (`%{message}`) captura el resto del texto, incluido el stack trace completo.
- `date` con `ISO8601` interpreta la fecha **y** su zona (`-06:00`) incluidas en el texto.
- Los `grok` con `ignore_failure` extraen datos opcionales del mensaje (duración, código, IP o versión) solo cuando existen.
- El `set` condicional copia el mensaje a `error.stack_trace` cuando tiene varias líneas.

**¿`dissect` o `grok`?** `dissect` divide por delimitadores fijos y es más rápido y fácil de leer; `grok` usa expresiones regulares y tolera formatos variables. Usa `dissect` cuando el formato es estable y `grok` cuando hay partes opcionales o variables.

#### Paso 6. Simula con un evento multilínea

```http
POST _ingest/pipeline/weblab-app/_simulate
{
  "docs": [
    { "_source": { "message": """2026-10-20T13:38:50.133-06:00 ERROR [checkout-service] [srv-app02] [trace=664595a620d2a6692d4bc1330b677eae] [user=vtorres] Fallo al confirmar pedido id=PED-268044
org.springframework.jdbc.CannotGetJdbcConnectionException: Failed to obtain JDBC Connection
	at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:84)
Caused by: java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 3000ms.""" } },
    { "_source": { "message": "2026-10-20T13:45:03.291-06:00 WARN [auth-service] [srv-app01] [trace=4a25b8960f7e87d6eab3494fbffa5359] [user=admin1] Login fallido usuario=admin1 ip=89.160.20.156 motivo=credenciales_invalidas" } },
    { "_source": { "message": "2026-10-20T07:58:00.000-06:00 INFO [catalog-service] [srv-app01] [trace=-] [user=-] Iniciando catalog-service version=1.9.3" } }
  ]
}
```

**Qué hace y por qué:** comprueba los tres casos típicos. El primero tiene `log.level: ERROR`, `trace.id` y `error.stack_trace`. El segundo tiene `source.ip: 89.160.20.156` con su geolocalización. El tercero tiene `service.version: 1.9.3` y no tiene `trace.id` ni `user.name`.

---

### Tarea 4. Preparar los data streams

Los logs se almacenan en **data streams**: un nombre lógico (por ejemplo, `logs-weblab.access-default`) respaldado por índices ocultos (*backing indices*, `.ds-logs-weblab.access-default-2026.10.21-000001`) que rotan con **rollover**. Solo se agregan documentos (`create`) y todos deben tener `@timestamp`.

#### Paso 7. Crea los index templates de los data streams

```http
PUT _index_template/logs-weblab.access
{
  "index_patterns": ["logs-weblab.access-*"],
  "data_stream": {},
  "priority": 200,
  "composed_of": ["logs@mappings", "logs@settings", "ecs@mappings"],
  "template": {
    "settings": {
      "index.default_pipeline": "weblab-access",
      "index.number_of_replicas": 0
    }
  },
  "_meta": { "descripcion": "Logs de acceso Nginx de la tienda (laboratorio)" }
}

PUT _index_template/logs-weblab.app
{
  "index_patterns": ["logs-weblab.app-*"],
  "data_stream": {},
  "priority": 200,
  "composed_of": ["logs@mappings", "logs@settings", "ecs@mappings"],
  "template": {
    "settings": {
      "index.default_pipeline": "weblab-app",
      "index.number_of_replicas": 0
    }
  },
  "_meta": { "descripcion": "Logs de aplicacion de la tienda (laboratorio)" }
}

POST _index_template/_simulate_index/logs-weblab.access-default
```

**Qué hace y por qué:**

- `"data_stream": {}` indica que los nombres que coincidan crean un **data stream**, no un índice normal.
- El esquema de nombres `logs-<dataset>-<namespace>` separa el **tipo** (`logs`), la **fuente** (`weblab.access`) y el **espacio** (`default`, `produccion`, `final`, etc.). Así se aplican permisos, retención y templates por fuente o por ambiente.
- `logs@mappings`, `logs@settings` y `ecs@mappings` son component templates **incluidos** en Elasticsearch 8.x. `ecs@mappings` asigna el tipo correcto a los campos ECS automáticamente: `*.ip` como `ip`, `source.geo.location` como `geo_point`, cadenas como `keyword`, etc.
- `index.default_pipeline` aplica el pipeline a **todo** documento que llegue, sin importar quién lo envíe (Filebeat, Logstash o una aplicación). Sobrescribe el pipeline por defecto de `logs@settings`.
- `priority: 200` gana sobre el template integrado `logs` (prioridad 100), que también coincide con `logs-*-*`.
- `number_of_replicas: 0` mantiene el estado `green` en un clúster de un nodo.

En la simulación verifica `index.default_pipeline: weblab-access`, `index.lifecycle.name: logs` (la política ILM integrada) y `source.ip` de tipo `ip` en `mappings.dynamic_templates`.

---

### Tarea 5. Instalar y ejecutar Filebeat en Windows

#### Paso 8. Descarga e instala Filebeat

En PowerShell:

```powershell
$VER = "8.19.21"
cd C:\elastic\descargas
curl.exe -L -O "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-$VER-windows-x86_64.zip"
tar -xf ".\filebeat-$VER-windows-x86_64.zip" -C C:\elastic
Get-ChildItem C:\elastic -Directory -Filter "filebeat-$VER*" | Rename-Item -NewName "filebeat"
Copy-Item C:\elastic\filebeat\filebeat.yml C:\elastic\filebeat\filebeat.yml.original
Copy-Item C:\labs\config\filebeat.yml C:\elastic\filebeat\filebeat.yml -Force
```

**Qué hace y por qué:** Filebeat es un recolector ligero escrito en Go: lee archivos, recuerda hasta dónde leyó (*registry*) y envía los eventos en lotes con la Bulk API. Se respalda el `filebeat.yml` original y se usa el del curso.

#### Paso 9. Revisa la configuración

```powershell
notepad C:\elastic\filebeat\filebeat.yml
```

**Qué hace y por qué:** observa los elementos clave:

- Dos entradas `filestream`, cada una con un `id` único, la ruta del archivo y la opción `index`, que define el **data stream** destino.
- En la entrada de aplicación, el parser `multiline` con `pattern: '^\d{4}-\d{2}-\d{2}T'`, `negate: true` y `match: after`: toda línea que **no** empieza con fecha se agrega a la anterior. Así cada stack trace llega como **un solo** evento.
- En `output.elasticsearch`, la URL HTTPS, el usuario `elastic`, la contraseña tomada del keystore (`${ES_PWD}`) y la CA del clúster.
- `setup.template.enabled: false` y `setup.ilm.enabled: false`: los templates y pipelines los administras tú (Paso 7), no Filebeat.

#### Paso 10. Guarda la contraseña en el keystore y prueba la configuración

```powershell
cd C:\elastic\filebeat
.\filebeat.exe keystore create
.\filebeat.exe keystore add ES_PWD
.\filebeat.exe test config -c filebeat.yml
.\filebeat.exe test output -c filebeat.yml
```

Cuando `keystore add` lo pida, escribe la contraseña de `elastic`.

**Qué hace y por qué:** el **keystore** guarda secretos cifrados fuera del archivo YAML, así que la contraseña no queda en texto plano ni se filtra si alguien comparte la configuración. `test config` valida la sintaxis y `test output` comprueba la conexión, el TLS y la autenticación. Debe terminar con `talk to server... OK`.

#### Paso 11. Ejecuta Filebeat

```powershell
.\filebeat.exe -e -c filebeat.yml
```

Espera a ver en la consola mensajes `Harvester started for paths: [C:\labs\logs\web\access.log]` y, cada 30 segundos, métricas con `"events":{"acked":...}`. **Deja esta ventana abierta**: Filebeat seguirá leyendo los archivos y se usará en las Prácticas 7, 8 y 10.

**Qué hace y por qué:** `-e` envía el log de Filebeat a la consola, útil para el laboratorio. En producción, Filebeat se instala como servicio con el script `install-service-filebeat.ps1` incluido en el paquete.

*Si necesitas reingerir desde cero: detén Filebeat con `Ctrl + C`, borra los data streams (`DELETE _data_stream/logs-weblab.*` en Dev Tools), borra la carpeta `C:\elastic\filebeat\data\registry` y vuelve a ejecutarlo.*

#### Paso 12. Verifica los data streams y los documentos

```http
GET _data_stream/logs-weblab.*

GET logs-weblab.access-default/_count
GET logs-weblab.app-default/_count

GET logs-weblab.access-default/_count
{ "query": { "term": { "event.kind": "pipeline_error" } } }

GET logs-weblab.app-default/_search
{
  "size": 1,
  "query": { "exists": { "field": "error.stack_trace" } },
  "_source": ["@timestamp", "log.level", "service.name", "host.name", "trace.id", "message"]
}
```

**Qué hace y por qué:** `_data_stream` muestra cada data stream, su índice de respaldo activo (`.ds-...-000001`), el template que lo creó, la política ILM y su estado de salud. Los conteos deben ser **3,421** documentos de acceso y **2,458** de aplicación: este último es menor que el número de líneas físicas del archivo, porque cada stack trace se agrupó en un solo evento. Ningún documento debe tener `pipeline_error`.

#### Paso 13. Monitorea las estadísticas de los pipelines

```http
GET _nodes/stats/ingest?filter_path=nodes.*.ingest.pipelines.weblab-*
```

**Qué hace y por qué:** para cada pipeline y cada procesador se informa `count` (documentos procesados), `time_in_millis` (tiempo acumulado) y `failed`. Sirve para detectar procesadores lentos, como un `grok` con muchos `GREEDYDATA`, o fallas silenciosas que terminan en `on_failure`. Observa que los `grok` opcionales del pipeline `weblab-app` muestran fallas: son esperadas, porque tienen `ignore_failure`.

---

### Resultado esperado

- Pipelines `weblab-access` y `weblab-app`, y templates `logs-weblab.access` y `logs-weblab.app`, creados.
- Data streams `logs-weblab.access-default` (**3,421** documentos) y `logs-weblab.app-default` (**2,458** documentos), en estado `GREEN`.
- Un documento de acceso indexado se ve así (extracto):

```json
{
  "@timestamp": "2026-10-20T19:38:51.000Z",
  "host": { "name": "srv-web02" },
  "source": { "ip": "177.228.46.19", "geo": { "country_iso_code": "MX", "...": "..." } },
  "http": { "request": { "method": "POST" }, "response": { "status_code": 503, "body": { "bytes": 187 } } },
  "url": { "original": "/api/checkout", "path": "/api/checkout" },
  "event": { "duration": 3041000000, "outcome": "failure", "dataset": "weblab.access", "kind": "event" },
  "nginx": { "access": { "upstream_host": "srv-app02", "upstream_port": 8080, "request_time": 3.041 } },
  "trace": { "id": "664595a620d2a6692d4bc1330b677eae" },
  "user_agent": { "name": "Safari", "os": { "name": "Mac OS X" } }
}
```

- Filebeat sigue en ejecución sin errores de conexión.

> **Nota sobre Elastic Agent:** con Elastic Agent y Fleet el flujo es equivalente. Se agrega la integración **Custom Logs (Filestream)** a una política de agente con `dataset` = `weblab.access` y `namespace` = `default`, y el pipeline se conecta mediante el pipeline personalizado `logs-weblab.access@custom`, que el pipeline integrado invoca automáticamente. Fleet requiere un Fleet Server; en redes sin Internet se configura además un *Elastic Package Registry* local y un *artifact registry* interno.

---

## Práctica 5b. Procesar logs de sistema con un pipeline de Logstash y validar su normalización con ECS

### Objetivo

Al finalizar la práctica, serás capaz de:

- Instalar Logstash en Windows y ejecutar un pipeline `input → filter → output`.
- Parsear logs syslog con `grok`, `date` y `mutate`, y extraer eventos de autenticación SSH y de uso de `sudo`.
- Enviar los eventos a un data stream con el plugin de salida de Elasticsearch.
- Activar colas persistentes y monitorear Logstash con su API.
- Validar que los campos resultantes cumplen ECS.

### Objetivo visual

![Pipeline de Logstash para syslog](img/cap05b.png)

### Duración aproximada

- 55 minutos.

### Prerrequisitos

- Práctica 5a completada.
- Archivo `C:\labs\logs\system\syslog.log` (663 líneas de syslog de los cinco servidores de la tienda).
- Archivo `C:\labs\config\logstash-syslog.conf`.

---

### Instrucciones

**Escenario:** los servidores Linux de la tienda envían su syslog a un servidor central, que lo deja en `syslog.log`. Necesitas procesar ese archivo con Logstash, normalizarlo con ECS y almacenarlo en el data stream `logs-syslog.lab-default` para analizar accesos SSH y uso de privilegios.

---

### Tarea 1. Instalar Logstash

#### Paso 1. Descarga y extrae Logstash

```powershell
$VER = "8.19.21"
cd C:\elastic\descargas
curl.exe -L -O "https://artifacts.elastic.co/downloads/logstash/logstash-$VER-windows-x86_64.zip"
tar -xf ".\logstash-$VER-windows-x86_64.zip" -C C:\elastic
Get-ChildItem C:\elastic -Directory -Filter "logstash-$VER*" | Rename-Item -NewName "logstash"
Get-ChildItem C:\elastic\logstash
```

**Qué hace y por qué:** Logstash incluye su propio JDK. Es el componente indicado cuando se necesita transformación compleja, enriquecimiento, múltiples destinos o **colas persistentes** que absorban picos y caídas de Elasticsearch.

#### Paso 2. Guarda la contraseña en el keystore de Logstash

```powershell
cd C:\elastic\logstash
.\bin\logstash-keystore.bat create
.\bin\logstash-keystore.bat add ES_PWD
.\bin\logstash-keystore.bat list
```

*Si `create` pregunta si deseas continuar sin contraseña para el keystore, responde `y`.*

**Qué hace y por qué:** igual que en Filebeat, la referencia `${ES_PWD}` del archivo `.conf` se resuelve desde el keystore, de modo que la contraseña no queda en texto plano.

---

### Tarea 2. Analizar y probar el pipeline de Logstash

#### Paso 3. Revisa el archivo de configuración

```powershell
notepad C:\labs\config\logstash-syslog.conf
```

**Qué hace y por qué:** un pipeline de Logstash tiene tres secciones:

| Sección | Plugin | Función |
|---|---|---|
| `input` | `file` | Lee `syslog.log` desde el inicio (`start_position => "beginning"`). `sincedb_path => "NUL"` evita recordar la posición, porque es un laboratorio. En Windows las rutas usan `/`. |
| `filter` | `grok` | Separa el encabezado syslog: fecha, host, proceso, PID y mensaje. Las partes temporales se guardan en `[@metadata]`, que no se envía a Elasticsearch. |
| `filter` | `date` | Convierte `Oct 20 13:50:03` a `@timestamp`. Syslog RFC 3164 **no** trae año ni zona horaria, así que se declara `timezone => "America/Mexico_City"`. Se incluyen dos formatos porque los días de un dígito llevan doble espacio (`Oct  5`). |
| `filter` | `mutate` | Reemplaza `message` por el texto del evento, corrige `host.name` y agrega `event.module` y `event.dataset`. |
| `filter` | condicional + `grok` | Solo para `sshd`: extrae `user.name`, `source.ip` y `source.port`, y clasifica el evento con `event.category: authentication` y `event.outcome: success/failure`. |
| `filter` | condicional + `grok` | Solo para `sudo`: extrae el usuario, el comando (`process.command_line`) y detecta el error `user NOT in sudoers`. |
| `output` | `elasticsearch` | Envía a un **data stream** `logs-syslog.lab-default` (tipo, dataset y namespace) con TLS y el usuario `elastic`. |
| `output` | `stdout` | Imprime un punto por evento para ver el avance. |

#### Paso 4. Valida la sintaxis

```powershell
cd C:\elastic\logstash
.\bin\logstash.bat -f C:\labs\config\logstash-syslog.conf --config.test_and_exit
```

**Qué hace y por qué:** compila el pipeline sin ejecutarlo. Debe terminar con `Configuration OK`. Detecta llaves sin cerrar, plugins mal escritos y opciones inválidas antes de procesar datos.

#### Paso 5. (Opcional) Prueba el filtro con entrada interactiva

Crea un archivo de prueba que lea desde el teclado:

```powershell
@"
input { stdin { } }
filter {
  grok { match => { "message" => "^%{SYSLOGTIMESTAMP:[@metadata][fecha_syslog]} %{SYSLOGHOST:[host][hostname]} %{DATA:[process][name]}(?:\[%{POSINT:[process][pid]:int}\])?: %{GREEDYDATA:[@metadata][mensaje]}$" } }
}
output { stdout { codec => rubydebug { metadata => true } } }
"@ | Set-Content -Encoding ascii C:\labs\config\prueba-stdin.conf

.\bin\logstash.bat -f C:\labs\config\prueba-stdin.conf
```

Cuando aparezca `Pipeline started`, pega esta línea y presiona `Enter`:

```text
Oct 20 13:58:41 srv-web01 sshd[25381]: Accepted password for soporte from 89.160.20.156 port 51234 ssh2
```

Observa el resultado y termina con `Ctrl + C`.

**Qué hace y por qué:** el codec `rubydebug` con `metadata => true` muestra el evento completo, incluidos los campos `[@metadata]`. Es la forma más rápida de desarrollar un patrón grok antes de integrarlo al pipeline real. También puedes probar patrones en **Kibana > Dev Tools > Grok Debugger**.

---

### Tarea 3. Ejecutar el pipeline con cola persistente

#### Paso 6. Activa la cola persistente

```powershell
notepad C:\elastic\logstash\config\logstash.yml
```

Agrega al final:

```yaml
queue.type: persisted
queue.max_bytes: 512mb
path.queue: C:\elastic\logstash\data\queue
api.http.host: 127.0.0.1
api.http.port: 9600
```

**Qué hace y por qué:** por defecto, Logstash usa una cola en **memoria**: si el proceso se detiene, se pierden los eventos en tránsito. La **cola persistente** los guarda en disco entre la entrada y los filtros, y los conserva si Elasticsearch no está disponible o si Logstash se reinicia. `queue.max_bytes` limita el disco usado; al llenarse, Logstash aplica *back-pressure* y deja de leer la entrada. La API de monitoreo queda en `http://127.0.0.1:9600`.

#### Paso 7. Ejecuta el pipeline

```powershell
cd C:\elastic\logstash
.\bin\logstash.bat -f C:\labs\config\logstash-syslog.conf
```

Espera a que aparezca `Pipeline started {"pipeline.id"=>"main"}` y, después, una fila de puntos (un punto por evento).

**Qué hace y por qué:** Logstash lee el archivo, aplica los filtros a cada línea y envía los eventos en lotes. Como el input `file` sigue esperando nuevas líneas (`mode => "tail"`), el proceso no termina solo: déjalo abierto mientras haces el Paso 8.

#### Paso 8. Consulta la API de monitoreo de Logstash

En **otra** ventana de PowerShell:

```powershell
curl.exe -s "http://127.0.0.1:9600/_node/stats/pipelines/main?pretty" | Select-String -Pattern '"in"|"out"|"filtered"|"queue_size_in_bytes"|"type"' | Select-Object -First 12
curl.exe -s "http://127.0.0.1:9600/_node/stats/pipelines/main?pretty" > C:\labs\logstash-stats.json
notepad C:\labs\logstash-stats.json
```

**Qué hace y por qué:** la API muestra los eventos de entrada (`in`), los filtrados y los de salida (`out`). Cuando los tres valen 663, todo el archivo se procesó. También muestra el tipo y tamaño de la cola (`persisted`) y estadísticas **por plugin**: los `id` declarados en el `.conf` (`grok_encabezado_syslog`, `grok_sshd`...) permiten identificar cuál filtro es lento o falla.

Cuando `out` sea igual a 663, detén Logstash con `Ctrl + C` en la primera ventana.

> **Importante:** si ejecutas el pipeline de nuevo, se reprocesa todo el archivo y se duplican los eventos, porque `sincedb_path` es `NUL`. Antes de repetirlo, borra el data stream con `DELETE _data_stream/logs-syslog.lab-default`.

---

### Tarea 4. Validar la normalización con ECS

#### Paso 9. Verifica el conteo y que no haya fallas de parseo

```http
GET logs-syslog.lab-default/_count

GET logs-syslog.lab-default/_count
{ "query": { "terms": { "tags": ["_grokparsefailure", "_dateparsefailure", "_grok_sudo_fallo"] } } }
```

**Qué hace y por qué:** debe haber **663** documentos y **0** con etiquetas de error. Logstash agrega `_grokparsefailure` o `_dateparsefailure` cuando un filtro no puede procesar el evento. Revisar estas etiquetas es el primer control de calidad de cualquier pipeline.

#### Paso 10. Comprueba los tipos de los campos ECS

```http
GET logs-syslog.lab-default/_field_caps?fields=source.ip,source.port,process.pid,user.name,host.name,event.outcome,@timestamp

GET logs-syslog.lab-default/_mapping/field/source.ip
```

**Qué hace y por qué:** `_field_caps` muestra el tipo y si el campo es buscable y agregable. Debes ver `source.ip` → `ip`, `process.pid` y `source.port` → `long`, `user.name`, `host.name` y `event.outcome` → `keyword`, y `@timestamp` → `date`. No declaraste estos tipos: los asignó `ecs@mappings` del template integrado `logs`, **porque los nombres de campo siguen ECS**. Esa es la ventaja práctica de normalizar: los dashboards, reglas de detección y consultas funcionan igual para cualquier fuente.

#### Paso 11. Compara un evento original con el normalizado

```http
GET logs-syslog.lab-default/_search
{
  "size": 1,
  "query": { "bool": { "filter": [
    { "term": { "event.action": "ssh_login" } },
    { "term": { "event.outcome": "success" } },
    { "term": { "source.ip": "89.160.20.156" } }
  ] } },
  "_source": ["@timestamp", "event.original", "host.name", "process.name", "process.pid", "user.name", "source.ip", "source.port", "event.category", "event.outcome", "sshd.auth_method", "data_stream"]
}
```

**Qué hace y por qué:** muestra lado a lado la línea original (`event.original`) y los campos ECS:

| Línea original | Campo ECS | Valor |
|---|---|---|
| `Oct 20 13:58:41` | `@timestamp` | `2026-10-20T19:58:41.000Z` |
| `srv-web01` | `host.name` | `srv-web01` |
| `sshd[25381]` | `process.name`, `process.pid` | `sshd`, `25381` |
| `Accepted password for soporte` | `event.outcome`, `user.name` | `success`, `soporte` |
| `from 89.160.20.156 port 51234` | `source.ip`, `source.port` | `89.160.20.156`, `51234` |

Este evento es relevante: un inicio de sesión **exitoso** desde la misma IP que hizo decenas de intentos fallidos. Lo investigarás en la Práctica 7.

#### Paso 12. Responde preguntas operativas con los campos normalizados

```http
GET logs-syslog.lab-default/_count
{ "query": { "bool": { "filter": [
  { "term": { "process.name": "sshd" } },
  { "term": { "event.outcome": "failure" } }
] } } }

GET logs-syslog.lab-default/_search
{
  "size": 5,
  "query": { "bool": { "filter": [
    { "term": { "process.name": "sudo" } },
    { "term": { "event.outcome": "failure" } }
  ] } },
  "_source": ["@timestamp", "host.name", "user.name", "process.command_line", "sudo.error"]
}
```

**Qué hace y por qué:** la primera consulta cuenta los inicios de sesión SSH fallidos (**145**). La segunda encuentra los intentos de `sudo` rechazados: el usuario `soporte` intentó abrir `/bin/bash` y leer `/etc/shadow` como root. Sin la normalización, estas preguntas exigirían buscar texto libre en cada mensaje.

---

### Resultado esperado

- Logstash procesó los 663 eventos (`in` = `out` = 663) con cola persistente.
- Data stream `logs-syslog.lab-default` con **663** documentos, sin etiquetas `_grokparsefailure` ni `_dateparsefailure`.
- Los campos `source.ip` (`ip`), `process.pid` (`long`), `user.name`, `host.name` y `event.outcome` (`keyword`) tienen tipos ECS correctos.
- Se identificaron **145** intentos SSH fallidos y **2** intentos de `sudo` rechazados del usuario `soporte`.

```http
GET _cat/indices/.ds-logs-*?v&h=index,health,docs.count&s=index&expand_wildcards=all
```

```text
index                                                health docs.count
.ds-logs-syslog.lab-default-2026.10.21-000001         green         663
.ds-logs-weblab.access-default-2026.10.21-000001      green        3421
.ds-logs-weblab.app-default-2026.10.21-000001         green        2458
```

(La fecha en el nombre del backing index corresponde al día en que se **creó** el data stream, no a la fecha de los eventos).

### Referencias

- Elastic. *Ingest pipelines [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/ingest.html
- Elastic. *Grok, Dissect, Date, GeoIP y User agent processors [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/processors.html
- Elastic. *Data streams [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/data-streams.html
- Elastic. *Filebeat Reference: filestream input y multiline [8.19].* https://www.elastic.co/guide/en/beats/filebeat/8.19/filebeat-input-filestream.html
- Elastic. *Logstash Reference: Elasticsearch output plugin y persistent queues [8.19].* https://www.elastic.co/guide/en/logstash/8.19/plugins-outputs-elasticsearch.html
- Elastic. *Elastic Common Schema (ECS).* https://www.elastic.co/guide/en/ecs/current/index.html
