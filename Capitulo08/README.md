# Investigar un incidente con Discover y consultas guardadas

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 50 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En este laboratorio se investigará un incidente controlado de `payments-api` utilizando Kibana Discover 7.17.29. Se inyectarán eventos ECS en el data stream `logs-ops-lab`, se delimitará una ventana temporal de incidente y se identificarán errores HTTP 5xx, timeouts y fallos de autenticación.

Al finalizar, se habrán creado o validado un data view, filtros estructurados, consultas KQL reutilizables y evidencias JSON para el informe técnico. Las consultas guardadas y la línea temporal obtenida se conservarán como evidencia previa a la actualización de la Práctica 9.

## Objetivos de aprendizaje

- [ ] Crear o validar el data view `logs-ops-lab*` usando `@timestamp` como campo temporal.
- [ ] Inyectar y verificar eventos controlados del incidente en el data stream `logs-ops-lab`.
- [ ] Investigar errores de `payments-api` mediante Discover, filtros estructurados, columnas y KQL.
- [ ] Identificar el periodo, hosts, endpoints, códigos HTTP y mensajes asociados al incidente.
- [ ] Guardar al menos tres consultas operativas normalizadas para triage, impacto y seguimiento.

## Prerrequisitos

### Conocimientos requeridos

- Uso básico de terminal Linux, Docker Compose, `curl` y `jq`.
- Navegación básica por Kibana Discover.
- Sintaxis básica de Kibana Query Language (KQL).
- Comprensión de los campos ECS: `@timestamp`, `service.name`, `log.level`, `host.name`, `event.dataset`, `http.response.status_code`, `trace.id` y `message`.

### Acceso requerido

- Práctica 7 completada.
- Data stream `logs-ops-lab` disponible.
- Política ILM `logs-ops-ilm-v1` disponible.
- Usuario `analyst_lab` y sus privilegios de lectura sobre `logs-ops-lab*`.
- Acceso a Kibana 7.17.29 en `https://localhost:5601`.
- Acceso administrativo local al host para ejecutar comandos en `/opt/elastic-labs`.
- Credenciales de laboratorio de Elasticsearch disponibles en `/opt/elastic-labs/.env`.

> **Nota de seguridad:** use el usuario `analyst_lab` para la investigación en Kibana. Use el usuario `elastic` únicamente desde la terminal para la carga controlada de datos y validaciones técnicas. No reutilice las credenciales del laboratorio fuera de este entorno.

## Entorno de laboratorio

### Componentes relevantes

| Componente | Valor esperado |
|---|---|
| Directorio raíz | `/opt/elastic-labs` |
| Directorio de trabajo | `/opt/elastic-labs/work` |
| Archivo de variables | `/opt/elastic-labs/.env` |
| Elasticsearch 7.17.29 | `https://localhost:9200` |
| Kibana 7.17.29 | `https://localhost:5601` |
| Contenedor Elasticsearch 7 | `es717-lab` |
| Contenedor Kibana 7 | `kibana717-lab` |
| Clúster Elasticsearch 7 | `es717-lab-cluster` |
| Data stream objetivo | `logs-ops-lab` |
| Data view esperado | `logs-ops-lab*` |
| Usuario de investigación | `analyst_lab` |

### Preparación inicial

1. Abra una terminal y acceda al directorio raíz obligatorio:

   ```bash
   cd /opt/elastic-labs
   ```

2. Verifique que el archivo de credenciales exista y tenga permisos restrictivos:

   ```bash
   ls -l /opt/elastic-labs/.env
   ```

   La salida debe indicar permisos equivalentes a:

   ```text
   -rw------- 1 root root ... /opt/elastic-labs/.env
   ```

3. Verifique el estado de los contenedores requeridos:

   ```bash
   docker compose ps
   ```

4. Si `es717-lab` o `kibana717-lab` no están en ejecución, inícielos sin eliminar volúmenes:

   ```bash
   docker compose up -d es717-lab kibana717-lab
   ```

5. Cargue las credenciales de laboratorio y valide la salud del clúster 7.17.29:

   ```bash
   set -a
   . /opt/elastic-labs/.env
   set +a

   curl -sk -u "elastic:${ELASTIC_PASSWORD}" \
     "https://localhost:9200/_cluster/health?pretty"
   ```

> **Importante:** no ejecute `docker compose down -v`. Ese comando eliminaría volúmenes persistentes y rompería la continuidad requerida entre prácticas.

---

## Procedimiento paso a paso

### Paso 1. Verificar el data stream y preparar el directorio de evidencias

**Objetivo:** confirmar que `logs-ops-lab` está disponible antes de inyectar los eventos del incidente.

**Instrucciones:**

1. Cree los directorios de trabajo para scripts y evidencias:

   ```bash
   mkdir -p /opt/elastic-labs/work/evidence
   chmod 700 /opt/elastic-labs/work/evidence
   ```

