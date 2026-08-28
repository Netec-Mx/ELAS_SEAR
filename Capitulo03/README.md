# Crear templates, mappings y un data stream para logs de aplicaciones

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 70 minutos |
| Complejidad | Media |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica se construirá una estructura reutilizable para almacenar logs de una API mediante un *data stream* de Elasticsearch 9.3.0. Se crearán dos *component templates*: uno para la configuración de shards y réplicas, y otro para mappings ECS mínimos y reglas dinámicas controladas para etiquetas.

Finalmente, se creará el *composable index template* para el patrón `logs-acme.api-*`, se indexarán eventos de ejemplo y se verificará el índice de respaldo, el destino lógico de escritura, los mappings resultantes y la distribución de shards.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Crear *component templates* reutilizables para settings y mappings.
- [ ] Definir un *composable index template* habilitado para un *data stream*.
- [ ] Aplicar mappings explícitos para campos ECS habituales de logs.
- [ ] Limitar el mapeo dinámico de etiquetas para reducir el riesgo de *mapping explosion*.
- [ ] Crear e inspeccionar el data stream `logs-acme.api-default` y su índice de respaldo.

## Prerrequisitos

### Conocimientos

- Comprensión básica de documentos JSON y solicitudes REST.
- Conocimiento conceptual de índices, aliases, data streams, shards y réplicas.
- Diferencia entre los tipos `text`, `keyword`, `date`, `integer`, `boolean` y `object`.
- Comprensión básica del Elastic Common Schema (ECS).

### Acceso y estado requerido

- Las prácticas 1 y 2 deben estar completadas.
- Elasticsearch 9.3.0 debe estar disponible en `https://localhost:9201`.
- El contenedor obligatorio `es930-lab` debe estar en ejecución.
- Debe existir el archivo `/opt/elastic-labs/.env` con permisos `0600`.
- El usuario de laboratorio es `elastic`.
- La contraseña se obtiene desde el archivo `.env`; no la escriba directamente en comandos ni la reutilice fuera del laboratorio.

## Entorno de laboratorio

| Componente | Valor requerido |
|---|---|
| Directorio raíz | `/opt/elastic-labs` |
| Directorio de trabajo | `/opt/elastic-labs/work` |
| Elasticsearch objetivo | Elasticsearch 9.3.0 |
| Contenedor Elasticsearch | `es930-lab` |
| Cluster | `es930-lab-cluster` |
| Nodo | `es930-node-1` |
| URL de Elasticsearch | `https://localhost:9201` |
| Red Docker | `elastic-lab-net` |
| Volumen persistente | `es930-data` |
| Data stream de esta práctica | `logs-acme.api-default` |

> **Importante:** esta práctica utiliza exclusivamente Elasticsearch 9.3.0 en el puerto `9201`. No ejecute operaciones contra Elasticsearch 7.17.29 en el puerto `9200`.

### Preparación de la terminal

1. Abra una terminal y acceda al directorio obligatorio del laboratorio.

   ```bash
   cd /opt/elastic-labs
   ```

2. Verifique que el archivo de variables existe, que tiene permisos restrictivos y que el contenedor de Elasticsearch 9.3.0 está activo.

   ```bash
   ls -l /opt/elastic-labs/.env
   docker ps --filter "name=es930-lab"
   ```

3. Exporte la contraseña desde el archivo `.env` sin mostrarla en pantalla.

   ```bash
   export ELASTIC_PASSWORD="$(sed -n 's/^ELASTIC_PASSWORD=//p' /opt/elastic-labs/.env | tail -n 1)"
   test -n "$ELASTIC_PASSWORD" && echo "Contraseña de laboratorio cargada."
   ```

4. Defina variables y una función auxiliar para las llamadas a la API.

   ```bash
   export ES_URL="https://localhost:9201"

   api() {
     curl -ksS --fail-with-body \
       -u "elastic:${ELASTIC_PASSWORD}" \
       "$@"
   }
   ```

5. Compruebe la conectividad y el estado del cluster.

   ```bash
   api "${ES_URL}/_cluster/health?pretty"
   ```

**Resultado esperado**

La respuesta debe indicar el nombre del cluster `es930-lab-cluster`. Antes de crear datos, el estado puede ser `green` o `yellow`, según el estado previo del laboratorio. Después de crear el data stream con cero réplicas, los shards primarios de esta práctica deben quedar en estado `green`.

**Verificación**

Ejecute:

```bash
api "${ES_URL}/_cat/nodes?v"
```

Debe aparecer un nodo llamado `es930-node-1`.

---

## Procedimiento paso a paso

### Paso 1. Crear el directorio de trabajo y validar recursos previos

**Objetivo:** preparar archivos JSON reproducibles y comprobar si existen recursos de una ejecución anterior.

**Instrucciones**

1. Cree el directorio de trabajo si no existe.

   ```bash
   mkdir -p /opt/elastic-labs/work
   cd /opt/elastic-labs/work
   ```

2. Consulte los templates existentes relacionados con el laboratorio.

   ```bash
   api "${ES_URL}/_component_template/ct-logs-acme-*?pretty"
   ```

3. Consulte si ya existe el composable template principal.

   ```bash
   api "${ES_URL}/_index_template/logs-acme-api-template?pretty" || true
   ```

4. Consulte si el data stream ya fue creado anteriormente.

   ```bash
   api "${ES_URL}/_data_stream/logs-acme.api-default?pretty" || true
   ```

**Resultado esperado**

En una ejecución inicial, las consultas de recursos inexistentes pueden devolver un error `404`. Si los recursos existen por una repetición de la práctica, se mostrarán sus definiciones actuales.

**Verificación**

No elimine recursos existentes todavía. Los comandos `PUT` de los pasos siguientes actualizan los templates de forma idempotente. Si el data stream ya existe, podrá reutilizarlo para las validaciones, siempre que su configuración coincida con la de esta guía.

---

### Paso 2. Crear el component template de settings

**Objetivo:** definir una configuración reutilizable para los índices de respaldo del data stream, adecuada para un cluster de un solo nodo.

**Instrucciones**

1. Cree el archivo `ct-logs-acme-settings.json`.

   ```bash
   cat > /opt/elastic-labs/work/ct-logs-acme-settings.json <<'JSON'
   {
     "template": {
       "settings": {
         "index.number_of_shards": 1,
         "index.number_of_replicas": 0,
         "index.refresh_interval": "5s"
       }
     },
     "_meta": {
       "description": "Settings reutilizables para logs de ACME en el laboratorio",
       "lab_id": "03-00-01",
       "managed_by": "elastic-labs"
     }
   }
   JSON
   ```

2. Cree el *component template* en Elasticsearch.

   ```bash
   api -X PUT "${ES_URL}/_component_template/ct-logs-acme-settings" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/ct-logs-acme-settings.json | jq
   ```

3. Consulte el recurso creado.

   ```bash
   api "${ES_URL}/_component_template/ct-logs-acme-settings?pretty"
   ```

**Resultado esperado**

La creación debe devolver una respuesta similar a:

```json
{
  "acknowledged": true
}
```

La consulta posterior debe mostrar:

- `index.number_of_shards` con valor `1`.
- `index.number_of_replicas` con valor `0`.
- `index.refresh_interval` con valor `5s`.

**Verificación**

Ejecute:

```bash
api "${ES_URL}/_component_template/ct-logs-acme-settings" \
  | jq '.component_templates[0].component_template.template.settings'
```

Confirme que el número de réplicas es `0`. Este valor evita shards réplica no asignados en el cluster de un solo nodo del laboratorio.

---

### Paso 3. Crear el component template de mappings ECS mínimos

**Objetivo:** definir mappings explícitos para los campos de log frecuentes y controlar las etiquetas dinámicas.

**Instrucciones**

