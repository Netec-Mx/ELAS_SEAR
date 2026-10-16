# Laboratorio 9: Simular una migración y ejecutar un runbook

- **Capítulo:** 9. Mantenimiento, migraciones y mejores prácticas operativas

- **Duración:** 54 min

- **Entorno:** Elastic Cloud Hosted (cuenta propia), Elasticsearch 9.x, Kibana Dev Tools y una terminal

- **Dependencias:** Laboratorio 3 (`weblogs-v1`, alias `weblogs` y templates), Laboratorio 4 (`weblogs-pipeline`, 5006 documentos). Recomendado: Laboratorio 8 (dashboard y data view `weblogs`).

## Objetivo

Simular una migración sin tiempo de inactividad dentro de un clúster 9.x: un cambio de mapping que no se puede aplicar en el lugar se resuelve con un índice nuevo, `_reindex` y un cambio atómico de alias. Después se ejecuta un runbook de verificación diaria, se atiende un incidente inducido y se automatiza la comprobación con un script. Al terminar, el participante podrá:

- Revisar la preparación previa a un cambio: versión, salud, versión de creación de los índices, deprecaciones y snapshot.

- Reconocer qué cambios de mapping exigen un reindex y cuáles no.

- Migrar con `_reindex` asíncrono, completar el delta con una segunda pasada y cambiar el alias de forma atómica, con ensayo de rollback.

- Ejecutar un runbook con pasos, resultado esperado y acción si falla.

- Automatizar una comprobación de salud con un script y un código de salida.

Alcance: la ruta de actualización desde 7.17 se estudia de forma conceptual en el capítulo. Este laboratorio no usa 7.17; el mismo patrón de reindex y alias se aplica a índices antiguos que deben migrarse antes de pasar a 9.x.

## Requisitos

- `GET _alias/weblogs` muestra `weblogs-v1`, y `GET weblogs/_count` devuelve 5006.

- Existen `weblogs-settings`, `weblogs-mappings`, `weblogs-template` y el pipeline `weblogs-pipeline`.

- Una terminal con `bash` (Linux, macOS o WSL) o con PowerShell (Windows) para la Parte E.

- La dirección de Elasticsearch (`<ES_URL>`) anotada en la ficha del deployment.

## Distribución del tiempo

| **Parte** | **Contenido**                                               | **Tiempo** |
|-----------|-------------------------------------------------------------|------------|
| Parte A   | Revisión previa: versión, índices, deprecaciones y snapshot | 8 min      |
| Parte B   | El cambio que no se puede hacer en el lugar                 | 6 min      |
| Parte C   | Migración con reindex y cambio de alias                     | 16 min     |
| Parte D   | Runbook de verificación diaria e incidente inducido         | 14 min     |
| Parte E   | Automatización con un script                                | 6 min      |
| Parte F   | Documentar el runbook y limpiar                             | 4 min      |

## Convenciones

- Los bloques con `GET`, `PUT`, `POST` o `DELETE` se ejecutan en Dev Tools. Los de la Parte E se ejecutan en la terminal.

- Cada paso importante indica el resultado esperado. Si no coincide, no se continúa: se aplica el criterio de un runbook (detenerse y diagnosticar).

- Los nombres `weblogs-v1` y `weblogs-v2` son versiones físicas; las aplicaciones usan siempre el alias `weblogs`.

## Parte A: revisión previa (8 min)

### Paso A1. Versión, salud e índices

``` console
GET /?filter_path=version.number,version.lucene_version

GET _cluster/health?filter_path=status,number_of_nodes,unassigned_shards

GET _cat/indices/weblogs*?v&h=health,index,pri,rep,docs.count,store.size

GET _alias/weblogs

GET weblogs-v1/_settings?filter_path=*.settings.index.version.created,*.settings.index.creation_date
```

**Qué hace:** recoge el estado de partida: versión del clúster, salud, índices del curso, a qué índice apunta el alias y con qué versión se creó `weblogs-v1`.

