# Laboratorio 2: Desplegar y comprobar un clúster en la nube

- **Capítulo:** 2. Instalación, versiones y despliegue en la nube

- **Duración:** 50 min

- **Entorno:** Elastic Cloud Hosted (cuenta de prueba propia), Elasticsearch 9.x, Kibana Dev Tools y una terminal con `curl`

- **Dependencias:** Guía previa de cuenta y deployment (el deployment ya existe y tiene su ficha iniciada). No requiere los índices del Laboratorio 1.

## Objetivo

Comprobar el estado del deployment propio, crear un índice e insertar y consultar documentos, revisar el mapping que Elasticsearch genera automáticamente y crear una API Key con privilegios limitados para usarla en una consulta externa. Al terminar, el participante podrá:

- Identificar versión, nodos, endpoints y zonas de un deployment desde la API y desde la consola.

- Interpretar los códigos de respuesta HTTP más frecuentes de Elasticsearch (200, 201, 400, 401, 403, 404 y 409).

- Explicar qué tipos de campo genera el mapeo dinámico y qué problemas introduce.

- Crear, usar y revocar una API Key restringida a un índice.

- Dejar cargado el índice `lab02-weblogs`, que el laboratorio 3.4 reutiliza.

## Requisitos

- Deployment Hosted 9.x disponible, con la contraseña del usuario `elastic` guardada.

- Ficha del deployment de la guía previa.

- Terminal con `curl`. En Windows, usar `curl.exe` (en PowerShell, `curl` es un alias de otro comando). Si no se dispone de terminal, se puede usar un cliente HTTP como Postman con el mismo encabezado.

## Distribución del tiempo

| **Parte** | **Contenido**                                      | **Tiempo** |
|-----------|----------------------------------------------------|------------|
| Parte A   | Comprobar el deployment                            | 8 min      |
| Parte B   | Índice, documentos, mapping y códigos de respuesta | 17 min     |
| Parte C   | API Key restringida                                | 25 min     |

## Convenciones

- Los comandos en bloques con `GET`, `PUT`, `POST` o `DELETE` se ejecutan en Kibana Dev Tools (Console).

- Los comandos que empiezan con `curl` se ejecutan en la terminal del equipo.

- Los valores entre ángulos, como `<ES_URL>` o `<ENCODED_KEY>`, se sustituyen por los datos propios.

- La respuesta de Dev Tools muestra el código HTTP en la parte superior del panel de resultados.

## Parte A: comprobar el deployment (8 min)

### Paso A1. Versión y salud del clúster

``` console
GET /

GET _cluster/health
```

**Qué hace:** devuelve la información del clúster y su estado de salud, como en el Laboratorio 1.

Campos a registrar:

- `version.number`: versión exacta de Elasticsearch.

- `version.lucene_version`: versión de Lucene. En 9.x corresponde a Lucene 10.

- `cluster_name`: nombre interno del clúster.

- `status`, `number_of_nodes` y `number_of_data_nodes`: salud y tamaño del clúster.

### Paso A2. Nodos y roles

``` console
GET _cat/nodes?v&h=name,node.role,master,heap.percent,disk.used_percent
```

**Qué hace:** lista los nodos con sus roles. Con 2 zonas de disponibilidad debe verse, además de los nodos de datos, un nodo de voto (rol `v`) que actúa como tiebreaker.

### Paso A3. Datos de la consola de Elastic Cloud

En la consola de Elastic Cloud, abrir el deployment y localizar en la página principal la sección de aplicaciones:

- **Elasticsearch endpoint:** URL HTTPS del clúster. Usar la opción para copiarla; se necesitará en la Parte C como `<ES_URL>`.

- **Kibana endpoint:** URL de acceso a Kibana.

- **Cloud ID:** identificador compacto que usan Beats, Logstash y los clientes oficiales para conectarse. Se usará en el capítulo 4.

Ampliar la ficha del deployment de la guía previa con estos datos:

| **Dato**                            | **Valor** |
|-------------------------------------|-----------|
| Elasticsearch endpoint (`<ES_URL>`) |           |
| Kibana endpoint                     |           |
| Cloud ID                            |           |
| Versión confirmada con `GET /`      |           |
| Nodos de datos y nodo tiebreaker    |           |

## Parte B: índice, documentos, mapping y códigos de respuesta (17 min)

Se trabaja con registros de acceso de una aplicación web. Este es el dataset base que se retoma en el capítulo 3.

