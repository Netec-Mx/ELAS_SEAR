# Laboratorio 7: Configurar snapshots y políticas de ILM

- **Capítulo:** 7. Seguridad, backups y gestión de datos

- **Duración:** 54 min

- **Entorno:** Elastic Cloud Hosted (cuenta propia), Elasticsearch 9.x, Kibana Dev Tools y una terminal con `curl`

- **Dependencias:** Laboratorio 1 (`orders-shared` con 6 documentos de 3 tenants) y Laboratorio 3 (`weblogs-v1` con el alias `weblogs`).

## Objetivo

Proteger y gobernar los datos del curso: aislar a dos tenants con seguridad a nivel de documento (DLS) y de campo (FLS), configurar snapshots y restaurar datos, y definir una política de ILM sobre un data stream. Al terminar, el participante podrá:

- Crear roles y usuarios con DLS y FLS, y comprobar un acceso autorizado, uno no autorizado (403) y una autenticación fallida (401).

- Crear una API key restringida a un tenant.

- Tomar un snapshot manual en el repositorio de Elastic Cloud, crear una política de snapshots (SLM) y restaurar datos con otro nombre.

- Definir una política de ILM con rollover, fase warm y borrado, aplicarla a un data stream y seguir su avance con `_ilm/explain`.

- Documentar la matriz de usuarios, roles y tenants.

## Requisitos

- `GET orders-shared/_count` devuelve 6 y `GET weblogs/_count` devuelve 5006.

- Los alias `orders-acme`, `orders-globex` y `orders-initech` existen (`GET _alias/orders-*`).

- La dirección de Elasticsearch (`<ES_URL>`) anotada en la ficha del deployment.

- Este laboratorio usa DLS y FLS, que son funciones de pago. La prueba de Elastic Cloud las incluye mientras esté vigente (paso A1 lo comprueba).

## Distribución del tiempo

| **Parte** | **Contenido**                                  | **Tiempo** |
|-----------|------------------------------------------------|------------|
| Parte A   | Licencia y punto de partida                    | 4 min      |
| Parte B   | Dos tenants: DLS, FLS y accesos no autorizados | 14 min     |
| Parte C   | ILM sobre un data stream (se deja corriendo)   | 8 min      |
| Parte D   | Snapshots, restore y SLM                       | 14 min     |
| Parte E   | Resultado de ILM                               | 6 min      |
| Parte F   | Auditoría, gobernanza y limpieza               | 8 min      |

## Convenciones

- Los bloques con `GET`, `PUT`, `POST` o `DELETE` se ejecutan en Dev Tools; los que empiezan con `curl.exe` se ejecutan en la terminal.

- En los comandos `curl.exe`, `-u usuario` hace que `curl` pida la contraseña de forma interactiva. Así no queda guardada en el historial de la terminal. Las URL van entre comillas dobles.

- Las contraseñas de los usuarios de práctica deben tener al menos 6 caracteres. Usa valores que no emplees en ningún otro sitio.

- La Parte C se inicia antes que la D porque ILM trabaja por tiempo y avanza mientras se hacen los snapshots.

## Parte A: licencia y punto de partida (4 min)

### Paso A1. Licencia e identidad

``` console
GET _license

GET _security/_authenticate
```

**Qué hace:** muestra la licencia activa del clúster y el usuario con el que estás autenticado en Dev Tools.

- `license.type` y `license.status`: tipo de licencia y si está vigente (`active`). Anota ambos y la fecha en `expiry_date`.

- `_authenticate`: devuelve el usuario (`username`), sus roles (`roles`) y cómo se autenticó (`authentication_type`). Debe aparecer `elastic` con el rol `superuser`.

Si más adelante la creación de un rol con `query` o `field_security` falla con un error de licencia no conforme, la prueba o la suscripción no incluyen DLS y FLS. Avisa al instructor: la Parte B se hará como demostración.

### Paso A2. Agregar un campo sensible a los pedidos