- `version.number`: versión de Elasticsearch (9.x). Anótala.

- `unassigned_shards`: debe ser 0. Un clúster que no está sano no se toca.

- `index.version.created`: identificador numérico de la versión con la que se creó el índice. Los dígitos son mayor, menor y parche; por ejemplo, `9000099` es 9.0.0. Un índice creado en 7.x no es compatible con 9.x y debe reindexarse, archivarse o eliminarse antes de actualizar.

- `creation_date`: milisegundos desde 1970 (hora UTC).

**Resultado esperado:** `status` `green`, el alias `weblogs` apuntando a `weblogs-v1` con 5006 documentos, y un `version.created` que empieza con 9.

### Paso A2. Deprecaciones y Upgrade Assistant

``` console
GET _migration/deprecations

GET _migration/system_features
```

**Qué hace:** la primera petición devuelve el informe de funciones obsoletas que usa el Upgrade Assistant: settings de clúster y de nodo, settings de índices, templates, entre otros. La segunda indica si los índices de sistema (Kibana, Fleet, seguridad) están listos para migrar.

- `migration_status`: estado de cada característica del sistema. `NO_MIGRATION_NEEDED` es lo normal.

- Cada deprecación incluye `level` (`warning` o `critical`), el mensaje, una URL a la documentación y los detalles.

- Un deployment recién creado en 9.x suele devolver poco o nada. Lo importante del paso es saber leer el informe antes de un upgrade real: lo `critical` bloquea la actualización.

1.  En Kibana, abrir **Stack Management** y luego **Upgrade Assistant**.

2.  Revisar el resumen y las secciones de deprecaciones de Elasticsearch y de Kibana.

3.  Anotar el estado: cuántos problemas críticos y cuántas advertencias hay.

### Paso A3. Snapshot previo al cambio

``` console
PUT _snapshot/found-snapshots/lab09-pre-migracion?wait_for_completion=true
{
  "indices": ["weblogs-v1"],
  "include_global_state": false,
  "metadata": { "motivo": "antes de cambiar el mapping de response_time_ms" }
}
```

**Qué hace:** toma un snapshot manual de `weblogs-v1`. Es la red de seguridad de cualquier cambio: si algo sale mal, se puede restaurar (laboratorio 7). **Resultado esperado:** `"state": "SUCCESS"`.

## Parte B: el cambio que no se puede hacer en el lugar (6 min)

Situación: el equipo de plataforma quiere reducir el almacenamiento de `response_time_ms`. Los datos ya vienen con una sola cifra decimal, por lo que un `scaled_float` con factor 10 (precisión de 0,1 ms) guarda la misma información como un entero y evita las imprecisiones de `float` (un valor como 189.8 se guarda en `float` como 189.80000305…).

### Paso B1. Intentar el cambio en el lugar

``` console
PUT weblogs-v1/_mapping
{
  "properties": {
    "response_time_ms": { "type": "scaled_float", "scaling_factor": 10 }
  }
}
```

**Qué hace:** intenta cambiar el tipo del campo en el índice existente.

**Resultado esperado:** error 400 de tipo `illegal_argument_exception`, con un mensaje equivalente a `mapper [response_time_ms] cannot be changed from type [float] to [scaled_float]`. Esto es una restricción de Lucene: los segmentos ya escritos usan el tipo anterior.

### Paso B2. Qué se puede cambiar sin reindexar

| **Cambio en un mapping existente**                                         | **Se puede en el lugar**                                                                                               |
|----------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Agregar un campo nuevo                                                     | Sí                                                                                                                     |
| Agregar un multi-field a un campo existente                                | Sí (afecta solo a los documentos nuevos o reindexados)                                                                 |
| Cambiar `ignore_above` de un `keyword`                                     | Sí                                                                                                                     |
| Cambiar el tipo de un campo (`float` a `scaled_float`, `text` a `keyword`) | No                                                                                                                     |
| Cambiar el `analyzer` de un campo `text`                                   | No                                                                                                                     |
| Cambiar parámetros estructurales (`doc_values`, `index`, `scaling_factor`) | No: se reindexa. Algunos parámetros pueden aceptarse; ante la duda, se comprueba con `_mapping` en un índice de prueba |

