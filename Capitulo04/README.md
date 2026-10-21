# Administración de Elasticsearch para gestión y análisis de logs

## Práctica 4. Diseñar mappings y templates para logs de aplicación y validarlos con la API de simulación

### Objetivo

Al finalizar la práctica, serás capaz de:

- Definir mappings explícitos con los tipos de campo adecuados para logs: `keyword`, `text`, `date`, `ip`, `long`, `boolean`, multi-fields y `flattened`.
- Probar analizadores con la API `_analyze` y crear un analizador personalizado para mensajes en español.
- Usar dynamic templates y runtime fields, y prevenir el *mapping explosion*.
- Construir component templates e index templates componibles, y comprobar su prioridad y precedencia con la API de simulación.
- Corregir un mapping mediante la Reindex API y un cambio atómico de alias.

### Objetivo visual

![Composición de templates y reindexación](img/cap04.png)

### Duración aproximada

- 65 minutos.

### Prerrequisitos

- Práctica 3 completada: índice `app-logs-lab` con 40 documentos y los aliases `app-logs` y `app-logs-errores`.
- Kibana > Dev Tools abierto.

---

### Instrucciones

**Escenario:** en la Práctica 3 el mapping dinámico generó tipos poco adecuados: `source.ip` quedó como texto, `log.level` y `service.name` como `text` con subcampo `keyword`, y la búsqueda de `pedido` no encontró `pedidos`. Diseñarás un mapping correcto, lo empaquetarás en templates reutilizables, lo validarás con la API de simulación y migrarás los datos sin interrumpir a los usuarios del alias `app-logs`.

---

### Tarea 1. Mappings explícitos y tipos de campo para logs

#### Paso 1. Revisa los problemas del mapping dinámico

```http
GET app-logs-lab/_mapping/field/source.ip,log.level,message,response_time_ms
```

**Qué hace y por qué:** la API `_mapping/field` muestra el tipo de campos específicos. Verás que `source.ip` y `log.level` son `text` con subcampo `keyword`. Esto causa tres problemas: no se pueden hacer consultas por rango o subred sobre la IP, se ocupa espacio indexando dos veces el nivel de log, y las consultas exactas exigen recordar el sufijo `.keyword`.

#### Paso 2. Crea un índice con mapping explícito

```http
PUT mapping-demo
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "@timestamp":       { "type": "date" },
      "service":          { "properties": { "name": { "type": "keyword" } } },
      "host":             { "properties": { "name": { "type": "keyword" } } },
      "log":              { "properties": { "level": { "type": "keyword" } } },
      "message": {
        "type": "text",
        "fields": { "raw": { "type": "keyword", "ignore_above": 1024 } }
      },
      "source":           { "properties": { "ip": { "type": "ip" } } },
      "http":             { "properties": { "response": { "properties": { "status_code": { "type": "long" } } } } },
      "response_time_ms": { "type": "long" },
      "error":            { "properties": { "handled": { "type": "boolean" } } },
      "tags":             { "type": "keyword" }
    }
  }
}
```

**Qué hace y por qué:**

| Tipo | Campos | Razón |
|---|---|---|
| `keyword` | `service.name`, `host.name`, `log.level`, `tags` | Valores exactos y de baja cardinalidad: filtros, agregaciones `terms` y ordenamiento. No se analizan. |
| `text` | `message` | Texto libre para búsqueda de texto completo; se analiza en tokens. |
| multi-field `message.raw` | `message` | El mismo valor indexado además como `keyword` para agrupar mensajes idénticos. `ignore_above` evita indexar mensajes enormes, como los stack traces. |
| `date` | `@timestamp` | Rangos de tiempo, `date_histogram`, ordenamiento. |
| `ip` | `source.ip` | Consultas por subred (CIDR) y rangos de IP. Ocupa menos espacio que el texto. |
| `long` | códigos HTTP, latencias | Rangos y métricas (`avg`, `percentiles`). |
| `boolean` | `error.handled` | Valores verdadero o falso. |

`"dynamic": "strict"` rechaza cualquier documento que traiga un campo no declarado. Es la opción más estricta; en el Paso 11 verás alternativas menos rígidas.

#### Paso 3. Prueba la validación del mapping