``` console
PUT orders-shared/_mapping
{
  "properties": {
    "customer_email": { "type": "keyword" }
  }
}

POST orders-shared/_update_by_query?refresh=true&conflicts=proceed
{
  "script": {
    "lang": "painless",
    "source": "ctx._source.customer_email = ctx._source.tenant_id + '-' + ctx._source.order_id + '@example.com'"
  }
}
```

**Qué hace:** añade el campo `customer_email` al mapping y lo completa en los 6 documentos con valores ficticios, como `acme-2001@example.com`. Será el dato que FLS oculte.

- `PUT _mapping`: agrega campos a un índice existente. No cambia campos ya definidos.

- `_update_by_query`: actualiza cada documento que coincide con la consulta (sin `query`, todos).

- `script.source`: código Painless que se ejecuta por documento. `ctx._source` es el documento.

- `conflicts=proceed`: si un documento cambia durante la operación, continúa en lugar de abortar.

**Resultado esperado:** `"updated": 6`, `"failures": []`.

## Parte B: dos tenants con DLS y FLS (14 min)

### Paso B1. Rol del tenant acme (DLS)

``` console
PUT _security/role/orders_acme_reader
{
  "indices": [
    {
      "names": ["orders-shared"],
      "privileges": ["read"],
      "query": "{\"term\": {\"tenant_id\": \"acme\"}}"
    }
  ]
}
```

**Qué hace:** crea un rol que puede leer `orders-shared`, pero solo los documentos cuyo `tenant_id` es `acme`.

- `names`: índices, alias o patrones sobre los que aplica el permiso.

- `privileges: ["read"]`: permite buscar y obtener documentos. No permite escribir ni borrar.

- `query`: consulta en forma de texto JSON (por eso las comillas escapadas). Es la regla de DLS: el rol solo ve los documentos que cumplen la consulta, en búsquedas, conteos, agregaciones y lecturas por ID.

### Paso B2. Rol del tenant globex (DLS y FLS)

``` console
PUT _security/role/orders_globex_reader
{
  "indices": [
    {
      "names": ["orders-shared"],
      "privileges": ["read"],
      "query": "{\"term\": {\"tenant_id\": \"globex\"}}",
      "field_security": {
        "grant": ["tenant_id", "order_id", "total"]
      }
    }
  ]
}
```

**Qué hace:** igual que el anterior para `globex`, y además limita los campos visibles.

- `field_security.grant`: lista de campos que el rol puede ver. Todo campo no incluido (aquí, `customer_email`) se oculta: no aparece en `_source`, no se puede buscar, ni agregar.

- También existe `except`, que concede todos los campos menos los indicados. `grant` es más seguro porque un campo nuevo queda oculto por defecto.

### Paso B3. Usuarios de práctica

``` console
POST _security/user/ana_acme
{
  "password": "<CONTRASEÑA_ANA>",
  "roles": ["orders_acme_reader"],
  "full_name": "Ana (tenant acme)"
}

POST _security/user/gus_globex
{
  "password": "<CONTRASEÑA_GUS>",
  "roles": ["orders_globex_reader"],
  "full_name": "Gus (tenant globex)"
}
```

**Qué hace:** crea dos usuarios nativos, cada uno con el rol de su tenant. **Resultado esperado:** `"created": true` en ambos.

### Paso B4. Acceso autorizado

``` console
curl.exe -s -u ana_acme "<ES_URL>/orders-shared/_search?filter_path=hits.total.value,hits.hits._source"

curl.exe -s -u gus_globex "<ES_URL>/orders-shared/_search?filter_path=hits.total.value,hits.hits._source"
```

**Qué hace:** cada usuario consulta el mismo índice, `orders-shared`, que contiene los 6 documentos de los tres tenants.

- `-u ana_acme`: autenticación básica; `curl` pide la contraseña.

- `filter_path`: reduce la respuesta al total y a los `_source`.

**Resultado esperado:**

- `ana_acme`: `total.value` de 2 (pedidos 2001 y 2002), con los campos `tenant_id`, `order_id`, `total` y `customer_email`.

- `gus_globex`: `total.value` de 2 (pedidos 3001 y 3002), **sin** `customer_email`.

El mismo índice devuelve resultados distintos según quién pregunta. Es la ventaja frente a los alias filtrados: el filtro viaja con el usuario y no depende de que la aplicación use el alias correcto.