Regla de diseño: cuando el cambio no se puede hacer en el lugar, el camino es crear un índice nuevo con el mapping correcto, copiar los datos y mover el alias. Es el patrón del resto del laboratorio.

## Parte C: migración con reindex y cambio de alias (16 min)

### Paso C1. Actualizar el component template

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
        "response_time_ms": { "type": "scaled_float", "scaling_factor": 10 },
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

**Qué hace:** reemplaza el component template con el mapping nuevo. Solo cambia `response_time_ms`. Un component template se reemplaza completo, por eso se envía el mapping entero.

- `scaling_factor: 10`: el valor se guarda como un entero multiplicado por 10 (189.8 se guarda como 1898). Con un factor de 10 se conserva una cifra decimal.

- El cambio no afecta a `weblogs-v1`. Solo lo recibirán los índices que se creen a partir de ahora con el patrón `weblogs-v*`.

### Paso C2. Crear el índice nuevo y comprobar su mapping

``` console
PUT weblogs-v2

GET weblogs-v2/_mapping/field/response_time_ms

GET weblogs-v2/_settings?filter_path=*.settings.index.number_of_shards,*.settings.index.number_of_replicas,*.settings.index.default_pipeline
```

**Qué hace:** crea `weblogs-v2` sin cuerpo (el template aporta settings y mappings) y lo verifica.

**Resultado esperado:** `response_time_ms` de tipo `scaled_float` con `scaling_factor` 10; 1 shard y 1 réplica; **sin** `default_pipeline`. El pipeline se activa más adelante: al copiar documentos que ya fueron procesados no se deben volver a procesar.

### Paso C3. Primera pasada de reindex (asíncrona)

``` console
POST _reindex?wait_for_completion=false
{
  "source": { "index": "weblogs-v1" },
  "dest":   { "index": "weblogs-v2", "op_type": "create" }
}
```

**Qué hace:** inicia el copiado y devuelve de inmediato el identificador de una tarea, sin esperar a que termine.

- `wait_for_completion=false`: la operación corre en segundo plano. Es la forma segura para índices grandes, porque un cliente con timeout no interrumpe la migración.

- `op_type: create`: solo crea documentos que no existen en el destino. Hace la operación repetible: una segunda ejecución no sobrescribe ni duplica.

- Los `_id` de los documentos se conservan.

Copia el valor de `task` (con el formato `nodo:número`) y consúltalo:

``` console
GET _tasks/<ID_DE_LA_TAREA>

POST weblogs-v2/_refresh

GET weblogs-v2/_count
```

**Qué hace:** consulta el avance de la tarea, hace visibles los documentos copiados y los cuenta.

- `completed`: `true` cuando termina. En la respuesta, `response` incluye `total`, `created`, `version_conflicts` y `failures`.

- Para operaciones largas: `requests_per_second` limita la velocidad para no competir con las búsquedas, y `slices` reparte el trabajo en paralelo. Una tarea en curso se cancela con `POST _tasks/<id>/_cancel`.

**Resultado esperado:** `created: 5006`, `failures: []` y `count` de 5006.

### Paso C4. Simular que llegan datos durante la migración

``` console
PUT weblogs/_doc/late-1?refresh=wait_for
{
  "@timestamp": "2026-10-14T12:00:00Z",
  "service": "search",
  "client_ip": "187.10.20.30",
  "method": "GET",
  "url": "/api/search?q=teclado",
  "status": 200,
  "bytes": 410,
  "response_time_ms": 120.5,
  "message": "GET /api/search?q=teclado 200 120.5ms ok"
}
```

**Qué hace:** indexa un documento por el alias `weblogs`, que todavía apunta a `weblogs-v1`. En una migración real, la aplicación sigue escribiendo mientras se copia. Ese documento no está en `weblogs-v2`.

