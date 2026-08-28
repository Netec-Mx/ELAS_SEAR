# 4 Laboratorio final: Diagnosticar y documentar un incidente en una plataforma de logs

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 60 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En este laboratorio final investigarás un incidente controlado sobre el clúster `es930-lab-cluster` y los datos restaurados en `logs-ops-lab`. Partirás de síntomas observables en Kibana Discover y recopilarás evidencia antes de realizar cambios, diferenciando entre fallos de ingestión, permisos, asignación de shards y capacidad.

La práctica concluye con una mitigación reversible, validación funcional y la redacción de un informe técnico de incidente que incorpore las líneas base, controles de seguridad, políticas ILM y garantías de recuperación de prácticas anteriores.

## Objetivos de aprendizaje

Al finalizar el laboratorio, podrás:

- [ ] Delimitar un incidente de logs mediante Kibana Discover, KQL, Query DSL y ES|QL.
- [ ] Recopilar evidencia no destructiva con las APIs de salud, shards, asignación, nodos, estadísticas, ILM y Bulk.
- [ ] Distinguir síntomas, impacto, causas inmediatas, hipótesis y causa raíz probable.
- [ ] Aplicar una mitigación segura y reversible para una réplica no viable o un problema controlado de permisos de ingestión.
- [ ] Elaborar un informe técnico de incidente con cronología, evidencias, decisiones, escalamiento y acciones preventivas.

## Prerrequisitos

### Conocimientos requeridos

- Metodología de diagnóstico de Elasticsearch: delimitación, impacto, evidencia, hipótesis, mitigación y validación.
- Uso básico de Kibana Discover, KQL, Elasticsearch Query DSL y ES|QL.
- Interpretación de `_cluster/health`, `_cat/shards`, `_cluster/allocation/explain`, `_nodes/stats` e ILM.
- Conceptos de roles, privilegios mínimos, API keys, ingestión Bulk, réplicas y clústeres de un nodo.
- Contenido de las prácticas 6, 7, 8 y 9.

### Acceso y estado requerido

Antes de comenzar, confirma lo siguiente:

- La Práctica 9 está completada.
- `es930-lab` y `kibana930-lab` están disponibles.
- El índice `logs-ops-lab` fue restaurado y contiene datos.
- Las consultas guardadas de la Práctica 8 están disponibles en Kibana.
- El runbook de actualización existe en:

```text
/opt/elastic-labs/reports/upgrade-runbook-717-to-930.md
```

- El archivo de credenciales existe y tiene permisos `0600`:

```text
/opt/elastic-labs/.env
```

> **Importante:** No ejecutes `docker compose down -v`. Esa operación eliminaría los volúmenes persistentes necesarios para la continuidad del curso.

## Entorno del laboratorio

### Componentes usados

| Componente | Valor esperado |
|---|---|
| Elasticsearch objetivo | `es930-lab` / Elasticsearch 9.3.0 |
| Kibana objetivo | `kibana930-lab` / Kibana 9.3.0 |
| Clúster | `es930-lab-cluster` |
| Nodo | `es930-node-1` |
| Tipo de descubrimiento | `single-node` |
| Índice de logs | `logs-ops-lab` |
| Elasticsearch | `http://localhost:9201` |
| Kibana | `http://localhost:5602` |
| Directorio raíz | `/opt/elastic-labs` |
| Directorio de evidencias | `/opt/elastic-labs/work/incident-10-00-01` |
| Directorio de informes | `/opt/elastic-labs/reports` |

### Preparación de la sesión

1. Abre una terminal y prepara un directorio privado para las evidencias.

   ```bash
   cd /opt/elastic-labs
   sudo mkdir -p /opt/elastic-labs/work/incident-10-00-01
   sudo mkdir -p /opt/elastic-labs/reports
   sudo chown -R "$USER":"$USER" /opt/elastic-labs/work/incident-10-00-01 /opt/elastic-labs/reports
   chmod 700 /opt/elastic-labs/work/incident-10-00-01
   umask 077
   ```

2. Carga las credenciales sin mostrarlas en pantalla.

   ```bash
   set -a
   source /opt/elastic-labs/.env
   set +a

   export ES_URL="http://localhost:9201"
   export KIBANA_URL="http://localhost:5602"
   export EVIDENCE_DIR="/opt/elastic-labs/work/incident-10-00-01"
   export REPORT="/opt/elastic-labs/reports/incident-10-00-01.md"
   ```

   Si la Práctica 9 dejó TLS habilitado en el endpoint HTTP, utiliza esta alternativa:

   ```bash
   export ES_URL="https://localhost:9201"
   ```

   En tal caso, añade `-k` a los comandos `curl` únicamente si el certificado es autofirmado dentro del laboratorio.

3. Comprueba conectividad, autenticación y versión.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/?pretty" | tee "${EVIDENCE_DIR}/00-cluster-identidad.json"
   ```

**Resultado esperado**

La respuesta incluye información similar a:

```json
{
  "name" : "es930-node-1",
  "cluster_name" : "es930-lab-cluster",
  "version" : {
    "number" : "9.3.0"
  }
}
```

**Verificación**

Confirma que:

```bash
jq -r '.cluster_name, .name, .version.number' \
  "${EVIDENCE_DIR}/00-cluster-identidad.json"
