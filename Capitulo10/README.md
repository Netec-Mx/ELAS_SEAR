# Laboratorio final: proyecto de integración y demostración

- **Capítulo:** 10. Rendimiento, ciclo de vida y preparación para producción

- **Duración:** 51 min (más un anexo opcional de demostración de 10 a 15 min)

- **Entorno:** Elastic Cloud Hosted (cuenta propia), Elasticsearch 9.x, Kibana Dev Tools y una terminal con `curl`

- **Dependencias:** Laboratorio 9 (alias `weblogs` sobre `weblogs-v2` con 5007 documentos; `weblogs-mappings` con `scaled_float`) y Laboratorio 3 (`weblogs-settings`). Archivo `proyecto_nuevos.ndjson` (500 documentos).

- **Uso:** mini-reto integrador y evaluación final del curso. Se trabaja de forma individual.

## Objetivo

Integrar lo aprendido en una plataforma de logs mínima pero completa: dimensionar, crear un data stream con su template y su política de ILM, cargar datos históricos y nuevos, asegurar el acceso, diagnosticar un problema inducido y completar el checklist de producción. Al terminar, el participante podrá:

- Dimensionar almacenamiento, shards y réplicas a partir de volumen y retención, y trasladarlo a una política de ILM.

- Crear un data stream con template compuesto y una política de ILM con rollover, fase warm y retención.

- Cargar datos históricos (con `_reindex`) y datos nuevos (con `_bulk` y una API key de mínimo privilegio).

- Diagnosticar y mitigar dos incidentes con evidencia, y documentar un plan de mitigación.

- Evaluar una plataforma contra un checklist de producción y presentar los resultados.

## Escenario

La empresa opera cuatro servicios (`checkout`, `catalog`, `payments` y `search`). El equipo de plataforma debe poner en producción un data stream de logs de acceso con estos requisitos:

| **Requisito**                  | **Valor**                                                                                                      |
|--------------------------------|----------------------------------------------------------------------------------------------------------------|
| Volumen estimado en producción | 40 GB por día                                                                                                  |
| Retención                      | 30 días: 7 en hot y 23 en warm                                                                                 |
| Disponibilidad                 | 1 réplica                                                                                                      |
| Factor de expansión medido     | 1.1 (el tamaño en disco es 1.1 veces el dato original)                                                         |
| Margen de capacidad            | 20 % (watermarks, merges y crecimiento)                                                                        |
| Rollover                       | Por tamaño de shard primario (hasta 50 GB) o 7 días, lo que ocurra primero                                     |
| Acceso                         | Un usuario de solo lectura para operaciones; una credencial de ingesta con permisos mínimos                    |
| Datos                          | Histórico: los logs ya cargados en `weblogs`. Nuevos: los del 15 de octubre de 2026 (`proyecto_nuevos.ndjson`) |

En el deployment de la prueba, la política usa los valores reales (50 GB, 7 días, 30 días), aunque el volumen del laboratorio sea mínimo. Lo que se evalúa es el diseño y su verificación.

## Distribución del tiempo y puntaje

| **Parte** | **Contenido**                          | **Tiempo** | **Puntos** |
|-----------|----------------------------------------|------------|------------|
| Parte A   | Dimensionamiento                       | 6 min      | 15         |
| Parte B   | ILM, template y data stream            | 11 min     | 25         |
| Parte C   | Carga de datos históricos y nuevos     | 8 min      | 15         |
| Parte D   | Acceso de solo lectura                 | 5 min      | 10         |
| Parte E   | Diagnóstico de problemas inducidos     | 12 min     | 25         |
| Parte F   | Checklist de producción y demostración | 9 min      | 10         |
| Total     |                                        | 51 min     | 100        |

## Convenciones

- Los bloques con `GET`, `PUT`, `POST` o `DELETE` se ejecutan en Dev Tools. Los `curl.exe` se ejecutan en la terminal.

- Cada parte indica la evidencia que se debe conservar (resultado o captura) para la entrega y la demostración.

- En este laboratorio la explicación de cada comando es breve: los comandos se conocen de laboratorios anteriores. Las novedades sí se explican.

## Parte A: dimensionamiento (6 min, 15 puntos)

Resuelve con los datos del escenario y la fórmula del capítulo 6. Completa la tabla (con calculadora o en papel).

``` console
almacenamiento = GB por día × días de retención × (1 + réplicas) × factor de expansión
capacidad a aprovisionar = almacenamiento / 0.80
```