```http
POST mapping-demo/_doc
{
  "@timestamp": "2026-10-20T10:00:00-06:00",
  "service": { "name": "auth-service" },
  "log": { "level": "WARN" },
  "message": "Intento de inicio de sesion fallido",
  "source": { "ip": "148.240.10.20" },
  "error": { "handled": true }
}

POST mapping-demo/_doc
{
  "@timestamp": "2026-10-20T10:01:00-06:00",
  "source": { "ip": "no-es-una-ip" }
}

POST mapping-demo/_doc
{
  "@timestamp": "2026-10-20T10:02:00-06:00",
  "campo_nuevo": "valor"
}
```

**Qué hace y por qué:** el primer documento se indexa. El segundo falla con `document_parsing_exception` (causa: `failed to parse field [source.ip] of type [ip]`) porque `no-es-una-ip` no es una IP válida. El tercero falla con `strict_dynamic_mapping_exception` porque `campo_nuevo` no está declarado. Un mapping explícito protege la calidad de los datos. Para no perder eventos con algún campo malformado, existe el parámetro `ignore_malformed: true`: el documento se indexa y el valor inválido se omite del índice.

#### Paso 4. Aprovecha el tipo `ip`

```http
GET mapping-demo/_search
{
  "query": { "term": { "source.ip": "148.240.0.0/16" } }
}
```

**Qué hace y por qué:** con el tipo `ip`, una consulta `term` acepta notación **CIDR** y encuentra todas las direcciones de la subred `148.240.x.x`. Con un campo de texto esto sería imposible. Es fundamental para el análisis de seguridad, por ejemplo, para buscar todo el tráfico de una red sospechosa.

---

### Tarea 2. Analizadores y la API `_analyze`

Un **analizador** transforma el texto en tokens al indexar y al buscar. Se compone de *character filters* (opcionales), un **tokenizer** y *token filters* (minúsculas, stop words, raíces, etc.).

#### Paso 5. Compara el analizador `standard` con el analizador `spanish`

```http
POST _analyze
{
  "analyzer": "standard",
  "text": "Timeout al conectar con la Base de Datos de pedidos"
}

POST _analyze
{
  "analyzer": "spanish",
  "text": "Timeout al conectar con la Base de Datos de pedidos"
}
```

**Qué hace y por qué:** `standard` separa por palabras y convierte a minúsculas: `timeout`, `al`, `conectar`, `con`, `la`, `base`, `de`, `datos`, `de`, `pedidos`. `spanish` además elimina *stop words* (`al`, `con`, `la`, `de`) y reduce las palabras a su raíz (`pedidos` → `pedid`, `conectar` → `conect`). Así, una búsqueda de `pedido` también encuentra `pedidos`.

#### Paso 6. Arma un analizador a la medida y pruébalo

```http
POST _analyze
{
  "tokenizer": "standard",
  "filter": [
    "lowercase",
    "asciifolding",
    { "type": "stop", "stopwords": "_spanish_" },
    { "type": "stemmer", "language": "light_spanish" }
  ],
  "text": "Sesión expirada: no se pudo actualizar los pedidos del carrito"
}
```

**Qué hace y por qué:** `asciifolding` quita acentos (`sesión` → `sesion`). Es importante porque muchos logs se escriben sin acentos y los usuarios buscan con ellos, o al revés. `light_spanish` es un *stemmer* menos agresivo que el del analizador `spanish`, lo que reduce falsos positivos. Probar con `_analyze` antes de crear el índice evita reindexar por un analizador mal diseñado.

#### Paso 7. Prueba un tokenizer especializado para rutas URL

```http
POST _analyze
{
  "tokenizer": "path_hierarchy",
  "text": "/api/catalog/items/1452"
}
```

**Qué hace y por qué:** `path_hierarchy` genera `/api`, `/api/catalog`, `/api/catalog/items` y `/api/catalog/items/1452`. Permite contar o filtrar logs web por cualquier nivel de la ruta; por ejemplo, todo lo que esté bajo `/api/catalog`.

---

### Tarea 3. Dynamic templates, runtime fields y prevención del mapping explosion

#### Paso 8. Define dynamic templates

