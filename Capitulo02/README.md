# Lab 2 · Versión, licencia, acceso y salud: inventario de `rutanorte-logs`

| | |
|---|---|
| **Duración** | 40 minutos |
| **Entorno** | Kibana › Dev Tools › Console (sesión de `rn_admin`) y navegador |
| **Archivos que usarás** | `cap02/lab02-inventario.txt` (comandos) · `cap02/ficha-inventario.md` (ficha) · `cap02/componentes-rutanorte.csv` (componentes a evaluar) |
| **Requisitos** | Actividad 0 y Lab 1 completos; `registro-deployment.md` y `hoja-arquitectura.md` a la mano |
| **Deja listo para** | Lab 3 (diseño de data streams) y Lab 9 (actualización y recuperación) |

---

## Objetivo

Consultar y documentar, con evidencia y fecha, la **versión, el build, la licencia, el acceso seguro y la salud** del deployment `rutanorte-logs`. Después, evaluar si los componentes que RutaNorte ya tiene en operación son **compatibles** con Elasticsearch 9.x.

## Contexto

Seguridad pidió al equipo de Plataforma un **acta de entrega** del deployment antes de autorizar que los servicios envíen logs. Quiere saber exactamente qué versión se recibió, con qué licencia, cómo se accede de forma segura y si lo que ya existe es compatible: agentes, Logstash y scripts de reportes. Algunos de esos componentes vienen del clúster anterior, todavía en 7.17.

Este inventario es también tu **línea base**. Cuando algo cambie (una actualización, un incidente, una auditoría), lo compararás contra esta ficha.

## Qué vas a hacer

| Paso | Actividad | Evidencia que produce |
|---|---|---|
| 2.1 | Verificar el acceso HTTPS y la confianza en el certificado | Emisor y vigencia del certificado; acceso sin desactivar TLS |
| 2.2 | Crear la API key de solo lectura `rn-inventario` | ID, privilegios y vencimiento de la API key |
| 2.3 | Registrar versión, build y compatibilidad mínima | Versión de Elasticsearch, Lucene y Kibana |
| 2.4 | Consultar la licencia y las funcionalidades disponibles | Tipo, estado y vencimiento de la licencia |
| 2.5 | Reunir estadísticas, plugins y salud | Conteos de índices, shards, documentos y almacenamiento |
| 2.6 | Revisar avisos de deprecación | Lista de deprecaciones de Elasticsearch y Kibana |
| 2.7 | Evaluar la compatibilidad de los componentes y cerrar la ficha | Ficha de inventario completa |

---

## Paso 2.1 · Verificar el acceso HTTPS y la confianza en el certificado

**Qué estás haciendo.** Compruebas que el endpoint de Elasticsearch usa TLS con un certificado **emitido por una autoridad pública de confianza**. En ECH no hay que distribuir ni instalar una CA propia: cualquier cliente moderno confía en el certificado sin opciones inseguras como `curl -k` o `verify_certs=False`.

1. Copia el **endpoint de Elasticsearch** de tu `registro-deployment.md` (tiene la forma `https://<id>.<región>.<proveedor>.cloud.es.io`).
2. Pégalo en una pestaña nueva del navegador.
3. El navegador pide usuario y contraseña. Entra como `rn_admin`. Verás el mismo JSON que devuelve `GET /` en Console.
4. Haz clic en el **candado** de la barra de direcciones › *La conexión es segura* › *El certificado es válido*. Registra en la sección 4 de la ficha:
   - **Emitido para:** el dominio comodín del proveedor en ECH.
   - **Emitido por:** la autoridad certificadora pública.
   - **Válido hasta:** la fecha de vencimiento. Elastic renueva el certificado; tú no tienes que hacerlo.

> **¿Por qué importa?** Un cliente que desactiva la verificación TLS acepta cualquier certificado, incluido el de un atacante que intercepte el tráfico. En RutaNorte la regla es: **nunca `-k`**.

## Paso 2.2 · Crear la API key de solo lectura `rn-inventario`

**Qué estás haciendo.** Creas la credencial que usarán las herramientas de monitoreo e inventario. A diferencia de un usuario:
- una API key **no tiene contraseña que alguien teclee**;
- puede **vencer sola**;
- sus privilegios pueden ser **menores** que los de quien la crea, nunca mayores.

