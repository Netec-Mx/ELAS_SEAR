# Lab 1 · Arquitectura, roles de nodos y salud de `rutanorte-logs`

| | |
|---|---|
| **Duración** | 55 minutos |
| **Entorno** | Kibana › Dev Tools › Console, con la sesión de `rn_admin` |
| **Archivos que usarás** | `cap01/lab01-arquitectura.txt` (comandos) · `cap01/hoja-arquitectura.md` (evidencia) |
| **Requisitos** | Actividad 0 completa: deployment `rutanorte-logs` en estado `green` y usuario `rn_admin` |
| **Deja listo para** | Lab 2 (inventario) y Lab 3 (diseño de data streams) |

---

## Objetivo

Analizar la arquitectura del deployment `rutanorte-logs` en Elastic Cloud Hosted:
- identificar sus nodos, roles, zonas y maestro elegido;
- observar cómo reparte Elasticsearch los shards primarios y las réplicas;
- provocar un estado **yellow**, diagnosticarlo con `_cluster/allocation/explain` y corregirlo con la acción adecuada.

## Contexto

El deployment `rutanorte-logs` ya existe, pero para el equipo de Plataforma es una caja negra: Elastic eligió cuántas máquinas asignar, cómo llamarlas y dónde ubicarlas. Antes de diseñar cómo guardar los logs de los seis servicios (Lab 3), necesitas responder tres preguntas que te hará cualquier auditor o compañero de guardia:

1. **¿De qué está hecho el clúster?** Nodos, roles y zonas.
2. **¿Qué pasa si se cae una zona?** Dónde quedan los primarios y las réplicas.
3. **¿Cómo sé si algo está mal y qué hago?** Estados de salud y su diagnóstico.

La evidencia de este lab queda en la **hoja de arquitectura**, el primer documento de operación de RutaNorte.

## Qué vas a hacer

| Paso | Actividad | Pregunta que responde |
|---|---|---|
| 1.1 | Listar nodos y roles con `_cat/nodes` | ¿Cuántos nodos hay y qué hace cada uno? |
| 1.2 | Ver zonas y atributos con `_cat/nodeattrs` | ¿Dónde vive cada nodo? |
| 1.3 | Identificar el maestro elegido y los nodos que votan | ¿Quién coordina el clúster y qué pasa si falla? |
| 1.4 | Revisar la salud con `_cluster/health` y `_health_report` | ¿Está sano el clúster? |
| 1.5 | Crear un data stream y observar el reparto de shards | ¿Cómo protege Elasticsearch los datos entre zonas? |
| 1.6 | Provocar y diagnosticar un estado yellow | ¿Qué significa yellow y cuál es la acción correcta? |
| 1.7 | Limpiar los recursos de práctica | ¿Cómo se borra de forma segura? |

---

## Paso 1.1 · Inventario de nodos y sus roles

**Qué estás haciendo.** Usas la familia de APIs `_cat`, que devuelve tablas en texto para leerlas rápido en la consola. `_cat/nodes` lista cada nodo con las columnas que elijas.

Ejecuta el bloque **1.1**:

```
GET _cat/nodes?v&h=name,node.role,master,heap.percent,ram.percent,cpu,disk.used_percent&s=name
```

| Parámetro | Explicación |
|---|---|
| `v` | *Verbose*: agrega la fila de encabezados |
| `h=` | Columnas a mostrar, separadas por comas. El bloque **1.1b** (`GET _cat/nodes?help`) lista todas las disponibles |
| `s=name` | Ordena por la columna `name` (agrega `:desc` para orden descendente) |
| `node.role` | Roles del nodo, abreviados con una letra cada uno |
| `master` | `*` marca al maestro elegido; `-` a los demás |
| `heap.percent` | Porcentaje de memoria heap de la JVM en uso |
| `disk.used_percent` | Porcentaje de disco usado; es clave para las marcas de agua (*watermarks*) |

**Resultado esperado** (ejemplo de un deployment de prueba de 2 zonas):

```
name                  node.role master heap.percent ram.percent cpu disk.used_percent
instance-0000000000   himrst    *                38          92   4              1.21
instance-0000000001   himrst    -                31          91   3              1.19
tiebreaker-0000000002 mv        -                22          88   1              0.35
```

**Cómo leer los roles**