```

muestra `es930-lab-cluster`, `es930-node-1` y `9.3.0`.

## Procedimiento paso a paso

### Paso 1. Registrar la apertura del incidente y preservar la evidencia inicial

**Objetivo:** Crear una referencia temporal del incidente y comprobar que los artefactos previos requeridos están disponibles antes de modificar el clúster.

**Instrucciones**

1. Registra fecha, zona horaria y contexto local.

   ```bash
   {
     echo "incident_id=INC-10-00-01"
     echo "investigador=$USER"
     echo "inicio_utc=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
     echo "host=$(hostname)"
     echo "es_url=$ES_URL"
     echo "kibana_url=$KIBANA_URL"
   } | tee "${EVIDENCE_DIR}/01-contexto-incidente.txt"
   ```

2. Confirma que el runbook de actualización y recuperación está presente.

   ```bash
   ls -l /opt/elastic-labs/reports/upgrade-runbook-717-to-930.md \
     | tee "${EVIDENCE_DIR}/01-runbook-disponible.txt"
   ```

3. Revisa el estado de los contenedores sin reiniciarlos ni recrearlos.

   ```bash
   docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}' \
     | tee "${EVIDENCE_DIR}/01-contenedores.txt"
   ```

4. Registra los índices existentes y verifica que `logs-ops-lab` está disponible.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_cat/indices?v&s=index" \
     | tee "${EVIDENCE_DIR}/01-cat-indices-inicial.txt"
   ```

**Resultado esperado**

- Los contenedores `es930-lab` y `kibana930-lab` aparecen en ejecución.
- Existe el archivo `upgrade-runbook-717-to-930.md`.
- El índice `logs-ops-lab` aparece en la salida de `_cat/indices`.

**Verificación**

```bash
grep -E 'es930-lab|kibana930-lab' "${EVIDENCE_DIR}/01-contenedores.txt"
grep 'logs-ops-lab' "${EVIDENCE_DIR}/01-cat-indices-inicial.txt"
```

No realices todavía cambios en réplicas, ILM, pipelines, nodos, índices ni contenedores.

---

### Paso 2. Delimitar el impacto funcional desde Kibana Discover

**Objetivo:** Identificar qué logs faltan o presentan errores, desde cuándo ocurre el problema y qué servicio está afectado.

**Instrucciones**

1. Abre Kibana en el navegador:

   ```text
   http://localhost:5602
   ```

2. Inicia sesión con el usuario `elastic` y la contraseña definida en `/opt/elastic-labs/.env`.

3. Accede a **Discover** y selecciona el data view asociado a `logs-ops-lab`.

4. Carga la consulta guardada de la Práctica 8 destinada a investigar errores o actividad de `payments-api`.

5. Establece un rango temporal que incluya los eventos del incidente. Si desconoces la hora exacta, comienza con **Last 24 hours** y ajusta el rango tras localizar el primer error.

6. Ejecuta la siguiente consulta KQL, adaptando el nombre de campo solo si la consulta guardada de la Práctica 8 documenta una estructura diferente:

   ```kql
   service.name : "payments-api" and log.level : "error"
   ```

7. Registra manualmente en un archivo:
   - rango temporal consultado;
   - cantidad aproximada de resultados;
   - primer y último evento relevante;
   - mensajes de error repetidos;
   - identificadores de correlación, si existen;
   - evidencia visual exportada o captura de pantalla.

   ```bash
   cat > "${EVIDENCE_DIR}/02-discover-observaciones.md" <<'EOF'
   # Observaciones de Discover

   - Consulta guardada utilizada:
   - KQL ejecutada: `service.name : "payments-api" and log.level : "error"`
   - Rango temporal:
   - Número aproximado de eventos:
   - Primer evento relevante:
   - Último evento relevante:
   - Mensajes repetidos:
   - Servicios afectados:
   - Impacto observado por el usuario:
   - Archivo de captura/exportación:
   EOF
   ```

8. Como corroboración por API, ejecuta una consulta Query DSL para contar errores por minuto. Ajusta el campo `log.level` si tu esquema usa otro campo equivalente.

   ```bash
   cat > "${EVIDENCE_DIR}/02-payments-errors-query.json" <<'EOF'
   {
     "size": 0,
     "query": {
       "bool": {
         "filter": [
           { "term": { "service.name.keyword": "payments-api" } },
           { "term": { "log.level.keyword": "error" } }
         ]
       }
     },
     "aggs": {
       "errores_por_minuto": {
         "date_histogram": {
           "field": "@timestamp",
           "fixed_interval": "1m"
         }
       }
     }
   }
   EOF

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     -X POST "${ES_URL}/logs-ops-lab/_search?pretty" \
     --data-binary @"${EVIDENCE_DIR}/02-payments-errors-query.json" \
     | tee "${EVIDENCE_DIR}/02-payments-errors-dsl.json"
   ```

9. Ejecuta una consulta ES|QL equivalente para obtener una tendencia resumida.

   ```bash
   cat > "${EVIDENCE_DIR}/02-payments-errors-esql.json" <<'EOF'
   {
     "query": "FROM logs-ops-lab | WHERE service.name == \"payments-api\" AND log.level == \"error\" | STATS errores = COUNT(*) BY bucket = DATE_TRUNC(1 minute, @timestamp) | SORT bucket ASC"
   }
   EOF

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     -X POST "${ES_URL}/_query?format=json" \
     --data-binary @"${EVIDENCE_DIR}/02-payments-errors-esql.json" \
     | tee "${EVIDENCE_DIR}/02-payments-errors-esql.json.out"
   ```