Ejecuta el bloque **2.2**:

```
POST _security/api_key
{
  "name": "rn-inventario",
  "expiration": "30d",
  "role_descriptors": {
    "rn_inventario_lectura": {
      "cluster": [ "monitor" ],
      "indices": [
        {
          "names": [ "logs-rutanorte.*" ],
          "privileges": [ "read", "view_index_metadata" ]
        }
      ]
    }
  },
  "metadata": {
    "proyecto": "rutanorte-logs",
    "uso": "inventario y monitoreo de solo lectura",
    "creado_en": "lab-02"
  }
}
```

| Parámetro | Explicación |
|---|---|
| `name` | Nombre descriptivo. No tiene que ser único, pero en RutaNorte lo será por convención |
| `expiration` | Vigencia: `30d` = 30 días. También acepta `h` y `m`. Sin este campo, la key **no vence**: evítalo |
| `role_descriptors` | Privilegios de la key. Si se omite, la key **hereda todos** los privilegios del creador. Como `rn_admin` es superuser, eso sería peligroso |
| `cluster: ["monitor"]` | Permite leer el estado, la salud y las estadísticas del clúster, sin modificar nada |
| `indices.names` | Índices o data streams a los que aplica. `logs-rutanorte.*` anticipa los data streams que crearás en el Lab 3 |
| `read`, `view_index_metadata` | Buscar documentos y ver mappings y settings, sin escribir ni borrar |
| `metadata` | Información libre para auditoría |

**Resultado esperado:**

```json
{
  "id": "VuaCfGcBCdbkQm-e5aOx",
  "name": "rn-inventario",
  "expiration": 1763520000000,
  "api_key": "ui2lp2axTNmsyakw9tvNnw",
  "encoded": "VnVhQ2ZHY0JDZGJrUW0tZTVhT3g6dWkybHAyYXhUTm1zeWFrdzl0dk5udw=="
}
```

> **El secreto solo se muestra una vez.** Copia el valor de `encoded` a tu gestor de contraseñas. Es el que se envía en el encabezado `Authorization: ApiKey <encoded>`. **No lo pegues en la ficha ni en ningún archivo compartido.**

Ejecuta el bloque **2.2b** para comprobar la key sin exponer el secreto:

```
GET _security/api_key?name=rn-inventario&active_only=true
```

| Parámetro | Explicación |
|---|---|
| `name` | Filtra por nombre. Admite comodines, por ejemplo `rn-*` |
| `active_only=true` | Excluye las keys vencidas o invalidadas |

Registra el `id`, la fecha de `expiration` y `role_descriptors` en la sección 4 de la ficha. También puedes ver la key en **Stack Management › Security › API keys**.

> **Prueba fuera de Kibana (opcional, solo si tienes terminal).** Ejecuta `curl -H "Authorization: ApiKey <encoded>" https://<endpoint>/_cluster/health`. Debe responder **sin `-k`**. Después prueba `curl -X DELETE -H "Authorization: ApiKey <encoded>" https://<endpoint>/cualquier-indice`: debe devolver **403**, porque la key es de solo lectura. En el Lab 7 harás estas pruebas de permitido y denegado de forma sistemática.

## Paso 2.3 · Versión, build y compatibilidad mínima

**Qué estás haciendo.** Registras la versión exacta. No basta con "9.3": hacen falta el número de parche y el build, porque las notas de versión, los avisos de seguridad y las rutas de actualización se publican por parche.

Ejecuta el bloque **2.3** (`GET /`) y registra en la sección 2 de la ficha:

| Campo | Ejemplo | Qué significa |
|---|---|---|
| `version.number` | `9.3.2` | Versión mayor.menor.parche |
| `version.build_flavor` | `default` | Distribución con todas las funciones de Elastic |
| `version.build_type` | `docker` | En ECH, los nodos corren como contenedores |
| `version.build_hash` | `a1b2c3d…` | Commit exacto del código compilado |
| `version.build_date` | `2026-…` | Fecha de compilación |
| `version.lucene_version` | `10.3.2` | Versión del motor de búsqueda subyacente |
| `minimum_wire_compatibility_version` | `8.19.0` | Versión mínima con la que estos nodos se comunican por la red de transporte. Por eso **8.19 es el puente** obligatorio desde 8.x hacia 9.x |
| `minimum_index_compatibility_version` | `8.0.0` | Versión mínima en que se pudo crear un índice para que 9.x lo lea. Los índices creados en **7.x** no se pueden abrir para escritura en 9.x: hay que reindexarlos o marcarlos como de solo lectura antes de actualizar (Lab 9) |