### Paso B5. Lo que cada usuario no puede ver

``` console
curl.exe -s -u ana_acme "<ES_URL>/orders-shared/_count?q=tenant_id:globex"

curl.exe -s -u gus_globex "<ES_URL>/orders-shared/_count?q=customer_email:globex-3001@example.com"

curl.exe -s -u gus_globex "<ES_URL>/orders-shared/_count"
```

**Qué hace:** intenta consultar datos fuera del alcance del usuario.

- `_count?q=...`: cuenta con una consulta de texto sencilla (`campo:valor`).

- Primera: `ana_acme` pregunta por pedidos de `globex`. **Resultado esperado:** `count` de 0. DLS filtra antes de evaluar la consulta; ni siquiera sabe si existen.

- Segunda: `gus_globex` busca por un campo oculto. **Resultado esperado:** `count` de 0, aunque el pedido existe. Con FLS, el campo no se puede usar para buscar.

- Tercera: **Resultado esperado:** `count` de 2. Cada usuario cuenta solo lo suyo.

### Paso B6. Acceso no autorizado y autenticación fallida

``` console
curl.exe -s -i -u ana_acme "<ES_URL>/weblogs/_search?size=0"

curl.exe -s -i -u ana_acme -X POST "<ES_URL>/orders-shared/_doc" -H "Content-Type: application/json" -d "{}"

curl.exe -s -i -u ana_acme:contraseña-incorrecta "<ES_URL>/orders-shared/_count"
```

**Qué hace:** tres intentos que deben fallar. `-i` muestra la línea de estado HTTP.

| **Intento**                 | **Estado** | **Interpretación**                                                                                                        |
|-----------------------------|------------|---------------------------------------------------------------------------------------------------------------------------|
| Leer `weblogs`              | 403        | El usuario está autenticado, pero su rol no incluye `weblogs`. El mensaje contiene `is unauthorized for user [ana_acme]`. |
| Escribir en `orders-shared` | 403        | El rol solo tiene `read`. Falta el privilegio `write`.                                                                    |
| Contraseña incorrecta       | 401        | La autenticación falla antes de evaluar permisos.                                                                         |

Regla para el diagnóstico: **401** es "no sé quién eres"; **403** es "sé quién eres, pero no tienes permiso".

### Paso B7. API key restringida a un tenant

``` console
POST _security/api_key
{
  "name": "orders-initech-app",
  "expiration": "1d",
  "role_descriptors": {
    "initech-reader": {
      "indices": [
        {
          "names": ["orders-shared"],
          "privileges": ["read"],
          "query": "{\"term\": {\"tenant_id\": \"initech\"}}"
        }
      ]
    }
  }
}
```

**Qué hace:** crea una credencial para una aplicación que solo debe ver los pedidos de `initech`. Copia el valor de `encoded`.

- `role_descriptors`: permisos de la clave. Una API key nunca tiene más permisos que el usuario que la crea; aquí se restringen aún más con DLS.

- `expiration`: vigencia. Las claves de una aplicación deben tener caducidad y rotarse.

``` console
curl.exe -s "<ES_URL>/orders-shared/_search?filter_path=hits.total.value,hits.hits._source.order_id" -H "Authorization: ApiKey <ENCODED_KEY>"
```

**Resultado esperado:** `total.value` de 2, con los pedidos 4001 y 4002.

## Parte C: ILM sobre un data stream (8 min)

ILM (index lifecycle management) mueve los índices por fases según su edad o tamaño. En este laboratorio los tiempos son de minutos para ver el resultado; en producción serían días o semanas.

### Paso C1. Acelerar la revisión de ILM

``` console
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval": "30s"
  }
}
```

**Qué hace:** ILM evalúa las políticas cada cierto intervalo, que por defecto es de 10 minutos. Con 30 segundos el laboratorio avanza rápido. Al final se restablece el valor.

Si Elastic Cloud rechaza el ajuste, continúa sin él: ILM seguirá funcionando y cada paso tardará hasta 10 minutos, de modo que la Parte E se hará con lo que haya avanzado.