**Resultado esperado**

- Discover muestra eventos de error asociados a `payments-api`.
- La agregación Query DSL revela uno o más intervalos con incremento de errores.
- ES|QL devuelve una serie temporal agrupada por minuto.
- Puede observarse que los errores de aplicación no prueban por sí solos una caída de Elasticsearch: son un síntoma que debe correlacionarse con ingestión y estado del clúster.

**Verificación**

Comprueba la cantidad total de errores devuelta por Query DSL:

```bash
jq '.hits.total' "${EVIDENCE_DIR}/02-payments-errors-dsl.json"
```

Revisa la salida ES|QL:

```bash
jq '.' "${EVIDENCE_DIR}/02-payments-errors-esql.json.out"
```

Conserva la captura de Discover en el directorio de evidencia con un nombre como:

```text
02-discover-payments-api-errors.png
```

---

### Paso 3. Recopilar evidencia técnica del clúster sin modificarlo

**Objetivo:** Determinar si el incidente incluye degradación de disponibilidad, shards no asignados, presión de recursos, crecimiento anómalo o incumplimiento de ILM.

**Instrucciones**

1. Captura la salud del clúster.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_cluster/health?pretty" \
     | tee "${EVIDENCE_DIR}/03-cluster-health-antes.json"
   ```

2. Registra los nodos, sus roles y métricas básicas.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_cat/nodes?v&h=name,roles,master,heap.percent,ram.percent,cpu,load_1m,disk.used_percent" \
     | tee "${EVIDENCE_DIR}/03-cat-nodes-antes.txt"
   ```

3. Captura el tamaño, salud y cantidad de documentos de los índices.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_cat/indices?v&s=store.size:desc" \
     | tee "${EVIDENCE_DIR}/03-cat-indices-antes.txt"
   ```

4. Examina los shards, ordenándolos por estado e índice.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_cat/shards?v&s=state,index" \
     | tee "${EVIDENCE_DIR}/03-cat-shards-antes.txt"
   ```

5. Si hay shards no asignados, solicita una explicación de asignación. No supongas la causa.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X GET "${ES_URL}/_cluster/allocation/explain?pretty" \
     | tee "${EVIDENCE_DIR}/03-allocation-explain-antes.json"
   ```

6. Obtén métricas de JVM, sistema de archivos, proceso e indexación del nodo.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_nodes/stats/jvm,fs,process,indices?pretty" \
     | tee "${EVIDENCE_DIR}/03-nodes-stats-antes.json"
   ```

7. Obtén estadísticas específicas del índice de logs.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/logs-ops-lab/_stats?pretty" \
     | tee "${EVIDENCE_DIR}/03-logs-ops-stats-antes.json"
   ```

8. Revisa el estado ILM y los pipelines configurados para el índice.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/logs-ops-lab/_ilm/explain?pretty" \
     | tee "${EVIDENCE_DIR}/03-ilm-explain-antes.json"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/logs-ops-lab/_settings?filter_path=*.settings.index.number_of_replicas,*.settings.index.default_pipeline,*.settings.index.final_pipeline&pretty" \
     | tee "${EVIDENCE_DIR}/03-index-settings-antes.json"
   ```

**Resultado esperado**

En el escenario previsto puede aparecer una combinación de estas condiciones:

- Estado `yellow`.
- Un shard réplica `UNASSIGNED`.
- Un mensaje de `_cluster/allocation/explain` que indique que una réplica no puede asignarse al mismo nodo que su shard primario.
- Un único nodo de datos en el clúster.
- Un índice `logs-ops-lab` con `number_of_replicas: "1"`.
- Métricas de heap, CPU y disco que deben compararse con las líneas base de la Práctica 6.
- ILM activo o configurado según la Práctica 7.

**Verificación**

Ejecuta los siguientes controles rápidos:

```bash
jq '{status, number_of_nodes, active_primary_shards, active_shards, unassigned_shards}' \
  "${EVIDENCE_DIR}/03-cluster-health-antes.json"

grep 'UNASSIGNED' "${EVIDENCE_DIR}/03-cat-shards-antes.txt" || true

jq '.' "${EVIDENCE_DIR}/03-allocation-explain-antes.json"
```

Documenta una primera separación entre hechos y conclusiones:

| Tipo | Ejemplo que debes adaptar a tu evidencia |
|---|---|
| Síntoma | Discover presenta errores de `payments-api` o ausencia de logs recientes. |
| Impacto | Operadores no pueden confiar plenamente en los paneles o en la trazabilidad de pagos. |
| Hecho técnico | El clúster tiene un shard réplica no asignado. |
| Hipótesis | La réplica no se asigna porque el clúster solo tiene un nodo. |
| Evidencia confirmatoria | `_cluster/allocation/explain` informa una decisión `same_shard` o equivalente. |

---

### Paso 4. Investigar errores de ingestión y permisos mediante respuestas Bulk

**Objetivo:** Confirmar si la ingestión falla por permisos insuficientes o una configuración de ingestión y separar ese fallo de la condición de réplicas.

**Instrucciones**

1. Busca primero artefactos de la simulación o respuestas Bulk que el instructor haya proporcionado.

   ```bash
   find /opt/elastic-labs/work -maxdepth 3 -type f \
     \( -iname '*bulk*' -o -iname '*ingest*' -o -iname '*incident*' \) \
     -printf '%TY-%Tm-%Td %TH:%TM %p\n' 2>/dev/null \
     | sort \
     | tee "${EVIDENCE_DIR}/04-artefactos-disponibles.txt"
   ```

