# Laboratorio 5: Consultas y agregaciones aplicadas a un set de logs

- **Capítulo:** 5. Consultas y búsqueda avanzada

- **Duración:** 54 min

- **Entorno:** Elastic Cloud Hosted (cuenta propia), Elasticsearch 9.x, Kibana Dev Tools y Lens

- **Dependencias:** Laboratorio 4 (alias `weblogs` con 5006 documentos). Si el conteo no coincide, revisar el Laboratorio 4 antes de continuar.

## Objetivo

Consultar el set de logs cargado en el laboratorio anterior con Query DSL y ES\|QL: búsquedas de texto y exactas con filtros combinados, ajuste de relevancia, agregaciones por categoría y por tiempo, comparación entre ambos lenguajes y una visualización en Lens. Al terminar, el participante podrá:

- Combinar `match`, `term`, `range` y `exists` en una consulta `bool`, separando lo que puntúa (`must`, `should`) de lo que solo filtra (`filter`, `must_not`).

- Ajustar la relevancia con `boost`, `fuzziness` y `match_phrase`, e interpretar `_score`.

- Construir agregaciones de bucket y de métrica, incluida una anidada y una de pipeline.

- Escribir consultas ES\|QL equivalentes y reconocer cuándo conviene cada lenguaje.

- Crear un data view y una visualización en Lens.

## Requisitos

- `GET weblogs/_count` devuelve 5006.

- Los documentos tienen los campos `service`, `team`, `tier`, `status`, `response_time_ms`, `message`, `detail` y `@timestamp`. Los 30 documentos con `parse_failure` no tienen `@timestamp` ni `service`, y quedan fuera de los filtros por fecha y de las agregaciones por servicio.

- Todas las fechas de los datos están en UTC. Dev Tools las muestra en UTC; Kibana (Discover y Lens) las muestra en la zona horaria del navegador.

## Distribución del tiempo

| **Parte** | **Contenido**                                       | **Tiempo** |
|-----------|-----------------------------------------------------|------------|
| Parte A   | Búsquedas de texto y exactas con filtros combinados | 12 min     |
| Parte B   | Ajuste de relevancia                                | 8 min      |
| Parte C   | Agregaciones por categoría y por tiempo             | 12 min     |
| Parte D   | ES\|QL comparado con Query DSL                      | 14 min     |
| Parte E   | Data view y visualización en Lens                   | 8 min      |

## Convenciones

- Los comandos se ejecutan en Dev Tools, salvo la Parte E, que se realiza en la interfaz de Kibana.

- En las consultas de conteo se usa `"size": 0` (sin documentos en la respuesta) y `"track_total_hits": true`. Por defecto el total se corta en 10 000; aquí se exige el conteo exacto.

- Los resultados esperados se derivan del dataset del curso. Los promedios y percentiles se indican de forma aproximada.

## Parte A: búsquedas de texto y exactas (12 min)

### Paso A1. Búsqueda de texto completo

``` console
GET weblogs/_search
{
  "track_total_hits": true,
  "size": 3,
  "_source": ["@timestamp", "service", "status", "message"],
  "query": {
    "match": { "message": "timeout" }
  }
}
```

**Qué hace:** busca el término `timeout` en el campo `message`, que es de tipo `text`.

- `match`: analiza el texto de la consulta con el mismo analyzer del campo y busca los tokens resultantes. Con varias palabras, por defecto basta con que aparezca una.

- `_source`: limita los campos que se devuelven.

- `size`: cantidad de documentos en la respuesta.

- `hits.hits[]._score`: relevancia (BM25) de cada resultado.

**Resultado esperado:** `hits.total.value` es 76 (75 documentos nuevos con `upstream timeout` y el documento original del laboratorio 2).

### Paso A2. Búsqueda exacta con term

``` console
GET weblogs/_search
{
  "size": 0,
  "track_total_hits": true,
  "query": {
    "term": { "service": "payments" }
  }
}
```

**Qué hace:** busca el valor exacto `payments` en un campo `keyword`. `term` no analiza el valor, por lo que debe coincidir tal como está indexado (mayúsculas incluidas).

**Resultado esperado:** 765.

Error frecuente: usar `term` sobre un campo `text` (por ejemplo, `"term": {"message": "Timeout"}`). No devuelve nada, porque el índice guarda `timeout` en minúsculas.

### Paso A3. Combinar con bool

``` console
GET weblogs/_search
{
  "size": 0,
  "track_total_hits": true,
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "timeout" } }
      ],
      "filter": [
        { "term":  { "service": "payments" } },
        { "range": { "status": { "gte": 500 } } },
        { "range": { "@timestamp": { "gte": "2026-10-13T00:00:00Z", "lt": "2026-10-14T00:00:00Z" } } }
      ]
    }
  }
}
```

