# Laboratorio 3: Crear índices y ajustar mapeos para logs

- **Capítulo:** 3. Índices, shards y mapeos avanzados

- **Duración:** 54 min

- **Entorno:** Elastic Cloud Hosted (cuenta propia), Elasticsearch 9.x, Kibana Dev Tools

- **Dependencias:** Laboratorio 2 (índice `lab02-weblogs` con 6 documentos). No requiere los índices del Laboratorio 1.

## Objetivo

Partir del índice `lab02-weblogs`, creado con mapeo dinámico, y construir el modelo definitivo para los logs del curso: un mapping explícito con multi-fields, un analyzer personalizado, templates reutilizables, un índice con alias y un data stream de prueba. Al terminar, el participante podrá:

- Identificar los problemas de un mapeo dinámico y decidir el tipo correcto de cada campo.

- Comparar analyzers con la API `_analyze` y definir un analyzer personalizado.

- Crear component templates e index templates y comprobar su efecto con `_simulate_index`.

- Crear un índice a partir de un template, asignarle un alias y poblarlo con `_reindex`.

- Crear un data stream y observar sus backing indices.

- Dejar listo el índice `weblogs-v1` (alias `weblogs`) y los templates que usan los laboratorios 4 y 5.

## Requisitos

- `lab02-weblogs` con 6 documentos. Verificar con `GET lab02-weblogs/_count`; debe devolver `count: 6`.

- Usuario `elastic` en Kibana Dev Tools (Console).

## Distribución del tiempo

| **Parte** | **Contenido**                        | **Tiempo** |
|-----------|--------------------------------------|------------|
| Parte A   | Analizar el mapping dinámico         | 8 min      |
| Parte B   | Comparar analyzers con `_analyze`    | 12 min     |
| Parte C   | Component templates e index template | 14 min     |
| Parte D   | Índice, alias y reindex              | 12 min     |
| Parte E   | Data stream de prueba                | 8 min      |

## Convenciones

- Todos los comandos se ejecutan en Kibana Dev Tools (Console).

- Cada comando va seguido de lo que hace y del significado de sus parámetros.

- Los nombres siguen el prefijo `weblogs-`; los que se conservan para otros laboratorios se indican en la tabla final.

## Parte A: analizar el mapping dinámico (8 min)

### Paso A1. Revisar el índice de partida

``` console
GET _cat/indices/lab02-weblogs?v&h=index,health,pri,rep,docs.count,store.size

GET lab02-weblogs/_mapping
```

**Qué hace:** muestra el tamaño del índice y el mapping que Elasticsearch generó automáticamente al indexar el primer documento.

### Paso A2. Decidir el tipo de cada campo

Completar la columna de decisión y comparar con la propuesta:

| **Campo**          | **Mapping dinámico** | **Propuesta**                             | **Motivo**                                                           |
|--------------------|----------------------|-------------------------------------------|----------------------------------------------------------------------|
| `@timestamp`       | `date`               | `date`                                    | Correcto.                                                            |
| `service`          | `text` + `.keyword`  | `keyword`                                 | Solo se filtra y agrega por valor exacto.                            |
| `client_ip`        | `text` + `.keyword`  | `ip`                                      | Permite búsquedas por rango y por CIDR.                              |
| `method`           | `text` + `.keyword`  | `keyword`                                 | Valor exacto de pocos valores.                                       |
| `url`              | `text` + `.keyword`  | `keyword` + subcampo `.path` (text)       | Exacto para agregar; analyzer de ruta para buscar por prefijo.       |
| `status`           | `long`               | `short`                                   | Rango 100 a 599; ahorra espacio y permite rangos.                    |
| `bytes`            | `long`               | `integer` con `index: false`              | Se agrega y se suma; consultar por valor es posible, pero más lento. |
| `response_time_ms` | `float`              | `float`                                   | Correcto.                                                            |
| `message`          | `text` + `.keyword`  | `text` + `.keyword` (`ignore_above: 512`) | Búsqueda de texto completo y valor exacto.                           |

## Parte B: comparar analyzers con \_analyze (12 min)

### Paso B1. Analyzer standard