### Paso B1. Insertar un documento con identificador propio

``` console
PUT lab02-weblogs/_doc/1?refresh=wait_for
{
  "@timestamp": "2026-10-14T09:15:02Z",
  "service": "checkout",
  "client_ip": "187.141.22.10",
  "method": "GET",
  "url": "/api/cart",
  "status": 200,
  "bytes": 1532,
  "response_time_ms": 42.5,
  "message": "GET /api/cart 200 42ms"
}
```

**Qué hace:** indexa el documento con `_id` igual a 1. Como el índice no existe, Elasticsearch lo crea automáticamente y genera el mapping a partir de los campos del documento.

Parámetros:

- `PUT` con `_id` en la ruta: el cliente decide el identificador. Si el documento ya existe, se reemplaza.

- `refresh=wait_for`: la respuesta espera al siguiente refresh, de modo que el documento ya aparece en las búsquedas siguientes.

**Resultado esperado:** código `201 Created`, `result: created`, `_version: 1`.

### Paso B2. Reemplazar el documento y recuperarlo

Ejecutar de nuevo exactamente la petición del paso B1 y luego recuperar el documento:

``` console
GET lab02-weblogs/_doc/1
```

**Resultado esperado:** la repetición del `PUT` devuelve código `200 OK`, `result: updated` y `_version: 2`. El `GET` devuelve `found: true` y el contenido en `_source`.

### Paso B3. Provocar un conflicto

``` console
PUT lab02-weblogs/_create/1
{
  "@timestamp": "2026-10-14T09:15:02Z",
  "service": "checkout"
}
```

**Qué hace:** `_create` solo indexa si el identificador no existe.

**Resultado esperado:** código `409 Conflict` con `version_conflict_engine_exception`.

### Paso B4. Cargar más documentos con la Bulk API

``` console
POST lab02-weblogs/_bulk?refresh=wait_for
{ "index": {} }
{ "@timestamp": "2026-10-14T09:15:07Z", "service": "checkout", "client_ip": "187.141.22.10", "method": "POST", "url": "/api/cart", "status": 201, "bytes": 310, "response_time_ms": 88.2, "message": "POST /api/cart 201 88ms" }
{ "index": {} }
{ "@timestamp": "2026-10-14T09:16:31Z", "service": "catalog", "client_ip": "201.103.7.55", "method": "GET", "url": "/api/products", "status": 200, "bytes": 20480, "response_time_ms": 120.4, "message": "GET /api/products 200 120ms" }
{ "index": {} }
{ "@timestamp": "2026-10-14T09:17:45Z", "service": "catalog", "client_ip": "201.103.7.55", "method": "GET", "url": "/api/products/9999", "status": 404, "bytes": 512, "response_time_ms": 15.1, "message": "GET /api/products/9999 404 15ms product not found" }
{ "index": {} }
{ "@timestamp": "2026-10-14T09:18:12Z", "service": "payments", "client_ip": "148.202.90.3", "method": "POST", "url": "/api/pay", "status": 500, "bytes": 740, "response_time_ms": 1503.9, "message": "POST /api/pay 500 1503ms upstream timeout" }
{ "index": {} }
{ "@timestamp": "2026-10-14T09:19:03Z", "service": "payments", "client_ip": "148.202.90.3", "method": "POST", "url": "/api/pay", "status": 200, "bytes": 655, "response_time_ms": 310.7, "message": "POST /api/pay 200 310ms" }
```

**Qué hace:** carga 5 documentos en una sola petición. La Bulk API se estudia a detalle en el capítulo 4.

- NDJSON: un objeto JSON por línea, alternando la línea de acción y la del documento.

- `{ "index": {} }`: acción de indexar; el índice viene de la ruta y el `_id` se genera solo.

- Verificar en la respuesta que aparezca `"errors": false`.

### Paso B5. Consultar los documentos

``` console
GET lab02-weblogs/_count

GET lab02-weblogs/_search
{
  "query": { "match": { "message": "timeout" } }
}

GET lab02-weblogs/_search
{
  "query": { "term": { "status": 404 } }
}
```

**Qué hace:** cuenta los documentos y ejecuta dos búsquedas: una de texto sobre `message` y otra por valor exacto sobre `status`.

Campos de la respuesta de `_search`:

- `took`: milisegundos que tardó la búsqueda.

- `timed_out`: si se agotó el tiempo límite.