2. Verifique que exista el data stream `logs-ops-lab`:

   ```bash
   curl -sk -u "elastic:${ELASTIC_PASSWORD}" \
     "https://localhost:9200/_data_stream/logs-ops-lab?pretty"
   ```

3. Verifique la política ILM aplicada al data stream o a su índice de respaldo:

   ```bash
   curl -sk -u "elastic:${ELASTIC_PASSWORD}" \
     "https://localhost:9200/logs-ops-lab/_ilm/explain?pretty"
   ```

4. Guarde la salida de la validación del data stream como evidencia inicial:

   ```bash
   curl -sk -u "elastic:${ELASTIC_PASSWORD}" \
     "https://localhost:9200/_data_stream/logs-ops-lab?pretty" \
     > /opt/elastic-labs/work/evidence/08-00-01-data-stream-inicial.json
   ```

**Resultado esperado:**

- La API `_data_stream` devuelve un objeto con el nombre `logs-ops-lab`.
- La API ILM devuelve información sobre el índice de respaldo del data stream.
- El archivo `08-00-01-data-stream-inicial.json` queda creado.

**Verificación:**

```bash
jq '.data_streams[0].name' \
  /opt/elastic-labs/work/evidence/08-00-01-data-stream-inicial.json
```

La salida esperada es:

```text
"logs-ops-lab"
```

---

### Paso 2. Inyectar eventos controlados del incidente

**Objetivo:** generar una secuencia reproducible de eventos de aplicación con errores HTTP 5xx, timeouts y errores de autenticación.

**Instrucciones:**

