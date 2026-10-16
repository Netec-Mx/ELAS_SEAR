# Laboratorio 1: Evaluación de topologías en la nube

- **Capítulo:** 1. Fundamentos y modelo distribuido de Elasticsearch

- **Duración:** 34 min

- **Entorno:** Elastic Cloud Hosted, Elasticsearch 9.x, Kibana Dev Tools

- **Dependencias:** ninguna. Este es el primer laboratorio de la cadena; los datos y la ficha que genera se reutilizan en laboratorios posteriores.

## Objetivo

Revisar la salud de un clúster en Elastic Cloud, observar shards y réplicas para entender por qué un índice puede quedar en estado yellow, y comparar estrategias de multi-tenancy sobre un caso. Al terminar, el participante podrá:

- Identificar la versión, los nodos y los roles de un deployment, tanto desde la API como desde la consola de Elastic Cloud.

- Provocar un estado yellow, diagnosticar su causa con la API de explicación de asignación y corregirlo.

- Comparar el consumo de shards entre un índice por tenant y un índice compartido con aliases filtrados.

- Dejar cargado el escenario de datos `orders-shared` que se reutiliza en laboratorios posteriores.

## Requisitos

- Deployment de Elastic Cloud **Hosted** con Elasticsearch 9.x y, como mínimo, 2 zonas de disponibilidad.

- Acceso a Kibana Dev Tools con un usuario con privilegios de administración del clúster (por ejemplo, el usuario `elastic`).

## Distribución del tiempo

| **Parte**             | **Contenido**                            | **Tiempo** |
|-----------------------|------------------------------------------|------------|
| Preparación y Parte A | Acceso al deployment y salud del clúster | 8 min      |
| Parte B               | Shards, réplicas y estado yellow         | 12 min     |
| Parte C               | Estrategias de multi-tenancy             | 14 min     |

## Convenciones del laboratorio

- Todos los comandos se ejecutan en la consola de Dev Tools (pestaña Console). Cada petición se ejecuta con el botón de reproducir o con Ctrl + Enter.

- La consola acepta la sintaxis `MÉTODO ruta` seguida del cuerpo JSON; no se usa `curl` ni la URL completa.

- Los índices del laboratorio usan prefijos fijos (`lab01-`, `orders-`) para identificarlos y limpiarlos.

- Cada comando va seguido de una explicación de lo que hace y del significado de sus parámetros.

- No eliminar el deployment ni índices que el laboratorio no pida borrar: los laboratorios siguientes dependen de ellos.

## Preparación: acceso al deployment

1.  Ingresar a la consola de Elastic Cloud (cloud.elastic.co) y abrir el deployment del curso.

2.  Verificar que el deployment sea de tipo **Hosted** (muestra nodos, zonas y tiers). Un proyecto Serverless no sirve para el curso: no expone nodos ni shards como se trabaja aquí.

3.  Seleccionar **Open Kibana**.

4.  En el menú principal, dentro de la sección Management, abrir **Dev Tools** y usar la pestaña **Console**.

## Parte A: salud del clúster

### Paso A1. Identificar el clúster

``` console
GET /
```

**Qué hace:** consulta la raíz del servidor y devuelve la información básica del clúster. Es equivalente a conectarse al puerto 9200 en una instalación local.

Campos a revisar en la respuesta:

- `name`: nodo que atendió la petición.

- `cluster_name`: nombre del clúster.

- `version.number`: versión de Elasticsearch. Debe ser 9.x.

- `version.build_flavor`: tipo de distribución.

- `version.lucene_version`: versión de Lucene sobre la que corre.

### Paso A2. Consultar la salud del clúster

``` console
GET _cluster/health
```

**Qué hace:** devuelve el estado global del clúster.

Campos a revisar:

- `status`: green, yellow o red.

- `number_of_nodes` y `number_of_data_nodes`: nodos totales y nodos que almacenan datos.

- `active_primary_shards` y `active_shards`: primaries activos y total de copias activas.

- `unassigned_shards`: copias sin nodo asignado.

- `active_shards_percent_as_number`: porcentaje de shards activos. Debe ser 100 cuando el estado es green.