``` console
GET _analyze
{
  "analyzer": "standard",
  "text": "GET /api/products/9999 404 product not found"
}
```

**Qué hace:** muestra los tokens que genera un analyzer para un texto, sin necesidad de indexarlo.

- `analyzer`: analyzer a probar (`standard` es el predeterminado de los campos `text`).

- `text`: cadena de entrada.

**Resultado esperado:** los tokens `get`, `api`, `products`, `9999`, `404`, `product`, `not`, `found`. El analyzer divide en las barras, quita la puntuación y pasa a minúsculas; la ruta original deja de existir como unidad.

### Paso B2. Analyzer keyword y tokenizer path_hierarchy

``` console
GET _analyze
{
  "analyzer": "keyword",
  "text": "/api/products/9999"
}

GET _analyze
{
  "tokenizer": "path_hierarchy",
  "text": "/api/products/9999"
}
```

**Qué hace:** compara dos formas de tratar una ruta.

- `keyword`: devuelve un solo token con el texto completo.

- `tokenizer`: en lugar de un analyzer completo, prueba un tokenizer directamente.

- `path_hierarchy`: genera un token por cada nivel de la ruta.

**Resultado esperado:** `keyword` devuelve `/api/products/9999`. `path_hierarchy` devuelve tres tokens: `/api`, `/api/products` y `/api/products/9999`.

### Paso B3. Cadena de filtros para texto en español

``` console
GET _analyze
{
  "tokenizer": "standard",
  "char_filter": ["html_strip"],
  "filter": ["lowercase", "asciifolding"],
  "text": "<b>Teléfono</b> PAGO Rechazado"
}
```

**Qué hace:** arma un analyzer temporal con sus tres etapas.

- `char_filter`: transforma el texto antes de dividirlo. `html_strip` elimina las etiquetas HTML.

- `tokenizer`: divide el texto en tokens.

- `filter` (token filters): transforman los tokens. `lowercase` pasa a minúsculas y `asciifolding` quita acentos.

**Resultado esperado:** `telefono`, `pago`, `rechazado`.

## Parte C: component templates e index template (14 min)

### Paso C1. Component template de settings

``` console
PUT _component_template/weblogs-settings
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "analysis": {
        "analyzer": {
          "url_path_analyzer": {
            "type": "custom",
            "tokenizer": "path_hierarchy",
            "filter": ["lowercase"]
          }
        }
      }
    }
  }
}
```

**Qué hace:** guarda un bloque reutilizable de settings. Un component template no se aplica por sí solo; se compone dentro de un index template.

- `number_of_shards` y `number_of_replicas`: un primary y una réplica, suficiente para el volumen del curso.

- `analysis.analyzer`: define el analyzer `url_path_analyzer` con el tokenizer `path_hierarchy` y un filtro `lowercase`. Los analyzers personalizados se definen en los settings del índice.

