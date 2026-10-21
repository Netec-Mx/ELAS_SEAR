# Administración de Elasticsearch para gestión y análisis de logs

## Práctica 3. Creación de índices, operaciones CRUD con documentos, búsquedas simples y booleanas

### Objetivo

Al finalizar la práctica, serás capaz de:

- Crear y administrar índices: settings, aliases, cierre y apertura, y eliminación.
- Ejecutar operaciones CRUD sobre documentos y cargas masivas con la Bulk API.
- Construir búsquedas con Query DSL: `match`, `match_phrase`, `term`, `terms`, `range`, `exists` y `wildcard`.
- Combinar condiciones con consultas `bool` y elegir correctamente entre `must`, `filter`, `should` y `must_not`.
- Ordenar, paginar (`from`/`size`, `search_after` con PIT) y filtrar el `_source` de los resultados.

### Objetivo visual

![Flujo de la práctica: índice, documentos y consultas](img/cap03.png)

### Duración aproximada

- 50 minutos.

### Prerrequisitos

- Práctica 2 completada: Elasticsearch 8.19 en ejecución como servicio y Kibana accesible en `http://localhost:5601`.
- Archivo `C:\labs\datos\cap03_app_logs.ndjson` (40 eventos de log de una tienda en línea).

---

### Instrucciones

Todas las peticiones de esta práctica se ejecutan en **Kibana > Dev Tools (Console)**, salvo la carga del archivo, que se hace con curl desde PowerShell. Para ejecutar una petición en Console, coloca el cursor sobre ella y presiona `Ctrl + Enter` o haz clic en el triángulo ▶.

**Escenario:** el equipo de operaciones de una tienda en línea necesita un índice para los logs de sus microservicios (`catalog-service`, `cart-service`, `checkout-service`, `payment-service`, `auth-service` e `inventory-service`) y consultas para responder preguntas operativas.

---

### Tarea 1. Gestionar índices

#### Paso 1. Crea el índice con settings explícitos

```http
PUT app-logs-lab
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "5s"
  }
}
```

**Qué hace y por qué:**

- `number_of_shards: 1`: un volumen pequeño no necesita más shards. Cada shard consume memoria y descriptores de archivo, así que demasiados shards pequeños (*oversharding*) es uno de los problemas más comunes.
- `number_of_replicas: 0`: el clúster tiene un solo nodo (ver Práctica 2) y así el índice queda `green`.
- `refresh_interval: 5s`: los documentos se vuelven visibles para la búsqueda cada 5 segundos. El valor por defecto es 1 s; aumentarlo reduce el trabajo de indexación en cargas intensivas de logs.

#### Paso 2. Consulta y modifica los settings

```http
GET app-logs-lab/_settings

PUT app-logs-lab/_settings
{
  "index": { "refresh_interval": "1s" }
}

GET app-logs-lab/_settings?filter_path=*.settings.index.refresh_interval,*.settings.index.number_of_*
```

**Qué hace y por qué:** algunos settings son **dinámicos** (`refresh_interval`, `number_of_replicas`) y se cambian en caliente; otros son **estáticos** (`number_of_shards`) y solo se definen al crear el índice. `filter_path` reduce la respuesta a los campos que te interesan, algo útil cuando la salida JSON es larga.

#### Paso 3. Crea aliases, incluido un alias filtrado

```http
POST _aliases
{
  "actions": [
    { "add": { "index": "app-logs-lab", "alias": "app-logs" } },
    { "add": { "index": "app-logs-lab", "alias": "app-logs-errores",
               "filter": { "term": { "log.level.keyword": "ERROR" } } } }
  ]
}

GET _cat/aliases/app-logs*?v
```

**Qué hace y por qué:** un **alias** es un nombre lógico que apunta a uno o varios índices. Las aplicaciones y los dashboards consultan el alias, y el administrador puede cambiar el índice real sin afectarlos (lo harás en la Práctica 4 con un reindex). El alias **filtrado** `app-logs-errores` aplica siempre una condición, de modo que un equipo puede consultar solo los errores sin escribir el filtro. Todas las acciones de `_aliases` se aplican de forma **atómica**.