1. Cree el script de inyección de eventos:

   ```bash
   cat > /opt/elastic-labs/work/inject_incident_08-00-01.sh <<'EOF'
   #!/usr/bin/env bash
   set -euo pipefail

   LAB_ROOT="/opt/elastic-labs"
   WORK_DIR="${LAB_ROOT}/work"
   BULK_FILE="${WORK_DIR}/incident-08-00-01.ndjson"
   WINDOW_FILE="${WORK_DIR}/incident-08-00-01-window.txt"

   set -a
   . "${LAB_ROOT}/.env"
   set +a

   # La ventana inicia hace 35 minutos. Los eventos críticos ocurren
   # aproximadamente entre los minutos +10 y +22 de la secuencia.
   BASE_EPOCH="$(date -u -d '35 minutes ago' +%s)"

   : > "${BULK_FILE}"

   event() {
     local offset="$1"
     local level="$2"
     local status="$3"
     local host="$4"
     local path="$5"
     local trace="$6"
     local error_type="$7"
     local error_message="$8"
     local message="$9"
     local user_id="${10}"

     local ts
     ts="$(date -u -d "@$((BASE_EPOCH + offset * 60))" '+%Y-%m-%dT%H:%M:%SZ')"

     jq -cn \
       --arg ts "${ts}" \
       --arg level "${level}" \
       --arg host "${host}" \
       --arg path "${path}" \
       --arg trace "${trace}" \
       --arg error_type "${error_type}" \
       --arg error_message "${error_message}" \
       --arg message "${message}" \
       --arg user_id "${user_id}" \
       --argjson status "${status}" \
       '
       {create:{_index:"logs-ops-lab"}},
       {
         "@timestamp": $ts,
         "message": $message,
         "log": {"level": $level},
         "service": {"name": "payments-api"},
         "event": {"dataset": "payments-api.log"},
         "environment": "production",
         "http": {"response": {"status_code": $status}},
         "host": {"name": $host},
         "url": {"path": $path},
         "trace": {"id": $trace},
         "error": {"type": $error_type, "message": $error_message},
         "user": {"id": $user_id}
       }
       ' >> "${BULK_FILE}"
   }

   # Actividad previa normal.
   event 0  "INFO"  200 "payments-app-01" "/v1/payments" "trace-baseline-001" "" "" "Payment request completed successfully" "customer-1001"
   event 3  "INFO"  201 "payments-app-02" "/v1/payments" "trace-baseline-002" "" "" "Payment created successfully" "customer-1002"
   event 6  "WARN"  401 "payments-app-01" "/v1/token"    "trace-auth-001" "AuthenticationError" "Invalid API token" "Authentication failed: invalid API token" "unknown"

   # Inicio del incidente: errores intermitentes por timeout.
   event 10 "ERROR" 504 "payments-app-01" "/v1/payments" "trace-pay-7f3a" "ConnectionTimeout" "Upstream gateway timeout after 30000ms" "Payment authorization timeout while contacting bank gateway" "customer-1003"
   event 11 "ERROR" 502 "payments-app-02" "/v1/payments" "trace-pay-7f3a" "ConnectionTimeout" "Connection timeout to bank gateway" "Payment request failed: upstream timeout" "customer-1003"
   event 12 "ERROR" 500 "payments-app-01" "/v1/payments" "trace-pay-8b91" "PaymentProviderError" "Provider response unavailable" "Payment processing failed after provider timeout" "customer-1004"
   event 13 "ERROR" 503 "payments-app-02" "/v1/refunds"  "trace-pay-55cc" "ConnectionTimeout" "Timeout while obtaining provider connection" "Refund request rejected: dependency timeout" "customer-1005"
   event 14 "ERROR" 504 "payments-app-01" "/v1/payments" "trace-pay-a201" "ConnectionTimeout" "Upstream gateway timeout after 30000ms" "Payment authorization timeout while contacting bank gateway" "customer-1006"
   event 15 "ERROR" 401 "payments-app-01" "/v1/token"    "trace-auth-002" "AuthenticationError" "Expired service token" "Authentication failed: service token expired" "service-checkout"
   event 16 "ERROR" 502 "payments-app-02" "/v1/payments" "trace-pay-c311" "ConnectionTimeout" "Connection timeout to bank gateway" "Payment request failed: upstream timeout" "customer-1007"
   event 17 "ERROR" 500 "payments-app-01" "/v1/payments" "trace-pay-dd20" "PaymentProviderError" "Provider response unavailable" "Payment provider returned an unavailable response" "customer-1008"
   event 18 "ERROR" 504 "payments-app-02" "/v1/payments" "trace-pay-ef31" "ConnectionTimeout" "Upstream gateway timeout after 30000ms" "Payment authorization timeout while contacting bank gateway" "customer-1009"
   event 19 "WARN"  429 "payments-app-02" "/v1/payments" "trace-pay-rate-01" "RateLimitError" "Retry threshold reached" "Payment request retried after upstream failures" "customer-1010"
   event 20 "ERROR" 503 "payments-app-01" "/v1/refunds"  "trace-pay-f010" "ConnectionTimeout" "Timeout while obtaining provider connection" "Refund request rejected: dependency timeout" "customer-1011"
   event 22 "INFO"  200 "payments-app-02" "/v1/health"   "trace-health-001" "" "" "Payments API health check recovered" "system"

   INCIDENT_START="$(date -u -d "@$((BASE_EPOCH + 10 * 60))" '+%Y-%m-%dT%H:%M:%SZ')"
   INCIDENT_END="$(date -u -d "@$((BASE_EPOCH + 22 * 60))" '+%Y-%m-%dT%H:%M:%SZ')"

   {
     echo "LAB_ID=08-00-01"
     echo "INCIDENT_START_UTC=${INCIDENT_START}"
     echo "INCIDENT_END_UTC=${INCIDENT_END}"
     echo "INCIDENT_SERVICE=payments-api"
     echo "INCIDENT_DATASET=payments-api.log"
     echo "INCIDENT_TRACE_EXAMPLE=trace-pay-7f3a"
   } > "${WINDOW_FILE}"

   curl --fail --silent --show-error -sk \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/x-ndjson' \
     -X POST "https://localhost:9200/_bulk" \
     --data-binary "@${BULK_FILE}" \
     | tee "${WORK_DIR}/evidence/08-00-01-bulk-response.json"

   curl --fail --silent --show-error -sk \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -X POST "https://localhost:9200/logs-ops-lab/_refresh" \
     > /dev/null

   echo
   echo "Ventana del incidente:"
   cat "${WINDOW_FILE}"
   EOF
   ```

2. Asigne permisos de ejecución al script:

   ```bash
   chmod 700 /opt/elastic-labs/work/inject_incident_08-00-01.sh
   ```

3. Ejecute el script:

   ```bash
   /opt/elastic-labs/work/inject_incident_08-00-01.sh
   ```

4. Consulte y conserve la ventana temporal exacta generada:

   ```bash
   cat /opt/elastic-labs/work/incident-08-00-01-window.txt
   ```

**Resultado esperado:**

- La respuesta Bulk contiene `"errors":false`.
- Se generan 15 documentos de prueba.
- El archivo `incident-08-00-01-window.txt` muestra los límites UTC del incidente.
- Los eventos críticos pertenecen a `payments-api`, al dataset `payments-api.log` y al entorno `production`.

**Verificación:**

```bash
jq '.errors' /opt/elastic-labs/work/evidence/08-00-01-bulk-response.json
```

La salida esperada es:

```text
false
```

Verifique el número de acciones procesadas:

```bash
jq '.items | length' /opt/elastic-labs/work/evidence/08-00-01-bulk-response.json
```

La salida esperada es:

```text
15
```

---

### Paso 3. Validar los eventos indexados mediante Elasticsearch Query DSL

**Objetivo:** confirmar que los datos están disponibles antes de comenzar la investigación visual en Discover.

**Instrucciones:**