### Paso C2. Crear la política

``` console
PUT _ilm/policy/curso-logs-ilm
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_docs": 20, "max_primary_shard_size": "50gb" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "1m",
        "actions": {
          "set_priority": { "priority": 50 },
          "readonly": {},
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "delete": {
        "min_age": "4m",
        "actions": { "delete": {} }
      }
    }
  }
}
```

**Qué hace:** define tres fases: hot (índice activo), warm (índice cerrado a escritura y compactado) y delete (retención vencida).

- `rollover`: crea un índice nuevo cuando se cumple cualquiera de las condiciones. `max_docs: 20` es solo para el laboratorio; en producción se usa `max_primary_shard_size` (hasta 50 GB por shard) y, si hace falta, `max_age`.

- `min_age`: edad mínima para entrar en la fase. Para un índice que hizo rollover se cuenta desde el momento del rollover, no desde su creación.

- `set_priority`: prioridad de recuperación tras un reinicio; los índices recientes se recuperan primero.

- `readonly`: bloquea las escrituras en el índice. `forcemerge` con `max_num_segments: 1`: compacta el índice (solo es seguro porque ya no recibe escrituras).

- `delete`: borra el índice completo. Borrar índices es mucho más barato que borrar documentos.

En un clúster con nodos warm, la fase warm también mueve los shards a ese tier. El deployment de la prueba puede tener solo nodos hot: en ese caso los shards permanecen donde están y la política avanza igual.

### Paso C3. Template y data stream

``` console
PUT _index_template/ilmtest-template
{
  "index_patterns": ["ilmtest-*"],
  "priority": 300,
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "curso-logs-ilm",
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service":    { "type": "keyword" },
        "message":    { "type": "text" }
      }
    }
  }
}
```

**Qué hace:** declara que los nombres `ilmtest-*` son data streams, con la política de ILM asociada mediante `index.lifecycle.name`.

- `data_stream: {}`: convierte el template en uno de data streams; el primer documento crea el data stream y su primer índice de respaldo.

- Si no se indicara `index.lifecycle.name`, un data stream usaría su ciclo de vida propio (data stream lifecycle). Cuando se indica una política de ILM, esta tiene prioridad.

### Paso C4. Cargar documentos

``` console
POST _reindex
{
  "max_docs": 25,
  "source": {
    "index": "weblogs",
    "query": { "term": { "service": "payments" } },
    "_source": ["@timestamp", "service", "message"]
  },
  "dest": { "index": "ilmtest-ds", "op_type": "create" }
}

GET ilmtest-ds/_count
```

**Qué hace:** copia 25 documentos de pagos a `ilmtest-ds`. Como no existe, se crea como data stream.

- `max_docs`: limita el número de documentos copiados.

- `_source` (dentro de `source`): copia solo esos campos.

- `op_type: create`: los data streams solo aceptan documentos nuevos (son append-only), no actualizaciones.

**Resultado esperado:** `created: 25` y `count` de 25. Supera el umbral de 20 documentos, así que el siguiente ciclo de ILM hará el rollover.

``` console
GET _data_stream/ilmtest-ds?filter_path=data_streams.name,data_streams.generation,data_streams.ilm_policy,data_streams.indices.index_name
```

**Resultado esperado:** `generation` de 1, `ilm_policy` igual a `curso-logs-ilm` y un índice de respaldo con nombre `.ds-ilmtest-ds-<fecha>-000001`. Revisa de nuevo en la Parte E.

## Parte D: snapshots, restore y SLM (14 min)

### Paso D1. El repositorio de Elastic Cloud

``` console
GET _snapshot

GET _snapshot/found-snapshots/*?size=3&sort=start_time&order=desc&filter_path=snapshots.snapshot,snapshots.state,snapshots.start_time

GET _slm/policy/cloud-snapshot-policy?filter_path=*.policy.schedule,*.policy.retention,*.last_success.snapshot_name
```

**Qué hace:** muestra los repositorios registrados, los tres snapshots más recientes y la política automática de la plataforma.

- `found-snapshots`: repositorio que Elastic Cloud registra en todo deployment Hosted.