| Letra | Rol | Letra | Rol |
|---|---|---|---|
| `h` | data_hot | `m` | master (elegible como maestro) |
| `i` | ingest | `v` | voting_only (vota, pero no puede ser maestro) |
| `r` | remote_cluster_client | `s` | data_content |
| `t` | transform | `w` / `c` / `f` | data_warm / data_cold / data_frozen |
| `l` | ml (machine learning) | `d` | data (genérico, sin tiers) |

**Interpretación.** Los nodos `instance-…` hacen casi todo: guardan datos del tier hot, ejecutan pipelines y son elegibles como maestro. El `tiebreaker-…` solo vota. ECH lo agrega en una tercera zona para que siempre haya mayoría en la elección del maestro. En producción, con deployments más grandes, ECH separa los maestros en nodos dedicados.

> Anota cada nodo en la sección 1 de `hoja-arquitectura.md`.

## Paso 1.2 · Zonas y atributos de cada nodo

**Qué estás haciendo.** ECH etiqueta cada nodo con **atributos**: pares clave-valor que describen dónde y cómo corre. Elasticsearch usa el atributo de zona para **no poner dos copias del mismo shard en la misma zona** (*allocation awareness*).

Ejecuta el bloque **1.2**:

```
GET _cat/nodeattrs?v&h=node,attr,value&s=node,attr
```

**Resultado esperado** (extracto):

```
node                  attr                       value
instance-0000000000   availability_zone          us-east-1a
instance-0000000000   data                       hot
instance-0000000000   instance_configuration     aws.es.datahot.c6gd
instance-0000000000   logical_availability_zone  zone-0
instance-0000000000   region                     us-east-1
instance-0000000001   availability_zone          us-east-1b
instance-0000000001   logical_availability_zone  zone-1
...
tiebreaker-0000000002 logical_availability_zone  tiebreaker
```

| Atributo | Significado |
|---|---|
| `availability_zone` | Zona física del proveedor cloud |
| `logical_availability_zone` | Zona lógica del deployment (`zone-0`, `zone-1`…). ECH la usa para el *awareness* |
| `data` | Tier de datos del nodo (`hot`, `warm`, `cold`, `frozen`) |
| `instance_configuration` | Tipo de instancia del catálogo de ECH |
| `region` | Región del deployment |

Para una vista compacta en JSON, ejecuta el bloque **1.2b**. Usa `filter_path` con el comodín `*`, que significa "cualquier ID de nodo":

```
GET _nodes?filter_path=nodes.*.name,nodes.*.roles,nodes.*.attributes.logical_availability_zone,nodes.*.attributes.availability_zone
```

> Los nombres de los atributos y de las instancias pueden variar por proveedor y región. Lo importante es que **cada nodo de datos está en una zona distinta**.

## Paso 1.3 · Maestro elegido y quórum

**Qué estás haciendo.** El nodo maestro mantiene el *estado del clúster* (índices, mappings, ubicación de shards) y lo publica a los demás. Solo hay un maestro elegido a la vez. Para elegirlo se necesita la **mayoría** de los nodos que votan.

Ejecuta el bloque **1.3**:

```
GET _cat/master?v
```

```
id                     host         ip           node
Xk2pQ8rTS3m0aBcDeFgHiJ 10.42.1.17   10.42.1.17   instance-0000000000
```

Ejecuta el bloque **1.3b** para ver la configuración de votación que el clúster usa actualmente:

```
GET _cluster/state/metadata?filter_path=metadata.cluster_coordination.last_committed_config
```

La respuesta es una lista de IDs de nodo. Con 3 nodos que votan (dos `instance` y el `tiebreaker`), la mayoría es **2**.

**Pregunta para tu hoja:** si falla la zona `zone-0` completa, ¿quedan votos suficientes para elegir maestro? ¿Y si fallan dos zonas?

## Paso 1.4 · Salud del clúster

**Qué estás haciendo.** Comparas dos APIs de salud. `_cluster/health` responde *qué tan bien están asignados los shards*. `_health_report` va más allá: evalúa indicadores como la estabilidad del maestro, el disco, la capacidad de shards, ILM o SLM, y propone diagnósticos con acciones concretas.

Ejecuta el bloque **1.4**:

```
GET _cluster/health

GET _health_report?verbose=false
```