1. Cree una consulta Query DSL que recupere los errores HTTP 5xx de `payments-api`:

   ```bash
   cat > /opt/elastic-labs/work/payments-5xx-query.json <<'EOF'
   {
     "size": 50,
     "sort": [
       {
         "@timestamp": {
           "order": "asc"
         }
       }
     ],
     "_source": [
       "@timestamp",
       "service.name",
       "event.dataset",
       "log.level",
       "http.response.status_code",
       "host.name",
       "url.path",
       "trace.id",
       "error.type",
       "error.message",
       "message"
     ],
     "query": {
       "bool": {
         "filter": [
           {
             "term": {
               "service.name.keyword": "payments-api"
             }
           },
           {
             "range": {
               "http.response.status_code": {
                 "gte": 500
               }
             }
           }
         ]
       }
     }
   }
   EOF
   ```

2. Ejecute la consulta y guarde el resultado:

   ```bash
   curl -sk -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     -X POST "https://localhost:9200/logs-ops-lab/_search?pretty" \
     -d @/opt/elastic-labs/work/payments-5xx-query.json \
     > /opt/elastic-labs/work/evidence/08-00-01-payments-5xx.json
   ```

3. Muestre un resumen de los eventos devueltos:

   ```bash
   jq -r '
     .hits.hits[]._source |
     [
       ."@timestamp",
       .service.name,
       .log.level,
       .http.response.status_code,
       .host.name,
       .url.path,
       .trace.id,
       .error.type
     ] | @tsv
   ' /opt/elastic-labs/work/evidence/08-00-01-payments-5xx.json
   ```

4. Revise la primera y la última marca temporal de los errores 5xx:

   ```bash
   jq -r '
     .hits.hits[]._source."@timestamp"
   ' /opt/elastic-labs/work/evidence/08-00-01-payments-5xx.json \
     | sort \
     | sed -n '1p;$p'
   ```

**Resultado esperado:**

- Se devuelven errores HTTP `500`, `502`, `503` y `504`.
- Los hosts implicados son `payments-app-01` y `payments-app-02`.
- Los endpoints más afectados son `/v1/payments` y `/v1/refunds`.
- Los errores de tipo `ConnectionTimeout` aparecen repetidamente.

**Verificación:**

```bash
jq '.hits.total' /opt/elastic-labs/work/evidence/08-00-01-payments-5xx.json
```

La propiedad `value` debe ser mayor que cero. En esta carga controlada, se esperan varios eventos 5xx.

---

### Paso 4. Crear o validar el data view en Kibana

**Objetivo:** disponer de una fuente de datos temporal para investigar `logs-ops-lab` en Discover.

**Instrucciones:**

1. Abra Firefox o un navegador compatible.

2. Acceda a:

   ```text
   https://localhost:5601
   ```

3. Si el navegador advierte que el certificado TLS es de laboratorio o autofirmado, acepte la excepción de seguridad únicamente para este entorno local.

4. Inicie sesión con el usuario:

   ```text
   analyst_lab
   ```

   Use la contraseña configurada durante la Práctica 7.

5. En el menú principal de Kibana, acceda a **Stack Management**.

6. Abra **Index Patterns**. En algunas vistas o configuraciones puede aparecer como **Data Views**.

7. Busque un data view existente cuyo patrón sea:

   ```text
   logs-ops-lab*
   ```

8. Si existe, ábralo y confirme que el campo temporal configurado es:

   ```text
   @timestamp
   ```

9. Si no existe, seleccione **Create index pattern** o **Create data view** y configure:

   | Campo | Valor |
   |---|---|
   | Patrón de índices | `logs-ops-lab*` |
   | Nombre sugerido | `logs-ops-lab*` |
   | Campo temporal | `@timestamp` |

10. Guarde la configuración.

11. Revise que estén disponibles, como mínimo, los siguientes campos:

   ```text
   @timestamp
   service.name
   event.dataset
   environment
   log.level
   http.response.status_code
   host.name
   url.path
   trace.id
   error.type
   error.message
   message
   ```

**Resultado esperado:**

- El data view `logs-ops-lab*` existe.
- El campo temporal seleccionado es `@timestamp`.
- Los campos ECS inyectados aparecen en la lista de campos disponibles.

**Verificación:**

Acceda a **Discover**, seleccione el data view `logs-ops-lab*` y confirme que se muestran documentos recientes cuando se elige un intervalo amplio, por ejemplo **Last 1 hour**.

---

### Paso 5. Delimitar el incidente y configurar columnas operativas

**Objetivo:** construir una vista de Discover que permita analizar cronológicamente el incidente.

**Instrucciones:**

1. En Kibana, abra **Discover**.

2. Seleccione el data view:

   ```text
   logs-ops-lab*
   ```

3. Abra el archivo de ventana temporal en una terminal para consultar los valores exactos:

   ```bash
   cat /opt/elastic-labs/work/incident-08-00-01-window.txt
   ```

4. En Discover, abra el selector de tiempo y seleccione **Absolute**.