#### Paso 4. Cierra y vuelve a abrir el índice

```http
POST app-logs-lab/_close
GET _cat/indices/app-logs-lab?v&h=index,status,health
GET app-logs-lab/_search

POST app-logs-lab/_open
GET _cat/indices/app-logs-lab?v&h=index,status,health
```

**Qué hace y por qué:** un índice **cerrado** conserva sus datos en disco, pero no consume memoria ni admite lecturas o escrituras: la búsqueda regresa el error `index_closed_exception`. Cerrar índices era una forma de "archivar" sin borrar, y también es necesario para cambiar ciertos settings estáticos, como los analizadores. En 8.x, para logs históricos se prefieren ILM y las fases cold y frozen (Práctica 10).

#### Paso 5. Comprueba la protección contra borrados masivos

```http
PUT borrar-prueba-1
PUT borrar-prueba-2

DELETE borrar-prueba-*

GET _cluster/settings?include_defaults=true&filter_path=*.action.destructive_requires_name

DELETE borrar-prueba-1,borrar-prueba-2
```

**Qué hace y por qué:** en Elasticsearch 8.x, `action.destructive_requires_name` vale `true` por defecto, por lo que un `DELETE` con comodines **falla** con un error. Es una salvaguarda contra borrados accidentales de muchos índices; para eliminar, debes nombrar cada índice explícitamente.

---

### Tarea 2. Operaciones CRUD sobre documentos

#### Paso 6. Indexa (crea) documentos con y sin ID

```http
PUT app-logs-lab/_doc/manual-1
{
  "@timestamp": "2026-10-20T08:00:00-06:00",
  "service": { "name": "checkout-service" },
  "host": { "name": "srv-app01" },
  "log": { "level": "INFO" },
  "message": "Documento de prueba creado manualmente",
  "tags": ["manual"]
}

POST app-logs-lab/_doc
{
  "@timestamp": "2026-10-20T08:01:00-06:00",
  "service": { "name": "cart-service" },
  "log": { "level": "WARN" },
  "message": "Documento con ID generado por Elasticsearch",
  "tags": ["manual"]
}
```

**Qué hace y por qué:** `PUT .../_doc/<id>` crea el documento con un ID propio, o lo **reemplaza** si ya existe. `POST .../_doc` genera un ID automático; es la forma más eficiente para logs, porque Elasticsearch no necesita verificar si el ID ya existe. Observa en la respuesta `_id`, `_version`, `_seq_no`, `_primary_term` y `result` (`created`).

#### Paso 7. Usa `_create` para evitar sobrescrituras

```http
PUT app-logs-lab/_create/manual-1
{
  "message": "Esto falla porque el ID ya existe"
}
```

**Qué hace y por qué:** `_create` solo indexa si el ID **no** existe; si existe, responde `409 version_conflict_engine_exception`. Garantiza que un evento no se sobrescriba por error, por ejemplo, al reprocesar logs.

#### Paso 8. Lee el documento

```http
GET app-logs-lab/_doc/manual-1

GET app-logs-lab/_source/manual-1

HEAD app-logs-lab/_doc/no-existe
```

**Qué hace y por qué:** `GET _doc` regresa el documento con sus metadatos; `_source` regresa solo el JSON original, y `HEAD` solo verifica si existe (200 o 404). La lectura por ID es **en tiempo real**: no depende del `refresh_interval`.

#### Paso 9. Actualiza parcialmente y con script

```http
POST app-logs-lab/_update/manual-1
{
  "doc": {
    "log": { "level": "WARN" },
    "response_time_ms": 1500
  }
}

POST app-logs-lab/_update/manual-1
{
  "script": {
    "source": "ctx._source.tags.add(params.etiqueta); ctx._source.response_time_ms += params.extra",
    "params": { "etiqueta": "revisado", "extra": 100 }
  }
}

GET app-logs-lab/_doc/manual-1
```