| **Cálculo**                                                                                                  | **Resultado** |
|--------------------------------------------------------------------------------------------------------------|---------------|
| Almacenamiento total a 30 días                                                                               |               |
| Capacidad a aprovisionar (con margen del 20 %)                                                               |               |
| Almacenamiento en hot (7 días)                                                                               |               |
| Almacenamiento en warm (23 días)                                                                             |               |
| Índices activos a 30 días si el rollover ocurre al llegar a 50 GB por shard primario (1 primario por índice) |               |
| Shards totales con réplica (primarios más réplicas)                                                          |               |
| Shards totales si el rollover fuera diario                                                                   |               |

Justifica en dos líneas por qué conviene el rollover por tamaño y no diario. Los resultados esperados están en la sección "Resumen de valores esperados".

## Parte B: ILM, template y data stream (11 min, 25 puntos)

### Paso B1. Política de ILM

``` console
PUT _ilm/policy/proyecto-ilm
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_primary_shard_size": "50gb", "max_age": "7d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "0d",
        "actions": {
          "set_priority": { "priority": 50 },
          "readonly": {},
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "delete": {
        "min_age": "23d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

**Qué hace:** implementa los requisitos de retención. `min_age` se cuenta desde el rollover. El índice activo recibe escrituras hasta 7 días (hot); al rotar pasa de inmediato a warm (`min_age` de 0) y se elimina 23 días después de rotar. Así, ningún dato supera los 30 días desde su escritura: 7 en hot y 23 en warm.

Pregunta de evaluación: ¿por qué `delete.min_age` es 23d y no 30d? ¿Qué pasaría con 30d? (Respuesta en el resumen.)

### Paso B2. Template del data stream

``` console
PUT _index_template/proyecto-logs-template
{
  "index_patterns": ["proyecto-logs-*"],
  "priority": 300,
  "data_stream": {},
  "composed_of": ["weblogs-settings", "weblogs-mappings"],
  "template": {
    "settings": {
      "index.lifecycle.name": "proyecto-ilm",
      "number_of_shards": 1,
      "number_of_replicas": 1
    }
  },
  "_meta": { "description": "Logs de acceso web del proyecto final" }
}

POST _index_template/_simulate_index/proyecto-logs-prod
```

**Qué hace:** crea el template del data stream componiendo los mismos bloques de mapping y settings del curso, y asocia la política de ILM. La segunda petición muestra qué recibiría el data stream, sin crearlo.

- `data_stream: {}`: los índices que coincidan con el patrón serán data streams.

- `composed_of`: reutiliza `weblogs-settings` (analyzer de ruta) y `weblogs-mappings` (mapping explícito, con `scaled_float` desde el laboratorio 9).

- `template.settings`: tiene prioridad sobre los component templates; aquí fija 1 shard, 1 réplica y la política.

**Resultado esperado de \`\_simulate_index\`:** `index.lifecycle.name` igual a `proyecto-ilm`, el mapping con `response_time_ms` de tipo `scaled_float` y sin otros templates en `overlapping`.

### Paso B3. Crear el data stream y verificar

``` console
PUT _data_stream/proyecto-logs-prod

GET _data_stream/proyecto-logs-prod?filter_path=data_streams.name,data_streams.generation,data_streams.ilm_policy,data_streams.status,data_streams.indices.index_name

GET proyecto-logs-prod/_ilm/explain?filter_path=indices.*.index,indices.*.phase,indices.*.action,indices.*.step
```

**Qué hace:** crea el data stream (con su primer índice de respaldo) y comprueba su estado y el de ILM.

**Resultado esperado:** `generation` de 1, `ilm_policy` `proyecto-ilm`, `status` `GREEN` (o `YELLOW` brevemente mientras se asigna la réplica), y en ILM la fase `hot` con la acción `rollover` en el paso `check-rollover-ready`.

## Parte C: carga de datos históricos y nuevos (8 min, 15 puntos)

### Paso C1. Datos históricos

``` console
POST _reindex?refresh=true
{
  "source": {
    "index": "weblogs",
    "query": { "exists": { "field": "@timestamp" } }
  },
  "dest": { "index": "proyecto-logs-prod", "op_type": "create" }
}