Ejecuta el bloque **2.3b** para confirmar que todos los nodos tienen la misma versión:

```
GET _cat/nodes?v&h=name,version,build,jdk&s=name
```

> Durante una actualización en ECH verás por un rato nodos con versiones distintas. Fuera de una actualización, una diferencia de versiones es una anomalía que debes reportar.

Ejecuta el bloque **2.3c** para la versión de **Kibana**:

```
GET kbn:/api/status
```

| Parte | Explicación |
|---|---|
| `kbn:` | Prefijo de Console que envía la solicitud **a la API de Kibana** en lugar de a Elasticsearch |
| `version.number` | Versión de Kibana. En ECH siempre coincide con la de Elasticsearch |
| `status.overall.level` | `available` si Kibana y sus servicios están sanos |

## Paso 2.4 · Licencia y funcionalidades

**Qué estás haciendo.** La licencia determina qué funciones puedes usar: seguridad avanzada, machine learning, snapshots buscables, modo LogsDB con *synthetic source*… En ECH, Elastic administra la licencia según la suscripción de la organización. Tú solo la consultas y la documentas.

Ejecuta el bloque **2.4**:

```
GET _license
```

| Campo | Qué registrar |
|---|---|
| `license.type` | `trial`, `basic`, `gold`, `platinum` o `enterprise`. En ECH normalmente verás `enterprise` en pruebas y en suscripciones Enterprise |
| `license.status` | `active` o `expired` |
| `license.issued_to` | Entidad a la que se emitió (en ECH, un identificador del deployment u organización) |
| `license.expiry_date` | Vencimiento. En ECH Elastic la renueva según la suscripción |

Ejecuta el bloque **2.4b**:

```
GET _xpack?categories=license,features&filter_path=license,features.security,features.ml,features.searchable_snapshots
```

| Parámetro | Explicación |
|---|---|
| `categories=license,features` | Devuelve solo la licencia y la lista de funcionalidades, sin la información del build |
| `features.<nombre>.available` | Si la licencia permite la función |
| `features.<nombre>.enabled` | Si la función está habilitada en el clúster |

## Paso 2.5 · Estadísticas, plugins y salud

**Qué estás haciendo.** Reúnes en una sola consulta los números que piden Finanzas y Capacidad: cuántos nodos, índices, shards y documentos hay, y cuánto almacenamiento se usa.

Ejecuta el bloque **2.5**:

```
GET _cluster/stats?human&filter_path=cluster_name,cluster_uuid,status,nodes.count,nodes.versions,nodes.jvm.versions.version,nodes.os.pretty_names,indices.count,indices.shards.total,indices.docs.count,indices.store.size
```

| Parámetro / campo | Explicación |
|---|---|
| `human` | Agrega versiones legibles de tamaños y tiempos (por ejemplo `"size": "12.4mb"`) |
| `nodes.count` | Total de nodos y su desglose por rol |
| `nodes.versions` | Lista de versiones presentes en el clúster: debe tener **un solo valor** |
| `nodes.jvm.versions.version` | Versión de la JVM incluida con Elasticsearch |
| `indices.count` / `indices.shards.total` | Índices y shards totales, incluidos los de sistema |
| `indices.store.size` | Almacenamiento total, réplicas incluidas |

Ejecuta el bloque **2.5b** (`GET _cat/plugins?v&h=name,component,version`). En ECH es normal que salga **vacío**: las funciones de snapshots en la nube ya vienen integradas como módulos.

Ejecuta el bloque **2.5c** para la salud resumida:

```
GET _cluster/health?filter_path=status,number_of_nodes,number_of_data_nodes,active_shards,unassigned_shards

GET _health_report?verbose=false&filter_path=status,indicators.*.status
```