- `cloud-snapshot-policy`: política SLM de la plataforma. Por defecto toma un snapshot de todos los índices cada 30 minutos.

- `size`, `sort`, `order`: limitan y ordenan el listado de snapshots.

- `schedule` y `retention`: cuándo se ejecuta y cuánto se conservan. Anótalos.

Los snapshots son incrementales: cada uno guarda solo los segmentos nuevos desde el anterior. Por eso un snapshot cada 30 minutos no multiplica el espacio.

### Paso D2. Snapshot manual

``` console
PUT _snapshot/found-snapshots/lab07-orders-1?wait_for_completion=true
{
  "indices": ["orders-shared", "weblogs-v1"],
  "include_global_state": false,
  "metadata": { "tomado_por": "laboratorio 7" }
}
```

**Qué hace:** toma un snapshot de dos índices del curso y espera a que termine.

- `lab07-orders-1`: nombre del snapshot dentro del repositorio.

- `wait_for_completion=true`: la respuesta llega al terminar. Sin él, la API devuelve de inmediato y el snapshot sigue en segundo plano.

- `indices`: qué incluir. Sin este campo se incluyen todos los índices y data streams.

- `include_global_state: false`: no incluye la configuración global del clúster (templates, políticas, etc.). Para restaurar datos suele ser lo adecuado; restaurar el estado global puede sobrescribir configuración.

- `metadata`: información libre para documentar el snapshot.

**Resultado esperado:** `"state": "SUCCESS"` y `shards.total` de 2 con `failed` en 0.

### Paso D3. Simular una pérdida y restaurar

``` console
POST orders-shared/_delete_by_query?refresh=true
{
  "query": { "term": { "tenant_id": "globex" } }
}

GET orders-shared/_count
```

**Qué hace:** simula un borrado accidental de los pedidos de `globex`. **Resultado esperado:** `deleted: 2` y `count` de 4.

``` console
POST _snapshot/found-snapshots/lab07-orders-1/_restore?wait_for_completion=true
{
  "indices": "orders-shared",
  "rename_pattern": "(.+)",
  "rename_replacement": "restored-$1",
  "include_aliases": false,
  "include_global_state": false
}

GET restored-orders-shared/_count
```

**Qué hace:** restaura el índice del snapshot con otro nombre, sin tocar el original.

- `rename_pattern` y `rename_replacement`: expresión regular y reemplazo para renombrar. `(.+)` captura el nombre completo y `$1` lo reutiliza con el prefijo `restored-`.

- `include_aliases: false`: no recrea los alias, que seguirían apuntando al índice original.

- Restaurar sobre el nombre original solo es posible si el índice está cerrado o no existe. Con otro nombre se puede comparar antes de decidir.

**Resultado esperado:** `count` de 6 en el índice restaurado.

``` console
POST _reindex
{
  "source": {
    "index": "restored-orders-shared",
    "query": { "term": { "tenant_id": "globex" } }
  },
  "dest": { "index": "orders-shared", "op_type": "create" }
}

GET orders-shared/_count
```

**Qué hace:** devuelve a `orders-shared` solo los documentos perdidos. `op_type: create` evita sobrescribir documentos existentes (los `_id` se conservan). **Resultado esperado:** `created: 2` y `count` de 6.

### Paso D4. Política de snapshots propia (SLM)

``` console
PUT _slm/policy/curso-diario
{
  "schedule": "0 30 1 * * ?",
  "name": "<curso-{now/d}>",
  "repository": "found-snapshots",
  "config": {
    "indices": ["orders-shared", "weblogs-v1"],
    "include_global_state": false
  },
  "retention": {
    "expire_after": "7d",
    "min_count": 2,
    "max_count": 10
  }
}

POST _slm/policy/curso-diario/_execute
```

**Qué hace:** crea una política que toma un snapshot de los dos índices cada día y la ejecuta una vez de inmediato.

- `schedule`: expresión cron de Elasticsearch (segundos, minutos, horas, día del mes, mes, día de la semana). `0 30 1 * * ?` es todos los días a la 01:30 UTC.