- `_shards`: shards consultados y su resultado.

- `hits.total.value`: coincidencias totales.

- `hits.hits[]._score`: relevancia calculada para cada documento.

**Resultado esperado:** `_count` devuelve 6. Cada búsqueda devuelve 1 documento (el de `upstream timeout` y el de estado 404).

### Paso B6. Revisar el mapping generado

``` console
GET lab02-weblogs/_mapping
```

**Qué hace:** muestra el mapping que Elasticsearch creó a partir del primer documento (mapeo dinámico). Comparar con esta tabla:

| **Campo**                             | **Tipo generado**             | **Observación**                                                                    |
|---------------------------------------|-------------------------------|------------------------------------------------------------------------------------|
| `@timestamp`                          | `date`                        | La cadena en formato ISO 8601 se detecta como fecha.                               |
| `service`, `method`, `url`, `message` | `text` con subcampo `keyword` | Toda cadena genera `text` más un subcampo `.keyword` con `ignore_above: 256`.      |
| `client_ip`                           | `text` con subcampo `keyword` | Una dirección IP no se reconoce como tipo `ip`; se corrige en el capítulo 3.       |
| `status`, `bytes`                     | `long`                        | Un entero genera `long`. Un código HTTP se suele modelar como `keyword` o `short`. |
| `response_time_ms`                    | `float`                       | Un decimal genera `float`.                                                         |

Anotar qué campos modelarías distinto. El laboratorio 3.4 parte de este índice para definir un mapping explícito.

### Paso B7. Provocar errores y leer los códigos

``` console
GET lab02-noexiste/_search

POST lab02-weblogs/_doc
{
  "@timestamp": "2026-10-14T09:20:00Z",
  "status": "no-es-un-numero"
}
```

**Qué hace:** la primera petición consulta un índice que no existe; la segunda intenta guardar texto en un campo ya mapeado como `long`.

- Primera petición: `404 Not Found` con `index_not_found_exception`.

- Segunda petición: `400 Bad Request` con `document_parsing_exception` (en versiones anteriores aparecía como `mapper_parsing_exception`). El mapping, una vez creado, rechaza valores incompatibles.

## Parte C: API Key restringida (25 min)

Hasta aquí todas las peticiones se hicieron con el usuario `elastic`, que es superusuario. En producción ese usuario no se usa para aplicaciones: se crean API Keys con los privilegios mínimos necesarios.

### Paso C1. Comprobar la identidad actual

``` console
GET _security/_authenticate
```

**Qué hace:** devuelve quién es el usuario autenticado y cómo se autenticó.

- `username`: `elastic`.

- `roles`: contiene `superuser`.

- `authentication_type`: `realm` (usuario y contraseña).

### Paso C2. Crear la API Key

``` console
POST _security/api_key
{
  "name": "lab02-weblogs-readonly",
  "expiration": "7d",
  "role_descriptors": {
    "weblogs_reader": {
      "cluster": [],
      "indices": [
        {
          "names": ["lab02-weblogs"],
          "privileges": ["read", "view_index_metadata"]
        }
      ]
    }
  },
  "metadata": { "lab": "02" }
}
```

**Qué hace:** crea una credencial que solo puede leer el índice `lab02-weblogs`.

Parámetros:

- `name`: nombre descriptivo de la clave.

- `expiration`: vigencia. Si se omite, la clave no expira nunca; en laboratorios y aplicaciones conviene siempre fijarla.

- `role_descriptors`: privilegios de la clave. Si se omite, la clave hereda los privilegios del usuario que la crea (aquí, un superusuario). Los privilegios efectivos nunca superan los del creador.

- `cluster`: privilegios a nivel de clúster. Vacío: ninguno.

- `indices[].names`: índices o patrones a los que aplica. Se pueden usar comodines, como `lab02-*`.

- `indices[].privileges`: `read` permite buscar, contar y recuperar documentos; `view_index_metadata` permite ver mappings y settings.

- `metadata`: datos propios para identificar la clave; no afectan a los permisos.

La respuesta contiene `id`, `name`, `expiration`, `api_key` y `encoded`.

- `encoded`: es el valor que se envía en el encabezado `Authorization`. Es la codificación en base64 de `id:api_key`.

- Copiar `encoded` y `id` de inmediato: el secreto de la clave solo se devuelve en la creación.

### Paso C3. Usar la API Key en una consulta

