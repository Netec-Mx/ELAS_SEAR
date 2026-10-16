# Laboratorio 6: Benchmark y tuning de un clúster de logs

- **Capítulo:** 6. Rendimiento, escalado y tuning operacional

- **Duración:** 54 min

- **Entorno:** Elastic Cloud Hosted (cuenta propia), Elasticsearch 9.x, Kibana Dev Tools y una terminal con `curl`

- **Dependencias:** Laboratorio 3 (component templates `weblogs-settings` y `weblogs-mappings`) y Laboratorio 4 (`weblogs` con 5006 documentos).

## Objetivo

Medir una carga de indexación y otra de búsqueda sobre el clúster del curso, aplicar ajustes de tuning y comparar los resultados con una línea base. Al terminar, el participante podrá:

- Leer el estado de un clúster con `_cluster/health`, `_cat` y `_nodes/stats`, incluidos los rechazos de los thread pools.

- Cargar un dataset de 30 000 documentos con la Bulk API y medir el tiempo de la carga.

- Comparar variantes de indexación (refresh, réplicas y tamaño de lote) y restaurar los valores de producción.

- Medir el costo de consultas y agregaciones con `took`, la caché de peticiones y `profile`.

- Observar el efecto de los segmentos y de `_forcemerge` en el tamaño del índice.

## Requisitos

- `GET weblogs/_count` devuelve 5006 y existen `weblogs-settings` y `weblogs-mappings` (`GET _component_template/weblogs-*`).

- Archivo `carga_logs.ndjson` (30 000 documentos, 11 MB) descargado en una carpeta local; los comandos `curl` se ejecutan desde esa carpeta.

- La dirección de Elasticsearch (`<ES_URL>`) anotada en la ficha del deployment.

## Distribución del tiempo

| **Parte** | **Contenido**                                     | **Tiempo** |
|-----------|---------------------------------------------------|------------|
| Parte A   | Línea base del clúster                            | 9 min      |
| Parte B   | Preparar el entorno de pruebas y cargar los datos | 8 min      |
| Parte C   | Experimentos de indexación                        | 14 min     |
| Parte D   | Experimentos de búsqueda                          | 11 min     |
| Parte E   | Segmentos y merge                                 | 8 min      |
| Parte F   | Conclusiones y limpieza                           | 4 min      |

## Convenciones y alcance de las mediciones

- Los bloques con `GET`, `PUT`, `POST` o `DELETE` se ejecutan en Dev Tools; los que empiezan con `curl` se ejecutan en la terminal. En Windows se usa `curl.exe`.

- Los tiempos dependen del hardware, de la región y de la carga del momento. **Este documento no fija tiempos esperados**: cada participante compara sus propias mediciones.

- El dataset es pequeño (30 000 documentos) y los nodos de la prueba son de 4 GB de RAM. En algunos experimentos la diferencia puede quedar dentro del ruido. Si pasa, ese es un resultado válido: significa que, a esta escala, el ajuste no justifica su riesgo. Para decidir hace falta repetir la prueba con volumen real, por ejemplo con Rally.

- Cada experimento se ejecuta una sola vez. Si tienes tiempo, repite uno y observa cuánto varía.

## Parte A: línea base del clúster (9 min)

### Paso A1. Salud y distribución

``` console
GET _cluster/health

GET _cat/nodes?v&h=name,node.role,heap.percent,ram.percent,cpu,load_1m,disk.used_percent

GET _cat/allocation?v
```

**Qué hace:** la primera petición resume el estado del clúster. La segunda lista los nodos con su uso de recursos. La tercera muestra cuántos shards y cuánto disco tiene cada nodo.

- `status`: `green` (todos los shards asignados), `yellow` (faltan réplicas) o `red` (falta algún primario).

- `unassigned_shards`, `initializing_shards`, `relocating_shards`: deben ser 0 en un clúster en reposo.

- `_cat`: familia de APIs con salida de texto para personas. `v` agrega los encabezados y `h` elige las columnas.

- `node.role`: letras de rol (por ejemplo, `d` datos, `m` master, `i` ingest). En Elastic Cloud los roles de tier aparecen como `h` (hot), `w` (warm) o `c` (cold).

- `heap.percent`: uso del heap de la JVM. Un valor sostenido por encima de 75 % indica presión de memoria.