| Parámetro / campo | Explicación |
|---|---|
| `_health_report?verbose=false` | Devuelve solo el estado y un resumen por indicador, sin el detalle de diagnóstico. Es más ligero para revisiones rutinarias |
| `indicators.master_is_stable` | ¿El maestro ha cambiado de forma anómala? |
| `indicators.shards_availability` | ¿Hay primarios o réplicas sin asignar? |
| `indicators.disk` | ¿Algún nodo se acerca a las marcas de agua de disco? |
| `indicators.shards_capacity` | ¿El clúster se acerca al límite de shards por nodo? |

**Resultado esperado:** `"status": "green"` en ambas APIs. Si algún indicador aparece en `unknown` (por ejemplo `repository_integrity` o `slm` antes de configurar snapshots propios), anótalo: no es un error.

## Paso 1.5 · Crear un data stream y observar el reparto de shards

**Qué estás haciendo.** Escribes eventos de log de prueba de `reservas-api` en un **data stream**, el contenedor recomendado para datos con marca de tiempo que solo se agregan. Todavía no creas plantillas propias (eso es el Lab 3), así que aprovechas la **plantilla integrada `logs`**, que se aplica a todo nombre con el patrón `logs-*-*`.

**1.5 · Primer evento.** Ejecuta el bloque **1.5**:

```
POST logs-lab01.reservas-default/_doc
{
  "@timestamp": "2026-03-27T18:00:00.000Z",
  "service.name": "reservas-api",
  "log.level": "info",
  "message": "Reserva RN-100231 creada para la corrida MTY-SLT 18:30"
}
```

| Parte | Explicación |
|---|---|
| `logs-lab01.reservas-default` | Nombre con el esquema `tipo-dataset-namespace`: `logs` · `lab01.reservas` · `default`. Coincide con `logs-*-*`, por eso Elasticsearch crea el data stream automáticamente |
| `POST …/_doc` | Indexa un documento y deja que Elasticsearch genere el `_id`. En los data streams **no** se puede usar `PUT …/_doc/<id>` para crear documentos: solo se agregan |
| `@timestamp` | Campo **obligatorio** en un data stream. Ordena los eventos en el tiempo |

Respuesta: `"result": "created"`, con `_index` igual a algo como `.ds-logs-lab01.reservas-default-2026.10.19-000001`. Ese es el **índice de respaldo** (*backing index*) donde quedó el documento.

**1.5b · Más eventos con `_bulk`.** Ejecuta el bloque **1.5b**. La API `_bulk` envía muchas operaciones en una sola solicitud:

```
POST logs-lab01.reservas-default/_bulk
{ "create": {} }
{ "@timestamp": "2026-03-27T18:00:05.000Z", "service.name": "reservas-api", "log.level": "info", "message": "..." }
...
```

| Parte | Explicación |
|---|---|
| Formato NDJSON | Una línea de **acción** seguida de una línea de **documento**. Cada línea es un JSON completo, sin saltos de línea dentro |
| `{ "create": {} }` | En data streams la única acción de escritura válida es `create`. El índice ya viene en la ruta |
| `errors` en la respuesta | `false` si todas las operaciones tuvieron éxito. Revísalo siempre: `_bulk` responde HTTP 200 aunque falle algún documento |

**1.5c · Anatomía del data stream.** Ejecuta el bloque **1.5c**:

```
GET _data_stream/logs-lab01.reservas-default
```

Registra en tu hoja:

| Campo | Valor esperado | Qué significa |
|---|---|---|
| `indices[0].index_name` | `.ds-logs-lab01.reservas-default-AAAA.MM.DD-000001` | Índice de respaldo actual (generación 1) |
| `template` | `logs` | Plantilla integrada que lo creó |
| `ilm_policy` | `logs` | Política de ciclo de vida integrada (rollover y retención por defecto) |
| `index_mode` | `logsdb` | Modo optimizado para logs, predeterminado en 9.x para `logs-*-*`: ordena y comprime para usar menos almacenamiento |
| `status` | `GREEN` | Salud del data stream |

**1.5d · ¿Dónde quedaron las copias?** Ejecuta el bloque **1.5d**:

```
GET _cat/shards/.ds-logs-lab01.reservas-default-*?v&h=index,shard,prirep,state,docs,store,node
```