**Qué hace:** busca errores por timeout del servicio de pagos el 13 de octubre.

- `must`: las cláusulas deben cumplirse y aportan al `_score`.

- `filter`: deben cumplirse pero no puntúan. Se pueden almacenar en caché, por lo que son más rápidas. Se usan para condiciones de sí o no.

- `range`: rango numérico o de fechas. `gte` es mayor o igual y `lt` menor que.

**Resultado esperado:** 13. Sin la cláusula `match` el resultado sería 17 (todos los errores 5xx de payments ese día; los 4 restantes son `db connection refused`).

### Paso A4. Excluir y comprobar existencia

``` console
GET weblogs/_search
{
  "size": 0,
  "track_total_hits": true,
  "query": {
    "bool": {
      "filter": [ { "range": { "status": { "gte": 500 } } } ],
      "must_not": [ { "term": { "service": "payments" } } ]
    }
  }
}

GET weblogs/_search
{
  "size": 0,
  "track_total_hits": true,
  "query": {
    "bool": { "must_not": [ { "exists": { "field": "@timestamp" } } ] }
  }
}

GET weblogs/_search
{
  "size": 0,
  "track_total_hits": true,
  "query": { "terms": { "status": [404, 429] } }
}
```

**Qué hace:** tres consultas.

- `must_not`: excluye los documentos que cumplen la cláusula, sin afectar la puntuación.

- `exists`: documentos que tienen un valor en el campo. Combinado con `must_not`, encuentra los que no lo tienen.

- `terms`: coincide con cualquiera de los valores de la lista.

**Resultado esperado:** 55 errores 5xx de servicios distintos de payments; 30 documentos sin `@timestamp` (los `parse_failure`); 53 documentos con estado 404 o 429.

## Parte B: ajuste de relevancia (8 min)

### Paso B1. Tolerar errores de escritura y buscar frases

``` console
GET weblogs/_search
{
  "size": 0,
  "track_total_hits": true,
  "query": {
    "match": { "message": { "query": "conection", "fuzziness": "AUTO" } }
  }
}

GET weblogs/_search
{
  "size": 0,
  "track_total_hits": true,
  "query": {
    "match_phrase": { "message": "product not found" }
  }
}
```

**Qué hace:** la primera consulta encuentra `connection` aunque se escriba mal; la segunda exige que las tres palabras aparezcan juntas y en orden.

- `fuzziness: AUTO`: tolera ediciones de caracteres según la longitud del término (0 ediciones hasta 2 caracteres, 1 hasta 5 y 2 desde 6). `conection` está a una edición de `connection`.

- `match_phrase`: busca la secuencia exacta de tokens. Es más estricta que `match`.

**Resultado esperado:** 19 documentos (`db connection refused`) y 43 documentos (42 nuevos más el original del laboratorio 2).

### Paso B2. Dar más peso a un término

``` console
GET weblogs/_search
{
  "size": 3,
  "track_total_hits": true,
  "_source": ["service", "message"],
  "explain": true,
  "query": {
    "bool": {
      "should": [
        { "match": { "message": { "query": "timeout", "boost": 1 } } },
        { "match": { "message": { "query": "refused", "boost": 3 } } }
      ]
    }
  }
}
```

**Qué hace:** devuelve los documentos que contienen `timeout` o `refused`, pero prioriza los que contienen `refused`.

- `should`: sin `must` ni `filter`, basta con que se cumpla al menos una cláusula, y cada una que se cumple suma puntuación.

- `boost`: multiplicador del peso de la cláusula.

- `explain: true`: añade a cada resultado la descomposición del `_score` (campo `_explanation`), con el `boost` visible.

**Resultado esperado:** `hits.total.value` es 95 (76 + 19). Los tres primeros resultados son de `db connection refused`, porque su `_score` se multiplica por 3.

### Paso B3. Bajar la prioridad sin excluir

``` console
GET weblogs/_search
{
  "size": 5,
  "_source": ["service", "message"],
  "query": {
    "boosting": {
      "positive": { "match": { "message": "timeout" } },
      "negative": { "term": { "service": "payments" } },
      "negative_boost": 0.1
    }
  }
}
```

**Qué hace:** busca `timeout` pero reduce la puntuación de los resultados de payments a una décima. A diferencia de `must_not`, no los elimina: los deja al final.