1. Cree el archivo de mappings.

   ```bash
   cat > /opt/elastic-labs/work/ct-logs-acme-mappings.json <<'JSON'
   {
     "template": {
       "mappings": {
         "dynamic": false,
         "dynamic_templates": [
           {
             "labels_flat_keywords": {
               "path_match": "labels.*",
               "path_unmatch": "labels.*.*",
               "match_mapping_type": "string",
               "mapping": {
                 "type": "keyword",
                 "ignore_above": 256
               }
             }
           }
         ],
         "properties": {
           "@timestamp": {
             "type": "date"
           },
           "message": {
             "type": "text",
             "fields": {
               "keyword": {
                 "type": "keyword",
                 "ignore_above": 1024
               }
             }
           },
           "log": {
             "properties": {
               "level": {
                 "type": "keyword"
               }
             }
           },
           "service": {
             "properties": {
               "name": {
                 "type": "keyword"
               },
               "environment": {
                 "type": "keyword"
               }
             }
           },
           "event": {
             "properties": {
               "dataset": {
                 "type": "keyword"
               }
             }
           },
           "host": {
             "properties": {
               "name": {
                 "type": "keyword"
               }
             }
           },
           "http": {
             "properties": {
               "response": {
                 "properties": {
                   "status_code": {
                     "type": "integer"
                   }
                 }
               }
             }
           },
           "error": {
             "properties": {
               "message": {
                 "type": "text",
                 "fields": {
                   "keyword": {
                     "type": "keyword",
                     "ignore_above": 1024
                   }
                 }
               }
             }
           },
           "labels": {
             "type": "object",
             "dynamic": "strict"
           }
         }
       }
     },
     "_meta": {
       "description": "Mappings ECS mínimos y etiquetas planas controladas",
       "lab_id": "03-00-01",
       "dynamic_mapping_policy": "Campos no definidos se conservan en _source pero no se agregan al mapping; labels solo admite cadenas planas"
     }
   }
   JSON
   ```

2. Cree el component template.

   ```bash
   api -X PUT "${ES_URL}/_component_template/ct-logs-acme-mappings" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/ct-logs-acme-mappings.json | jq
   ```

3. Consulte el template y visualice sus propiedades principales.

   ```bash
   api "${ES_URL}/_component_template/ct-logs-acme-mappings" \
     | jq '.component_templates[0].component_template.template.mappings'
   ```

**Resultado esperado**

Debe obtener una confirmación con `"acknowledged": true`.

El mapping debe incluir:

- `@timestamp` como `date`.
- `message` como `text` y el subcampo `message.keyword`.
- `log.level`, `service.name`, `service.environment`, `event.dataset` y `host.name` como `keyword`.
- `http.response.status_code` como `integer`.
- `error.message` como `text`.
- `labels` como objeto con comportamiento dinámico estricto.

**Verificación**

Observe dos controles complementarios:

1. El mapping raíz tiene `"dynamic": false`. Por ello, campos no definidos, como objetos de depuración arbitrarios, no se incorporan automáticamente al mapping.
2. El objeto `labels` admite únicamente nuevas etiquetas planas de tipo cadena mediante el *dynamic template*. Valores complejos, como objetos anidados, deben ser rechazados.

> **Nota técnica:** el contenido de un campo no mapeado puede permanecer en `_source`, pero Elasticsearch no lo indexará como campo consultable. Esto permite conservar evidencia original sin crear campos ilimitados en el mapping.

---

### Paso 4. Crear el composable index template para el data stream

**Objetivo:** combinar settings y mappings en un template aplicable al patrón `logs-acme.api-*`.

**Instrucciones**

1. Cree el archivo del composable index template.

   ```bash
   cat > /opt/elastic-labs/work/logs-acme-api-template.json <<'JSON'
   {
     "index_patterns": [
       "logs-acme.api-*"
     ],
     "data_stream": {},
     "composed_of": [
       "ct-logs-acme-settings",
       "ct-logs-acme-mappings"
     ],
     "priority": 350,
     "_meta": {
       "description": "Template composable para el data stream de logs de la API ACME",
       "lab_id": "03-00-01",
       "owner": "equipo-plataforma"
     }
   }
   JSON
   ```

2. Cree el template.

   ```bash
   api -X PUT "${ES_URL}/_index_template/logs-acme-api-template" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/logs-acme-api-template.json | jq
   ```

3. Consulte su definición.

   ```bash
   api "${ES_URL}/_index_template/logs-acme-api-template?pretty"
   ```

4. Simule la aplicación del template sobre el nombre del futuro data stream.

   ```bash
   api -X POST "${ES_URL}/_index_template/_simulate_index/logs-acme.api-default" \
     | jq '{
       template_settings: .template.settings,
       template_mappings: .template.mappings,
       overlapping_templates: .overlapping
     }'
   ```

**Resultado esperado**

La simulación debe mostrar:

- Un shard primario.
- Cero réplicas.
- El mapping explícito de `@timestamp`, `message`, `log`, `service`, `event`, `host`, `http`, `error` y `labels`.
- La aplicación del template `logs-acme-api-template`.

**Verificación**

Ejecute la siguiente consulta focalizada:

```bash
api -X POST "${ES_URL}/_index_template/_simulate_index/logs-acme.api-default" \
  | jq '{
    shards: .template.settings["index.number_of_shards"],
    replicas: .template.settings["index.number_of_replicas"],
    timestamp_type: .template.mappings.properties["@timestamp"].type,
    log_level_type: .template.mappings.properties.log.properties.level.type,
    status_code_type: .template.mappings.properties.http.properties.response.properties.status_code.type
  }'
```

Debe mostrar los valores `1`, `0`, `date`, `keyword` e `integer`.

---

### Paso 5. Crear el data stream mediante una operación create

**Objetivo:** crear automáticamente el data stream `logs-acme.api-default` y su primer índice de respaldo.

**Instrucciones**

1. Cree el primer evento de aplicación. La operación `/_create/` impone semántica de solo creación y permite la creación automática del data stream cuando existe un template compatible.

   ```bash
   cat > /opt/elastic-labs/work/log-app-0001.json <<'JSON'
   {
     "@timestamp": "2026-08-25T09:00:00Z",
     "message": "Solicitud procesada correctamente",
     "log": {
       "level": "INFO"
     },
     "service": {
       "name": "acme-api",
       "environment": "production"
     },
     "event": {
       "dataset": "acme.api"
     },
     "host": {
       "name": "api-01"
     },
     "http": {
       "response": {
         "status_code": 200
       }
     },
     "labels": {
       "release": "2026.08.25",
       "region": "eu-west-1"
     }
   }
   JSON
   ```

2. Envíe el documento al nombre lógico del data stream.

   ```bash
   api -X PUT "${ES_URL}/logs-acme.api-default/_create/app-0001" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/log-app-0001.json | jq
   ```

3. Consulte el data stream creado.

   ```bash
   api "${ES_URL}/_data_stream/logs-acme.api-default?pretty"
   ```

**Resultado esperado**

La respuesta de indexación debe incluir:

```json
{
  "result": "created"
}
```

El atributo `_index` debe contener un nombre similar a:

```text
.ds-logs-acme.api-default-2026.08.25-000001
```

La fecha concreta del nombre físico puede variar según el entorno. El sufijo inicial de generación debe ser `000001`.

**Verificación**

Extraiga el nombre del primer índice de respaldo:

```bash
export BACKING_INDEX="$(
  api "${ES_URL}/_data_stream/logs-acme.api-default" \
  | jq -r '.data_streams[0].indices[-1].index_name'
)"

echo "$BACKING_INDEX"
```

El valor debe comenzar por:

```text
.ds-logs-acme.api-default-
```

> **Importante:** los productores deben escribir en `logs-acme.api-default`, no directamente en `$BACKING_INDEX`. El data stream utiliza el índice de respaldo actual como destino interno de escritura.

---

### Paso 6. Indexar eventos adicionales y comprobar mappings dinámicos controlados

**Objetivo:** añadir logs representativos y demostrar el comportamiento de campos explícitos, etiquetas controladas y campos no mapeados.

**Instrucciones**

1. Cree un evento de error HTTP.

   ```bash
   cat > /opt/elastic-labs/work/log-app-0002.json <<'JSON'
   {
     "@timestamp": "2026-08-25T09:02:14Z",
     "message": "No fue posible validar el token de acceso",
     "log": {
       "level": "ERROR"
     },
     "service": {
       "name": "acme-api",
       "environment": "production"
     },
     "event": {
       "dataset": "acme.api"
     },
     "host": {
       "name": "api-02"
     },
     "http": {
       "response": {
         "status_code": 401
       }
     },
     "error": {
       "message": "JWT expired"
     },
     "labels": {
       "release": "2026.08.25",
       "component": "authentication"
     }
   }
   JSON
   ```

2. Cree un evento de advertencia que incluya un objeto de diagnóstico no definido en el mapping raíz.

   ```bash
   cat > /opt/elastic-labs/work/log-app-0003.json <<'JSON'
   {
     "@timestamp": "2026-08-25T09:05:45Z",
     "message": "Latencia elevada durante la consulta de clientes",
     "log": {
       "level": "WARN"
     },
     "service": {
       "name": "acme-api",
       "environment": "production"
     },
     "event": {
       "dataset": "acme.api"
     },
     "host": {
       "name": "api-01"
     },
     "http": {
       "response": {
         "status_code": 200
       }
     },
     "labels": {
       "release": "2026.08.25",
       "component": "customers"
     },
     "debug_context": {
       "query_time_ms": 847,
       "temporary_trace": "trace-4f9a"
     }
   }
   JSON
   ```