5. Configure el intervalo usando los valores de:

   ```text
   INCIDENT_START_UTC
   INCIDENT_END_UTC
   ```

6. Amplíe el intervalo un minuto antes y un minuto después para obtener contexto operativo. Por ejemplo:

   - Inicio: un minuto antes de `INCIDENT_START_UTC`.
   - Fin: un minuto después de `INCIDENT_END_UTC`.

7. Añada las siguientes columnas a la tabla de documentos. Use la lista de campos y la opción **Add** o el icono `+`:

   ```text
   @timestamp
   service.name
   log.level
   http.response.status_code
   host.name
   url.path
   trace.id
   error.type
   message
   ```

8. Ordene inicialmente por `@timestamp` de forma ascendente para identificar el primer evento relevante.

9. Revise el histograma temporal y observe la concentración de eventos de error dentro de la ventana del incidente.

**Resultado esperado:**

- La tabla muestra una secuencia temporal de actividad previa, errores y recuperación.
- Se observa un aumento de eventos de error entre los minutos definidos como incidente.
- Las columnas permiten relacionar tiempo, host, endpoint, código HTTP, trazabilidad y mensaje.

**Verificación:**

Identifique manualmente en la tabla:

- Un evento previo exitoso con código `200` o `201`.
- El primer evento `ERROR` con código HTTP 5xx.
- Un evento de autenticación con código `401`.
- El evento de recuperación con ruta `/v1/health` y código `200`.

---

### Paso 6. Aplicar filtros estructurados y ejecutar consultas KQL

**Objetivo:** aislar los eventos relevantes del incidente mediante campos ECS y consultas KQL.

**Instrucciones:**

1. Mantenga el mismo intervalo temporal absoluto configurado en el paso anterior.

2. Agregue los siguientes filtros estructurados desde **Add filter**:

   | Campo | Operador | Valor |
   |---|---|---|
   | `environment` | is | `production` |
   | `event.dataset` | is | `payments-api.log` |
   | `service.name` | is | `payments-api` |

3. En la barra KQL, ejecute la consulta de triage:

   ```kql
   log.level: ERROR and (http.response.status_code >= 500 or error.message: "*timeout*")
   ```

4. Observe el número de resultados y el histograma.

5. Añada temporalmente un filtro estructurado adicional:

   | Campo | Operador | Valor |
   |---|---|---|
   | `host.name` | is | `payments-app-01` |

6. Compare el número de eventos y los códigos de respuesta de `payments-app-01`.

7. Deshabilite el filtro de `host.name` utilizando el control de activación del filtro, sin eliminarlo.

8. Cambie el valor del filtro `host.name` a:

   ```text
   payments-app-02
   ```

9. Compare los eventos del segundo host.

10. Elimine o deshabilite el filtro de host antes de continuar para recuperar la vista global del incidente.

11. Ejecute la consulta KQL específica para impacto HTTP 5xx:

   ```kql
   http.response.status_code >= 500
   ```

12. Ejecute la consulta KQL específica para timeouts:

   ```kql
   error.type: "ConnectionTimeout" or message: "*timeout*"
   ```

13. Ejecute la consulta KQL de autenticación:

   ```kql
   http.response.status_code: 401 or error.type: "AuthenticationError"
   ```

**Resultado esperado:**

- Los errores 5xx se concentran en `payments-api`.
- Los dos hosts presentan errores, por lo que el incidente no se limita a una sola instancia.
- Los mensajes de timeout se asocian principalmente con los endpoints `/v1/payments` y `/v1/refunds`.
- Los eventos de autenticación `401` son distinguibles del grupo principal de errores 5xx.

**Verificación:**

Complete esta tabla en las notas del informe técnico:

| Pregunta de investigación | Evidencia esperada |
|---|---|
| ¿Qué servicio presenta el incidente? | `payments-api` |
| ¿Qué entorno está afectado? | `production` |
| ¿Qué hosts están implicados? | `payments-app-01` y `payments-app-02` |
| ¿Qué códigos HTTP predominan en los errores? | `500`, `502`, `503`, `504` |
| ¿Cuál es el patrón de error predominante? | `ConnectionTimeout` / mensajes de timeout |
| ¿Qué endpoints están afectados? | `/v1/payments` y `/v1/refunds` |
| ¿Existe evidencia de recuperación? | Evento `200` en `/v1/health` |

---

### Paso 7. Correlacionar la secuencia con `trace.id`

**Objetivo:** utilizar un identificador de trazabilidad para correlacionar eventos relacionados y demostrar una investigación de detalle.

**Instrucciones:**

1. En la tabla de Discover, localice un evento con el siguiente valor de trazabilidad:

   ```text
   trace-pay-7f3a
   ```

2. Abra el documento y revise sus campos JSON o la vista detallada del documento.