**Resultado esperado:** los primeros resultados son timeouts de servicios distintos de payments (por ejemplo, checkout); los de payments aparecen más abajo.

## Parte C: agregaciones por categoría y por tiempo (12 min)

### Paso C1. Por servicio: conteo, promedio y percentil 95

``` console
GET weblogs/_search
{
  "size": 0,
  "aggs": {
    "por_servicio": {
      "terms": { "field": "service" },
      "aggs": {
        "rt_promedio": { "avg": { "field": "response_time_ms" } },
        "rt_p95": { "percentiles": { "field": "response_time_ms", "percents": [95] } }
      }
    }
  }
}
```

**Qué hace:** agrupa por servicio (agregación de bucket) y calcula métricas dentro de cada grupo (agregaciones anidadas).

- `terms`: un bucket por cada valor distinto del campo. Devuelve los 10 más frecuentes por defecto.

- `aggs` dentro de otra agregación: se evalúa una vez por bucket.

- `avg`: promedio. `percentiles`: valores por debajo de los cuales cae el porcentaje indicado; el cálculo es aproximado.

- `size: 0` en la raíz: la respuesta lleva solo agregaciones.

| **Servicio** | **Documentos** | **Promedio (ms)** | **Percentil 95 (ms)** |
|--------------|----------------|-------------------|-----------------------|
| catalog      | 1782           | ≈ 113             | ≈ 193                 |
| checkout     | 1499           | ≈ 94              | ≈ 137                 |
| search       | 930            | ≈ 185             | ≈ 339                 |
| payments     | 765            | ≈ 404             | ≈ 1404                |

Payments tiene el promedio y el percentil 95 más altos: sus errores por timeout tardan más de 1.2 segundos.

### Paso C2. Errores por hora y detección del pico

``` console
GET weblogs/_search
{
  "size": 0,
  "query": { "range": { "status": { "gte": 500 } } },
  "aggs": {
    "por_hora": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h",
        "min_doc_count": 1
      }
    },
    "pico": {
      "max_bucket": { "buckets_path": "por_hora>_count" }
    }
  }
}
```

**Qué hace:** cuenta los errores 5xx por hora y luego identifica la hora con más errores.

- `date_histogram`: un bucket por cada intervalo de tiempo.

- `fixed_interval`: duración exacta del intervalo (`1h`, `30m`, `1d`). Para meses o años se usa `calendar_interval`.

- `min_doc_count: 1`: omite los intervalos sin documentos.

- `max_bucket` (agregación de pipeline): opera sobre el resultado de otra agregación. `buckets_path` indica cuál: `por_hora>_count` es el conteo de cada bucket de `por_hora`.

**Resultado esperado:** `pico.value` es 12 y `pico.keys` es `["2026-10-13T14:00:00.000Z"]`. En esa hora hubo un incidente en payments: 11 de sus 25 peticiones fallaron.

### Paso C3. Rangos de tiempo de respuesta y usuarios distintos

``` console
GET weblogs/_search
{
  "size": 0,
  "aggs": {
    "latencia": {
      "range": {
        "field": "response_time_ms",
        "ranges": [
          { "key": "rapida (<100 ms)",      "to": 100 },
          { "key": "normal (100-500 ms)",   "from": 100,  "to": 500 },
          { "key": "lenta (500-1000 ms)",   "from": 500,  "to": 1000 },
          { "key": "muy lenta (>=1000 ms)", "from": 1000 }
        ]
      }
    },
    "ips_distintas": { "cardinality": { "field": "client_ip" } },
    "por_criticidad": { "terms": { "field": "tier" } }
  }
}
```

**Qué hace:** agrupa las peticiones por rangos de latencia, cuenta direcciones IP distintas y agrupa por criticidad del servicio.

- `range`: buckets por rangos numéricos. `from` es inclusivo y `to` es exclusivo.

- `cardinality`: cuenta valores distintos con un algoritmo aproximado (exacto con volúmenes pequeños como este).

**Resultado esperado:** rápida 2583, normal 2240, lenta 61 y muy lenta 92 (suman 4976); 63 IP distintas; criticidad `standard` 2712 y `critical` 2264.

## Parte D: ES\|QL comparado con Query DSL (14 min)

ES\|QL es un lenguaje de consulta por tuberías: cada comando recibe una tabla y pasa otra al siguiente. Se ejecuta con `POST _query` o desde Discover.

### Paso D1. Primera consulta

``` console
POST _query?format=txt
{
  "query": """
    FROM weblogs
    | WHERE service == "payments" AND status >= 500
    | KEEP @timestamp, service, status, detail
    | SORT @timestamp DESC
    | LIMIT 3
  """
}
```