- `disk.used_percent`: porcentaje de disco usado. Los watermarks de disco (85 % bajo, 90 % alto, 95 % inundación) limitan la asignación de shards.

Anota en la tabla de resultados el número de nodos de datos, su `heap.percent` y el espacio libre.

### Paso A2. Índices y shards

``` console
GET _cat/indices?v&s=store.size:desc&h=health,index,pri,rep,docs.count,store.size

GET _cat/shards/weblogs-v1?v&h=index,shard,prirep,state,node,store
```

**Qué hace:** lista los índices ordenados por tamaño y el detalle de los shards de `weblogs-v1`.

- `s=store.size:desc`: ordena por la columna indicada, de mayor a menor.

- `prirep`: `p` primario, `r` réplica.

- `state`: `STARTED` es el estado normal.

### Paso A3. Explicar la asignación de un shard

``` console
GET _cluster/allocation/explain
{
  "index": "weblogs-v1",
  "shard": 0,
  "primary": false
}
```

**Qué hace:** explica por qué el shard indicado está donde está. Sin cuerpo, la API explica el primer shard sin asignar y devuelve error si no hay ninguno.

- `current_state`: estado del shard.

- `node_allocation_decisions`: decisión de cada nodo y el decider que la tomó (por ejemplo, `same_shard`, que impide poner primario y réplica en el mismo nodo).

- `can_remain_on_current_node` y `can_rebalance_cluster`: si el shard puede quedarse y si el clúster puede reubicarlo.

### Paso A4. Thread pools y presión de memoria

``` console
GET _cat/thread_pool/write,search?v&h=node_name,name,size,active,queue,queue_size,rejected,completed

GET _nodes/stats/jvm,os?filter_path=nodes.*.name,nodes.*.jvm.mem.heap_used_percent,nodes.*.os.cpu.percent
```

**Qué hace:** muestra el estado de los pools `write` (indexación) y `search` (búsqueda), y el uso de heap y CPU de cada nodo.

- `size`: hilos del pool. Depende de los procesadores asignados al nodo.

- `active`: hilos trabajando ahora. `queue` y `queue_size`: tareas en espera y capacidad de la cola.

- `rejected`: tareas rechazadas desde el arranque del nodo. Si crece, los clientes reciben respuestas 429. Debe estar en 0 al inicio; anótalo para compararlo al final.

- `filter_path`: devuelve solo las rutas indicadas del JSON; `*` sustituye el identificador del nodo.

### Paso A5. Monitoreo en la consola de Elastic Cloud

1.  En la consola de Elastic Cloud, abrir el deployment.

2.  Revisar las opciones de monitoreo que ofrece tu cuenta (por ejemplo, AutoOps o Logs and metrics). AutoOps está disponible en Elastic Cloud Hosted sin costo, aunque puede no aparecer en todas las regiones.

3.  Anotar qué métricas muestra (CPU, memoria, indexación, búsqueda) y en qué se parecen a las que acabas de consultar con las APIs.

## Parte B: preparar el entorno de pruebas y cargar los datos (8 min)

### Paso B1. Template para los índices de prueba

``` console
PUT _index_template/bench-template
{
  "index_patterns": ["bench-*"],
  "priority": 150,
  "composed_of": ["weblogs-settings", "weblogs-mappings"],
  "template": {
    "settings": { "number_of_replicas": 1 }
  }
}
```

**Qué hace:** todo índice cuyo nombre empiece con `bench-` recibe los mismos settings y mappings que `weblogs-v1`, con un primario y una réplica. Así los experimentos comparan solo lo que se cambia a propósito.

- `composed_of`: reutiliza los component templates del laboratorio 3.

- `template.settings`: en un index template, este bloque prevalece sobre los component templates.

### Paso B2. Crear la API key de carga

``` console
POST _security/api_key
{
  "name": "lab06-bench",
  "expiration": "1d",
  "role_descriptors": {
    "bench-writer": {
      "indices": [
        { "names": ["bench-*"], "privileges": ["write"] }
      ]
    }
  }
}
```

**Qué hace:** crea una credencial que solo puede escribir en índices `bench-*` y vence en un día. Copia el valor del campo `encoded` de la respuesta; se usa en el paso siguiente como `<ENCODED_KEY>`.