- `name`: nombre de cada snapshot, con fecha (`<curso-{now/d}>` se resuelve a `curso-<fecha>`) más un sufijo único.

- `retention.expire_after`: antigüedad a partir de la cual un snapshot de esta política es elegible para borrado.

- `min_count` y `max_count`: mínimo que siempre se conserva y máximo permitido.

- `_execute`: dispara la política ahora, sin esperar al horario. Devuelve el nombre del snapshot.

``` console
GET _slm/policy/curso-diario?filter_path=*.last_success,*.last_failure,*.next_execution_millis
```

**Resultado esperado:** `last_success` con el nombre del snapshot. La retención se aplica según `slm.retention_schedule` (por defecto, a la 01:30 UTC) y solo afecta a los snapshots de esta política.

## Parte E: resultado de ILM (6 min)

``` console
GET _data_stream/ilmtest-ds?filter_path=data_streams.generation,data_streams.indices.index_name

GET ilmtest-ds/_ilm/explain?filter_path=indices.*.index,indices.*.phase,indices.*.action,indices.*.step,indices.*.age
```

**Qué hace:** muestra los índices de respaldo del data stream y en qué punto de su ciclo está cada uno.

- `phase`, `action`, `step`: fase, acción y paso en curso. El paso `complete` indica que terminó la fase.

- `age`: edad del índice (desde el rollover).

- `step_info` (si aparece): detalle del error cuando el paso está en `ERROR`. Se reintenta con `POST <índice>/_ilm/retry`.

Cronología esperada con `poll_interval` de 30 s:

| **Momento**                | **Qué se observa**                                                                                    |
|----------------------------|-------------------------------------------------------------------------------------------------------|
| Primer ciclo               | Rollover: `generation` de 2; aparece el índice `...-000002` como índice de escritura.                 |
| 1 min después del rollover | El índice `...-000001` pasa a la fase `warm`: solo lectura y compactado a un segmento.                |
| 4 min después del rollover | El índice `...-000001` entra en `delete` y desaparece. Quedan solo los documentos del índice vigente. |

Si tras 3 minutos no hay rollover, fuerza uno con `POST ilmtest-ds/_rollover` y vuelve a consultar `_ilm/explain`. La política se aplicará a partir de ese momento.

Pregunta para observar: después de que el primer índice se borra, `GET ilmtest-ds/_count` devuelve menos documentos que los 25 iniciales. ¿Por qué? Esto es lo que significa la retención por índices completos.

## Parte F: auditoría, gobernanza y limpieza (8 min)

### Paso F1. Auditoría (demostración del instructor)

La auditoría registra eventos de seguridad: autenticaciones fallidas, accesos denegados, cambios de roles. Está desactivada por defecto. En Elastic Cloud Hosted se activa en la consola, en el deployment, con la opción **Edit**, y se añade el ajuste `xpack.security.audit.enabled: true` en la configuración de usuario de Elasticsearch y de Kibana. Guardar el cambio ejecuta un cambio de plan en el deployment, y los registros se entregan al deployment de monitoreo.

Por eso no se hace en el laboratorio: requiere monitoreo configurado y un cambio de plan. El instructor puede mostrarlo en su propio deployment y revisar los eventos de los pasos B5 y B6 (accesos denegados del usuario `ana_acme`). Los eventos útiles para revisar son `authentication_failed` y `access_denied`.

### Paso F2. Matriz de usuarios, roles y tenants

Completa la matriz con lo que configuraste y los resultados de tus pruebas. Es el documento que un auditor pediría primero.

| **Identidad**              | **Rol**                     | **Tenant** | **Alcance**               | **DLS / FLS** | **Prueba y resultado**           |
|----------------------------|-----------------------------|------------|---------------------------|---------------|----------------------------------|
| `ana_acme`                 | `orders_acme_reader`        | acme       | `read` en `orders-shared` | DLS / no      |                                  |
| `gus_globex`               | `orders_globex_reader`      | globex     | `read` en `orders-shared` | DLS / FLS     |                                  |
| Clave `orders-initech-app` | Descriptor `initech-reader` | initech    | `read` en `orders-shared` | DLS / no      |                                  |
| `elastic`                  | `superuser`                 | todos      | Todo                      | No aplica     | Uso restringido a administración |