**Qué hace:** muestra los tres errores más recientes de payments.

- `POST _query`: API de ES\|QL. El cuerpo lleva la consulta en `query`; las comillas triples permiten escribirla en varias líneas.

- `format=txt`: respuesta como tabla de texto. Otras opciones: `csv`, `tsv` y `json` (predeterminado).

- `FROM`: índice, alias o data stream de origen.

- `WHERE`: filtra filas. `KEEP`: elige columnas. `SORT`: ordena. `LIMIT`: limita las filas (por defecto 1000).

### Paso D2. Errores por servicio

``` console
POST _query?format=txt
{
  "query": """
    FROM weblogs
    | WHERE status >= 500
    | STATS errores = COUNT(*) BY service
    | SORT errores DESC
  """
}
```

**Qué hace:** cuenta los errores 5xx por servicio. `STATS ... BY` equivale a una agregación `terms` con una métrica.

**Resultado esperado:** payments 40, checkout 25, search 16 y catalog 14 (suman 95).

Equivalente en Query DSL:

``` console
GET weblogs/_search
{
  "size": 0,
  "query": { "range": { "status": { "gte": 500 } } },
  "aggs": {
    "por_servicio": { "terms": { "field": "service" } }
  }
}
```

Los conteos coinciden. Con ES\|QL la consulta es más corta, el resultado ya viene como tabla y los pasos intermedios son visibles; con Query DSL se tiene control fino de la relevancia (`_score`, `boost`, `fuzziness`) y se integra con todas las agregaciones.

### Paso D3. Errores por hora en el día del incidente

``` console
POST _query?format=txt
{
  "query": """
    FROM weblogs
    | WHERE service == "payments"
        AND @timestamp >= "2026-10-13T00:00:00Z"
        AND @timestamp <  "2026-10-14T00:00:00Z"
    | EVAL hora = DATE_TRUNC(1 hour, @timestamp),
           es_error = CASE(status >= 500, 1, 0)
    | STATS peticiones = COUNT(*), errores = SUM(es_error), rt_prom = AVG(response_time_ms) BY hora
    | SORT errores DESC
    | LIMIT 2
  """
}
```

**Qué hace:** calcula, para cada hora, el número de peticiones, los errores y el tiempo de respuesta promedio de payments.

- `EVAL`: crea columnas calculadas. `DATE_TRUNC(1 hour, ...)` redondea cada fecha a la hora. `CASE(condición, valor_si, valor_no)` marca los errores con 1.

- `STATS ... BY hora`: agrupa y agrega, como `date_histogram` con métricas.

- Las fechas se comparan con cadenas ISO 8601.

**Resultado esperado:** la primera fila es `2026-10-13T14:00:00.000Z` con 25 peticiones y 11 errores, con un `rt_prom` muy superior al de las demás horas. La segunda es la hora de las 10:00 con 22 peticiones y 2 errores.

### Paso D4. Percentiles por servicio

``` console
POST _query?format=txt
{
  "query": """
    FROM weblogs
    | STATS p50 = PERCENTILE(response_time_ms, 50),
            p95 = PERCENTILE(response_time_ms, 95)
        BY service
    | SORT p95 DESC
  """
}
```

**Qué hace:** repite el cálculo del percentil 95 del paso C1 con ES\|QL.

**Resultado esperado:** payments primero (p95 cercano a 1400 ms), después search, catalog y checkout. Los valores pueden diferir ligeramente de los del paso C1 porque ambos cálculos son aproximados. Los 30 documentos sin `service` generan una fila con valor nulo.

Completar la comparación:

| **Criterio**                     | **Query DSL**                                      | **ES\|QL**              |
|----------------------------------|----------------------------------------------------|-------------------------|
| Búsqueda de texto con relevancia | Completa (`match`, `boost`, `fuzziness`, `_score`) | Más limitada            |
| Agregar y transformar columnas   | Agregaciones anidadas y JSON extenso               | Tubería corta y legible |
| Formato de resultado             | JSON con hits y buckets                            | Tabla                   |
| Caso en que lo prefiero          |                                                    |                         |

## Parte E: data view y visualización en Lens (8 min)

### Paso E1. Crear el data view

1.  En Kibana, abrir el menú principal y entrar a **Stack Management**. Dentro de la sección de Kibana, seleccionar **Data Views** y **Create data view**.

2.  Nombre: `weblogs`. Index pattern: `weblogs`. Campo de marca de tiempo: `@timestamp`.

3.  Guardar con **Save data view to Kibana**.