### Paso B3. Crear el índice base y cargar los datos

``` console
PUT bench-base
```

**Qué hace:** crea el índice sin cuerpo; los settings y mappings vienen de `bench-template`.

``` console
curl.exe -s -X POST "<ES_URL>/bench-base/_bulk?filter_path=took,errors" -H "Authorization: ApiKey <ENCODED_KEY>" -H "Content-Type: application/x-ndjson" --data-binary "@carga_logs.ndjson" -w "\nTiempo total del cliente: %{time_total} s\n"
```

**Qué hace:** envía los 30 000 documentos en una sola petición Bulk y mide el tiempo total.

- `/bench-base/_bulk`: el índice va en la ruta, por eso las líneas de acción del archivo son `{"index":{}}` sin nombre de índice. Los `_id` se generan automáticamente.

- `filter_path=took,errors`: reduce la respuesta a esos dos campos. Sin él, la respuesta lista 30 000 elementos.

- `took`: milisegundos que Elasticsearch dedicó a la petición.

- `-w`: imprime al final el tiempo total medido por `curl`, que incluye la red.

- `--data-binary`: envía el archivo sin modificar los saltos de línea; el formato NDJSON los necesita.

**Resultado esperado:** `"errors": false`. Anota `took` y el tiempo del cliente como **medición B**. Si la petición falla con 401, revisa la API key; si falla por tamaño, avisa al instructor.

``` console
GET bench-base/_count

GET _cat/indices/bench-base?v&h=index,health,pri,rep,docs.count,store.size
```

**Resultado esperado:** `count` de 30 000.

## Parte C: experimentos de indexación (14 min)

Cada experimento copia los 30 000 documentos de `bench-base` a un índice nuevo con `_reindex` y compara el campo `took` de la respuesta. La API lee y escribe dentro del clúster, de modo que mide el costo de indexar sin la variable de la red. Incluye también el costo de leer el origen, que es igual en todas las variantes.

| **Índice** | **Settings**                                  | **Tamaño de lote** | **Qué se prueba**            |
|------------|-----------------------------------------------|--------------------|------------------------------|
| `bench-e1` | Valores del template (refresh 1 s, 1 réplica) | 1000               | Línea base                   |
| `bench-e2` | refresh desactivado, 0 réplicas               | 1000               | Efecto de refresh y réplicas |
| `bench-e3` | Valores del template                          | 5000               | Efecto del tamaño de lote    |
| `bench-e4` | refresh desactivado, 0 réplicas               | 5000               | Combinación de ambos         |

### Paso C1. Línea base (e1)

``` console
POST _reindex
{
  "source": { "index": "bench-base", "size": 1000 },
  "dest":   { "index": "bench-e1" }
}
```

**Qué hace:** copia los documentos de `bench-base` a `bench-e1`, que se crea con los valores del template.

- `source.size`: documentos por lote; el valor por defecto es 1000.

- `took`: milisegundos totales. Anótalo en la tabla de resultados.

- `created`: documentos creados. Debe ser 30 000.

### Paso C2. Sin refresh y sin réplicas (e2)

``` console
PUT bench-e2
{
  "settings": {
    "index.refresh_interval": "-1",
    "index.number_of_replicas": 0
  }
}

POST _reindex
{
  "source": { "index": "bench-base", "size": 1000 },
  "dest":   { "index": "bench-e2" }
}
```

**Qué hace:** crea el índice con el refresh desactivado y sin réplicas, y repite la copia.

- `index.refresh_interval: -1`: no se generan segmentos nuevos de forma periódica; los documentos no son visibles para las búsquedas hasta el siguiente refresh. Cada refresh cuesta CPU y E/S.

- `index.number_of_replicas: 0`: se escribe una sola copia. Es más rápido y usa la mitad del disco, pero si el nodo falla durante este periodo, los datos del índice se pierden. Solo es aceptable para cargas que se pueden repetir.

``` console
GET bench-e2/_count
```

**Qué hace:** cuenta los documentos visibles. **Resultado esperado:** un valor menor a 30 000, normalmente 0, porque no ha habido refresh. Los documentos están indexados, pero no son visibles.

``` console
POST bench-e2/_refresh

GET bench-e2/_count
```