2. Examina los ajustes de pipeline capturados en el paso anterior. Si `default_pipeline` o `final_pipeline` contiene un nombre, inspecciónalo sin modificarlo.

   ```bash
   PIPELINE_NAME=$(jq -r '
     to_entries[]
     | .value.settings.index.default_pipeline // .value.settings.index.final_pipeline // empty
   ' "${EVIDENCE_DIR}/03-index-settings-antes.json" | head -n 1)

   echo "pipeline_detectado=${PIPELINE_NAME:-ninguno}" \
     | tee "${EVIDENCE_DIR}/04-pipeline-detectado.txt"

   if [ -n "${PIPELINE_NAME}" ]; then
     curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
       "${ES_URL}/_ingest/pipeline/${PIPELINE_NAME}?pretty" \
       | tee "${EVIDENCE_DIR}/04-pipeline-inspeccion.json"
   fi
   ```

3. Si el instructor ya proporcionó una respuesta Bulk fallida, cópiala al directorio de evidencias y analízala con `jq`. Si no existe, ejecuta la siguiente reproducción controlada autorizada. Esta prueba crea una API key sin privilegio de escritura y no debe indexar ningún documento.

   ```bash
   RUN_ID="lab-$(date -u +%Y%m%dT%H%M%SZ)"

   cat > "${EVIDENCE_DIR}/04-api-key-denied-request.json" <<EOF
   {
     "name": "incident-denied-${RUN_ID}",
     "role_descriptors": {
       "lectura_sin_ingestion": {
         "cluster": [],
         "indices": [
           {
             "names": ["logs-ops-lab"],
             "privileges": ["read"]
           }
         ]
       }
     }
   }
   EOF

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     -X POST "${ES_URL}/_security/api_key" \
     --data-binary @"${EVIDENCE_DIR}/04-api-key-denied-request.json" \
     | tee "${EVIDENCE_DIR}/04-api-key-denied-response.json"

   export DENIED_KEY_ID
   export DENIED_KEY
   DENIED_KEY_ID=$(jq -r '.id' "${EVIDENCE_DIR}/04-api-key-denied-response.json")
   DENIED_KEY=$(jq -r '.encoded' "${EVIDENCE_DIR}/04-api-key-denied-response.json")

   cat > "${EVIDENCE_DIR}/04-bulk-denied.ndjson" <<EOF
   {"create":{"_id":"${RUN_ID}-denied"}}
   {"@timestamp":"$(date -u +%Y-%m-%dT%H:%M:%SZ)","service":{"name":"payments-api"},"log":{"level":"error"},"message":"Prueba controlada de API key sin privilegio create_doc","event":{"kind":"event","dataset":"lab.incident"}}
   EOF

   curl -sS \
     -H "Authorization: ApiKey ${DENIED_KEY}" \
     -H 'Content-Type: application/x-ndjson' \
     -X POST "${ES_URL}/logs-ops-lab/_bulk?refresh=true" \
     --data-binary @"${EVIDENCE_DIR}/04-bulk-denied.ndjson" \
     | tee "${EVIDENCE_DIR}/04-bulk-denied-response.json"
   ```

4. Analiza la respuesta Bulk.

   ```bash
   jq '{
     errors,
     items: [
       .items[]
       | {
           operation: (keys[0]),
           status: (.[] .status),
           error_type: (.[] .error.type // null),
           reason: (.[] .error.reason // null)
         }
     ]
   }' "${EVIDENCE_DIR}/04-bulk-denied-response.json"
   ```

5. Clasifica el resultado:
   - Si aparece `security_exception`, `authorization_exception` o estado `403`, la causa inmediata es insuficiencia de privilegios.
   - Si aparece un error de procesador, `illegal_argument_exception` o referencia a pipeline, la causa inmediata es una configuración de ingestión.
   - Si la operación es exitosa, registra que la reproducción no confirma el fallo de permisos y usa la evidencia suministrada por el instructor para continuar el diagnóstico.

**Resultado esperado**

La reproducción controlada con una API key de solo lectura devuelve una respuesta Bulk con:

```json
{
  "errors": true
}
```

y al menos un ítem con estado `403` y error de seguridad. La operación no crea un documento porque la API key no dispone de privilegios de escritura.

**Verificación**

```bash
jq -r '.errors' "${EVIDENCE_DIR}/04-bulk-denied-response.json"

jq '.items[] | .[] | {status, error}' \
  "${EVIDENCE_DIR}/04-bulk-denied-response.json"
```

La evidencia debe permitir formular una afirmación precisa:

> La respuesta Bulk confirma que una API key sin privilegio de indexación no puede crear documentos en `logs-ops-lab`. Esto explica un fallo de ingestión cuando el cliente utiliza dicha key, pero no explica por sí solo un shard réplica no asignado.

---

### Paso 5. Formular hipótesis, seleccionar la mitigación y ejecutar un cambio reversible

**Objetivo:** Aplicar la acción de menor riesgo que reduzca el impacto confirmado, sin ocultar la causa ni eliminar evidencia.

**Instrucciones**