3. Confirme los siguientes valores:

   | Campo | Valor esperado |
   |---|---|
   | `service.name` | `payments-api` |
   | `http.response.status_code` | `504` o `502` |
   | `error.type` | `ConnectionTimeout` |
   | `url.path` | `/v1/payments` |
   | `trace.id` | `trace-pay-7f3a` |

4. Cierre el documento y reemplace la consulta KQL actual por:

   ```kql
   trace.id: "trace-pay-7f3a"
   ```

5. Ordene por `@timestamp` ascendente.

6. Compare los dos eventos obtenidos. Deben reflejar el progreso de una misma operación entre hosts distintos.

7. Tome una captura de pantalla donde se aprecien la consulta KQL, el intervalo temporal y las columnas relevantes.

8. Guarde la captura con un nombre normalizado, por ejemplo:

   ```text
   /opt/elastic-labs/work/evidence/08-00-01-trace-pay-7f3a.png
   ```

**Resultado esperado:**

- Se localizan varios eventos asociados al mismo `trace.id`.
- Los eventos muestran un timeout de dependencia o gateway en hosts diferentes.
- La correlación confirma que no se trata de eventos aislados sin relación.

**Verificación:**

La investigación debe permitir sostener la siguiente hipótesis técnica inicial:

> El incidente afectó a `payments-api` en producción y presenta un patrón consistente con indisponibilidad o latencia excesiva de una dependencia externa, evidenciada por respuestas 5xx y errores `ConnectionTimeout` en ambos hosts de la aplicación.

> Esta es una hipótesis basada en logs; no constituye una confirmación definitiva de la causa raíz sin métricas, trazas distribuidas completas o registros de la dependencia.

---

### Paso 8. Guardar consultas operativas reutilizables

**Objetivo:** crear consultas normalizadas que puedan reutilizarse durante triage, análisis de impacto y seguimiento posterior.

**Instrucciones:**

1. Mantenga seleccionados el data view `logs-ops-lab*` y el intervalo absoluto del incidente.

2. Asegúrese de conservar estos filtros estructurados habilitados:

   ```text
   environment is production
   event.dataset is payments-api.log
   service.name is payments-api
   ```

3. Configure la siguiente consulta KQL de triage:

   ```kql
   log.level: ERROR and (http.response.status_code >= 500 or error.message: "*timeout*")
   ```

4. En el menú de consultas de Discover, seleccione **Save query**.

5. Guarde la consulta con el nombre:

   ```text
   OPS-INC-08-00-01-PAYMENTS-TRIAGE
   ```

6. Si Kibana muestra la opción para incluir filtros y rango temporal, active ambas opciones para conservar el contexto de investigación.

7. Configure la consulta de impacto:

   ```kql
   http.response.status_code >= 500
   ```

8. Guárdela con el nombre:

   ```text
   OPS-INC-08-00-01-PAYMENTS-IMPACT-5XX
   ```

9. Configure la consulta de seguimiento de timeouts:

   ```kql
   error.type: "ConnectionTimeout" or message: "*timeout*"
   ```

10. Guárdela con el nombre:

    ```text
    OPS-INC-08-00-01-PAYMENTS-SEGUIMIENTO-TIMEOUT
    ```

11. Como consulta adicional de autenticación, configure:

    ```kql
    http.response.status_code: 401 or error.type: "AuthenticationError"
    ```

12. Guarde la consulta adicional con el nombre:

    ```text
    OPS-INC-08-00-01-PAYMENTS-AUTH-401
    ```

13. Vuelva a cargar cada consulta guardada desde el menú de consultas y confirme que restaura la KQL y los filtros esperados.

14. Para preservar columnas, data view y contexto de Discover, guarde también la sesión de Discover, si la interfaz presenta la opción **Save**:

    ```text
    OPS-INC-08-00-01-PAYMENTS-EVIDENCIA-DISCOVER
    ```

**Resultado esperado:**

- Existen al menos tres consultas guardadas con nombres normalizados.
- Las consultas conservan el contexto de `payments-api` en producción.
- La consulta de impacto puede ser utilizada posteriormente como base de un dashboard o visualización.
- La sesión guardada de Discover conserva las columnas seleccionadas para la evidencia funcional.

**Verificación:**

Abra el menú de consultas guardadas y confirme la presencia de, como mínimo:

```text
OPS-INC-08-00-01-PAYMENTS-TRIAGE
OPS-INC-08-00-01-PAYMENTS-IMPACT-5XX
OPS-INC-08-00-01-PAYMENTS-SEGUIMIENTO-TIMEOUT
```

---

### Paso 9. Exportar evidencia JSON y documentar la línea temporal

**Objetivo:** conservar evidencia técnica independiente de la interfaz gráfica para el informe del incidente y las prácticas posteriores.

**Instrucciones:**

1. Consulte nuevamente los límites exactos del incidente:

   ```bash
   cat /opt/elastic-labs/work/incident-08-00-01-window.txt
   ```