**Resultado esperado:** ahora la cuenta es 30 000.

### Paso C3. Lotes más grandes (e3)

``` console
POST _reindex
{
  "source": { "index": "bench-base", "size": 5000 },
  "dest":   { "index": "bench-e3" }
}
```

**Qué hace:** igual que la línea base, pero con lotes de 5000 documentos. Menos peticiones internas, cada una más grande. La guía de Elastic recomienda probar tamaños crecientes hasta que la velocidad deje de mejorar, sin superar unas decenas de MB por petición.

### Paso C4. Combinación (e4)

``` console
PUT bench-e4
{
  "settings": {
    "index.refresh_interval": "-1",
    "index.number_of_replicas": 0
  }
}

POST _reindex
{
  "source": { "index": "bench-base", "size": 5000 },
  "dest":   { "index": "bench-e4" }
}
```

### Paso C5. Restaurar los valores de producción

``` console
PUT bench-e2,bench-e4/_settings
{
  "index.refresh_interval": null,
  "index.number_of_replicas": 1
}

GET _cluster/health?wait_for_status=green&timeout=60s

GET _cat/indices/bench-*?v&s=index&h=index,health,docs.count,pri.store.size,store.size
```

**Qué hace:** devuelve los dos índices a su configuración normal y espera a que las réplicas se creen.

- `null`: elimina el valor puesto a mano y el índice vuelve al valor por defecto (1 s para `refresh_interval`).

- `number_of_replicas: 1`: Elasticsearch copia cada primario a otro nodo (recuperación de la réplica).

- `wait_for_status=green`: la petición espera hasta 60 s a que el clúster esté en verde.

- `pri.store.size` y `store.size`: tamaño de los primarios y total. Con réplica, el total es el doble del primario.

Regla de la parte C: un ajuste de carga se aplica solo mientras dura la carga y se revierte al terminar. Un índice que se deja en `refresh_interval: -1` y 0 réplicas no queda protegido ni es visible para búsquedas.

## Parte D: experimentos de búsqueda (11 min)

### Paso D1. Costo de una agregación y caché de peticiones

``` console
GET bench-base/_search?request_cache=false&filter_path=took
{
  "size": 0,
  "aggs": {
    "por_ip": {
      "terms": { "field": "client_ip", "size": 400 },
      "aggs": {
        "por_hora": {
          "date_histogram": { "field": "@timestamp", "fixed_interval": "1h" },
          "aggs": {
            "p95": { "percentiles": { "field": "response_time_ms", "percents": [95] } }
          }
        }
      }
    }
  }
}
```

**Qué hace:** agrupa por IP de cliente (400 valores distintos), luego por hora y calcula un percentil 95 en cada celda. Es una agregación anidada de alta cardinalidad: unos 29 000 buckets.

- `request_cache=false`: desactiva la caché de peticiones del shard para esta ejecución.

- `filter_path=took`: devuelve solo el tiempo. La respuesta completa sería enorme.

- `size: 0`: sin documentos en la respuesta.

Ejecútala tres veces y anota el menor `took` y el mayor. Ahora compara con una agregación sencilla:

``` console
GET bench-base/_search?request_cache=false&filter_path=took
{
  "size": 0,
  "aggs": { "por_servicio": { "terms": { "field": "service" } } }
}
```

**Qué hace:** agrupa solo por servicio (4 valores). **Resultado esperado:** `took` claramente menor que el de la agregación anidada. El costo crece con el número de buckets, no con el número de documentos.

``` console
GET bench-base/_search?request_cache=true&filter_path=took
{
  "size": 0,
  "aggs": {
    "por_ip": {
      "terms": { "field": "client_ip", "size": 400 },
      "aggs": {
        "por_hora": {
          "date_histogram": { "field": "@timestamp", "fixed_interval": "1h" },
          "aggs": {
            "p95": { "percentiles": { "field": "response_time_ms", "percents": [95] } }
          }
        }
      }
    }
  }
}

GET bench-base/_stats/request_cache?filter_path=_all.total.request_cache.hit_count,_all.total.request_cache.miss_count
```

**Qué hace:** repite la agregación pesada con la caché de peticiones activa y consulta sus contadores. Ejecuta la primera petición dos veces.

- Primera ejecución: fallo de caché (`miss_count` aumenta) y `took` similar al anterior.