**Resultado esperado:** `result: created`. El pipeline por defecto de `weblogs-v1` añade `team` y `tier`.

### Paso C5. Segunda pasada: copiar solo el delta

``` console
POST _reindex?refresh=true
{
  "conflicts": "proceed",
  "source": { "index": "weblogs-v1" },
  "dest":   { "index": "weblogs-v2", "op_type": "create" }
}
```

**Qué hace:** repite el reindex. Los 5006 documentos que ya existen en el destino producen conflictos y se omiten; solo se crea el nuevo.

- `conflicts: proceed`: ante un conflicto de versión, la operación continúa en lugar de abortar. Los conflictos se cuentan en `version_conflicts`.

- `refresh=true`: refresca el destino al terminar.

**Resultado esperado:** `created: 1` y `version_conflicts: 5006`.

### Paso C6. Validar antes del cambio

``` console
GET weblogs-v1,weblogs-v2/_search?size=0
{
  "aggs": {
    "por_indice": {
      "terms": { "field": "_index" },
      "aggs": {
        "rt_prom": { "avg":   { "field": "response_time_ms" } },
        "rt_max":  { "max":   { "field": "response_time_ms" } },
        "servicios": { "terms": { "field": "service" } }
      }
    }
  }
}
```

**Qué hace:** compara los dos índices con la misma agregación, separados por el campo especial `_index`.

**Resultado esperado:** ambos buckets con `doc_count` de 5007, los mismos conteos por servicio (payments 765, checkout 1499, search 931, catalog 1782) y promedio y máximo del tiempo de respuesta iguales hasta el primer decimal. En `weblogs-v1` pueden verse cifras como `3199.89990234375`, que es la imprecisión de `float`; en `weblogs-v2` el mismo valor aparece como `3199.9`.

### Paso C7. Activar el pipeline en el índice nuevo

``` console
PUT weblogs-v2/_settings
{
  "index.default_pipeline": "weblogs-pipeline"
}
```

**Qué hace:** a partir de ahora, todo documento nuevo que llegue a `weblogs-v2` pasa por `weblogs-pipeline`, igual que en `weblogs-v1`. Se hace después del copiado para no reprocesar documentos migrados.

### Paso C8. Cambio atómico del alias

``` console
POST _aliases
{
  "actions": [
    { "remove": { "index": "weblogs-v1", "alias": "weblogs" } },
    { "add":    { "index": "weblogs-v2", "alias": "weblogs" } }
  ]
}

GET _alias/weblogs

GET weblogs/_count
```

**Qué hace:** quita el alias de un índice y lo agrega al otro en una sola operación atómica: ningún cliente ve un instante sin alias ni con los dos.

- `actions`: lista de operaciones que se aplican juntas o no se aplica ninguna.

- Las aplicaciones, el data view `weblogs` y el dashboard del laboratorio 8 siguen funcionando sin cambios.

**Resultado esperado:** el alias apunta a `weblogs-v2` y `count` es 5007. Si abres el dashboard `Operación weblogs`, la métrica de peticiones muestra ahora **4977** (los 4976 de antes más el documento nuevo, que tiene `@timestamp`).

### Paso C9. Ensayo de rollback

``` console
POST _aliases
{
  "actions": [
    { "remove": { "index": "weblogs-v2", "alias": "weblogs" } },
    { "add":    { "index": "weblogs-v1", "alias": "weblogs" } }
  ]
}

GET weblogs/_search?size=1&filter_path=hits.hits._index

POST _aliases
{
  "actions": [
    { "remove": { "index": "weblogs-v1", "alias": "weblogs" } },
    { "add":    { "index": "weblogs-v2", "alias": "weblogs" } }
  ]
}

GET weblogs/_search?size=1&filter_path=hits.hits._index
```