3. Indexe ambos documentos con IDs únicos.

   ```bash
   api -X PUT "${ES_URL}/logs-acme.api-default/_create/app-0002" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/log-app-0002.json | jq

   api -X PUT "${ES_URL}/logs-acme.api-default/_create/app-0003" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/log-app-0003.json | jq
   ```

4. Fuerce una actualización de búsqueda para hacer visibles los documentos inmediatamente.

   ```bash
   api -X POST "${ES_URL}/logs-acme.api-default/_refresh" | jq
   ```

5. Consulte el documento con el objeto no mapeado.

   ```bash
   api "${ES_URL}/logs-acme.api-default/_doc/app-0003?pretty"
   ```

**Resultado esperado**

Los documentos `app-0002` y `app-0003` deben crearse correctamente.

El campo `debug_context` debe aparecer dentro de `_source` del documento `app-0003`, pero no debe convertirse en un campo indexado del mapping, debido a `"dynamic": false` en la raíz.

**Verificación**

1. Compruebe que una etiqueta dinámica plana quedó como `keyword`.

   ```bash
   api "${ES_URL}/logs-acme.api-default/_field_caps?fields=labels.release,labels.component" \
     | jq '.fields'
   ```

   Debe aparecer el tipo `keyword`.

2. Consulte los tipos de campos principales.

   ```bash
   api "${ES_URL}/logs-acme.api-default/_field_caps?fields=@timestamp,message,message.keyword,log.level,service.name,http.response.status_code,error.message" \
     | jq '.fields'
   ```

3. Revise el mapping completo relevante.

   ```bash
   api "${ES_URL}/logs-acme.api-default/_mapping" \
     | jq '.[].mappings.properties | {
       "@timestamp": .["@timestamp"],
       message: .message,
       log: .log,
       service: .service,
       http: .http,
       error: .error,
       labels: .labels
     }'
   ```

---

### Paso 7. Validar la restricción de etiquetas complejas

**Objetivo:** comprobar que el objeto `labels` rechaza estructuras no permitidas y protege el mapping ante una expansión no controlada.

**Instrucciones**

1. Cree un documento intencionalmente inválido: `labels.release` contiene un objeto cuando solo debe ser una cadena plana.

   ```bash
   cat > /opt/elastic-labs/work/log-app-invalid.json <<'JSON'
   {
     "@timestamp": "2026-08-25T09:10:00Z",
     "message": "Evento de prueba con etiqueta no permitida",
     "log": {
       "level": "WARN"
     },
     "service": {
       "name": "acme-api",
       "environment": "production"
     },
     "event": {
       "dataset": "acme.api"
     },
     "labels": {
       "release": {
         "major": 2026,
         "minor": 8
       }
     }
   }
   JSON
   ```

2. Envíe la solicitud sin usar la función `api`, ya que se espera una respuesta HTTP `400`.

   ```bash
   curl -ksS \
     -o /opt/elastic-labs/work/invalid-label-response.json \
     -w "HTTP %{http_code}\n" \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -X PUT "${ES_URL}/logs-acme.api-default/_create/app-invalid" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/log-app-invalid.json
   ```

3. Examine el detalle del error.

   ```bash
   jq . /opt/elastic-labs/work/invalid-label-response.json
   ```

**Resultado esperado**

La salida debe indicar:

```text
HTTP 400
```

La respuesta debe contener un error de parsing o mapping relacionado con el comportamiento `strict` de `labels`.

**Verificación**

Confirme que el documento inválido no fue creado:

```bash
api "${ES_URL}/logs-acme.api-default/_doc/app-invalid?pretty" || true
```

Debe devolver un error `404`.

---

### Paso 8. Inspeccionar el índice de respaldo, el destino de escritura y los shards

**Objetivo:** verificar que el data stream administra un backing index y que su configuración coincide con la definida por los templates.

**Instrucciones**

1. Muestre la estructura del data stream.

   ```bash
   api "${ES_URL}/_data_stream/logs-acme.api-default" \
     | jq '.data_streams[0] | {
       name,
       timestamp_field,
       generation,
       indices
     }'
   ```

2. Actualice la variable con el índice de respaldo de escritura actual.

   ```bash
   export BACKING_INDEX="$(
     api "${ES_URL}/_data_stream/logs-acme.api-default" \
     | jq -r '.data_streams[0].indices[-1].index_name'
   )"

   echo "Índice de respaldo actual: ${BACKING_INDEX}"
   ```