- Segunda ejecución: acierto de caché (`hit_count` aumenta) y `took` mucho menor. El resultado completo se tomó de memoria.

- La caché solo guarda búsquedas con `size: 0`, y se invalida cuando el shard cambia (por ejemplo, tras un refresh con documentos nuevos). Por eso rinde más en índices que ya no se escriben.

### Paso D2. Costo de un comodín

``` console
GET bench-base/_search?filter_path=took,hits.total.value
{
  "size": 0,
  "track_total_hits": true,
  "query": { "match": { "message": "timeout" } }
}

GET bench-base/_search?filter_path=took,hits.total.value
{
  "size": 0,
  "track_total_hits": true,
  "query": { "wildcard": { "message.keyword": { "value": "*timeout*" } } }
}
```

**Qué hace:** busca la misma palabra de dos maneras. La primera usa el índice invertido del campo `text`. La segunda recorre los términos del subcampo `keyword` para evaluar el patrón.

**Resultado esperado:** ambas devuelven 416 documentos. Compara `took`: a esta escala la diferencia puede ser pequeña, pero el costo del comodín crece con el número de términos distintos del campo. Con el patrón `*timeout*` (comodín al inicio) no hay forma de acotar el recorrido.

### Paso D3. Dónde se va el tiempo: profile

``` console
GET bench-base/_search?filter_path=profile.shards.searches.query.type,profile.shards.searches.query.time_in_nanos
{
  "profile": true,
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "service": "payments" } },
        { "range": { "status": { "gte": 500 } } }
      ]
    }
  }
}
```

**Qué hace:** ejecuta la consulta y devuelve el árbol de ejecución con el tiempo de cada componente en nanosegundos.

- `profile: true`: activa el perfilado. Agrega sobrecarga, por lo que se usa para diagnosticar, no en producción.

- `type`: la clase interna de Lucene que ejecuta cada parte de la consulta.

- `time_in_nanos`: tiempo de esa parte. En Kibana, la herramienta Search Profiler (en Dev Tools) muestra el mismo resultado de forma gráfica.

Para comprobar el resultado: `GET bench-base/_count` con esa misma consulta debe devolver 183 (payments con estado 5xx).

## Parte E: segmentos y merge (8 min)

### Paso E1. Estadísticas de indexación, refresh y merge

``` console
GET bench-e1,bench-e2/_stats/indexing,refresh,merge?level=indices&filter_path=indices.*.primaries.indexing.index_total,indices.*.primaries.indexing.index_time_in_millis,indices.*.primaries.refresh.total,indices.*.primaries.merges.total,indices.*.primaries.merges.total_time_in_millis
```

**Qué hace:** compara lo que hizo cada índice mientras se cargó: documentos indexados, tiempo de indexación, refreshes y merges.

- `level=indices`: separa los resultados por índice.

- `indexing.index_total` y `index_time_in_millis`: documentos indexados y tiempo acumulado de indexación en los primarios.

- `refresh.total`: número de refreshes. `bench-e2` debe tener muchos menos que `bench-e1`.

- `merges.total`: fusiones de segmentos realizadas. Lucene fusiona segmentos pequeños en segundo plano.

### Paso E2. Segmentos y force merge

``` console
GET _cat/segments/bench-e1?v&h=index,shard,prirep,segment,docs.count,size

POST bench-e1/_forcemerge?max_num_segments=1

GET _cat/segments/bench-e1?v&h=index,shard,prirep,segment,docs.count,size

GET _cat/indices/bench-e1?v&h=index,docs.count,pri.store.size,store.size
```

**Qué hace:** lista los segmentos del índice, los fusiona en uno y vuelve a listar.

- `_cat/segments`: un segmento es un conjunto inmutable de archivos de Lucene. Cada búsqueda debe consultar todos los segmentos del shard.

- `_forcemerge`: fusiona segmentos hasta el número indicado en `max_num_segments`.

- `max_num_segments=1`: un segmento por shard. Reduce el overhead y suele acelerar las búsquedas, y el tamaño del índice puede bajar.

**Resultado esperado:** el número de segmentos por shard baja a 1. Compara `pri.store.size` con el valor que anotaste en el paso C5.