GET proyecto-logs-prod/_count
```

**Qué hace:** copia al data stream los documentos de `weblogs` que tienen `@timestamp`. Los 30 documentos de `parse_failure` no tienen fecha y no se copian: un data stream exige `@timestamp`.

**Resultado esperado:** `created: 4977` y `count` de 4977.

### Paso C2. Credencial de ingesta

``` console
POST _security/api_key
{
  "name": "proyecto-ingesta",
  "expiration": "1d",
  "role_descriptors": {
    "ingesta": {
      "indices": [
        { "names": ["proyecto-logs-*"], "privileges": ["create_doc", "auto_configure"] }
      ]
    }
  }
}
```

**Qué hace:** crea una clave que solo puede añadir documentos nuevos a los data streams `proyecto-logs-*`. Copia el valor de `encoded`.

### Paso C3. Datos nuevos

``` console
curl.exe -s -X POST "<ES_URL>/proyecto-logs-prod/_bulk?filter_path=took,errors" -H "Authorization: ApiKey <ENCODED_KEY>" -H "Content-Type: application/x-ndjson" --data-binary "@proyecto_nuevos.ndjson"
```

**Qué hace:** carga los 500 documentos del 15 de octubre. En un data stream, las líneas de acción del archivo son `create`.

**Resultado esperado:** `"errors": false`. Si fuera `true`, repite la petición sin `filter_path` para ver el error de cada elemento.

### Paso C4. Verificación

``` console
POST proyecto-logs-prod/_refresh

GET proyecto-logs-prod/_search?size=0&track_total_hits=true
{
  "aggs": {
    "por_servicio": { "terms": { "field": "service" } },
    "errores_5xx": { "filter": { "range": { "status": { "gte": 500 } } } },
    "por_dia": { "date_histogram": { "field": "@timestamp", "calendar_interval": "day" } }
  }
}
```

**Qué hace:** refresca el data stream y cuenta por servicio, errores 5xx y día, para comprobar la carga completa.

**Resultado esperado:** total de 5477; payments 827, checkout 1661, search 1027, catalog 1962; 111 errores 5xx; por día: 12 de octubre 1708, 13 de octubre 1621, 14 de octubre 1648 y 15 de octubre 500.

En Kibana, crea el data view `proyecto-logs-prod` (campo de tiempo `@timestamp`) y comprueba en Discover, con el rango del 12 al 16 de octubre de 2026, que el total coincide.

## Parte D: acceso de solo lectura (5 min, 10 puntos)

``` console
PUT _security/role/proyecto_viewer
{
  "indices": [
    { "names": ["proyecto-logs-*"], "privileges": ["read", "view_index_metadata"] }
  ]
}

POST _security/user/ops_viewer
{
  "password": "<CONTRASEÑA_OPS>",
  "roles": ["proyecto_viewer"],
  "full_name": "Operaciones (solo lectura)"
}
```

**Qué hace:** crea un rol de solo lectura sobre el data stream y un usuario con ese rol. Pruébalo desde la terminal:

``` console
curl.exe -s -u ops_viewer "<ES_URL>/proyecto-logs-prod/_count"

curl.exe -s -i -u ops_viewer -X POST "<ES_URL>/proyecto-logs-prod/_doc" -H "Content-Type: application/json" -d "{}"

curl.exe -s -i -u ops_viewer "<ES_URL>/weblogs/_count"
```

**Resultado esperado:** la primera devuelve `count` de 5477; la segunda y la tercera, 403 (`is unauthorized for user [ops_viewer]`): sin permiso de escritura y sin acceso a `weblogs`. Evidencia: los tres resultados.

## Parte E: diagnóstico de problemas inducidos (12 min, 25 puntos)

Cada incidente tiene una instrucción de inducción y un síntoma. Si trabajas en pareja, tu compañero ejecuta la inducción sin decirte qué hace y tú diagnosticas solo con el síntoma. Si trabajas solo, ejecuta la inducción y diagnostica como si no la conocieras. En cada incidente se documenta: síntoma, evidencia, causa, mitigación, verificación y prevención.

### Incidente 1: la ingesta se rechaza con 403

**Inducción:**

``` console
PUT proyecto-logs-prod/_settings
{
  "index.blocks.write": true
}
```

**Síntoma:** al intentar ingestar, la aplicación recibe un error.

``` console
POST proyecto-logs-prod/_doc
{
  "@timestamp": "2026-10-16T00:00:00Z",
  "service": "search",
  "status": 200,
  "message": "prueba de ingesta"
}
```

**Diagnóstico guiado:**

- Lee el mensaje de error: ¿es 401, 403 o 429? ¿Cuál es el tipo de excepción?

- Comprueba los bloqueos del índice con `GET proyecto-logs-prod/_settings?filter_path=*.settings.index.blocks`.

- Descarta las otras causas: salud (`_cluster/health`), disco (`_cat/allocation`) y permisos (`_security/_authenticate`).

**Resultado esperado:** error 403 de tipo `cluster_block_exception` con el texto `blocked by: [FORBIDDEN/8/index write (api)]`. No es un problema de credenciales: es un bloqueo de escritura del índice.

**Mitigación y verificación:**

``` console
PUT proyecto-logs-prod/_settings
{
  "index.blocks.write": null
}