### Paso C2. Component template de mappings

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
        "url": {
          "type": "keyword",
          "ignore_above": 512,
          "fields": {
            "path": { "type": "text", "analyzer": "url_path_analyzer" }
          }
        },
        "status":           { "type": "short" },
        "bytes":            { "type": "integer", "index": false },
        "response_time_ms": { "type": "float" },
        "message": {
          "type": "text",
          "fields": {
            "keyword": { "type": "keyword", "ignore_above": 512 }
          }
        }
      }
    }
  }
}
```

**Qué hace:** define el mapping explícito de los logs.

- `fields`: multi-field; el mismo valor se indexa de más de una forma. `url` es `keyword` y `url.path` es `text` con el analyzer de ruta.

- `ignore_above: 512`: los valores más largos no se indexan como `keyword` (siguen en `_source`).

- `index: false` en `bytes`: no crea la estructura de búsqueda rápida. El campo sigue guardando `doc_values`, de modo que se puede agregar y ordenar. En los tipos numéricos también se puede consultar con `term` o `range`, pero recorriendo los valores, más lento que con el índice.

- `status` como `short` y `client_ip` como `ip`: tipos más compactos y más precisos que los generados dinámicamente.

### Paso C3. Index template

``` console
PUT _index_template/weblogs-template
{
  "index_patterns": ["weblogs-v*"],
  "priority": 200,
  "composed_of": ["weblogs-settings", "weblogs-mappings"],
  "_meta": { "description": "Logs de acceso web del curso" }
}
```

**Qué hace:** asocia los dos component templates a todos los índices cuyo nombre empiece con `weblogs-v`.

- `index_patterns`: patrones de nombre a los que aplica. Solo actúa al crear el índice; no modifica índices existentes.

- `priority`: si un índice coincide con más de un template, gana el de mayor prioridad. Dos templates con patrones solapados no pueden tener la misma prioridad.

- `composed_of`: lista ordenada de component templates. Si hay conflicto, el último prevalece.

- `_meta`: información libre para documentar el template.

### Paso C4. Comprobar el resultado sin crear el índice

``` console
POST _index_template/_simulate_index/weblogs-v1
```

**Qué hace:** devuelve los settings, mappings y aliases que recibiría un índice llamado `weblogs-v1`, sin crearlo.

**Resultado esperado:** aparecen `number_of_shards: "1"`, el analyzer `url_path_analyzer` y todos los campos del mapping. En `overlapping` no debe figurar ningún otro template.

## Parte D: índice, alias y reindex (12 min)

### Paso D1. Crear el índice con su alias

``` console
PUT weblogs-v1
{
  "aliases": {
    "weblogs": {}
  }
}
```

**Qué hace:** crea `weblogs-v1`. El cuerpo no incluye settings ni mappings: los toma del template. Además crea el alias `weblogs`.

- `aliases`: nombres alternativos del índice. Las aplicaciones usan `weblogs`; el índice físico puede cambiar de versión (`weblogs-v2`) sin que los clientes lo noten. Este patrón se usa en el capítulo 9.

``` console
GET weblogs-v1/_mapping/field/client_ip

GET weblogs-v1/_settings?filter_path=*.settings.index.number_of_shards,*.settings.index.analysis
```

**Qué hace:** confirma que el índice heredó el mapping y los settings. `filter_path` limita la respuesta a las rutas indicadas.

### Paso D2. Poblar el índice con \_reindex

``` console
POST _reindex?refresh=true
{
  "source": { "index": "lab02-weblogs" },
  "dest":   { "index": "weblogs-v1" }
}
```

**Qué hace:** copia los documentos de un índice a otro. Como el destino tiene un mapping distinto, cada campo se convierte al tipo nuevo. La API se estudia a detalle en el capítulo 4.

- `source.index`: índice de origen.

- `dest.index`: índice de destino.

- `refresh=true`: refresca el destino al terminar, para consultar de inmediato.

- Los `_id` se conservan. Esto permite repetir la operación más adelante sobre los mismos documentos.

**Resultado esperado:** `total: 6`, `created: 6`, `failures: []`.

### Paso D3. Comprobar el efecto de los tipos y del analyzer

``` console
GET weblogs/_search
{
  "query": { "term": { "client_ip": "187.141.0.0/16" } }
}

GET weblogs/_search
{
  "query": { "term": { "url.path": "/api/products" } }
}

GET weblogs-v1/_analyze
{
  "field": "url.path",
  "text": "/API/Cart/Items"
}
```

**Qué hace:** tres comprobaciones.

- Búsqueda por CIDR sobre `client_ip`. Solo es posible con el tipo `ip`.

- `term` sobre `url.path`: encuentra todas las rutas que cuelgan de `/api/products`, porque el analyzer indexó cada nivel como token.

- `_analyze` con `field`: usa el analyzer configurado en ese campo del índice.

**Resultado esperado:** la primera búsqueda devuelve 2 documentos (la IP `187.141.22.10`); la segunda devuelve 2 (`/api/products` y `/api/products/9999`); `_analyze` devuelve `/api`, `/api/cart` y `/api/cart/items`.

## Parte E: data stream de prueba (8 min)

### Paso E1. Crear el template del data stream

``` console
PUT _index_template/weblogs-ds-template
{
  "index_patterns": ["weblogs-ds-*"],
  "data_stream": {},
  "priority": 200,
  "composed_of": ["weblogs-settings", "weblogs-mappings"]
}
```

**Qué hace:** reutiliza los mismos component templates para un data stream.

- `data_stream: {}`: convierte el patrón en data streams. Es obligatorio y exige un campo `@timestamp`, que el mapping ya define.

- El patrón `weblogs-ds-*` no se solapa con `weblogs-v*`. No usar nombres `logs-*-*`: ese patrón lo reservan los templates integrados de Elastic.

### Paso E2. Crear el data stream indexando un documento

``` console
POST weblogs-ds-test/_doc
{
  "@timestamp": "2026-10-14T10:00:00Z",
  "service": "checkout",
  "client_ip": "187.141.22.10",
  "method": "GET",
  "url": "/api/cart",
  "status": 200,
  "bytes": 900,
  "response_time_ms": 35.0,
  "message": "GET /api/cart 200 35ms"
}