Advertencia: el force merge es costoso y solo debe hacerse sobre índices que ya no reciben escrituras. En un data stream se aplica a los índices ya rotados (con ILM, la acción `forcemerge` de la fase warm). Hacerlo sobre un índice activo genera un segmento enorme que el merge automático deja de fusionar.

### Paso E3. Rechazos al final de las pruebas

``` console
GET _cat/thread_pool/write,search?v&h=node_name,name,active,queue,rejected,completed
```

**Qué hace:** repite la lectura del paso A4. Compara `completed` (creció con tus pruebas) y `rejected`. **Resultado esperado:** `rejected` igual que al inicio. Si hubiera crecido, la carga superaba la capacidad del pool de escritura y habría que reducir el lote o la concurrencia de la carga.

## Parte F: conclusiones y limpieza (4 min)

### Paso F1. Tabla de resultados

Completa la tabla con tus mediciones y escribe una conclusión por fila.

| **Medición**                               | **took (ms)** | **Docs por s** | **Conclusión** |
|--------------------------------------------|---------------|----------------|----------------|
| B: carga con curl (una petición)           |               |                |                |
| C1: e1, línea base                         |               |                |                |
| C2: e2, sin refresh ni réplicas            |               |                |                |
| C3: e3, lotes de 5000                      |               |                |                |
| C4: e4, combinación                        |               |                |                |
| D1: agregación pesada, sin caché           |               |                |                |
| D1: agregación pesada, con caché (2.ª vez) |               |                |                |
| D2: `match` frente a `wildcard`            |               |                |                |

Docs por segundo = 30 000 / (`took` en segundos). Por ejemplo, con `took` de 3000 ms son 10 000 docs por segundo.

### Paso F2. Limpieza

``` console
DELETE bench-base,bench-e1,bench-e2,bench-e3,bench-e4

DELETE _index_template/bench-template

DELETE _security/api_key
{ "name": "lab06-bench" }
```

**Qué hace:** elimina los índices de prueba, el template y la API key.

- Los índices se nombran uno por uno porque, desde la versión 8.0, `action.destructive_requires_name` es `true` por defecto y rechaza borrados con comodín o `_all`.

- `DELETE _security/api_key` con `name`: invalida las API keys con ese nombre.

## Resumen de valores esperados

| **Comprobación**                                    | **Resultado**                                            |
|-----------------------------------------------------|----------------------------------------------------------|
| `GET bench-base/_count`                             | 30000                                                    |
| `errors` en la carga Bulk                           | false                                                    |
| `match` `message: timeout` (D2)                     | 416                                                      |
| `wildcard` `*timeout*` sobre `message.keyword` (D2) | 416                                                      |
| payments con estado 5xx (D3)                        | 183                                                      |
| Documentos de `bench-base` por servicio             | catalog 10361, checkout 9146, search 5947, payments 4546 |
| Estado 5xx en `bench-base`                          | 548 (500: 416, 503: 132)                                 |
| IP de cliente distintas                             | 400                                                      |

## Estado final

| **Elemento**                                           | **Se conserva**     | **Lo reutiliza**                        |
|--------------------------------------------------------|---------------------|-----------------------------------------|
| `weblogs` con 5006 documentos                          | Sí                  | Laboratorios 7, 8 y 9, y proyecto final |
| `bench-*`, `bench-template` y la API key `lab06-bench` | No (se eliminan)    | \-                                      |
| Tabla de resultados                                    | Sí (en tu cuaderno) | Discusión final y proyecto final        |

## Preguntas de discusión

- ¿Qué ajuste dio la mayor mejora de indexación? ¿Cuál es su riesgo y cuándo lo aplicarías en producción?

- ¿Por qué los documentos de `bench-e2` no eran visibles antes del `_refresh`?

- ¿Por qué la caché de peticiones acelera un panel que repite la misma agregación y no una búsqueda con `size` mayor que 0?

- ¿Qué cambiarías en el diseño del mapping para que la agregación por IP y hora fuera menos costosa?

- ¿Qué indicador revisarías primero si los clientes empezaran a recibir respuestas 429?

- Con 500 GB diarios de logs, ¿qué cambia respecto de este laboratorio en el número de shards, los tiers y la retención?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 6: Benchmark y tuning de un clúster de logs](imagenes/lab06_resultado_final.png)