**Qué hace y por qué:** `_update` con `doc` fusiona los campos indicados con el documento existente. Con `script` (lenguaje Painless) se aplica lógica, como agregar una etiqueta o sumar un valor. Internamente, Elasticsearch **reindexa** el documento completo, porque los segmentos de Lucene son inmutables. Por eso los logs, que casi nunca se modifican, se tratan como datos *append-only*.

#### Paso 10. Aplica control de concurrencia optimista

```http
GET app-logs-lab/_doc/manual-1?filter_path=_seq_no,_primary_term

# Sustituye 3 y 1 por los valores de _seq_no y _primary_term que obtuviste
PUT app-logs-lab/_doc/manual-1?if_seq_no=3&if_primary_term=1
{
  "@timestamp": "2026-10-20T08:00:00-06:00",
  "message": "Reemplazo controlado",
  "tags": ["manual"]
}
```

Ejecuta el `PUT` dos veces. La segunda vez responde `409` porque `_seq_no` ya cambió.

**Qué hace y por qué:** `if_seq_no` e `if_primary_term` aseguran que solo se modifique la versión que leíste. Si otro proceso cambió el documento entretanto, la operación falla en lugar de perder cambios.

#### Paso 11. Usa la Bulk API con varias operaciones

```http
POST _bulk
{ "index": { "_index": "app-logs-lab", "_id": "manual-2" } }
{ "@timestamp": "2026-10-20T08:02:00-06:00", "log": { "level": "INFO" }, "message": "Alta por bulk", "tags": ["manual"] }
{ "create": { "_index": "app-logs-lab", "_id": "manual-3" } }
{ "@timestamp": "2026-10-20T08:03:00-06:00", "log": { "level": "ERROR" }, "message": "Creado por bulk", "tags": ["manual"] }
{ "update": { "_index": "app-logs-lab", "_id": "manual-2" } }
{ "doc": { "log": { "level": "WARN" } } }
{ "delete": { "_index": "app-logs-lab", "_id": "manual-3" } }
```

**Qué hace y por qué:** `_bulk` ejecuta muchas operaciones en una sola petición HTTP, con el formato NDJSON (una línea de acción y, para `index`, `create` y `update`, una línea de datos). Es la forma en que Filebeat, Logstash y Elastic Agent envían los logs. Revisa en la respuesta `"errors": false` y el `status` de cada elemento: un bulk **no es transaccional**, y cada operación puede fallar o tener éxito por separado.

#### Paso 12. Elimina los documentos de prueba

```http
DELETE app-logs-lab/_doc/manual-1

POST app-logs-lab/_delete_by_query?refresh=true
{
  "query": { "term": { "tags.keyword": "manual" } }
}

GET app-logs-lab/_count
```

**Qué hace y por qué:** `DELETE` borra un documento por ID; `_delete_by_query` borra todos los que cumplen una consulta. Aquí elimina los documentos de prueba, incluido el de ID automático, para que el índice quede vacío (`count: 0`) antes de la carga masiva. En logs, el borrado selectivo es poco común; la retención se maneja borrando índices completos con ILM, que es mucho más eficiente.

#### Paso 13. Carga el archivo de logs con curl

En PowerShell:

```powershell
. C:\labs\scripts\entorno.ps1
es POST "/app-logs-lab/_bulk?refresh=true&filter_path=errors,took" -File C:\labs\datos\cap03_app_logs.ndjson
```

Después, en Dev Tools:

```http
GET app-logs-lab/_count
GET app-logs-lab/_search?size=2
```

**Qué hace y por qué:** el archivo tiene 80 líneas: 40 acciones `{"index":{}}` y 40 documentos. Como el índice va en la URL, las líneas de acción no repiten `_index`. `--data-binary` (dentro de la función `es`) envía el archivo sin alterar los saltos de línea, que son obligatorios en NDJSON. `refresh=true` hace visibles los documentos de inmediato. La respuesta esperada es `{"took":..,"errors":false}` y el conteo debe ser **40**.

#### Paso 14. Revisa el mapping dinámico

```http
GET app-logs-lab/_mapping
```

**Qué hace y por qué:** no definiste un mapping, así que Elasticsearch lo **infirió**:

- `@timestamp` es `date`, y `response_time_ms` y `http.response.status_code` son `long`.
- Las cadenas (`message`, `log.level`, `service.name`, etc.) son `text` con un subcampo `keyword` (*multi-field*).
- `source.ip` quedó como `text`/`keyword`, **no** como `ip`, porque la inferencia no detecta direcciones IP.

Esta es la razón por la que en esta práctica usarás `campo.keyword` para comparaciones exactas. En la Práctica 4 diseñarás un mapping explícito que corrige estos tipos.

---

### Tarea 3. Búsquedas simples con Query DSL

#### Paso 15. Busca texto completo con `match` y `match_phrase`

```http
GET app-logs-lab/_search
{
  "query": { "match": { "message": "timeout" } }
}

GET app-logs-lab/_count
{
  "query": { "match": { "message": "base de datos" } }
}

GET app-logs-lab/_count
{
  "query": { "match": { "message": { "query": "base de datos", "operator": "and" } } }
}

GET app-logs-lab/_count
{
  "query": { "match_phrase": { "message": "base de datos" } }
}
```

**Qué hace y por qué:** `match` **analiza** el texto de búsqueda con el mismo analizador del campo (`standard`: minúsculas y separación por palabras) y busca los términos resultantes:

| Consulta | Lógica | Resultado |
|---|---|---|
| `match "timeout"` | contiene el término `timeout` | 3 |
| `match "base de datos"` | `base` **OR** `de` **OR** `datos` | 21 (la palabra común `de` aparece en muchos mensajes) |
| `match ... operator: and` | contiene los tres términos, en cualquier orden | 5 |
| `match_phrase` | los tres términos juntos y en ese orden | 5 |

Para frases en logs, usa `match_phrase` u `operator: and`; un `match` simple con palabras comunes devuelve ruido.

#### Paso 16. Compara `term` sobre `text` y sobre `keyword`

```http
GET app-logs-lab/_count
{ "query": { "term": { "log.level": "ERROR" } } }

GET app-logs-lab/_count
{ "query": { "term": { "log.level.keyword": "ERROR" } } }
```

**Qué hace y por qué:** `term` **no analiza** el valor buscado. El campo `log.level` (tipo `text`) se indexó en minúsculas (`error`), así que buscar `ERROR` da **0**. El subcampo `log.level.keyword` guarda el valor exacto y da **8**. Regla práctica: usa `term`, `terms` y `range` sobre campos `keyword`, numéricos, de fecha o IP, y usa `match` sobre campos `text`.

#### Paso 17. Busca varios valores exactos con `terms`

```http
GET app-logs-lab/_count
{
  "query": {
    "terms": { "service.name.keyword": ["checkout-service", "payment-service"] }
  }
}
```

**Qué hace y por qué:** `terms` equivale a un `IN` de SQL: devuelve los documentos cuyo valor coincide con **cualquiera** de la lista. El resultado es **16** (9 de checkout y 7 de payment).

#### Paso 18. Filtra por rangos numéricos y de fechas

```http
GET app-logs-lab/_count
{
  "query": { "range": { "response_time_ms": { "gte": 1000 } } }
}

GET app-logs-lab/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2026-10-20T12:00:00-06:00",
        "lt":  "2026-10-20T13:00:00-06:00"
      }
    }
  },
  "_source": ["@timestamp", "service.name", "message"]
}
```

**Qué hace y por qué:** `range` admite `gt`, `gte`, `lt` y `lte`. Las fechas se guardan en UTC; al incluir la zona `-06:00` en el valor, la consulta se interpreta en hora de la Ciudad de México. Otra opción es usar el parámetro `"time_zone": "America/Mexico_City"`. Resultados: **9** eventos tardaron 1,000 ms o más, y **8** eventos ocurrieron entre las 12:00 y las 13:00. En logs, el rango de tiempo es el filtro más usado: Kibana lo agrega en cada consulta.

#### Paso 19. Verifica la existencia de campos y usa comodines

```http
GET app-logs-lab/_count
{ "query": { "exists": { "field": "user.name" } } }

GET app-logs-lab/_search
{
  "query": { "wildcard": { "user.name.keyword": { "value": "*ez" } } },
  "_source": ["user.name", "message"]
}
```