1. Completa la siguiente matriz en un archivo de trabajo antes de modificar configuración.

   ```bash
   cat > "${EVIDENCE_DIR}/05-matriz-diagnostico.md" <<'EOF'
   # Matriz de diagnóstico

   | Elemento | Hallazgo | Evidencia | Confianza |
   |---|---|---|---|
   | Síntoma en Discover |  | 02-discover-observaciones.md |  |
   | Impacto operativo |  | 02-discover-observaciones.md |  |
   | Salud del clúster |  | 03-cluster-health-antes.json |  |
   | Shard no asignado |  | 03-cat-shards-antes.txt |  |
   | Motivo de asignación |  | 03-allocation-explain-antes.json |  |
   | Estado de recursos |  | 03-nodes-stats-antes.json |  |
   | ILM |  | 03-ilm-explain-antes.json |  |
   | Ingestión Bulk |  | 04-bulk-denied-response.json |  |
   | Causa inmediata probable |  |  |  |
   | Causa raíz probable |  |  |  |
   EOF
   ```

2. Confirma que el clúster tiene exactamente un nodo y que el shard no asignado es una réplica no viable. No reduzcas réplicas si la explicación identifica otra causa, como falta de disco, filtros de asignación o un primario sin asignar.

   ```bash
   jq '{number_of_nodes, status, unassigned_shards}' \
     "${EVIDENCE_DIR}/03-cluster-health-antes.json"

   jq '{
     index,
     shard,
     primary,
     current_state,
     unassigned_info,
     can_allocate,
     allocate_explanation,
     node_allocation_decisions
   }' "${EVIDENCE_DIR}/03-allocation-explain-antes.json"
   ```

3. Si la evidencia confirma una réplica no asignable en el clúster de un solo nodo, aplica la mitigación reversible estableciendo cero réplicas en el índice de laboratorio.

   ```bash
   cat > "${EVIDENCE_DIR}/05-replicas-cero-request.json" <<'EOF'
   {
     "index": {
       "number_of_replicas": 0
     }
   }
   EOF

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     -X PUT "${ES_URL}/logs-ops-lab/_settings?pretty" \
     --data-binary @"${EVIDENCE_DIR}/05-replicas-cero-request.json" \
     | tee "${EVIDENCE_DIR}/05-replicas-cero-response.json"
   ```

4. Para el fallo de permisos, no reutilices la API key denegada. Crea una API key temporal con privilegio mínimo `create_doc`, válida únicamente para la prueba de recuperación de ingestión.

   ```bash
   cat > "${EVIDENCE_DIR}/05-api-key-write-request.json" <<EOF
   {
     "name": "incident-create-doc-${RUN_ID}",
     "role_descriptors": {
       "ingestion_minima": {
         "cluster": [],
         "indices": [
           {
             "names": ["logs-ops-lab"],
             "privileges": ["create_doc"]
           }
         ]
       }
     }
   }
   EOF

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     -X POST "${ES_URL}/_security/api_key" \
     --data-binary @"${EVIDENCE_DIR}/05-api-key-write-request.json" \
     | tee "${EVIDENCE_DIR}/05-api-key-write-response.json"

   export ALLOWED_KEY_ID
   export ALLOWED_KEY
   ALLOWED_KEY_ID=$(jq -r '.id' "${EVIDENCE_DIR}/05-api-key-write-response.json")
   ALLOWED_KEY=$(jq -r '.encoded' "${EVIDENCE_DIR}/05-api-key-write-response.json")

   export OK_DOC_ID="${RUN_ID}-allowed"

   cat > "${EVIDENCE_DIR}/05-bulk-allowed.ndjson" <<EOF
   {"create":{"_id":"${OK_DOC_ID}"}}
   {"@timestamp":"$(date -u +%Y-%m-%dT%H:%M:%SZ)","service":{"name":"payments-api"},"log":{"level":"info"},"message":"Prueba controlada de recuperación con privilegio create_doc","event":{"kind":"event","dataset":"lab.incident"},"labels":{"incident_id":"INC-10-00-01"}}
   EOF

   curl -sS \
     -H "Authorization: ApiKey ${ALLOWED_KEY}" \
     -H 'Content-Type: application/x-ndjson' \
     -X POST "${ES_URL}/logs-ops-lab/_bulk?refresh=true" \
     --data-binary @"${EVIDENCE_DIR}/05-bulk-allowed.ndjson" \
     | tee "${EVIDENCE_DIR}/05-bulk-allowed-response.json"
   ```

5. Registra la decisión técnica tomada. La reducción a cero réplicas en un clúster de un nodo elimina el estado `yellow`, pero también elimina redundancia. Es una mitigación aceptable únicamente para el entorno de laboratorio o como riesgo temporal formalmente aceptado.

   ```bash
   cat > "${EVIDENCE_DIR}/05-decision-mitigacion.md" <<'EOF'
   # Decisión de mitigación

   - Cambio aplicado:
   - Evidencia que lo justifica:
   - Riesgo introducido o aceptado:
   - Reversibilidad:
   - Criterio para revertir:
   - Responsable de aprobación:
   - Hora UTC de aplicación:
   EOF
   ```

**Resultado esperado**

- La actualización de settings es reconocida con `"acknowledged": true`.
- El Bulk ejecutado con la API key de privilegio mínimo devuelve `"errors": false`.
- La mitigación de réplicas solo se aplica si el diagnóstico confirmó que el problema era una réplica no viable en un clúster de nodo único.

**Verificación**