3. Consulte los settings aplicados al índice de respaldo.

   ```bash
   api "${ES_URL}/${BACKING_INDEX}/_settings?filter_path=*.settings.index.number_of_shards,*.settings.index.number_of_replicas,*.settings.index.refresh_interval" \
     | jq
   ```

4. Consulte los shards del índice de respaldo.

   ```bash
   api "${ES_URL}/_cat/shards/${BACKING_INDEX}?v"
   ```

5. Verifique el destino lógico de escritura creando un cuarto documento.

   ```bash
   cat > /opt/elastic-labs/work/log-app-0004.json <<'JSON'
   {
     "@timestamp": "2026-08-25T09:15:00Z",
     "message": "Comprobación del destino lógico de escritura",
     "log": {
       "level": "INFO"
     },
     "service": {
       "name": "acme-api",
       "environment": "production"
     },
     "event": {
       "dataset": "acme.api"
     },
     "host": {
       "name": "api-01"
     },
     "http": {
       "response": {
         "status_code": 201
       }
     },
     "labels": {
       "release": "2026.08.25",
       "test_case": "write-target"
     }
   }
   JSON

   api -X PUT "${ES_URL}/logs-acme.api-default/_create/app-0004" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/log-app-0004.json | jq
   ```

**Resultado esperado**

Debe observar:

- Un índice de respaldo cuyo nombre comienza por `.ds-logs-acme.api-default-`.
- Un único shard primario (`p`) en estado `STARTED`.
- Cero shards réplica para este índice.
- `index.number_of_shards: 1`.
- `index.number_of_replicas: 0`.
- El documento `app-0004` creado en el backing index actual.

**Verificación**

Ejecute:

```bash
api "${ES_URL}/logs-acme.api-default/_search?size=10&sort=@timestamp:asc" \
  | jq '{
    total: .hits.total.value,
    documents: [.hits.hits[] | {
      id: ._id,
      index: ._index,
      timestamp: ._source["@timestamp"],
      level: ._source.log.level,
      status: ._source.http.response.status_code
    }]
  }'
```

Los documentos deben mostrar como `_index` el índice físico `.ds-logs-acme.api-default-...`.

> **Aclaración:** un data stream ofrece un nombre lógico de escritura, pero no equivale a un alias convencional configurado manualmente con `is_write_index: true`. Elasticsearch administra internamente cuál es el backing index de escritura. En esta práctica, al existir una única generación, el índice `.ds-...-000001` es el destino de escritura.

## Validación y pruebas

Ejecute la siguiente secuencia para confirmar que la configuración y los datos cumplen los requisitos de la práctica.

1. Compruebe la existencia de los dos component templates y del composable index template.

   ```bash
   api "${ES_URL}/_component_template/ct-logs-acme-settings,ct-logs-acme-mappings" \
     | jq '.component_templates[].name'

   api "${ES_URL}/_index_template/logs-acme-api-template" \
     | jq '.index_templates[].name'
   ```

2. Compruebe que el data stream existe y tiene al menos un backing index.

   ```bash
   api "${ES_URL}/_data_stream/logs-acme.api-default" \
     | jq '{
       name: .data_streams[0].name,
       generation: .data_streams[0].generation,
       backing_indices: [.data_streams[0].indices[].index_name]
     }'
   ```

3. Compruebe que se han indexado cuatro documentos válidos.

   ```bash
   api "${ES_URL}/logs-acme.api-default/_count" | jq
   ```

   El valor esperado de `count` es `4`.

4. Busque eventos con nivel `ERROR`.

   ```bash
   api -X POST "${ES_URL}/logs-acme.api-default/_search" \
     -H "Content-Type: application/json" \
     -d '{
       "query": {
         "term": {
           "log.level": "ERROR"
         }
       }
     }' | jq '{
       total: .hits.total.value,
       hits: [.hits.hits[] | {
         id: ._id,
         message: ._source.message,
         error: ._source.error.message
       }]
     }'
   ```

   Debe recuperar el documento `app-0002`.

5. Busque eventos mediante una etiqueta dinámica controlada.

   ```bash
   api -X POST "${ES_URL}/logs-acme.api-default/_search" \
     -H "Content-Type: application/json" \
     -d '{
       "query": {
         "term": {
           "labels.component": "authentication"
         }
       }
     }' | jq '{
       total: .hits.total.value,
       ids: [.hits.hits[]._id]
     }'
   ```

   Debe recuperar el documento `app-0002`.