**Qué hace y por qué:** `exists` devuelve los documentos que tienen un valor en el campo: **29** de 40 eventos tienen usuario, porque las consultas anónimas al catálogo no lo tienen. `wildcard` acepta `*` y `?` sobre campos `keyword`; devuelve **13** eventos de usuarios cuyo nombre termina en "ez". Los comodines al **inicio** del patrón (`*ez`) obligan a recorrer todo el diccionario de términos y son costosos en índices grandes; úsalos con moderación o prefiere el tipo `wildcard`, que se ve en la Práctica 4.

---

### Tarea 4. Consultas booleanas: `must`, `filter`, `should` y `must_not`

La consulta `bool` combina otras consultas mediante cuatro cláusulas. Elegir la cláusula correcta afecta el **resultado**, la **relevancia** (`_score`) y el **rendimiento**.

| Cláusula | ¿El documento debe cumplirla? | ¿Calcula `_score`? | Contexto | ¿Se puede cachear? | Uso recomendado en logs |
|---|---|---|---|---|---|
| `must` | Sí (AND) | **Sí**, suma relevancia | query | No | Búsqueda de texto completo donde importa qué tan bien coincide el texto, como un `match` sobre `message`. |
| `filter` | Sí (AND) | **No** (score 0) | filter | **Sí** | Condiciones exactas: rango de tiempo, `service.name`, `log.level`, `host.name`, códigos HTTP o entorno. Es la cláusula **más usada** en análisis de logs. |
| `should` | Opcional; con `minimum_should_match` se vuelve obligatoria (OR) | **Sí**, aumenta el score de lo que coincide | query | No | Expresar un OR (por ejemplo, "es 5xx **o** tardó más de 2 s") o dar prioridad a ciertos resultados sin excluir los demás. |
| `must_not` | Debe **no** cumplirse (NOT) | **No** | filter | **Sí** | Excluir ruido: health checks, entorno de pruebas o servicios que no interesan. |

> **Importante:** si una `bool` **no** tiene `must` ni `filter`, al menos una cláusula `should` debe cumplirse (`minimum_should_match` vale 1 de forma implícita). Si **sí** tiene `must` o `filter`, las `should` son opcionales y solo modifican el orden por relevancia, a menos que indiques `minimum_should_match`.

#### Paso 20. Combina `must` y `filter`

```http
GET app-logs-lab/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "pedido" } }
      ],
      "filter": [
        { "term":  { "environment.keyword": "produccion" } },
        { "range": { "response_time_ms": { "gte": 1000 } } }
      ]
    }
  },
  "_source": ["@timestamp", "service.name", "response_time_ms", "message"]
}
```

**Qué hace y por qué:** `must` busca la palabra `pedido` en el mensaje y calcula relevancia; `filter` exige el entorno de producción y una latencia de 1 s o más, sin afectar el score. El resultado son **2** documentos: "Latencia alta al guardar el pedido…" y "Reintento de guardado de pedido…". Los mensajes "…base de datos de pedidos" **no** aparecen: `pedidos` es un término distinto de `pedido`, porque el analizador `standard` no reduce palabras a su raíz. En la Práctica 4 verás cómo un analizador en español lo resuelve.

#### Paso 21. Excluye con `must_not`

```http
GET app-logs-lab/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "log.level.keyword": "ERROR" } }
      ],
      "must_not": [
        { "term": { "service.name.keyword": "catalog-service" } }
      ]
    }
  },
  "_source": ["service.name", "message"]
}
```

**Qué hace y por qué:** obtiene los errores de todos los servicios **excepto** catálogo: **7** documentos. `must_not` se ejecuta en contexto de filtro, así que no calcula relevancia y su resultado puede cachearse.

#### Paso 22. Expresa un OR con `should` y `minimum_should_match`

```http
GET app-logs-lab/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "service.name.keyword": "checkout-service" } }
      ],
      "should": [
        { "match": { "message": "timeout" } },
        { "match": { "message": "latencia" } }
      ],
      "minimum_should_match": 1
    }
  },
  "_source": ["@timestamp", "message", "response_time_ms"]
}
```