```bash
jq '.' "${EVIDENCE_DIR}/05-replicas-cero-response.json"

jq '{
  errors,
  items: [.items[] | .[] | {status, result, error}]
}' "${EVIDENCE_DIR}/05-bulk-allowed-response.json"
```

---

### Paso 6. Validar recuperación técnica y funcional

**Objetivo:** Demostrar con métricas, APIs y búsqueda funcional que la mitigación produjo el resultado esperado y que no generó un efecto secundario relevante.

**Instrucciones**

1. Espera hasta 60 segundos a que la salud del clúster alcance el estado esperado.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_cluster/health?wait_for_status=green&timeout=60s&pretty" \
     | tee "${EVIDENCE_DIR}/06-cluster-health-despues.json"
   ```

2. Captura nuevamente el estado de shards, índices, nodos y estadísticas del índice.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_cat/shards?v&s=state,index" \
     | tee "${EVIDENCE_DIR}/06-cat-shards-despues.txt"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_cat/indices?v&s=store.size:desc" \
     | tee "${EVIDENCE_DIR}/06-cat-indices-despues.txt"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/_nodes/stats/jvm,fs,process,indices?pretty" \
     | tee "${EVIDENCE_DIR}/06-nodes-stats-despues.json"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/logs-ops-lab/_stats?pretty" \
     | tee "${EVIDENCE_DIR}/06-logs-ops-stats-despues.json"
   ```

3. Comprueba que el documento de prueba autorizado fue indexado.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "${ES_URL}/logs-ops-lab/_doc/${OK_DOC_ID}?pretty" \
     | tee "${EVIDENCE_DIR}/06-documento-prueba.json"
   ```

4. En Kibana Discover, busca el identificador de la prueba controlada:

   ```kql
   labels.incident_id : "INC-10-00-01"
   ```

5. Repite la consulta guardada de la Práctica 8 y la consulta KQL de `payments-api`. Comprueba si:
   - los nuevos documentos son visibles;
   - no hay retraso inesperado de ingestión;
   - la búsqueda responde;
   - los errores históricos siguen disponibles para investigación;
   - el panel o visualización relacionado recuperó comportamiento normal.

6. Compara las métricas de antes y después con la línea base de la Práctica 6. Registra como mínimo:
   - `heap.percent`;
   - `cpu`;
   - `disk.used_percent`;
   - `store.size`;
   - total de documentos;
   - `unassigned_shards`;
   - estado de salud;
   - resultado Bulk.

**Resultado esperado**

- El clúster pasa a `green` si la única causa del estado `yellow` era una réplica no asignable.
- No quedan líneas `UNASSIGNED` para `logs-ops-lab`.
- El Bulk autorizado devuelve `errors: false`.
- El documento de prueba existe y es visible en Discover.
- Los recursos del nodo no muestran una desviación importante frente a la línea base de la Práctica 6.

**Verificación**

```bash
jq '{status, number_of_nodes, active_shards, unassigned_shards}' \
  "${EVIDENCE_DIR}/06-cluster-health-despues.json"

grep 'UNASSIGNED' "${EVIDENCE_DIR}/06-cat-shards-despues.txt" || true

jq '._source | {timestamp: ."@timestamp", service, log, labels, message}' \
  "${EVIDENCE_DIR}/06-documento-prueba.json"