```http
PUT dyn-demo
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "dynamic_templates": [
      { "ips":         { "path_match": "*.ip", "match_mapping_type": "string", "mapping": { "type": "ip" } } },
      { "etiquetas":   { "path_match": "labels.*", "match_mapping_type": "string", "mapping": { "type": "keyword" } } },
      { "cadenas_como_keyword": { "match_mapping_type": "string", "mapping": { "type": "keyword", "ignore_above": 1024 } } }
    ]
  }
}

POST dyn-demo/_doc?refresh=true
{
  "client": { "ip": "201.141.20.5" },
  "labels": { "equipo": "pagos", "region": "centro" },
  "environment": "produccion",
  "retries": 3
}

GET dyn-demo/_mapping
```

**Qué hace y por qué:** los dynamic templates deciden el tipo de los campos **nuevos** según reglas: por nombre (`match`), por ruta (`path_match`) o por tipo detectado (`match_mapping_type`). Se evalúan **en orden** y gana la primera regla que coincide. El resultado es `client.ip` → `ip`, `labels.*` → `keyword` y `environment` → `keyword` (en lugar de `text` + `keyword`). `retries` no coincide con ninguna regla y queda como `long`. Es el mecanismo que usan las integraciones de Elastic (`ecs@mappings`) para normalizar millones de campos sin declararlos uno por uno.

#### Paso 9. Crea runtime fields en la consulta

```http
GET app-logs-lab/_search
{
  "runtime_mappings": {
    "response_time_s": {
      "type": "double",
      "script": "if (doc['response_time_ms'].size() > 0) { emit(doc['response_time_ms'].value / 1000.0); }"
    },
    "status_class": {
      "type": "keyword",
      "script": "if (doc['http.response.status_code'].size() > 0) { emit((doc['http.response.status_code'].value / 100) + 'xx'); }"
    }
  },
  "query": { "term": { "status_class": "5xx" } },
  "fields": ["response_time_s", "status_class", "service.name"],
  "_source": false
}
```

**Qué hace y por qué:** un **runtime field** se calcula al consultar ("schema on read") y no ocupa espacio en el índice. Permite agregar campos sin reindexar, por ejemplo, para corregir o derivar datos de logs ya almacenados. Su costo es que cada consulta ejecuta el script, por lo que conviene volverlo un campo indexado si se usa con frecuencia. La consulta devuelve los 6 eventos 5xx con la latencia en segundos.

#### Paso 10. Extrae datos del mensaje con un runtime field con grok

```http
GET app-logs-lab/_search
{
  "runtime_mappings": {
    "producto_id": {
      "type": "keyword",
      "script": """
        if (doc['message.keyword'].size() > 0) {
          def m = grok('%{GREEDYDATA} producto %{NUMBER:id}').extract(doc['message.keyword'].value);
          if (m != null) { emit(m.id); }
        }
      """
    }
  },
  "query": { "exists": { "field": "producto_id" } },
  "fields": ["producto_id", "message"],
  "_source": false
}
```

**Qué hace y por qué:** `grok` dentro de un runtime field extrae partes de un texto con patrones, aquí el número de producto de los mensajes "Stock bajo para el producto 1452". Las triples comillas `"""` de Console permiten escribir scripts de varias líneas. Es una forma rápida de investigar un dato que no se extrajo al ingerir. En la Práctica 5 harás lo mismo de forma permanente con un ingest pipeline.

#### Paso 11. Provoca y previene un mapping explosion

```http
PUT explosion-demo
{
  "settings": {
    "number_of_shards": 1, "number_of_replicas": 0,
    "index.mapping.total_fields.limit": 10
  }
}

POST explosion-demo/_doc
{
  "a1":"x","a2":"x","a3":"x","a4":"x","a5":"x","a6":"x","a7":"x","a8":"x"
}
```

El documento falla con `Limit of total fields [10] has been exceeded`. Ahora corrige el diseño:

```http
DELETE explosion-demo

PUT explosion-demo
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0, "index.mapping.total_fields.limit": 10 },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "labels":     { "type": "flattened" }
    }
  }
}

POST explosion-demo/_doc?refresh=true
{
  "@timestamp": "2026-10-20T10:00:00-06:00",
  "labels": { "a1":"x","a2":"x","a3":"x","a4":"x","a5":"x","a6":"x","a7":"x","a8":"x","a9":"x","a10":"x","a11":"x" }
}

GET explosion-demo/_search
{ "query": { "term": { "labels.a11": "x" } } }
```