Repite la consulta **sin** `"minimum_should_match": 1` y compara el número de resultados.

**Qué hace y por qué:** con `minimum_should_match: 1`, el documento debe cumplir al menos una `should`: **3** resultados (dos timeouts y una latencia alta). Sin ese parámetro, como existe un `filter`, las `should` se vuelven opcionales y regresan **los 9** eventos de checkout. Los que coinciden con `timeout` o `latencia` aparecen primero por tener mayor score. Es el error más frecuente al construir consultas `bool`.

#### Paso 23. Usa `should` para priorizar sin excluir

```http
GET app-logs-lab/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "pedido" } }
      ],
      "should": [
        { "term": { "log.level.keyword": { "value": "ERROR", "boost": 5 } } }
      ]
    }
  },
  "_source": ["log.level", "message"]
}
```

**Qué hace y por qué:** regresa los **7** documentos con la palabra `pedido`, pero el de nivel `ERROR` aparece **primero** porque la `should` suma score (multiplicado por `boost`). Es útil en búsquedas interactivas: "muéstrame todo, pero los errores arriba".

#### Paso 24. Construye una consulta operativa completa

Pregunta: *"¿Qué eventos de producción entre las 09:00 y las 12:00 fueron errores del servidor (5xx) o tardaron 2 segundos o más?"*

```http
GET app-logs-lab/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term":  { "environment.keyword": "produccion" } },
        { "range": { "@timestamp": { "gte": "2026-10-20T09:00:00", "lt": "2026-10-20T12:00:00", "time_zone": "America/Mexico_City" } } }
      ],
      "should": [
        { "range": { "http.response.status_code": { "gte": 500 } } },
        { "range": { "response_time_ms": { "gte": 2000 } } }
      ],
      "minimum_should_match": 1,
      "must_not": [
        { "term": { "service.name.keyword": "inventory-service" } }
      ]
    }
  },
  "sort": [ { "@timestamp": "asc" } ],
  "_source": ["@timestamp", "service.name", "http.response.status_code", "response_time_ms", "message"]
}
```

**Qué hace y por qué:** combina las cuatro cláusulas como se recomienda: las condiciones exactas van en `filter`, el OR en `should` con `minimum_should_match`, y la exclusión en `must_not`. El resultado son **6** eventos. Al ordenar por `@timestamp`, el score no se usa, así que no hace falta `must`: en análisis de logs, la mayoría de consultas son puramente de filtro.

#### Paso 25. Compara el contexto query y el contexto filter

```http
GET app-logs-lab/_search
{
  "query": { "bool": { "must":   [ { "term": { "log.level.keyword": "ERROR" } } ] } },
  "size": 3, "_source": false
}

GET app-logs-lab/_search
{
  "query": { "bool": { "filter": [ { "term": { "log.level.keyword": "ERROR" } } ] } },
  "size": 3, "_source": false
}
```

**Qué hace y por qué:** ambas regresan los mismos 8 documentos. Con `must`, cada resultado tiene un `_score` mayor que 0; con `filter`, el `_score` es **0.0**. El contexto filter evita el cálculo de relevancia y permite que Elasticsearch **cachee** el conjunto de documentos que cumplen la condición, lo que acelera consultas repetidas, como las de los dashboards que se actualizan cada minuto.

---

### Tarea 5. Ordenar, paginar y filtrar el `_source`

#### Paso 26. Ordena y pagina con `from` y `size`

```http
GET app-logs-lab/_search
{
  "from": 0,
  "size": 5,
  "sort": [
    { "response_time_ms": "desc" },
    { "@timestamp": "asc" }
  ],
  "_source": ["service.name", "response_time_ms", "message"]
}
```

Cambia `"from": 0` por `"from": 5` para ver la segunda página.

**Qué hace y por qué:** muestra los 5 eventos más lentos (5000, 4200, 3010, 3005 y 2450 ms). El segundo criterio de orden desempata. `from`/`size` es sencillo, pero está limitado por `index.max_result_window` (10,000 por defecto), porque cada página profunda obliga a cada shard a ordenar `from + size` documentos.