El comodín `indicators.*.status` devuelve solo el estado de cada indicador: una línea por indicador, ideal para la ficha.

## Paso 2.6 · Avisos de deprecación

**Qué estás haciendo.** Una *deprecación* es una configuración o función que todavía funciona pero que se eliminará en una versión futura. Revisarlas hoy evita sorpresas en la próxima actualización.

Ejecuta el bloque **2.6**:

```
GET _migration/deprecations
```

La respuesta agrupa los avisos por origen: `cluster_settings`, `node_settings`, `index_settings`, `data_streams`, `templates`, `ilm_policies` y `ml_settings`. Cada aviso tiene:

| Campo | Significado |
|---|---|
| `level` | `critical`: bloquea la actualización a la siguiente versión mayor. `warning`: conviene corregirlo |
| `message` | Descripción breve |
| `url` | Documentación con la corrección |
| `details` | Detalle del recurso afectado |

En un deployment nuevo lo normal es que todas las listas estén **vacías** (`[]`). Anótalo así en la sección 6 de la ficha: "Sin deprecaciones a la fecha" también es evidencia.

Ejecuta el bloque **2.6b** para los avisos de Kibana:

```
GET kbn:/api/deprecations/
```

> También puedes revisar las deprecaciones en **Stack Management › Upgrade Assistant**, que reúne las de Elasticsearch y Kibana en una vista, si tu versión lo muestra.

## Paso 2.7 · Evaluar la compatibilidad de los componentes y cerrar la ficha