### Paso A3. Listar los nodos y sus roles

``` console
GET _cat/nodes?v&h=name,node.role,master,heap.percent,disk.used_percent
```

**Qué hace:** lista los nodos en formato de tabla legible.

Parámetros:

- `_cat`: familia de APIs que devuelve texto tabulado en lugar de JSON, pensada para lectura en consola.

- `v`: incluye la fila de encabezados.

- `h=`: columnas a mostrar, separadas por coma.

- `node.role`: abreviatura de roles. Los más frecuentes en Elastic Cloud son `m` (master-eligible), `v` (voting-only, nodo tiebreaker), `h` (data hot), `w` (data warm), `c` (data cold), `s` (data content) e `i` (ingest).

- `master`: un asterisco (`*`) marca al master elegido.

- `heap.percent`: uso del heap de la JVM.

- `disk.used_percent`: uso de disco, que se compara con los watermarks de 85 %, 90 % y 95 % vistos en el tema 1.2.

### Paso A4. Listar los índices

``` console
GET _cat/indices?v
```

**Qué hace:** lista los índices con su salud, número de shards y tamaño.

Columnas relevantes: `health`, `index`, `pri` (primaries), `rep` (réplicas por primary), `docs.count` y `store.size`. Los índices cuyo nombre empieza con punto son del sistema y no se modifican.

### Paso A5. Contrastar con la consola y completar la ficha del deployment

En la consola de Elastic Cloud, abrir la configuración del deployment (opción **Edit**) y comparar con lo observado en `_cat/nodes`:

- Zonas de disponibilidad configuradas.

- Tamaño y tiers de datos habilitados (hot, warm, cold).

- Presencia de nodos master dedicados o de un tiebreaker.

Completar la ficha del deployment. Se usará en todos los laboratorios del curso:

| **Dato**                   | **Valor** |
|----------------------------|-----------|
| Nombre del deployment      |           |
| Versión de Elasticsearch   |           |
| Zonas de disponibilidad    |           |
| Número de nodos de datos   |           |
| Tiers habilitados          |           |
| Estado inicial del clúster |           |

## Parte B: shards, réplicas y estado yellow

### Paso B1. Crear un índice con más réplicas de las que el clúster puede alojar

``` console
PUT lab01-demo
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  }
}
```

**Qué hace:** crea un índice que pide más copias de las que el clúster puede alojar. Con 2 nodos de datos, cada shard necesita 3 nodos (1 primary y 2 réplicas) y solo hay 2.

Parámetros:

- `number_of_shards`: cantidad de primary shards. No se puede cambiar después sin split, shrink o reindex.

- `number_of_replicas`: copias por cada primary. Se puede modificar en cualquier momento.

Si el deployment tiene 3 nodos de datos, usar `"number_of_replicas": 3` para provocar la misma situación.

### Paso B2. Observar la ubicación de los shards

``` console
GET _cat/shards/lab01-demo?v&h=index,shard,prirep,state,node,unassigned.reason
```

**Qué hace:** muestra cada copia de cada shard y su ubicación.

Columnas:

- `shard`: número de shard (0 a 2).

- `prirep`: `p` indica primary y `r` indica réplica.

- `state`: STARTED, INITIALIZING o UNASSIGNED.

- `node`: nodo donde está alojada la copia; vacío si no tiene.

- `unassigned.reason`: motivo registrado por el master (por ejemplo, INDEX_CREATED).

**Resultado esperado:** los 3 primaries y 3 réplicas en STARTED, y 3 réplicas en UNASSIGNED.

### Paso B3. Confirmar el estado del índice

``` console
GET _cluster/health/lab01-demo
```

**Qué hace:** devuelve el mismo informe de salud, limitado a un índice.

**Resultado esperado:** `status: yellow`. Todos los datos están disponibles, pero la tolerancia a fallos está reducida.

### Paso B4. Diagnosticar la causa

``` console
GET _cluster/allocation/explain
{
  "index": "lab01-demo",
  "shard": 0,
  "primary": false
}
```

**Qué hace:** explica por qué una copia de shard no está asignada.

Parámetros del cuerpo:

- `index`: índice a analizar.

- `shard`: número de shard.

- `primary`: `false` para analizar una réplica, `true` para analizar el primary.

Sin cuerpo, la API analiza la primera copia sin asignar que encuentre.

Campos a revisar en la respuesta:

- `current_state`: `unassigned`.

- `unassigned_info.reason`: motivo del estado.

- `can_allocate`: `no`.

- `node_allocation_decisions[].deciders`: decisión de cada decider por nodo. Debe aparecer `same_shard`, que indica que el nodo ya tiene una copia de ese shard.

### Paso B5. Corregir y verificar

``` console
PUT lab01-demo/_settings
{
  "index": {
    "number_of_replicas": 1
  }
}
```

**Qué hace:** actualiza un setting dinámico del índice sin recrearlo ni reindexar.

``` console
GET _cluster/health/lab01-demo?wait_for_status=green&timeout=30s
```

Parámetros:

- `wait_for_status=green`: la petición espera hasta que el índice alcance ese estado.

- `timeout=30s`: tiempo máximo de espera. Si se agota, la respuesta trae `timed_out: true`.

**Resultado esperado:** `status: green`.

## Parte C: estrategias de multi-tenancy

Caso: una aplicación registra pedidos para tres tenants (acme, globex e initech). Se comparan dos estrategias y se conserva la segunda para los laboratorios siguientes.

### Paso C1. Estrategia 1: un índice por tenant

``` console
PUT orders-idx-acme
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}

PUT orders-idx-globex
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}
```

**Qué hace:** crea un índice independiente por tenant, con un primary y una réplica. Aun vacíos, cada uno ocupa 2 shards.

### Paso C2. Indexar un documento

``` console
POST orders-idx-acme/_doc?refresh=wait_for
{
  "order_id": 1001,
  "total": 250.5
}
```

**Qué hace:** indexa un documento con `_id` generado automáticamente.

Parámetros:

- `POST` sin `_id` en la ruta: Elasticsearch genera el identificador. Con `PUT orders-idx-acme/_doc/1001` se fijaría el `_id`.

- `refresh=wait_for`: la respuesta espera al siguiente refresh, de modo que el documento ya aparece en la búsqueda siguiente.

Campos a revisar en la respuesta:

- `result`: `created`.

- `_shards.total` y `_shards.successful`: copias en las que se intentó y se logró escribir (2 y 2).

- `_seq_no` y `_primary_term`: control de versiones de la operación.

### Paso C3. Estrategia 2: un índice compartido con mapeo explícito

``` console
PUT orders-shared
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "tenant_id": { "type": "keyword" },
      "order_id":  { "type": "long" },
      "total":     { "type": "double" }
    }
  }
}
```

**Qué hace:** crea un único índice para todos los tenants, con mapeo explícito.

Tipos de campo:

- `keyword`: valor exacto, sin análisis. Es el adecuado para filtrar por `tenant_id`.

- `long`: entero de 64 bits.

- `double`: decimal de 64 bits.

### Paso C4. Crear un alias filtrado por tenant

``` console
POST _aliases
{
  "actions": [
    { "add": { "index": "orders-shared", "alias": "orders-acme",
               "filter": { "term": { "tenant_id": "acme" } },
               "routing": "acme" } },
    { "add": { "index": "orders-shared", "alias": "orders-globex",
               "filter": { "term": { "tenant_id": "globex" } },
               "routing": "globex" } },
    { "add": { "index": "orders-shared", "alias": "orders-initech",
               "filter": { "term": { "tenant_id": "initech" } },
               "routing": "initech" } }
  ]
}
```

**Qué hace:** crea un alias por tenant que se comporta como un índice, pero solo ve los documentos de ese tenant.

Parámetros:

- `actions`: lista de operaciones sobre aliases, aplicadas de forma atómica.

- `add`: crea el alias.

- `index` y `alias`: índice destino y nombre del alias.

- `filter`: consulta que se aplica automáticamente en cada búsqueda por el alias.

- `routing`: dirige escrituras y búsquedas al shard que corresponde a ese valor. Con un solo shard no cambia el resultado, pero es el patrón que se usa con índices de varios shards.