```

Si el clúster no llega a `green`, no ocultes el resultado: conserva la evidencia, revisa `_cluster/allocation/explain` otra vez y considera el caso para escalamiento.

---

### Paso 7. Redactar el informe técnico de incidente

**Objetivo:** Crear un informe técnico trazable que permita a otro equipo comprender el incidente, validar las decisiones y prevenir su recurrencia.

**Instrucciones**

1. Crea el informe usando la siguiente plantilla.

   ```bash
   cat > "${REPORT}" <<'EOF'
   # Informe técnico de incidente INC-10-00-01

   ## 1. Identificación

   | Campo | Valor |
   |---|---|
   | Identificador | INC-10-00-01 |
   | Fecha y hora de detección (UTC) | |
   | Fecha y hora de mitigación (UTC) | |
   | Investigador | |
   | Entorno | Laboratorio Elasticsearch 9.3.0 |
   | Clúster | es930-lab-cluster |
   | Nodo | es930-node-1 |
   | Índice afectado | logs-ops-lab |
   | Severidad propuesta | |

   ## 2. Resumen ejecutivo

   Describa en 3 a 5 líneas qué ocurrió, qué usuarios o procesos fueron afectados,
   cuál fue el impacto y cuál fue el estado final.

   ## 3. Impacto

   - Impacto en ingestión:
   - Impacto en búsqueda y Discover:
   - Impacto en disponibilidad:
   - Impacto en integridad de datos:
   - Impacto en seguridad:
   - Riesgo de continuidad o recuperación:

   ## 4. Línea temporal

   | Hora UTC | Evento | Evidencia |
   |---|---|---|
   | | Detección inicial | |
   | | Consulta de Discover | |
   | | Captura de salud del clúster | |
   | | Confirmación de Bulk o permisos | |
   | | Mitigación aplicada | |
   | | Validación final | |

   ## 5. Evidencia técnica

   ### Evidencia funcional

   - Consulta guardada de Kibana utilizada:
   - KQL:
   - Rango temporal:
   - Resultado observado:
   - Archivo de captura:

   ### Evidencia de Elasticsearch

   - Salud antes:
   - Salud después:
   - Shards no asignados:
   - Resultado de allocation explain:
   - Métricas de heap, CPU y disco:
   - Estadísticas del índice:
   - Estado ILM:
   - Respuesta Bulk denegada:
   - Respuesta Bulk autorizada:

   ## 6. Diagnóstico

   | Categoría | Descripción |
   |---|---|
   | Síntoma | |
   | Causa inmediata | |
   | Causa raíz probable | |
   | Evidencia que confirma la hipótesis | |
   | Evidencia que descarta hipótesis alternativas | |
   | Riesgo residual | |

   ## 7. Mitigación aplicada

   - Cambio:
   - Justificación:
   - Comando o API utilizada:
   - Reversibilidad:
   - Riesgo aceptado:
   - Resultado:

   ## 8. Validación de recuperación

   - Estado del clúster:
   - Shards:
   - Resultado de Bulk:
   - Verificación en Discover:
   - Comparación con línea base de Práctica 6:
   - Relación con ILM y seguridad de Práctica 7:
   - Garantía de recuperación consultada en el runbook de Práctica 9:

   ## 9. Escalamiento

   - ¿Requiere escalamiento?: Sí / No
   - Equipo destinatario:
   - Criterio de escalamiento:
   - Información que debe adjuntarse:
   - Decisión final:

   ## 10. Acciones preventivas

   | Acción | Prioridad | Responsable | Fecha objetivo | Criterio de cierre |
   |---|---|---|---|---|
   | Revisar privilegios mínimos de API keys de ingestión | Alta | | | |
   | Definir política explícita de réplicas para clústeres de un nodo | Alta | | | |
   | Alertar sobre errores Bulk y shards no asignados | Media | | | |
   | Validar ILM y capacidad frente a crecimiento de logs | Media | | | |
   | Ensayar restauración según runbook | Media | | | |

   ## 11. Lecciones aprendidas

   - Qué evidencia fue más útil:
   - Qué decisión podría automatizarse:
   - Qué documentación debe mejorarse:
   - Qué se validará en el siguiente ejercicio:
   EOF
   ```

2. Completa el informe usando exclusivamente evidencias verificables almacenadas en:

   ```text
   /opt/elastic-labs/work/incident-10-00-01
   ```

3. Añade al informe referencias explícitas a:
   - líneas base de rendimiento de la Práctica 6;
   - roles, API keys, TLS e ILM de la Práctica 7;
   - consultas y observaciones de Discover de la Práctica 8;
   - runbook de actualización y recuperación de la Práctica 9.

4. Determina si el incidente requiere escalamiento. Debe escalarse, por ejemplo, si se observa alguno de estos criterios:
   - shards primarios no asignados o estado `red`;
   - uso de disco cercano a los umbrales de Elasticsearch;
   - heap o CPU sostenidamente por encima de la línea base;
   - pérdida no explicada de eventos;
   - permisos de producción asignados incorrectamente;
   - fallo de ILM que compromete retención o cumplimiento;
   - necesidad de modificar infraestructura, añadir nodos o restaurar desde snapshot.

**Resultado esperado**

Existe un informe completo en:

```text
/opt/elastic-labs/reports/incident-10-00-01.md
```

El documento distingue claramente entre observaciones, evidencias, hipótesis, causa inmediata, causa raíz probable, mitigación, riesgo residual y acciones preventivas.

**Verificación**

```bash
test -s "${REPORT}" && echo "Informe creado correctamente"
grep -E '^## (3|4|5|6|7|8|9|10)\.' "${REPORT}"
```

## Validación y pruebas

Completa esta lista de validación final antes de entregar el laboratorio.

| Validación | Comando o evidencia | Resultado esperado |
|---|---|---|
| Identidad del clúster | `00-cluster-identidad.json` | Clúster `es930-lab-cluster`, nodo `es930-node-1`, versión 9.3.0 |
| Datos restaurados | `_cat/indices` | Existe `logs-ops-lab` |
| Impacto funcional | Discover y KQL | Se identifican eventos o ausencia de logs relacionados con el incidente |
| Tendencia de errores | Query DSL y ES|QL | Se puede cuantificar la tendencia de `payments-api` |
| Salud inicial | `03-cluster-health-antes.json` | Se documenta estado y shards no asignados |
| Asignación | `03-allocation-explain-antes.json` | Se identifica la causa real de no asignación |
| Recursos | `_nodes/stats` | Heap, CPU y disco comparados contra Práctica 6 |
| ILM | `03-ilm-explain-antes.json` | Estado ILM documentado |
| Permisos | Respuesta Bulk denegada | Error `403` o error de seguridad confirmado, si aplica |
| Recuperación de ingestión | Respuesta Bulk autorizada | `errors: false` |
| Mitigación de réplicas | Salud posterior y CAT shards | Estado `green` y ausencia de réplica no asignada, si la condición era de nodo único |
| Documento de prueba | GET por ID y Discover | Documento visible y consultable |
| Informe final | `incident-10-00-01.md` | Incluye impacto, cronología, evidencia, diagnóstico, mitigación, escalamiento y prevención |

Ejecuta este resumen técnico final:

```bash
echo "=== Salud posterior ==="
jq '{status, number_of_nodes, active_primary_shards, active_shards, unassigned_shards}' \
  "${EVIDENCE_DIR}/06-cluster-health-despues.json"

echo
echo "=== Resultado Bulk autorizado ==="
jq '{errors, items: [.items[] | .[] | {status, result, error}]}' \
  "${EVIDENCE_DIR}/05-bulk-allowed-response.json"