En la terminal del equipo, sustituir `<ES_URL>` por el endpoint de Elasticsearch de la ficha (sin barra final) y `<ENCODED_KEY>` por el valor `encoded`:

``` console
curl.exe -i -H "Authorization: ApiKey <ENCODED_KEY>" "<ES_URL>/lab02-weblogs/_count?pretty"
```

Parámetros:

- `curl.exe`: en macOS y Linux se escribe `curl`.

- `-i`: incluye en la salida la línea de estado y los encabezados HTTP.

- `-H "Authorization: ApiKey ..."`: encabezado de autenticación con la clave.

- `?pretty`: formatea el JSON de la respuesta.

**Resultado esperado:** `HTTP/2 200` y `"count" : 6`.

### Paso C4. Comprobar los límites de la clave

``` console
curl.exe -i -X DELETE -H "Authorization: ApiKey <ENCODED_KEY>" "<ES_URL>/lab02-weblogs/_doc/1"

curl.exe -i -H "Authorization: ApiKey <ENCODED_KEY>" "<ES_URL>/lab01-demo/_count"

curl.exe -i "<ES_URL>/"
```

**Qué hace:** prueba tres situaciones que deben fallar.

- Borrar un documento: `403 Forbidden` con `security_exception`. La clave tiene lectura, no escritura.

- Leer otro índice: `403 Forbidden`. La clave solo cubre `lab02-weblogs`. (Se obtiene el mismo resultado si el índice ya no existe.)

- Petición sin credenciales: `401 Unauthorized`. No se envió ningún encabezado de autenticación.

Regla para recordar: **401** significa que no se sabe quién eres; **403** significa que se sabe quién eres, pero no tienes permiso.

### Paso C5. Verificar la identidad de la clave

``` console
curl.exe -H "Authorization: ApiKey <ENCODED_KEY>" "<ES_URL>/_security/_authenticate?pretty"
```

**Qué hace:** repite el paso C1, ahora con la clave.

**Resultado esperado:** `authentication_type` es `api_key` y aparece el nombre `lab02-weblogs-readonly` en `api_key.name`.

### Paso C6. Listar y revocar la clave

``` console
GET _security/api_key?name=lab02-weblogs-readonly

DELETE _security/api_key
{
  "ids": ["<ID_DE_LA_CLAVE>"]
}
```

**Qué hace:** la primera petición muestra los metadatos de la clave (`id`, `creation`, `expiration`, `invalidated`). La segunda la invalida.

- `ids`: lista de identificadores a invalidar. Alternativamente, se puede usar `name` para invalidar por nombre.

- La respuesta incluye `invalidated_api_keys` con los identificadores revocados.

- Las claves también se administran desde Kibana: Stack Management, sección Security, **API keys**.

Repetir el paso C3. **Resultado esperado:** `401 Unauthorized`. La clave ya no es válida.

## Limpieza y estado final

Este laboratorio no requiere borrar índices. La API Key de prueba quedó invalidada en el paso C6.

| **Elemento**                                  | **Se conserva** | **Lo reutiliza**                                                 |
|-----------------------------------------------|-----------------|------------------------------------------------------------------|
| Deployment propio y ficha ampliada            | Sí              | Todos los laboratorios                                           |
| `lab02-weblogs` con 6 documentos              | Sí              | Laboratorio 3.4 (comparar mapeo dinámico y explícito, analyzers) |
| `orders-shared` y sus aliases (Laboratorio 1) | Sí              | Laboratorio 7.4                                                  |
| API Key `lab02-weblogs-readonly`              | No (invalidada) | Solo se usó para practicar                                       |

Verificación final:

``` console
GET _cat/indices/lab02-*,orders-*?v&h=index,health,pri,rep,docs.count
```

Debe verse `lab02-weblogs` con 6 documentos y `orders-shared` con los documentos del Laboratorio 1.

## Preguntas de discusión

- ¿Por qué conviene no usar el usuario `elastic` en aplicaciones y preferir API Keys?

- ¿Qué privilegios tendría una API Key creada sin `role_descriptors` por un superusuario? ¿Es una buena práctica?

- ¿Qué diferencia práctica hay entre recibir un 401 y un 403?

- ¿Qué problemas esperas con `client_ip` como `text` y con `status` como `long`?

- Si la prueba gratuita vence antes de terminar el curso, ¿qué se pierde y qué se puede recuperar?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 2: Desplegar y comprobar un clúster en la nube](imagenes/lab02_resultado_final.png)