**Qué hace y por qué:** cada campo nuevo agrega una entrada al mapping, que vive en el *cluster state* y se replica a todos los nodos. Logs con claves arbitrarias (IDs como nombres de campo, encabezados HTTP, etiquetas de contenedores) pueden generar miles de campos y degradar todo el clúster: eso es el **mapping explosion**. Defensas disponibles:

- `index.mapping.total_fields.limit` (1000 por defecto), como límite de seguridad.
- Tipo `flattened`: todo el objeto ocupa **un solo** campo en el mapping y sus claves siguen siendo consultables como `keyword`.
- `"dynamic": false`: los campos nuevos se guardan en `_source`, pero no se indexan.
- `"dynamic": "runtime"`: los campos nuevos se crean como runtime fields.
- `index.mapping.total_fields.ignore_dynamic_beyond_limit: true`: ignora los campos que exceden el límite en lugar de rechazar el documento.

---

### Tarea 4. Component templates e index templates componibles

Un **index template** define settings, mappings y aliases que se aplican automáticamente al **crear** un índice cuyo nombre coincide con sus `index_patterns`. Los **component templates** son bloques reutilizables que un index template combina con `composed_of`.

#### Paso 12. Crea el component template de configuración base

```http
PUT _component_template/ct-tienda-base
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "5s",
      "index.mapping.total_fields.limit": 500
    },
    "mappings": {
      "dynamic_templates": [
        { "ips":     { "path_match": "*.ip", "match_mapping_type": "string", "mapping": { "type": "ip" } } },
        { "cadenas": { "match_mapping_type": "string", "mapping": { "type": "keyword", "ignore_above": 1024 } } }
      ],
      "properties": {
        "@timestamp":       { "type": "date" },
        "service":          { "properties": { "name": { "type": "keyword" } } },
        "host":             { "properties": { "name": { "type": "keyword" } } },
        "log":              { "properties": { "level": { "type": "keyword" } } },
        "user":             { "properties": { "name": { "type": "keyword" } } },
        "source":           { "properties": { "ip": { "type": "ip" } } },
        "http":             { "properties": { "response": { "properties": { "status_code": { "type": "long" } } } } },
        "response_time_ms": { "type": "long" },
        "tags":             { "type": "keyword" },
        "labels":           { "type": "flattened" }
      },
      "runtime": {
        "status_class": {
          "type": "keyword",
          "script": "if (doc['http.response.status_code'].size() > 0) { emit((doc['http.response.status_code'].value / 100) + 'xx'); }"
        }
      }
    }
  },
  "_meta": { "descripcion": "Settings y campos comunes de logs de la tienda", "version": 1 }
}
```

**Qué hace y por qué:** agrupa lo que comparten todos los índices de logs de la tienda: settings, campos ECS con su tipo correcto, dynamic templates y un runtime field. Si mañana cambia una regla, se actualiza **un** component template y todos los index templates que lo usan heredan el cambio en los **nuevos** índices.

#### Paso 13. Crea el component template de análisis de texto

```http
PUT _component_template/ct-tienda-texto
{
  "template": {
    "settings": {
      "analysis": {
        "filter": {
          "es_stop": { "type": "stop", "stopwords": "_spanish_" },
          "es_stem": { "type": "stemmer", "language": "light_spanish" }
        },
        "analyzer": {
          "analizador_logs_es": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["lowercase", "asciifolding", "es_stop", "es_stem"]
          }
        }
      }
    },
    "mappings": {
      "properties": {
        "message": {
          "type": "text",
          "analyzer": "analizador_logs_es",
          "fields": { "raw": { "type": "keyword", "ignore_above": 1024 } }
        }
      }
    }
  }
}
```

**Qué hace y por qué:** el analizador y el campo que lo usa se definen en el **mismo** component template, porque cada component template se valida por separado y un mapping no puede referirse a un analizador que no conoce.

#### Paso 14. Crea el index template

```http
PUT _index_template/it-tienda-logs
{
  "index_patterns": ["tienda-logs-*"],
  "priority": 200,
  "composed_of": ["ct-tienda-base", "ct-tienda-texto"],
  "template": {
    "settings": { "refresh_interval": "2s" },
    "aliases": { "tienda-logs": {} }
  },
  "_meta": { "responsable": "equipo-operaciones" }
}
```