**Qué estás haciendo.** Aplicas las reglas de compatibilidad de Elastic a los componentes que RutaNorte ya tiene funcionando, listados en `componentes-rutanorte.csv`. Consulta la matriz oficial (<https://www.elastic.co/support/matrix>) y estas reglas:

| Regla | Detalle |
|---|---|
| Ingesta (Elastic Agent, Beats, Logstash) | Deben estar en **8.19** o en una versión **9.x no mayor** que la del deployment. Las versiones 8.0 a 8.18 y las 7.x no son compatibles con 9.x |
| Clientes de lenguaje | Lo recomendado es usar la **misma versión mayor** que el clúster. Los clientes 8.x funcionan con 9.x gracias a la compatibilidad REST, pero debes planear su actualización. Los clientes 7.x y el *High Level REST Client* de Java no son compatibles |
| Kibana | Siempre en la misma versión que Elasticsearch. ECH lo garantiza |

Completa las columnas `compatible_con_9x` y `accion_recomendada` del CSV y cópialas a la sección 7 de la ficha. Después escribe la conclusión de la sección 8.

**Resultado esperado de la evaluación**

| ID | Componente | Versión | ¿Compatible? | Acción recomendada |
|---|---|---|---|---|
| C01 | Filebeat (api-gateway) | 7.17.29 | **No** | Actualizar a 8.19 o 9.x, o reemplazar por Elastic Agent |
| C02 | Elastic Agent (reservas-api) | 8.19.4 | Sí | Planear la actualización a 9.x desde Fleet |
| C03 | Elastic Agent (asientos-api) | 9.3.1 | Sí* | *Siempre que no sea mayor que la versión del deployment |
| C04 | Logstash (pagos-api) | 8.12.2 | **No** | Actualizar a 8.19 o 9.x antes de apuntarlo a `rutanorte-logs` |
| C05 | elasticsearch-py (portal-web) | 8.15.1 | Con condiciones | Funciona por compatibilidad REST; actualizar al cliente 9.x |
| C06 | Java High Level REST Client | 7.17.29 | **No** | Migrar al *Elasticsearch Java API Client* 9.x |
| C07 | Kibana | = deployment | Sí | Ninguna (lo administra ECH) |
| C08 | Elastic Agent (auth-api) | 8.17.2 | **No** | Actualizar a 8.19 o 9.x desde Fleet |

**Conclusión esperada:** la plataforma está lista y sana. Antes de conectar `api-gateway`, `pagos-api` y `auth-api` hay que actualizar sus componentes de ingesta (C01, C04 y C08), y el proceso batch de `asientos-api` (C06) requiere migrar de cliente.

---

## Verificación

- [ ] El acceso HTTPS funciona con un certificado válido, sin desactivar TLS.
- [ ] La API key `rn-inventario` existe, vence en 30 días y solo tiene `monitor` y lectura sobre `logs-rutanorte.*`.
- [ ] El secreto de la API key está guardado fuera de la ficha.
- [ ] La ficha registra la versión, el build, la compatibilidad mínima, la licencia y la salud, cada dato con su evidencia.
- [ ] La evaluación de compatibilidad identifica los cuatro componentes que no son compatibles con 9.x.
- [ ] La ficha tiene fecha y conclusión.

## Pantalla de resultado final

![Consulta de licencia y versión en Kibana Dev Tools](img/lab02-resultado-final.png)

*Pantalla de referencia de los Pasos 2.3 y 2.4: `GET /` y `GET _license`. La respuesta muestra la versión, la compatibilidad mínima en red y el tipo y estado de la licencia. Los valores exactos cambian en cada deployment.*

## Preguntas de reflexión

1. ¿Por qué no basta con anotar "Elasticsearch 9" en el inventario? Da un ejemplo en el que el número de parche importe.
2. Si Seguridad pide revocar de inmediato la key `rn-inventario`, ¿qué API usarías? (Pista: `DELETE _security/api_key` con el `id` o el `name`).
3. ¿Qué componente de la tabla de compatibilidad bloquearía hoy la ingesta del gateway? ¿Qué alternativas tiene RutaNorte?

## Lo que dejas listo para el Lab 3

- La **ficha de inventario** con la línea base del deployment.
- La API key **`rn-inventario`**, que podrá leer los data streams `logs-rutanorte.*` en cuanto existan.

En el Lab 3 vas a decidir **dónde y cómo** se guardan los logs de RutaNorte: diseñarás los templates y los mappings que darán forma a los data streams de los seis servicios.

---

## Archivo completo · `cap02/lab02-inventario.txt`

```
# =====================================================================
# Lab 2 · Versión, licencia, acceso y salud: inventario de rutanorte-logs
# Archivo: cap02/lab02-inventario.txt
# Entorno: Kibana › Dev Tools › Console (sesión de rn_admin)
# =====================================================================

### 2.2 Crear la API key de solo lectura rn-inventario
POST _security/api_key
{
  "name": "rn-inventario",
  "expiration": "30d",
  "role_descriptors": {
    "rn_inventario_lectura": {
      "cluster": [ "monitor" ],
      "indices": [
        {
          "names": [ "logs-rutanorte.*" ],
          "privileges": [ "read", "view_index_metadata" ]
        }
      ]
    }
  },
  "metadata": {
    "proyecto": "rutanorte-logs",
    "uso": "inventario y monitoreo de solo lectura",
    "creado_en": "lab-02"
  }
}

### 2.2b Consultar la API key (nunca devuelve el secreto)
GET _security/api_key?name=rn-inventario&active_only=true

### 2.3 Versión, build y compatibilidad mínima
GET /

### 2.3b Todos los nodos deben tener la misma versión
GET _cat/nodes?v&h=name,version,build,jdk&s=name

### 2.3c Versión de Kibana (API de Kibana desde Console con el prefijo kbn:)
GET kbn:/api/status

### 2.4 Licencia activa
GET _license

### 2.4b Funcionalidades disponibles con la licencia
GET _xpack?categories=license,features&filter_path=license,features.security,features.ml,features.searchable_snapshots

### 2.5 Estadísticas del clúster para la ficha de inventario
GET _cluster/stats?human&filter_path=cluster_name,cluster_uuid,status,nodes.count,nodes.versions,nodes.jvm.versions.version,nodes.os.pretty_names,indices.count,indices.shards.total,indices.docs.count,indices.store.size

### 2.5b Plugins instalados
GET _cat/plugins?v&h=name,component,version

### 2.5c Salud resumida
GET _cluster/health?filter_path=status,number_of_nodes,number_of_data_nodes,active_shards,unassigned_shards

GET _health_report?verbose=false&filter_path=status,indicators.*.status

### 2.6 Avisos de deprecación de Elasticsearch
GET _migration/deprecations

### 2.6b Avisos de deprecación de Kibana
GET kbn:/api/deprecations/
```