2. Cree una consulta Query DSL acotada a la ventana del incidente. Sustituya los valores `gte` y `lte` por los valores obtenidos en el archivo anterior, ampliados un minuto si se desea contexto:

   ```bash
   cat > /opt/elastic-labs/work/incident-timeline-query.json <<'EOF'
   {
     "size": 100,
     "sort": [
       {
         "@timestamp": {
           "order": "asc"
         }
       }
     ],
     "_source": [
       "@timestamp",
       "service.name",
       "event.dataset",
       "environment",
       "log.level",
       "http.response.status_code",
       "host.name",
       "url.path",
       "trace.id",
       "error.type",
       "error.message",
       "message"
     ],
     "query": {
       "bool": {
         "filter": [
           {
             "term": {
               "service.name.keyword": "payments-api"
             }
           },
           {
             "term": {
               "environment.keyword": "production"
             }
           }
         ]
       }
     }
   }
   EOF
   ```

3. Ejecute la consulta y guarde los resultados:

   ```bash
   curl -sk -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     -X POST "https://localhost:9200/logs-ops-lab/_search?pretty" \
     -d @/opt/elastic-labs/work/incident-timeline-query.json \
     > /opt/elastic-labs/work/evidence/08-00-01-linea-temporal-payments.json
   ```

4. Genere una versión tabular legible de la línea temporal:

   ```bash
   jq -r '
     .hits.hits[]._source |
     [
       ."@timestamp",
       .log.level,
       .http.response.status_code,
       .host.name,
       .url.path,
       .trace.id,
       .error.type,
       .message
     ] | @tsv
   ' /opt/elastic-labs/work/evidence/08-00-01-linea-temporal-payments.json \
     > /opt/elastic-labs/work/evidence/08-00-01-linea-temporal-payments.tsv
   ```

5. Revise el archivo tabular:

   ```bash
   column -ts $'\t' \
     /opt/elastic-labs/work/evidence/08-00-01-linea-temporal-payments.tsv
   ```

6. Cree un archivo de notas para el informe técnico:

   ```bash
   cat > /opt/elastic-labs/work/evidence/08-00-01-hallazgos.md <<'EOF'
   # Hallazgos iniciales del incidente

   - Servicio afectado: payments-api
   - Entorno: production
   - Hosts observados: payments-app-01, payments-app-02
   - Endpoints afectados: /v1/payments, /v1/refunds
   - Patrón predominante: errores HTTP 5xx asociados a ConnectionTimeout.
   - Eventos adicionales: errores de autenticación HTTP 401.
   - Hipótesis inicial: una dependencia de pagos o gateway externo presentó latencia excesiva o indisponibilidad.
   - Evidencia de recuperación: evento exitoso HTTP 200 en /v1/health.
   - Limitación: los logs no demuestran por sí solos la causa raíz de la dependencia externa.
   EOF
   ```

**Resultado esperado:**

- Existen evidencias JSON y TSV en `/opt/elastic-labs/work/evidence`.
- La línea temporal permite identificar actividad normal, inicio de errores, concentración de 5xx, eventos de autenticación y recuperación.
- El informe contiene una hipótesis explícita y una limitación de la evidencia.

**Verificación:**

```bash
find /opt/elastic-labs/work/evidence -maxdepth 1 -type f -printf '%f\n' | sort
```

Como mínimo, deben aparecer archivos similares a:

```text
08-00-01-bulk-response.json
08-00-01-data-stream-inicial.json
08-00-01-hallazgos.md
08-00-01-linea-temporal-payments.json
08-00-01-linea-temporal-payments.tsv
08-00-01-payments-5xx.json
```

## Validación y pruebas

Complete las siguientes comprobaciones antes de dar por finalizada la práctica.

| Validación | Método | Resultado esperado |
|---|---|---|
| Data stream disponible | API `_data_stream/logs-ops-lab` | El data stream existe. |
| Eventos inyectados | Respuesta Bulk | `"errors": false` y 15 acciones procesadas. |
| Datos visibles en Discover | Data view `logs-ops-lab*` | Se observan documentos en el rango temporal absoluto. |
| Triage de errores | Consulta KQL de triage | Se muestran eventos `ERROR` con 5xx o timeout. |
| Impacto HTTP | Consulta KQL de impacto | Se muestran códigos 500, 502, 503 y 504. |
| Correlación | Consulta por `trace.id` | Se recuperan eventos asociados a `trace-pay-7f3a`. |
| Análisis por host | Filtro `host.name` | Se observan eventos en ambos hosts. |
| Consultas reutilizables | Menú de consultas guardadas | Existen al menos tres consultas normalizadas. |
| Evidencias técnicas | Archivos en `work/evidence` | Existen JSON, TSV y hallazgos documentados. |

Ejecute esta comprobación final de datos 5xx:

```bash
jq -r '
  .hits.hits[]._source.http.response.status_code
' /opt/elastic-labs/work/evidence/08-00-01-payments-5xx.json \
  | sort -n \
  | uniq -c
```

La salida debe mostrar una distribución de códigos 5xx, por ejemplo:

```text
      2 500
      2 502
      2 503
      3 504
```

Los conteos concretos pueden variar únicamente si se realizaron cargas adicionales de la práctica.

## Solución de problemas

### Problema 1: Discover no muestra eventos aunque la carga Bulk indica éxito

**Síntomas:**

- La respuesta Bulk contiene `"errors": false`.
- El data view `logs-ops-lab*` existe.
- Discover muestra el mensaje de que no hay resultados.

**Causa probable:**

El selector de tiempo no incluye la ventana temporal generada por el script. Los eventos se insertan con marcas temporales UTC situadas aproximadamente entre 35 y 13 minutos antes de la ejecución, por lo que un rango como “Last 15 minutes” puede excluir parte o todos los eventos.

**Corrección:**

1. Consulte la ventana real:

   ```bash
   cat /opt/elastic-labs/work/incident-08-00-01-window.txt
   ```

2. En Discover, seleccione un rango temporal absoluto que incluya `INCIDENT_START_UTC` e `INCIDENT_END_UTC`.

3. Si aún no aparecen documentos, use temporalmente **Last 1 hour**.

4. Confirme que el data view usa `@timestamp` como campo temporal.

5. Fuerce una actualización del data view desde **Stack Management > Index Patterns/Data Views** si los campos fueron creados después del data view.

### Problema 2: El usuario `analyst_lab` puede consultar datos, pero no puede crear data views o guardar consultas

**Síntomas:**

- El usuario puede abrir Discover y ver documentos.
- Kibana muestra un error de permisos al crear el data view o al usar **Save query**.
- Algunas opciones de guardado no están disponibles.

**Causa probable:**

El rol asignado durante la Práctica 7 no incluye privilegios suficientes sobre objetos guardados de Kibana, el espacio actual de Kibana, o la gestión de data views.

**Corrección:**

1. No cambie a `elastic` para realizar la investigación cotidiana, ya que el objetivo es validar los controles de mínimo privilegio.

2. Documente el mensaje exacto del error y el usuario afectado.

3. Solicite al instructor o al administrador de la Práctica 7 que revise el rol de `analyst_lab`. Debe disponer, como mínimo, de:
   - Privilegios de lectura sobre `logs-ops-lab*`.
   - Acceso a Discover.
   - Privilegios sobre objetos guardados necesarios para consultas guardadas.
   - Privilegios para crear o usar el data view definido para el laboratorio.

4. Tras el ajuste, cierre sesión, vuelva a iniciar sesión como `analyst_lab` y repita el guardado de las consultas.

## Limpieza

No elimine el data stream, las consultas guardadas, el data view, las evidencias ni los volúmenes Docker. Estos elementos se utilizarán como evidencia funcional antes de la actualización de la Práctica 9 y durante el diagnóstico final de la Práctica 10.

Realice únicamente las siguientes acciones de limpieza:

1. Proteja los archivos de evidencia:

   ```bash
   chmod 700 /opt/elastic-labs/work/evidence
   chmod 600 /opt/elastic-labs/work/evidence/*
   ```

2. Verifique que los archivos de trabajo permanezcan disponibles:

   ```bash
   ls -lh /opt/elastic-labs/work/evidence
   ```

3. Cierre la sesión de Kibana del usuario `analyst_lab` cuando termine la práctica.

4. Mantenga los contenedores en ejecución si la siguiente práctica se realizará en el mismo equipo.

> **No ejecute:** `docker compose down -v`  
> **No elimine:** `es717-data`, `kibana717-data`, `logs-ops-lab` ni los objetos guardados de Kibana.

## Resumen

En esta práctica se investigó un incidente controlado de `payments-api` sobre el data stream `logs-ops-lab`. Se utilizó un data view temporal basado en `@timestamp`, filtros estructurados por entorno, dataset, servicio y host, y consultas KQL para separar triage, impacto HTTP 5xx, timeouts y autenticación.

La evidencia indica un patrón de errores `ConnectionTimeout` distribuido entre `payments-app-01` y `payments-app-02`, con impacto principal sobre `/v1/payments` y `/v1/refunds`. Se guardaron consultas operativas reutilizables y se exportaron resultados JSON y TSV para respaldar el informe técnico y validar la continuidad funcional antes de la actualización de Elasticsearch.

### Recursos opcionales

- [Documentación oficial de Kibana Discover](https://www.elastic.co/docs/explore-analyze/discover)
- [Documentación oficial de Kibana Query Language (KQL)](https://www.elastic.co/docs/explore-analyze/query-filter/languages/kql)
- [Referencia de Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html)
- [Documentación de data views de Kibana](https://www.elastic.co/docs/explore-analyze/find-and-organize/data-views)