El filtro actúa solo en las búsquedas. Al escribir por el alias, el documento debe llevar su propio `tenant_id`.

### Paso C5. Cargar datos de los tres tenants

``` console
POST orders-shared/_bulk?refresh=wait_for
{ "index": {} }
{ "tenant_id": "acme",    "order_id": 2001, "total": 80.0 }
{ "index": {} }
{ "tenant_id": "acme",    "order_id": 2002, "total": 120.0 }
{ "index": {} }
{ "tenant_id": "globex",  "order_id": 3001, "total": 45.5 }
{ "index": {} }
{ "tenant_id": "globex",  "order_id": 3002, "total": 300.0 }
{ "index": {} }
{ "tenant_id": "initech", "order_id": 4001, "total": 99.9 }
{ "index": {} }
{ "tenant_id": "initech", "order_id": 4002, "total": 15.0 }
```

**Qué hace:** carga 6 documentos en una sola petición con la Bulk API, que se estudia a detalle en el capítulo 4.

Formato:

- NDJSON: un objeto JSON por línea, alternando una línea de acción y una línea de documento.

- `{ "index": {} }`: acción de indexar. El índice viene de la ruta y el `_id` se genera solo.

- Revisar en la respuesta que aparezca `"errors": false`.

### Paso C6. Consultar por el alias y por el índice base

``` console
GET orders-initech/_search

GET orders-shared/_search
```

**Qué hace:** busca por el alias y por el índice base.

**Resultado esperado** (campo `hits.total.value`):

- Por el alias: 2 documentos, solo de initech.

- Por el índice base: 6 documentos, de los tres tenants.

### Paso C7. Comparar el consumo de shards

``` console
GET _cat/shards/orders-*?v&h=index,shard,prirep,state,node
```

Parámetros:

- `orders-*`: patrón que selecciona todos los índices cuyo nombre empiece con `orders-`.

**Resultado esperado:** 4 shards para los dos índices individuales (`orders-idx-acme` y `orders-idx-globex`) frente a 2 shards para `orders-shared`, aun con tres tenants. Con 500 tenants serían 1000 shards contra 2.

## Limpieza

Eliminar solo lo que no se reutiliza:

``` console
DELETE lab01-demo
DELETE orders-idx-acme
DELETE orders-idx-globex
```

**Qué hace:** elimina los índices que solo sirvieron para la comparación. `orders-shared` y sus aliases se conservan.

## Estado final y dependencias

| **Elemento**                                              | **Se conserva** | **Lo reutiliza**                                                    |
|-----------------------------------------------------------|-----------------|---------------------------------------------------------------------|
| Deployment Hosted 9.x con mínimo 2 zonas                  | Sí              | Todos los laboratorios                                              |
| Ficha del deployment completada                           | Sí              | Laboratorios 2 a 10                                                 |
| `orders-shared` con 6 documentos                          | Sí              | Laboratorio 7.4 (DLS y FLS con 2 tenants)                           |
| Aliases `orders-acme`, `orders-globex` y `orders-initech` | Sí              | Laboratorio 7.4 y, opcionalmente, laboratorio 9.4 (reindex y alias) |
| `lab01-demo`, `orders-idx-acme`, `orders-idx-globex`      | No              | Solo se usaron para comparar                                        |

Verificación final:

``` console
GET _cat/indices/orders-*?v&h=index,health,pri,rep,docs.count

GET _alias/orders-*
```

Debe verse `orders-shared` con 6 documentos y los tres aliases apuntando a él.

## Preguntas de discusión

- ¿Por qué `orders-shared/_search` devuelve datos de todos los tenants aunque existan aliases filtrados?

- ¿Cuántos shards consumiría cada estrategia con 500 tenants?

- ¿Qué estrategia elegirías para 5 tenants grandes? ¿Y para 2000 tenants pequeños?

- ¿Qué cambia si cada tenant exige una política de retención distinta?

- Con 2 zonas de disponibilidad, ¿qué pasa con el estado del clúster si se pierde una zona completa?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 1: Evaluación de topologías en la nube](imagenes/lab01_resultado_final.png)