echo
echo "=== Réplicas configuradas ==="
curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
  "${ES_URL}/logs-ops-lab/_settings?filter_path=*.settings.index.number_of_replicas&pretty"
```

## Solución de problemas

### 1. El clúster permanece en `yellow` después de establecer `number_of_replicas: 0`

**Síntomas**

- `_cluster/health` continúa mostrando `yellow`.
- `_cat/shards` todavía presenta shards `UNASSIGNED`.
- La explicación de asignación no corresponde a una réplica del índice `logs-ops-lab`.

**Causa**

La condición no está relacionada únicamente con la réplica del índice intervenido. Puede existir otro índice con réplicas configuradas, un shard primario sin asignar, un filtro de asignación, una restricción de tier o una condición de disco.

**Corrección**

No apliques cambios masivos de réplicas. Identifica el índice y shard exactos:

```bash
curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
  "${ES_URL}/_cat/shards?v&s=state,index"

curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
  "${ES_URL}/_cluster/allocation/explain?pretty"
```

Actualiza el informe con el nuevo hallazgo y escala el caso si la causa requiere capacidad, cambio de topología, corrección de filtros o intervención fuera del alcance del laboratorio.

### 2. La prueba Bulk autorizada devuelve `403`, `401` o `errors: true`

**Síntomas**

- La respuesta Bulk con la API key de prueba no devuelve `errors: false`.
- Aparece `security_exception`, `unable to authenticate`, `authorization_exception` o un error de pipeline.

**Causa**

La API key puede estar mal copiada, expirada, creada con un rol incorrecto, enviada sin la cabecera `Authorization: ApiKey`, o el índice puede tener un pipeline obligatorio que falla antes de indexar el documento.

**Corrección**

Primero inspecciona la respuesta exacta sin exponer la API key:

```bash
jq '.items[] | .[] | {status, error}' \
  "${EVIDENCE_DIR}/05-bulk-allowed-response.json"
```

Si el error es de permisos, crea nuevamente una API key con `create_doc` sobre `logs-ops-lab` y repite solo la prueba controlada. Si el error menciona un pipeline, revisa el pipeline detectado en `04-pipeline-inspeccion.json`, documenta el hallazgo y no lo modifiques sin una corrección aprobada y reversible.

## Limpieza

La limpieza debe eliminar únicamente artefactos temporales de la prueba controlada. Debes conservar las evidencias y el informe final.

1. Elimina el documento creado por la prueba de ingestión autorizada.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X DELETE "${ES_URL}/logs-ops-lab/_doc/${OK_DOC_ID}?refresh=true&pretty" \
     | tee "${EVIDENCE_DIR}/cleanup-delete-documento-prueba.json"
   ```

2. Revoca las API keys temporales creadas durante el laboratorio.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X DELETE "${ES_URL}/_security/api_key?id=${DENIED_KEY_ID}&pretty" \
     | tee "${EVIDENCE_DIR}/cleanup-delete-denied-key.json"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X DELETE "${ES_URL}/_security/api_key?id=${ALLOWED_KEY_ID}&pretty" \
     | tee "${EVIDENCE_DIR}/cleanup-delete-allowed-key.json"
   ```

3. Elimina variables sensibles de la sesión actual.

   ```bash
   unset DENIED_KEY DENIED_KEY_ID ALLOWED_KEY ALLOWED_KEY_ID ELASTIC_PASSWORD
   ```

4. Mantén el ajuste de cero réplicas si fue la mitigación aprobada para este clúster de laboratorio. No lo reviertas a `1` mientras el clúster siga teniendo un único nodo, porque el estado volvería previsiblemente a `yellow`.

5. No ejecutes ninguno de estos comandos:

   ```bash
   docker compose down -v
   docker volume rm es717-data es930-data kibana717-data kibana930-data
   ```

## Resumen

En este laboratorio aplicaste una metodología de diagnóstico estructurada: partiste de síntomas observables en Discover, mediste el impacto, capturaste evidencia de Elasticsearch antes de modificar el entorno, formulaste hipótesis y validaste una mitigación reversible.

Las conclusiones principales deben reflejar que:

- Un error de aplicación o de ingestión no demuestra por sí mismo una caída del clúster.
- Un estado `yellow` requiere identificar qué shard está sin asignar y solicitar una explicación de asignación.
- Una réplica en un clúster de un nodo representa un riesgo de disponibilidad, no necesariamente una pérdida de servicio inmediata.
- Las respuestas de Bulk son evidencia directa para distinguir problemas de permisos, autenticación y pipelines.
- Una mitigación operativa solo está completa cuando se valida con salud del clúster, shards, métricas, consultas funcionales y documentación.
- El informe técnico convierte la investigación en conocimiento reutilizable, facilita el escalamiento y define acciones preventivas verificables.

### Recursos opcionales

- [Diagnóstico de clústeres Elasticsearch en estado rojo o amarillo](https://www.elastic.co/docs/troubleshoot/elasticsearch/red-yellow-cluster-status)
- [API Cluster Health](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-health)
- [API Cluster Allocation Explain](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-allocation-explain)
- [API Bulk](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-bulk)
- [API Explain ILM](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ilm-explain-lifecycle)
- [Seguridad con API keys de Elasticsearch](https://www.elastic.co/docs/deploy-manage/api-keys)