### Paso F3. Limpieza

``` console
DELETE _data_stream/ilmtest-ds
DELETE _index_template/ilmtest-template
DELETE _ilm/policy/curso-logs-ilm
DELETE restored-orders-shared

DELETE _slm/policy/curso-diario
DELETE _snapshot/found-snapshots/lab07-orders-1
DELETE _snapshot/found-snapshots/curso-*

DELETE _security/api_key
{ "name": "orders-initech-app" }

PUT _cluster/settings
{
  "persistent": { "indices.lifecycle.poll_interval": null }
}
```

**Qué hace:** elimina lo creado en este laboratorio, en el orden que exigen las dependencias: la política de ILM no se puede borrar mientras un índice la use, por eso primero se elimina el data stream.

- Borrar un snapshot en `found-snapshots` es permanente. Los comandos de esta limpieza se limitan a los snapshots que creaste (`lab07-orders-1` y los de `curso-*`); nunca borres los de `cloud-snapshot-policy`.

- `null` en el ajuste de clúster: restablece el intervalo de ILM a su valor por defecto.

Se conservan `orders-shared` (con el nuevo campo `customer_email`), los roles y usuarios de práctica, y todo lo de laboratorios anteriores.

## Resumen de valores esperados

| **Comprobación**                                        | **Resultado**                                |
|---------------------------------------------------------|----------------------------------------------|
| `_update_by_query` en `orders-shared`                   | updated: 6                                   |
| `ana_acme`: búsqueda en `orders-shared`                 | 2 pedidos (2001, 2002), con `customer_email` |
| `gus_globex`: búsqueda en `orders-shared`               | 2 pedidos (3001, 3002), sin `customer_email` |
| `ana_acme` busca pedidos de globex                      | count: 0                                     |
| `gus_globex` busca por `customer_email`                 | count: 0                                     |
| `ana_acme` lee `weblogs` / escribe en `orders-shared`   | 403 / 403                                    |
| Contraseña incorrecta                                   | 401                                          |
| Clave `orders-initech-app`                              | 2 pedidos (4001, 4002)                       |
| Snapshot `lab07-orders-1`                               | SUCCESS, 2 shards                            |
| Tras el borrado accidental / tras restaurar y reindexar | count: 4 / count: 6                          |
| Data stream tras el primer ciclo de ILM                 | generation: 2                                |

## Estado final

| **Elemento**                                                                            | **Se conserva**  | **Lo reutiliza**                   |
|-----------------------------------------------------------------------------------------|------------------|------------------------------------|
| `orders-shared` con 6 documentos y `customer_email`                                     | Sí               | Laboratorio 10 (proyecto final)    |
| Roles `orders_acme_reader` y `orders_globex_reader`, usuarios `ana_acme` y `gus_globex` | Sí               | Proyecto final (pruebas de acceso) |
| `weblogs` con 5006 documentos                                                           | Sí               | Laboratorios 8 y 9                 |
| `ilmtest-ds`, `restored-orders-shared`, snapshots, SLM y clave de initech               | No (se eliminan) | \-                                 |

## Preguntas de discusión

- ¿Qué ventajas y qué costos tiene DLS frente a un alias filtrado por tenant?

- ¿Qué pasaría si el rol de `gus_globex` usara `except: ["customer_email"]` en lugar de `grant`, y mañana se añadiera el campo `phone`?

- El snapshot tomado cada 30 minutos, ¿es una estrategia de recuperación ante desastres suficiente? ¿Qué falta (RPO, RTO, copia en otra región)?

- ¿Por qué conviene borrar índices completos con ILM en lugar de borrar documentos con `_delete_by_query`?

- ¿Qué retención propondrías para logs de acceso web si el negocio exige 90 días y la regulación pide poder borrar los datos de una persona?

- Si un auditor pide demostrar quién accedió a `orders-shared` la semana pasada, ¿qué necesitas tener activado hoy?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 7: Configurar snapshots y políticas de ILM](imagenes/lab07_resultado_final.png)