```
index                                               shard prirep state   docs store node
.ds-logs-lab01.reservas-default-2026.10.19-000001   0     p      STARTED    5 9.8kb instance-0000000001
.ds-logs-lab01.reservas-default-2026.10.19-000001   0     r      STARTED    5 9.8kb instance-0000000000
```

| Columna | Significado |
|---|---|
| `prirep` | `p` = primario, `r` = réplica |
| `state` | `STARTED` (activo), `INITIALIZING`, `RELOCATING` o `UNASSIGNED` |
| `docs` | Documentos en esa copia. El primario y la réplica deben coincidir |

**1.5e · Reparto con varios primarios.** Ejecuta el bloque **1.5e**, que crea un índice clásico con 2 primarios y 1 réplica y luego muestra sus shards:

```
PUT lab01-reparto
{
  "settings": {
    "index.number_of_shards": 2,
    "index.number_of_replicas": 1
  }
}
```

| Setting | Explicación |
|---|---|
| `index.number_of_shards` | Primarios del índice. **No se puede cambiar** después de crearlo (solo con *split*, *shrink* o reindexando) |
| `index.number_of_replicas` | Réplicas por primario. **Sí se puede cambiar** en caliente |

Cruza los nodos del resultado con las zonas del Paso 1.2. Vas a comprobar la regla: **el primario y la réplica de un mismo shard nunca comparten nodo ni zona**. Por eso, si falla una zona, la otra conserva una copia completa de los datos.

## Paso 1.6 · Provocar y diagnosticar un estado yellow

**Qué estás haciendo.** Reproduces de forma controlada una situación que verás en producción: un índice que pide más copias de las que el clúster puede alojar. Aprendes a diagnosticarla **con evidencia** y no por intuición.

**1.6 · Provocar.** Ejecuta el bloque **1.6**:

```
PUT lab01-amarillo
{
  "settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 2
  }
}

GET _cluster/health?filter_path=status,number_of_data_nodes,active_shards,unassigned_shards

GET _cat/shards/lab01-amarillo?v&h=index,shard,prirep,state,unassigned.reason,node
```

Pides 3 copias (1 primario + 2 réplicas) del shard 0, pero solo hay **2 nodos de datos**. Resultado:

```json
{ "status": "yellow", "number_of_data_nodes": 2, "active_shards": 24, "unassigned_shards": 1 }
```

```
index          shard prirep state      unassigned.reason node
lab01-amarillo 0     p      STARTED                      instance-0000000000
lab01-amarillo 0     r      STARTED                      instance-0000000001
lab01-amarillo 0     r      UNASSIGNED INDEX_CREATED
```

**1.6b · Diagnosticar.** Ejecuta el bloque **1.6b**:

```
GET _cluster/allocation/explain
{
  "index": "lab01-amarillo",
  "shard": 0,
  "primary": false
}
```

| Campo del cuerpo | Explicación |
|---|---|
| `index`, `shard` | Shard a explicar |
| `primary` | `false` para preguntar por una réplica, `true` para el primario |
| *(sin cuerpo)* | Elasticsearch explica el **primer** shard sin asignar que encuentre. Útil cuando no sabes cuál falla |

**Resultado esperado** (extracto):

```json
{
  "index": "lab01-amarillo",
  "shard": 0,
  "primary": false,
  "current_state": "unassigned",
  "unassigned_info": { "reason": "INDEX_CREATED", "last_allocation_status": "no_attempt" },
  "can_allocate": "no",
  "allocate_explanation": "Elasticsearch isn't allowed to allocate this shard to any of the nodes in the cluster...",
  "node_allocation_decisions": [
    {
      "node_name": "instance-0000000000",
      "node_decision": "no",
      "deciders": [
        { "decider": "same_shard", "decision": "NO",
          "explanation": "a copy of this shard is already allocated to this node [[lab01-amarillo][0], ...]" }
      ]
    },
    { "node_name": "instance-0000000001", "node_decision": "no", "deciders": [ { "decider": "same_shard", "decision": "NO" } ] }
  ]
}
```

**Interpretación.** El *decider* `same_shard` prohíbe dos copias del mismo shard en un nodo. Ambos nodos ya tienen una copia, así que no hay dónde colocar la tercera. También puede aparecer el decider `awareness`, que aplica la misma lógica por zona.