POST proyecto-logs-prod/_doc
{
  "@timestamp": "2026-10-16T00:00:00Z",
  "service": "search",
  "status": 200,
  "message": "prueba de ingesta"
}
```

Nota de contexto: un bloqueo parecido, `index.blocks.read_only_allow_delete`, lo pone el clúster solo cuando un nodo supera el watermark de inundación del disco (95 %). Aquí el bloqueo fue manual; la prevención es controlar quién puede cambiar settings de índices (privilegio `manage`) y auditarlo.

### Incidente 2: faltan documentos tras una carga

**Inducción y síntoma:** una aplicación envía dos eventos con `_bulk` y recibe un código HTTP 200.

``` console
POST proyecto-logs-prod/_bulk
{ "create": {} }
{ "@timestamp": "2026-10-16T00:01:00Z", "service": "payments", "status": "error-interno", "message": "pago rechazado" }
{ "create": {} }
{ "@timestamp": "2026-10-16T00:02:00Z", "service": "payments", "status": 500, "message": "pago con timeout" }
```

**Diagnóstico guiado:**

- La respuesta es 200, pero ¿qué dice el campo `errors`? Revisa el arreglo `items`, elemento por elemento.

- Lee el `status` y la `reason` del elemento fallido.

- Compara el valor enviado con el tipo del campo en el mapping: `GET proyecto-logs-prod/_mapping/field/status`.

**Resultado esperado:** `"errors": true`; el primer elemento falla con estado 400 y un error de tipo `document_parsing_exception` (o `mapper_parsing_exception`): no se pudo interpretar `error-interno` como `short`. El segundo se indexó. La causa está en el productor (envía texto en un campo numérico), no en Elasticsearch.

**Mitigación:** corregir el productor y reenviar solo el documento rechazado. Para eventos en tránsito, reparar con un ingest pipeline que normalice el campo. **Prevención:** revisar `errors` en toda respuesta de `_bulk` y guardar los elementos rechazados para reprocesarlos (cola de eventos fallidos). Usar `ignore_malformed` evita el rechazo, a costa de no indexar el campo defectuoso; es una decisión de diseño, no un parche.

**Limpieza de los documentos de prueba de los incidentes 1 y 2:**

``` console
POST proyecto-logs-prod/_delete_by_query?refresh=true
{
  "query": { "range": { "@timestamp": { "gte": "2026-10-16T00:00:00Z" } } }
}