6. Verifique que `debug_context` no se añadió al mapping indexado.

   ```bash
   api "${ES_URL}/logs-acme.api-default/_mapping" \
     | jq '[.[].mappings.properties | has("debug_context")]'
   ```

   El resultado esperado es:

   ```json
   [
     false
   ]
   ```

## Resolución de problemas

### Problema 1: El data stream no se crea y Elasticsearch devuelve un error relacionado con template o índice

**Síntoma**

Al enviar el primer documento a `logs-acme.api-default`, Elasticsearch devuelve un error `404`, `400` o indica que no existe un template de data stream aplicable.

**Causa**

El composable index template no existe, no contiene `"data_stream": {}`, o su patrón no coincide exactamente con `logs-acme.api-default`. También puede ocurrir si se usó una operación de indexación incompatible en vez de `/_create/`.

**Solución**

1. Compruebe el template:

   ```bash
   api "${ES_URL}/_index_template/logs-acme-api-template?pretty"
   ```

2. Confirme que contiene:

   ```json
   "index_patterns": ["logs-acme.api-*"],
   "data_stream": {}
   ```

3. Simule el nombre objetivo:

   ```bash
   api -X POST "${ES_URL}/_index_template/_simulate_index/logs-acme.api-default" | jq
   ```

4. Repita la creación usando una operación `PUT` con `/_create/` y un documento que incluya `@timestamp`.

### Problema 2: Un documento es rechazado con `strict_dynamic_mapping_exception` en `labels`

**Síntoma**

La indexación devuelve HTTP `400` y el error menciona `strict_dynamic_mapping_exception`, `labels` o un campo no permitido.

**Causa**

El objeto `labels` fue diseñado para admitir únicamente etiquetas planas de texto, por ejemplo `"region": "eu-west-1"`. Un valor objeto, un arreglo o una estructura anidada no cumple la política definida por el dynamic template y por `"dynamic": "strict"`.

**Solución**

1. Revise el documento JSON.
2. Transforme la etiqueta a un valor de texto plano:

   ```json
   "labels": {
     "release": "2026.08"
   }
   ```

3. No use estructuras como la siguiente:

   ```json
   "labels": {
     "release": {
       "major": 2026
     }
   }
   ```

4. Si necesita almacenar información estructurada adicional, defina explícitamente un campo con mapping controlado en una futura revisión del component template, en lugar de permitir objetos dinámicos ilimitados.

## Limpieza

Los datos generados en esta práctica serán utilizados y ampliados por prácticas posteriores. Por tanto, **no elimine** el data stream, los templates, los volúmenes ni los contenedores.

1. Conserve los archivos de evidencia y configuración en el directorio de trabajo:

   ```bash
   ls -lh /opt/elastic-labs/work
   ```

2. Si desea guardar una evidencia de estado para el informe técnico, exporte la información del data stream:

   ```bash
   api "${ES_URL}/_data_stream/logs-acme.api-default?pretty" \
     > /opt/elastic-labs/work/evidencia-data-stream-logs-acme-api.json

   api "${ES_URL}/logs-acme.api-default/_mapping?pretty" \
     > /opt/elastic-labs/work/evidencia-mapping-logs-acme-api.json
   ```

3. No ejecute el siguiente comando:

   ```bash
   docker compose down -v
   ```

   Este comando eliminaría volúmenes persistentes y rompería la continuidad requerida entre prácticas.

## Resumen

En esta práctica se creó una arquitectura de ingesta para logs de aplicaciones basada en un data stream de Elasticsearch 9.3.0:

- Se creó el component template `ct-logs-acme-settings` con un shard primario y cero réplicas.
- Se creó el component template `ct-logs-acme-mappings` con campos ECS mínimos y controles contra *mapping explosion*.
- Se creó el composable index template `logs-acme-api-template` con prioridad `350` y patrón `logs-acme.api-*`.
- Se creó automáticamente el data stream `logs-acme.api-default` mediante una operación `create`.
- Se verificó el backing index `.ds-logs-acme.api-default-...`, el destino lógico de escritura, los shards y los mappings.
- Se validó que las etiquetas planas se mapean como `keyword`, mientras que objetos arbitrarios en `labels` se rechazan.

Los documentos generados permanecerán disponibles para su procesamiento mediante pipelines en la siguiente práctica y para consultas con Query DSL, KQL y ES|QL en prácticas posteriores.