**1.6c · Segunda opinión.** Ejecuta el bloque **1.6c**:

```
GET _health_report/shards_availability
```

El indicador aparece en `yellow`. En `diagnosis` propone acciones concretas, como reducir réplicas o agregar nodos, y en `affected_resources` lista los índices afectados.

**1.6d · Corregir con la acción correcta.** Ejecuta el bloque **1.6d**:

```
PUT lab01-amarillo/_settings
{
  "index.number_of_replicas": 1
}

GET _cluster/health?filter_path=status,unassigned_shards
```

El estado vuelve a `green` y `unassigned_shards` a `0`.

> **¿Por qué no agregar un nodo?** Porque el problema no era de capacidad, sino de **configuración**: 2 réplicas no aportan protección adicional en un clúster de 2 zonas. Con 1 réplica, cada zona ya tiene una copia completa. Agregar nodos solo para que quepa una copia inútil aumenta el costo sin aumentar la disponibilidad.

**Impacto de un yellow:** no se pierden datos y se puede escribir y buscar con normalidad, pero se **reduce la redundancia**. Si un nodo fallara en este momento, algún shard podría quedar sin copias. Con `red` (primario sin asignar), sí hay datos inaccesibles y escrituras rechazadas para ese índice.

> Completa la sección 4 de `hoja-arquitectura.md`.

## Paso 1.7 · Limpieza

**Qué estás haciendo.** Borras los recursos de práctica. En Elasticsearch 9.x, `action.destructive_requires_name` vale `true` por defecto: **no se permiten comodines** al borrar índices, para evitar accidentes como `DELETE lab01-*` o `DELETE *`.

Ejecuta el bloque **1.7**:

```
DELETE lab01-amarillo

DELETE lab01-reparto

DELETE _data_stream/logs-lab01.reservas-default

GET _cluster/health?filter_path=status
```

| Comando | Explicación |
|---|---|
| `DELETE <índice>` | Borra el índice y sus datos de forma permanente |
| `DELETE _data_stream/<nombre>` | Borra el data stream **y todos sus índices de respaldo**. Un índice de respaldo `.ds-…` no se puede borrar por separado mientras sea el índice de escritura |

---

## Verificación

- [ ] La hoja de arquitectura tiene completos los nodos, roles, zonas y el maestro elegido.
- [ ] Explicaste por qué hay un nodo tiebreaker y cuántos votos forman la mayoría.
- [ ] Documentaste en qué nodo y zona quedaron el primario y la réplica del data stream.
- [ ] Explicaste el estado yellow: causa (`same_shard`), impacto (menos redundancia, sin pérdida) y acción (bajar a 1 réplica).
- [ ] `GET _cluster/health` termina en `green` y no quedan índices `lab01-*`.

## Pantalla de resultado final

![Diagnóstico del estado yellow con allocation/explain en Kibana Dev Tools](img/lab01-resultado-final.png)

*Pantalla de referencia del Paso 1.6b. La respuesta de `_cluster/allocation/explain` muestra `can_allocate: "no"` y el decider `same_shard` en cada nodo de datos. Los nombres de nodo e identificadores cambian en cada deployment.*

## Preguntas de reflexión

1. RutaNorte quiere sobrevivir a la caída de **una** zona sin perder escritura. ¿Basta con la topología actual? ¿Qué cambiaría con 3 zonas?
2. ¿Por qué un patrón de búsqueda como `logs-*` es más costoso que `logs-rutanorte.pagos-*`? Relaciónalo con el patrón *scatter-gather*.
3. ¿Qué tareas de esta práctica harías tú y cuáles hace Elastic en ECH? Completa la sección 5 de la hoja.

## Lo que dejas listo para el Lab 2

- La **hoja de arquitectura** con la topología real del deployment.
- Un clúster limpio, en `green`.

En el Lab 2 vas a completar el expediente del deployment: versión exacta, licencia, acceso seguro con API key y compatibilidad de los componentes que RutaNorte ya tiene funcionando.

---

## Archivo completo · `cap01/lab01-arquitectura.txt`