GET _data_stream/weblogs-ds-test
```

**Qué hace:** al indexar en un nombre que coincide con un template de data stream, Elasticsearch crea el data stream y su primer backing index.

Campos a revisar en la respuesta de `_data_stream`:

- `indices[].index_name`: backing index, con el formato `.ds-weblogs-ds-test-<fecha UTC>-000001`.

- `generation`: 1.

- `timestamp_field`: `@timestamp`.

- `template`: `weblogs-ds-template`.

### Paso E3. Comprobar las restricciones y hacer un rollover

``` console
PUT weblogs-ds-test/_doc/1
{ "@timestamp": "2026-10-14T10:05:00Z", "service": "checkout" }

POST weblogs-ds-test/_rollover

GET _data_stream/weblogs-ds-test

GET weblogs-ds-test/_search?size=0
```

**Qué hace:** la primera petición intenta indexar con `PUT` e identificador; la segunda crea un nuevo backing index.

- Primera petición: `400 Bad Request`. Los data streams solo admiten escrituras `create` (solo agregar). Sirven para datos que no se modifican.

- `_rollover`: crea el backing index `...-000002`. Las escrituras nuevas van a él; las búsquedas abarcan todos.

- Después del rollover, `generation` pasa a 2 y la búsqueda sigue devolviendo 1 documento.

## Limpieza y estado final

``` console
DELETE _data_stream/weblogs-ds-test

DELETE _index_template/weblogs-ds-template
```

**Qué hace:** elimina el data stream (y sus backing indices) y su template. Se vuelven a crear con ILM en el laboratorio 10.

| **Elemento**                                                | **Se conserva** | **Lo reutiliza**                     |
|-------------------------------------------------------------|-----------------|--------------------------------------|
| `weblogs-v1` (alias `weblogs`) con 6 documentos             | Sí              | Laboratorios 4, 5 y 6                |
| `weblogs-settings`, `weblogs-mappings` y `weblogs-template` | Sí              | Laboratorios 4, 6 y 9                |
| `lab02-weblogs`                                             | Sí              | Laboratorio 4 (reindex con pipeline) |
| `weblogs-ds-test` y `weblogs-ds-template`                   | No              | Solo para practicar                  |

Verificación final:

``` console
GET _cat/indices/weblogs-*,lab02-*?v&h=index,health,pri,rep,docs.count

GET _alias/weblogs

GET _index_template/weblogs-template?filter_path=index_templates.name,index_templates.index_template.composed_of
```

Debe verse `weblogs-v1` con 6 documentos, el alias `weblogs` y el template compuesto por los dos component templates.

## Preguntas de discusión

- ¿Qué ventaja tiene guardar `url` como `keyword` con un subcampo `text` en lugar de un solo `text`?

- ¿Qué ocurre si más adelante se necesita cambiar `client_ip` a otro tipo? ¿Por qué no basta con modificar el mapping?

- ¿Qué cambia para `bytes` al poner `index: false`? ¿Qué se conserva y qué se vuelve más lento?

- ¿Qué ventajas tiene que las aplicaciones usen el alias `weblogs` en lugar del nombre del índice?

- ¿Por qué un data stream rechaza `PUT` con identificador? ¿Para qué tipo de datos es adecuado?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 3: Crear índices y ajustar mapeos para logs](imagenes/lab03_resultado_final.png)