#### Paso 27. Pagina con `search_after` y un Point in Time (PIT)

```http
POST app-logs-lab/_pit?keep_alive=2m
```

Copia el `id` de la respuesta y úsalo en lugar de `<PIT_ID>`:

```http
GET _search
{
  "size": 10,
  "pit": { "id": "<PIT_ID>", "keep_alive": "2m" },
  "sort": [ { "@timestamp": "asc" } ],
  "_source": ["@timestamp", "message"]
}
```

En la respuesta, copia el arreglo `sort` del **último** resultado (por ejemplo, `[1760976067000, 9]`) y pide la siguiente página:

```http
GET _search
{
  "size": 10,
  "pit": { "id": "<PIT_ID>", "keep_alive": "2m" },
  "sort": [ { "@timestamp": "asc" } ],
  "search_after": [1760976067000, 9],
  "_source": ["@timestamp", "message"]
}
```

Al terminar, libera el PIT:

```http
DELETE _pit
{ "id": "<PIT_ID>" }
```

**Qué hace y por qué:** un **PIT** congela una vista consistente del índice, de modo que los documentos que llegan durante la paginación no desplazan las páginas. `search_after` continúa desde los valores de orden del último resultado, sin el costo de `from`, y no tiene límite de profundidad. Es el método recomendado para exportar o recorrer muchos logs. Con PIT, Elasticsearch agrega un desempate implícito (`_shard_doc`), por eso el arreglo `sort` tiene dos valores. La petición se hace a `_search` sin nombre de índice, porque el PIT ya lo incluye.

#### Paso 28. Filtra los campos devueltos

```http
GET app-logs-lab/_search
{
  "size": 3,
  "_source": {
    "includes": ["@timestamp", "service.*", "log.level"],
    "excludes": ["service.version"]
  }
}

GET app-logs-lab/_search
{
  "size": 3,
  "_source": false,
  "fields": ["@timestamp", "service.name", "response_time_ms"]
}
```

**Qué hace y por qué:** `_source` con `includes` y `excludes` (se permiten comodines) reduce el tamaño de la respuesta y el tráfico de red. `fields` devuelve los valores tal como están **mapeados**, por ejemplo, fechas normalizadas, y también funciona con runtime fields (Práctica 4). Kibana Discover usa `fields`.

---

### Resultado esperado

- El índice `app-logs-lab` existe, está en estado `green`, tiene los aliases `app-logs` y `app-logs-errores` y contiene **40** documentos.

```http
GET _cat/indices/app-logs-lab?v&h=index,health,status,docs.count
GET app-logs-errores/_count
```

```text
index        health status docs.count
app-logs-lab green  open           40
```

`app-logs-errores/_count` regresa **8**.

- Resumen de resultados de las consultas:

| Paso | Consulta | Resultado |
|---|---|---|
| 15 | `match` "timeout" / "base de datos" (OR) / (AND) / `match_phrase` | 3 / 21 / 5 / 5 |
| 16 | `term` en `log.level` / `log.level.keyword` | 0 / 8 |
| 17 | `terms` checkout o payment | 16 |
| 18 | `range` ≥ 1000 ms / de 12:00 a 13:00 | 9 / 8 |
| 19 | `exists user.name` / `wildcard *ez` | 29 / 13 |
| 20 | `bool` must + filter | 2 |
| 21 | `bool` filter + must_not | 7 |
| 22 | `should` con / sin `minimum_should_match` | 3 / 9 |
| 23 | `should` con boost | 7 (el ERROR primero) |
| 24 | consulta operativa completa | 6 |

- Puedes explicar con tus palabras cuándo usar `must`, `filter`, `should` y `must_not`, y por qué en el análisis de logs predominan `filter` y `must_not`.

### Referencias

- Elastic. *Index APIs [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/indices.html
- Elastic. *Document APIs y Bulk API [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/docs-bulk.html
- Elastic. *Boolean query [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/query-dsl-bool-query.html
- Elastic. *Query and filter context [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/query-filter-context.html
- Elastic. *Paginate search results [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/paginate-search-results.html