**Qué hace y por qué:** el template se aplica a cualquier índice nuevo cuyo nombre empiece con `tienda-logs-`. **Precedencia** al construir el índice, de menor a mayor:

1. Los component templates, en el orden de `composed_of`: un componente posterior sobrescribe a uno anterior.
2. El bloque `template` del propio index template.
3. El cuerpo de la petición de creación del índice, si se crea manualmente.

Por eso el `refresh_interval` final será `2s` y no `5s`.

#### Paso 15. Simula el índice resultante antes de crearlo

```http
POST _index_template/_simulate_index/tienda-logs-2026.10.20
```

**Qué hace y por qué:** la API de **simulación** devuelve los settings, mappings y aliases que tendría el índice **sin crearlo**. Verifica en la respuesta:

- `settings.index.refresh_interval` = `"2s"` (sobrescrito por el index template).
- `settings.index.analysis.analyzer.analizador_logs_es` presente.
- `mappings.properties.source.ip.type` = `"ip"` y `message.analyzer` = `"analizador_logs_es"`.
- `aliases.tienda-logs`.

Valida siempre un template con `_simulate_index` antes de que lleguen datos reales. Un error en un template de logs se descubre, si no, con el primer rollover a medianoche.

#### Paso 16. Comprueba la prioridad entre templates que se traslapan

```http
PUT _index_template/it-tienda-logs-pagos
{
  "index_patterns": ["tienda-logs-pagos-*"],
  "priority": 300,
  "composed_of": ["ct-tienda-base", "ct-tienda-texto"],
  "template": {
    "settings": { "refresh_interval": "30s" },
    "mappings": {
      "properties": {
        "payment": { "properties": { "amount": { "type": "scaled_float", "scaling_factor": 100 } } }
      }
    }
  }
}

POST _index_template/_simulate_index/tienda-logs-pagos-2026.10.20
```

Ahora intenta crear un template con la **misma prioridad** y patrones que se traslapan:

```http
PUT _index_template/it-conflicto
{
  "index_patterns": ["tienda-logs-*"],
  "priority": 200,
  "template": { "settings": { "number_of_replicas": 1 } }
}
```

**Qué hace y por qué:** el índice `tienda-logs-pagos-2026.10.20` coincide con **ambos** templates. Elasticsearch aplica **solo uno**, el de mayor `priority` (300); los index templates **no** se mezclan entre sí. En la simulación, el campo `overlapping` lista el template de menor prioridad que fue ignorado. El último `PUT` falla porque dos templates con patrones traslapados no pueden tener la misma prioridad: el resultado sería ambiguo. Una buena convención es usar prioridades separadas por decenas o centenas y documentarlas.

---

### Tarea 5. Cambios de mapping y Reindex API

#### Paso 17. Intenta cambiar el tipo de un campo existente

```http
PUT app-logs-lab/_mapping
{
  "properties": { "source": { "properties": { "ip": { "type": "ip" } } } }
}

PUT app-logs-lab/_mapping
{
  "properties": { "pipeline_version": { "type": "keyword" } }
}
```

**Qué hace y por qué:** el primer cambio falla (`mapper [source.ip] cannot be changed from type [text] to [ip]`): el tipo de un campo existente **no** se puede modificar, porque los datos ya están indexados con la estructura anterior. Sí se permiten cambios compatibles, como **agregar** campos nuevos (segundo `PUT`), agregar multi-fields o cambiar `ignore_above`. Para cambiar un tipo hay que crear un índice nuevo y **reindexar**.

#### Paso 18. Reindexa hacia un índice gobernado por el template

```http
POST _reindex?wait_for_completion=true
{
  "source": { "index": "app-logs-lab" },
  "dest":   { "index": "tienda-logs-2026.10.20", "op_type": "create" }
}

GET tienda-logs-2026.10.20/_count
GET tienda-logs-2026.10.20/_mapping/field/source.ip,log.level,environment,message
```