**Qué hace:** vuelve al índice anterior, comprueba que las búsquedas llegan a `weblogs-v1`, y regresa a `weblogs-v2`. El rollback de un cambio de alias es una operación de un paso, y por eso el índice anterior se conserva durante una ventana de observación.

**Resultado esperado:** primero `_index` igual a `weblogs-v1`, después `weblogs-v2`. Deja el alias en `weblogs-v2`.

### Paso C10. Proteger el índice anterior

``` console
PUT weblogs-v1/_settings
{
  "index.blocks.write": true
}
```

**Qué hace:** bloquea las escrituras en `weblogs-v1`. Se conserva para un posible rollback, pero ya no debe recibir datos que `weblogs-v2` no tenga. En producción se elimina después de la ventana de observación (días o semanas); en el laboratorio se elimina en la Parte F.

## Parte D: runbook de verificación diaria e incidente inducido (14 min)

### Paso D1. Ejecutar el runbook de verificación

Ejecuta cada paso en orden. Si el resultado no coincide, se aplica la columna "Si falla".

| **N.º** | **Comando**                                                                                                      | **Esperado**                             | **Si falla**                                                       |
|---------|------------------------------------------------------------------------------------------------------------------|------------------------------------------|--------------------------------------------------------------------|
| 1       | `GET _cluster/health?filter_path=status,number_of_nodes,unassigned_shards,initializing_shards,relocating_shards` | `green`, shards sin asignar 0            | `yellow` o `red`: ir al paso de diagnóstico (`allocation/explain`) |
| 2       | `GET _cat/nodes?v&h=name,heap.percent,cpu,disk.used_percent`                                                     | Heap bajo 75 %, disco bajo 80 %          | Escalar o liberar disco; revisar el capítulo 6                     |
| 3       | `GET _cat/indices?v&health=yellow` y luego `health=red`                                                          | Solo encabezado (sin índices)            | Diagnosticar cada índice con `allocation/explain`                  |
| 4       | `GET _cat/thread_pool/write,search?v&h=node_name,name,rejected`                                                  | `rejected` sin crecer entre lecturas     | Revisar carga de indexación o consultas costosas                   |
| 5       | `GET */_ilm/explain?only_errors=true`                                                                            | `"indices": {}`                          | Corregir la causa y `POST <índice>/_ilm/retry`                     |
| 6       | `GET _slm/stats?filter_path=total_snapshots_taken,total_snapshots_failed`                                        | `total_snapshots_failed` sin crecer      | Revisar `last_failure` de la política                              |
| 7       | `GET _security/api_key?active_only=true&filter_path=api_keys.name,api_keys.expiration`                           | Ninguna clave a punto de vencer sin plan | Rotar la clave                                                     |
| 8       | `GET _license?filter_path=license.status,license.expiry_date_in_millis`                                          | `active`, con fecha lejana               | Renovar o contactar a Elastic                                      |

Los pasos del 1 al 8 se ejecutan en Dev Tools, uno por uno (el comando del paso 3 son dos peticiones). Anota los resultados. **Resultado esperado:** todo en orden, con `status` `green`.

- `health=yellow` o `health=red` en `_cat/indices`: filtra solo los índices con ese estado.

- `*/_ilm/explain?only_errors=true`: muestra solo los índices con un error en su política de ILM.

- `_slm/stats`: contadores de snapshots tomados y fallidos por las políticas SLM.

- `api_key?active_only=true`: lista las claves vigentes con su caducidad.

### Paso D2. Provocar el incidente

``` console
PUT runbook-test
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 5
  }
}
```

**Qué hace:** crea un índice que pide 5 réplicas. El clúster no tiene tantos nodos de datos, de modo que no podrá asignarlas.

### Paso D3. Diagnosticar con el runbook

``` console
GET _cluster/health?filter_path=status,unassigned_shards

GET _cat/indices?v&health=yellow

GET _cluster/allocation/explain
{
  "index": "runbook-test",
  "shard": 0,
  "primary": false
}
```