GET proyecto-logs-prod/_count
```

**Resultado esperado:** `deleted: 2` y `count` de 5477 (los documentos de los incidentes 1 y 2 que sí se indexaron). Los datos nuevos del 15 de octubre quedan intactos.

### Incidente 3 (opcional): el clúster pasa a yellow

Si terminaste antes de tiempo. **Inducción:** `PUT proyecto-logs-prod/_settings` con `{"index.number_of_replicas": 4}`. **Síntoma:** `GET _cluster/health` devuelve `yellow`. Diagnostica con `_cat/indices?health=yellow`, `_cat/shards/proyecto-logs-prod?v` y `_cluster/allocation/explain`, y resuelve devolviendo las réplicas a 1. Es el incidente del laboratorio 9; aquí se resuelve sin la guía.

### Plan de mitigación (entregable)

Completa la tabla del incidente 1 o del 2, según la indique el instructor:

| **Campo**                                    | **Contenido** |
|----------------------------------------------|---------------|
| Incidente                                    |               |
| Impacto (qué dejó de funcionar y para quién) |               |
| Síntoma observado                            |               |
| Evidencia (comando y resultado)              |               |
| Causa raíz                                   |               |
| Mitigación inmediata                         |               |
| Verificación                                 |               |
| Acción correctiva y preventiva               |               |
| Responsable                                  |               |

## Parte F: checklist de producción y demostración (9 min, 10 puntos)

Evalúa tu plataforma contra el checklist. Para cada control, anota la evidencia (comando y resultado) y marca Cumple, No cumple o No aplica. Un control sin evidencia cuenta como no cumplido.

| **Área**         | **Control**                                                                  | **Evidencia**                                                                | **Estado** |
|------------------|------------------------------------------------------------------------------|------------------------------------------------------------------------------|------------|
| Salud            | Clúster `green` y sin shards sin asignar                                     | `GET _cluster/health`                                                        |            |
| Topología        | Al menos 2 zonas de disponibilidad y 1 réplica en todos los índices de datos | Consola de Elastic Cloud; `GET _cat/indices?v&h=index,rep&s=rep`             |            |
| Mapping          | Mapping explícito en el template; sin campos dinámicos inesperados           | `GET proyecto-logs-prod/_mapping`                                            |            |
| Ciclo de vida    | Política ILM asociada y sin errores                                          | `GET proyecto-logs-prod/_ilm/explain`; `GET */_ilm/explain?only_errors=true` |            |
| Dimensionamiento | Shards entre 10 y 50 GB en producción; cálculo documentado                   | Parte A                                                                      |            |
| Ingesta          | Credencial de ingesta con permisos mínimos y caducidad                       | `GET _security/api_key?active_only=true`                                     |            |
| Ingesta          | Respuestas `_bulk` revisadas; manejo de rechazados                           | Incidente 2                                                                  |            |
| Seguridad        | Usuario de solo lectura probado (200 y 403)                                  | Parte D                                                                      |            |
| Seguridad        | Aplicaciones sin usar el usuario `elastic`                                   | `GET _security/_authenticate` en cada aplicación                             |            |
| Backups          | Snapshot reciente y política SLM                                             | `GET _snapshot/found-snapshots/*?size=1&sort=start_time&order=desc`          |            |
| Backups          | Restore ensayado                                                             | Laboratorio 7                                                                |            |
| Monitoreo        | Dashboard operativo y alerta de errores                                      | Laboratorio 8                                                                |            |
| Operación        | Runbook de verificación diaria y de cambios de mapping                       | Laboratorio 9                                                                |            |
| Gobernanza       | Retención documentada y matriz de accesos                                    | Laboratorio 7 y Parte B                                                      |            |
| Diagnóstico      | Plan de mitigación documentado                                               | Parte E                                                                      |            |
| Costos           | Retención y réplicas justificadas                                            | Parte A                                                                      |            |

### Demostración

Prepara una demostración de 2 minutos con: el diseño elegido y su dimensionamiento, el data stream en Discover o en el dashboard, el resultado de un incidente y los controles que no cumples y cómo los cerrarías.

### Cierre

``` console
DELETE _security/api_key
{ "name": "proyecto-ingesta" }
```

Se conservan el data stream, el template, la política de ILM, el rol y el usuario de solo lectura. La prueba gratuita dura 14 días; al vencer, el deployment se suspende. Antes de que termine, exporta lo que quieras conservar: templates, pipelines y políticas (con `GET`), y los dashboards desde Kibana en **Saved Objects**.

## Anexo opcional: búsqueda semántica y vectorial (demostración, 10 a 15 min)

Solo demostración del instructor, fuera de la evaluación. Antes de la clase hay que verificar en la documentación oficial qué incluye por defecto la versión usada y si el deployment de la prueba tiene disponible un endpoint de inferencia.

### Paso X1. Endpoints de inferencia disponibles

``` console
GET _inference/_all
```

**Qué hace:** lista los endpoints de inferencia del clúster. Cada uno identifica un modelo y el servicio que lo ejecuta (por ejemplo, ELSER en nodos de machine learning o en el Elastic Inference Service). **Resultado esperado:** al menos un endpoint de embeddings dispersos; si la lista está vacía o el modelo no puede desplegarse en la prueba, se muestra la demostración con capturas.

### Paso X2. Índice con semantic_text

``` console
PUT semantic-demo
{
  "mappings": {
    "properties": {
      "texto": { "type": "semantic_text" }
    }
  }
}

POST semantic-demo/_bulk?refresh=wait_for
{ "index": { "_id": "1" } }
{ "texto": "Tips to speed up a laptop: close unused apps and add more memory." }
{ "index": { "_id": "2" } }
{ "texto": "How to bake sourdough bread at home." }
{ "index": { "_id": "3" } }
{ "texto": "Cluster performance: reducing query latency with filters and caching." }
{ "index": { "_id": "4" } }
{ "texto": "Resetting your account password step by step." }
```

**Qué hace:** crea un índice con un campo `semantic_text`. Al indexar, Elasticsearch genera los embeddings con el endpoint de inferencia por defecto; no hace falta definir un pipeline.

- `semantic_text`: campo que gestiona la inferencia y el almacenamiento de los vectores.

- Sin `inference_id`, se usa el endpoint por defecto de la versión y el deployment. En producción se fija de forma explícita para que todos los índices usen el mismo modelo.

- ELSER es un modelo en inglés; por eso los textos están en inglés. Para español se usa un modelo multilingüe, con otro `inference_id`.

### Paso X3. Consulta semántica

``` console
GET semantic-demo/_search
{
  "query": {
    "match": { "texto": "my computer is very slow" }
  }
}
```

**Qué hace:** busca por significado. La consulta no comparte palabras con el documento 1, pero el modelo relaciona "computer is slow" con "speed up a laptop". **Resultado esperado:** el documento 1 en primer lugar, seguido probablemente del 3. El orden depende del modelo.

Comparación: `match` sobre un campo `text` normal no encontraría el documento 1 con esa consulta, porque no hay términos en común. Las aplicaciones reales combinan ambas técnicas (búsqueda híbrida: léxica y semántica, fusionadas, por ejemplo, con reciprocal rank fusion).

``` console
DELETE semantic-demo
```

## Resumen de valores esperados

| **Elemento**                                             | **Resultado**                                                                                           |
|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Almacenamiento total a 30 días                           | 40 × 30 × 2 × 1.1 = 2640 GB                                                                             |
| Capacidad a aprovisionar                                 | 2640 / 0.80 = 3300 GB                                                                                   |
| Hot (7 días) y warm (23 días)                            | 616 GB y 2024 GB                                                                                        |
| Índices a 30 días con rollover a 50 GB (1 primario)      | Alrededor de 24 (1200 GB / 50 GB)                                                                       |
| Shards con réplica: rollover por tamaño / diario         | Alrededor de 48 / 60                                                                                    |
| Por qué `delete.min_age` es 23d                          | Cuenta desde el rollover: 7d de escritura en hot más 23d = 30d. Con 30d, un dato podría vivir hasta 37d |
| Carga histórica                                          | created 4977                                                                                            |
| Total del data stream tras la carga nueva                | 5477                                                                                                    |
| Por servicio                                             | payments 827, checkout 1661, search 1027, catalog 1962                                                  |
| Errores 5xx                                              | 111                                                                                                     |
| Por día (UTC)                                            | 12 oct: 1708, 13 oct: 1621, 14 oct: 1648, 15 oct: 500                                                   |
| Usuario de solo lectura: lectura / escritura / `weblogs` | 200 / 403 / 403                                                                                         |
| Incidente 1                                              | 403 `cluster_block_exception`, bloqueo `index write`                                                    |
| Incidente 2                                              | 200 con `errors: true`; fallo 400 por tipo de `status`                                                  |

Nota sobre el índice con rollover por tamaño: el cálculo supone que el tráfico es constante. La estimación de índices (24) cambia si el volumen real difiere del estimado, por eso se revisa con el consumo medido.

## Rúbrica de evaluación

| **Criterio**                | **Qué se espera**                                                                                                          | **Puntos** |
|-----------------------------|----------------------------------------------------------------------------------------------------------------------------|------------|
| A. Dimensionamiento         | Cálculos correctos y justificación del rollover por tamaño                                                                 | 15         |
| B. Diseño                   | Política ILM coherente con la retención; template compuesto; data stream verificado con `_simulate_index` y `_ilm/explain` | 25         |
| C. Carga                    | Histórico y nuevos cargados, con totales verificados por servicio, errores y día                                           | 15         |
| D. Acceso                   | Rol y usuario de solo lectura probados, con 200 y 403                                                                      | 10         |
| E. Diagnóstico              | Dos incidentes con síntoma, evidencia, causa, mitigación y verificación; plan de mitigación completo                       | 25         |
| F. Checklist y demostración | Controles con evidencia; demostración clara de 2 minutos                                                                   | 10         |
| Total                       |                                                                                                                            | 100        |

## Preguntas de discusión

- ¿Qué decisión de diseño cambiarías si el volumen fuera 20 veces mayor? ¿Y si hubiera 2000 tenants pequeños?

- ¿Qué parte del checklist fue la más difícil de comprobar y qué automatizarías primero?

- ¿Qué incidente costó más diagnosticar y qué señal lo habría detectado antes?

- ¿Qué runbook añadirías antes de poner esta plataforma en producción?

- ¿Qué dejarías fuera de la primera versión productiva y por qué?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 10: Proyecto final de integración y demostración](imagenes/lab10_resultado_final.png)