Un data view le dice a Kibana qué índices consultar y qué campo usar como tiempo. Se estudia a detalle en el capítulo 8.

### Paso E2. Explorar en Discover

1.  Abrir **Discover** y elegir el data view `weblogs`.

2.  Establecer el selector de tiempo en un rango absoluto: del 11 al 16 de octubre de 2026 (un rango amplio evita problemas con la zona horaria).

3.  En la barra de búsqueda (KQL), escribir `service : "payments" and status >= 500` y ejecutar.

**Resultado esperado:** 40 documentos. Agregar los campos `status`, `detail` y `message` a la tabla para revisarlos.

### Paso E3. Crear la visualización

1.  Abrir el menú principal, entrar a **Visualize Library** (o a **Dashboards**) y crear una visualización con **Lens**.

2.  Elegir el data view `weblogs` y el tipo de gráfico **Line**.

3.  Eje horizontal: `@timestamp`, con intervalo de una hora. Eje vertical: **Count of records**. Dividir las series por `service` (4 valores).

4.  Aplicar el filtro `status >= 500` en la barra de búsqueda de Lens.

5.  Guardar con el título `Errores 5xx por hora (lab 5)`.

**Resultado esperado:** una línea por servicio, con un pico claro de payments el 13 de octubre. Si el navegador está en la zona horaria de Ciudad de México (UTC−6), el pico de las 14:00 UTC aparece a las 08:00.

## Extensión opcional: paginación profunda con search_after

Si sobra tiempo. `from` + `size` no puede superar 10 000 resultados (`index.max_result_window`). Para recorrer más datos se usa `search_after` con un point in time (PIT).

``` console
POST weblogs/_pit?keep_alive=2m

GET _search
{
  "size": 3,
  "_source": ["@timestamp", "service"],
  "query": { "exists": { "field": "@timestamp" } },
  "pit": { "id": "<ID_DEL_PIT>", "keep_alive": "2m" },
  "sort": [ { "@timestamp": "asc" } ]
}
```

**Qué hace:** abre un point in time (una vista congelada del índice) y consulta contra él. La petición no lleva índice en la ruta: el índice lo define el PIT.

Para la página siguiente, copiar el arreglo `sort` del último resultado y agregarlo como `"search_after": [ ... ]` en la misma consulta. Al terminar, cerrar el PIT:

``` console
DELETE _pit
{
  "id": "<ID_DEL_PIT>"
}
```

## Resumen de valores esperados

| **Consulta**                                                 | **Resultado**                                   |
|--------------------------------------------------------------|-------------------------------------------------|
| Total de documentos (`GET weblogs/_count`)                   | 5006                                            |
| `match` `message: timeout`                                   | 76                                              |
| payments, 5xx, 13 de octubre, con `timeout`                  | 13                                              |
| 5xx excluyendo payments                                      | 55                                              |
| Documentos sin `@timestamp`                                  | 30                                              |
| Estado 404 o 429                                             | 53                                              |
| `fuzziness` `conection` / `match_phrase` `product not found` | 19 / 43                                         |
| `should` con `boost` (`timeout` o `refused`)                 | 95                                              |
| Hora pico de errores (UTC)                                   | 2026-10-13T14:00, 12 errores                    |
| Errores 5xx por servicio (ES\|QL)                            | payments 40, checkout 25, search 16, catalog 14 |

## Limpieza y estado final

Este laboratorio no elimina datos.

| **Elemento**                                 | **Se conserva** | **Lo reutiliza**                      |
|----------------------------------------------|-----------------|---------------------------------------|
| `weblogs` con 5006 documentos                | Sí              | Laboratorios 6, 8, 9 y proyecto final |
| Data view `weblogs`                          | Sí              | Laboratorio 8 (dashboards y alertas)  |
| Visualización `Errores 5xx por hora (lab 5)` | Sí              | Laboratorio 8 (dashboard)             |

## Preguntas de discusión

- ¿Por qué la consulta A3 pone los criterios de servicio, estado y fecha en `filter` y no en `must`?

- ¿Por qué un `term` sobre `message` no encuentra `Timeout`, y uno sobre `service` sí encuentra `payments`?

- ¿Qué diferencia práctica hay entre `must_not` y `boosting`?

- ¿Qué decisión tomarías con el pico de las 14:00 UTC del 13 de octubre? ¿Qué más consultarías?

- ¿En qué casos usarías ES\|QL y en cuáles Query DSL?

- Si el índice tuviera 50 millones de documentos, ¿qué consultas de este laboratorio serían las más costosas?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 5: Consultas y agregaciones sobre logs](imagenes/lab05_resultado_final.png)