```
# =====================================================================
# Lab 1 · Arquitectura, roles de nodos y salud de rutanorte-logs
# Archivo: cap01/lab01-arquitectura.txt
# Entorno: Kibana › Dev Tools › Console (sesión de rn_admin)
# =====================================================================

### 1.1 Inventario de nodos y sus roles
GET _cat/nodes?v&h=name,node.role,master,heap.percent,ram.percent,cpu,disk.used_percent&s=name

### 1.1b Leyenda completa de columnas disponibles en _cat/nodes
GET _cat/nodes?help

### 1.2 Zonas y atributos que ECH asigna a cada nodo
GET _cat/nodeattrs?v&h=node,attr,value&s=node,attr

### 1.2b Vista compacta: solo la zona lógica y la física
GET _nodes?filter_path=nodes.*.name,nodes.*.roles,nodes.*.attributes.logical_availability_zone,nodes.*.attributes.availability_zone

### 1.3 Maestro elegido
GET _cat/master?v

### 1.3b Nodos que votan en la elección del maestro
GET _cluster/state/metadata?filter_path=metadata.cluster_coordination.last_committed_config

### 1.4 Salud: vista rápida y reporte detallado
GET _cluster/health

GET _health_report?verbose=false

### 1.5 Crear un data stream de prueba (usa la plantilla integrada "logs")
POST logs-lab01.reservas-default/_doc
{
  "@timestamp": "2026-03-27T18:00:00.000Z",
  "service.name": "reservas-api",
  "log.level": "info",
  "message": "Reserva RN-100231 creada para la corrida MTY-SLT 18:30"
}

### 1.5b Agregar más eventos con la API _bulk
POST logs-lab01.reservas-default/_bulk
{ "create": {} }
{ "@timestamp": "2026-03-27T18:00:05.000Z", "service.name": "reservas-api", "log.level": "info", "message": "Reserva RN-100232 creada para la corrida MTY-CUU 21:00" }
{ "create": {} }
{ "@timestamp": "2026-03-27T18:00:09.000Z", "service.name": "reservas-api", "log.level": "warn", "message": "Asiento 14 ya bloqueado en la corrida MTY-SLT 18:30, se reintenta" }
{ "create": {} }
{ "@timestamp": "2026-03-27T18:00:12.000Z", "service.name": "reservas-api", "log.level": "info", "message": "Reserva RN-100233 creada para la corrida TIJ-HMO 07:15" }
{ "create": {} }
{ "@timestamp": "2026-03-27T18:00:20.000Z", "service.name": "reservas-api", "log.level": "error", "message": "Tiempo de espera agotado al consultar asientos-api (3000 ms)" }

### 1.5c ¿Qué se creó? Data stream, índice de respaldo, plantilla y modo de índice
GET _data_stream/logs-lab01.reservas-default

### 1.5d ¿Dónde quedaron el primario y la réplica?
GET _cat/shards/.ds-logs-lab01.reservas-default-*?v&h=index,shard,prirep,state,docs,store,node

### 1.5e Índice con 2 primarios y 1 réplica para observar el reparto
PUT lab01-reparto
{
  "settings": {
    "index.number_of_shards": 2,
    "index.number_of_replicas": 1
  }
}

GET _cat/shards/lab01-reparto?v&h=index,shard,prirep,state,node&s=shard,prirep

### 1.6 Provocar un estado yellow: pedir más réplicas que nodos de datos
PUT lab01-amarillo
{
  "settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 2
  }
}

GET _cluster/health?filter_path=status,number_of_data_nodes,active_shards,unassigned_shards

GET _cat/shards/lab01-amarillo?v&h=index,shard,prirep,state,unassigned.reason,node

### 1.6b Preguntar a Elasticsearch por qué no asigna la réplica
GET _cluster/allocation/explain
{
  "index": "lab01-amarillo",
  "shard": 0,
  "primary": false
}

### 1.6c El indicador de disponibilidad de shards del health report
GET _health_report/shards_availability

### 1.6d Acción correcta: alinear las réplicas con la topología
PUT lab01-amarillo/_settings
{
  "index.number_of_replicas": 1
}

GET _cluster/health?filter_path=status,unassigned_shards

### 1.7 Limpieza (los nombres deben ser exactos: los comodines no se permiten al borrar)
DELETE lab01-amarillo

DELETE lab01-reparto

DELETE _data_stream/logs-lab01.reservas-default

GET _cluster/health?filter_path=status
```