**Qué hace y por qué:** `_reindex` lee los documentos (`_source`) del índice origen y los indexa en el destino. Como el destino no existe y su nombre coincide con `tienda-logs-*`, se crea **con el template**: `source.ip` es `ip`, `log.level` es `keyword`, `environment` es `keyword` (por el dynamic template) y `message` usa el analizador en español. `op_type: create` evita sobrescribir si el documento ya existía. Para volúmenes grandes se ejecuta con `wait_for_completion=false` y se sigue con la API `_tasks`.

#### Paso 19. Cambia los aliases de forma atómica

```http
POST _aliases
{
  "actions": [
    { "remove": { "index": "app-logs-lab", "alias": "app-logs" } },
    { "add":    { "index": "tienda-logs-2026.10.20", "alias": "app-logs" } },
    { "remove": { "index": "app-logs-lab", "alias": "app-logs-errores" } },
    { "add":    { "index": "tienda-logs-2026.10.20", "alias": "app-logs-errores",
                  "filter": { "term": { "log.level": "ERROR" } } } }
  ]
}

GET _cat/aliases/app-logs*,tienda-logs?v
```

**Qué hace y por qué:** quien consulta `app-logs` empieza a leer el nuevo índice **sin cambiar nada** y sin un instante en que el alias no exista, porque todas las acciones se aplican juntas. El filtro del alias de errores ahora usa `log.level` directamente, ya que el campo es `keyword`.

#### Paso 20. Comprueba las mejoras sobre el alias

```http
GET app-logs/_count
{ "query": { "match": { "message": "pedido" } } }

GET app-logs/_count
{ "query": { "term": { "log.level": "ERROR" } } }

GET app-logs/_search
{
  "query": { "term": { "source.ip": "89.160.20.0/24" } },
  "_source": ["user.name", "message", "source.ip"]
}

GET app-logs/_search
{
  "size": 0,
  "aggs": { "por_clase": { "terms": { "field": "status_class" } } }
}
```

**Qué hace y por qué:**

- La búsqueda de `pedido` ahora regresa **9** documentos (en la Práctica 3 eran 7), porque el analizador en español también encuentra `pedidos`.
- `term log.level: ERROR` regresa **8** sin el sufijo `.keyword`.
- La consulta CIDR encuentra los **2** intentos de inicio de sesión de la red `89.160.20.0/24` (usuarios `admin` y `root`).
- La agregación usa el runtime field `status_class` definido en el component template: 28 `2xx`, 6 `4xx` y 6 `5xx`, sin haberlo indexado.

#### Paso 21. Limpia los índices de prueba

```http
DELETE mapping-demo
DELETE dyn-demo
DELETE explosion-demo
DELETE _index_template/it-tienda-logs-pagos
```

**Qué hace y por qué:** elimina los objetos que solo sirvieron para experimentar. Conserva `app-logs-lab`, `tienda-logs-2026.10.20`, los component templates y el template `it-tienda-logs`: se usan en prácticas posteriores.

---

### Resultado esperado

- Los component templates `ct-tienda-base` y `ct-tienda-texto` y el index template `it-tienda-logs` existen:

```http
GET _component_template/ct-tienda-*?filter_path=component_templates.name
GET _index_template/it-tienda-logs?filter_path=index_templates.name,index_templates.index_template.composed_of
```

- `POST _index_template/_simulate_index/tienda-logs-2026.10.20` muestra `source.ip` de tipo `ip`, el analizador `analizador_logs_es` y el alias `tienda-logs`.
- El índice `tienda-logs-2026.10.20` contiene **40** documentos y los aliases `app-logs`, `app-logs-errores` y `tienda-logs` apuntan a él:

```text
alias            index                  filter
app-logs         tienda-logs-2026.10.20 -
app-logs-errores tienda-logs-2026.10.20 *
tienda-logs      tienda-logs-2026.10.20 -
```

- Puedes explicar por qué el tipo de un campo no puede cambiarse en sitio, cómo se resuelve con reindex y alias, y cómo se decide qué template se aplica cuando hay traslapes.

### Referencias

- Elastic. *Mapping [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/mapping.html
- Elastic. *Text analysis [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/analysis.html
- Elastic. *Dynamic templates [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/dynamic-templates.html
- Elastic. *Runtime fields [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/runtime.html
- Elastic. *Index templates y Simulate index API [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/index-templates.html
- Elastic. *Reindex API [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/docs-reindex.html