**Qué hace:** repite los pasos 1 y 3 del runbook y pregunta al clúster por qué una réplica no se asigna.

- `status`: ahora `yellow`, con varios shards sin asignar.

- `_cat/indices`: aparece `runbook-test` como `yellow` con `rep` igual a 5.

- `allocation/explain`: `can_allocate` en `no` y, en `node_allocation_decisions`, el decider `same_shard`: una copia de ese shard ya está en el nodo, y no puede haber dos copias en el mismo nodo.

**Causa:** el número de réplicas debe ser, como máximo, el de nodos de datos menos 1.

### Paso D4. Resolver y verificar

``` console
PUT runbook-test/_settings
{
  "index.number_of_replicas": 1
}

GET _cluster/health?wait_for_status=green&timeout=60s&filter_path=status
```

**Qué hace:** ajusta las réplicas a un valor que el clúster puede asignar y espera el estado `green`. **Resultado esperado:** `"status": "green"`. Un incidente no termina hasta verificar que el síntoma desapareció.

## Parte E: automatización con un script (6 min)

### Paso E1. Crear una API key de solo monitoreo

``` console
POST _security/api_key
{
  "name": "lab09-monitor",
  "expiration": "1d",
  "role_descriptors": {
    "solo-monitoreo": { "cluster": ["monitor"] }
  }
}
```

**Qué hace:** crea una credencial con el privilegio `monitor`, que permite leer el estado del clúster y nada más. Un script de verificación no necesita permisos de escritura. Copia el valor de `encoded`.

### Paso E2. El script

Guarda el contenido que corresponda a tu sistema en un archivo.

**bash** (archivo `salud.sh`):

``` console
#!/usr/bin/env bash
ES_URL="$1"
API_KEY="$2"
estado=$(curl -s -H "Authorization: ApiKey $API_KEY" "$ES_URL/_cluster/health?filter_path=status" | sed -E 's/.*"status":"([a-z]+)".*/\1/')
echo "Estado del clúster: $estado"
[ "$estado" = "green" ] || exit 1
```

**PowerShell** (archivo `salud.ps1`):

``` console
param($EsUrl, $ApiKey)
$salud = Invoke-RestMethod -Uri "$EsUrl/_cluster/health?filter_path=status" -Headers @{ Authorization = "ApiKey $ApiKey" }
Write-Output "Estado del clúster: $($salud.status)"
if ($salud.status -ne "green") { exit 1 }
```

**Qué hace:** consulta el estado del clúster con la API key, lo muestra y termina con código de salida 0 si es `green` y con 1 en cualquier otro caso.

- `filter_path=status`: pide solo el campo `status`.

- El código de salida es lo que usan los programadores de tareas (cron, CI/CD) para decidir si una ejecución falló y avisar.

- `$1` y `$2` (bash) o `-EsUrl` y `-ApiKey` (PowerShell): parámetros, para que el script no contenga secretos.

### Paso E3. Ejecutar con el clúster sano y con una falla

``` console
bash salud.sh "<ES_URL>" "<ENCODED_KEY>" ; echo "Código de salida: $?"

./salud.ps1 -EsUrl "<ES_URL>" -ApiKey "<ENCODED_KEY>" ; echo "Código de salida: $LASTEXITCODE"
```

Ejecuta el comando de tu sistema. **Resultado esperado:** `Estado del clúster: green` y código de salida 0. Ahora provoca la falla otra vez:

``` console
PUT runbook-test/_settings
{
  "index.number_of_replicas": 5
}
```

Espera unos segundos, vuelve a ejecutar el script. **Resultado esperado:** `Estado del clúster: yellow` y código de salida 1. Restaura el valor y confirma el regreso al estado `green`:

``` console
PUT runbook-test/_settings
{
  "index.number_of_replicas": 1
}
```

## Parte F: documentar el runbook y limpiar (4 min)

### Paso F1. Runbook del cambio de mapping

Completa el siguiente runbook con los comandos que ejecutaste. Debe poder ejecutarlo otra persona sin ayuda.

| **Sección**                     | **Contenido**                                                                  |
|---------------------------------|--------------------------------------------------------------------------------|
| Nombre y propósito              | Cambio de mapping incompatible con reindex y alias                             |
| Cuándo usarlo                   |                                                                                |
| Prerrequisitos                  | Clúster `green`; snapshot reciente; espacio en disco para una copia del índice |
| Pasos                           | Numerados, con comando y resultado esperado en cada uno (partes A a C)         |
| Verificación                    |                                                                                |
| Rollback                        |                                                                                |
| Escalamiento                    | A quién se avisa y cuándo                                                      |
| Responsable y fecha de revisión |                                                                                |

### Paso F2. Limpieza

``` console
DELETE weblogs-v1

DELETE runbook-test

DELETE _snapshot/found-snapshots/lab09-pre-migracion

DELETE _security/api_key
{ "name": "lab09-monitor" }
```

**Qué hace:** elimina el índice anterior (ya terminó la ventana de observación del laboratorio), el índice de prueba, el snapshot previo y la API key.

- Con `weblogs-v1` eliminado, el alias `weblogs` apunta solo a `weblogs-v2` y sigue funcionando.

- Antes de borrar un índice en producción, se confirma que el nuevo funciona y que el snapshot es restaurable.

- Elimina también los archivos `salud.sh` o `salud.ps1` si contienen la clave.

## Resumen de valores esperados

| **Comprobación**                                    | **Resultado**                                         |
|-----------------------------------------------------|-------------------------------------------------------|
| `version.created` de `weblogs-v1`                   | Empieza con 9                                         |
| Cambio de tipo en el lugar (B1)                     | Error 400 `illegal_argument_exception`                |
| Reindex, primera pasada                             | created 5006                                          |
| Reindex, segunda pasada                             | created 1, version_conflicts 5006                     |
| `_count` de `weblogs-v1` y `weblogs-v2`             | 5007 y 5007                                           |
| Conteos por servicio                                | payments 765, checkout 1499, search 931, catalog 1782 |
| Alias `weblogs` tras el cambio                      | `weblogs-v2`                                          |
| Panel de peticiones del dashboard del laboratorio 8 | 4977                                                  |
| Incidente con 5 réplicas                            | `yellow`; decider `same_shard`                        |
| Script con el clúster sano / con la falla           | Código de salida 0 / 1                                |

## Estado final

| **Elemento**                                                          | **Se conserva**     | **Lo reutiliza**                                    |
|-----------------------------------------------------------------------|---------------------|-----------------------------------------------------|
| `weblogs-v2` con 5007 documentos y alias `weblogs`                    | Sí                  | Laboratorio 10 (carga histórica del proyecto final) |
| `weblogs-mappings` con `scaled_float`                                 | Sí                  | Laboratorio 10 (template del data stream)           |
| `weblogs-v1`, `runbook-test`, snapshot previo y clave `lab09-monitor` | No (se eliminan)    | \-                                                  |
| Runbook documentado (F1)                                              | Sí (en tu cuaderno) | Laboratorio 10 (checklist de producción)            |

## Preguntas de discusión

- ¿Por qué el índice nuevo se crea sin `default_pipeline` y se activa después de copiar los datos?

- ¿Qué habría pasado si en el paso C5 se hubiera usado `op_type` distinto de `create` y sin `conflicts: proceed`?

- ¿Por qué el cambio de alias con `_aliases` es más seguro que borrar un índice y crear otro con el mismo nombre?

- ¿Qué cambia en esta migración si el índice tuviera 2 TB y recibiera 20 000 documentos por segundo?

- ¿Qué paso añadirías al runbook diario para detectar que una fuente dejó de enviar datos?

- Un índice de `weblogs-v1` creado en 7.x, ¿qué opciones tendría antes de pasar a 9.x?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 9: Simular una migración y ejecutar un runbook](imagenes/lab09_resultado_final.png)
