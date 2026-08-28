# Configurar roles, privilegios y una política de ciclo de vida para logs.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 75 minutos |
| Complejidad | Alta |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica se protege el acceso al clúster `es717-lab` mediante TLS, autenticación nativa, roles con privilegios mínimos y una API key limitada para ingestión. Después se implementa una política ILM para datos de logs, se crea un data stream administrado por dicha política y se reinyectan eventos con campos ECS mínimos.

Al finalizar, el data stream `logs-ops-lab` quedará disponible como fuente segura para consultas y visualización en Discover durante la práctica posterior.

## Objetivos de aprendizaje

- [ ] Verificar TLS HTTP, TLS de transporte y autenticación en Elasticsearch 7.17.29.
- [ ] Crear los roles `log_ingestor_lab`, `log_analyst_lab` y `log_operator_lab` aplicando mínimo privilegio.
- [ ] Crear los usuarios nativos `ingest_lab` y `analyst_lab`, y validar sus permisos efectivos.
- [ ] Generar una API key restringida para escribir documentos en `logs-ops-lab`.
- [ ] Configurar ILM, templates componibles y un data stream con rollover y retención.

## Prerrequisitos

### Conocimientos requeridos

- Comprensión básica de índices, aliases, shards, documentos y búsquedas en Elasticsearch.
- Conocimiento introductorio de JSON, `curl`, `jq` y autenticación HTTP Basic.
- Comprensión de los conceptos de roles, privilegios de clúster, privilegios de índice y API keys.
- Práctica 6 completada, con documentos disponibles en `logs-app-lab-v1`.

### Acceso y archivos requeridos

- Acceso con permisos administrativos al host Docker.
- Directorio raíz obligatorio: `/opt/elastic-labs`.
- Archivo de composición disponible en `/opt/elastic-labs/compose.yaml`.
- Certificado CA disponible en `/opt/elastic-labs/certs/ca/ca.crt`.
- Archivo `/opt/elastic-labs/.env` con permisos `0600`.
- Clúster `es717-lab` disponible en `https://localhost:9200`.
- Usuario administrativo de laboratorio: `elastic`.
- Contraseña administrativa de laboratorio: declarada en `/opt/elastic-labs/.env`.

> **Importante:** No ejecute `docker compose down -v`. Este comando eliminaría volúmenes persistentes y el estado requerido por prácticas posteriores.

## Entorno de laboratorio

| Componente | Valor esperado |
|---|---|
| Elasticsearch | 7.17.29 |
| Contenedor Elasticsearch | `es717-lab` |
| Clúster | `es717-lab-cluster` |
| Nodo | `es717-node-1` |
| Endpoint HTTPS | `https://localhost:9200` |
| Kibana | `https://localhost:5601` o la configuración definida por la composición |
| Red Docker | `elastic-lab-net` |
| Volumen Elasticsearch | `es717-data` |
| Certificado CA | `/opt/elastic-labs/certs/ca/ca.crt` |
| Directorio de trabajo | `/opt/elastic-labs/work` |

### Preparación inicial

1. Abra una terminal y sitúese en el directorio del laboratorio.

   ```bash
   cd /opt/elastic-labs
   ```

2. Verifique que el archivo de credenciales tenga permisos restrictivos.

   ```bash
   sudo chmod 0600 /opt/elastic-labs/.env
   stat -c '%a %n' /opt/elastic-labs/.env
   ```

   **Salida esperada**

   ```text
   600 /opt/elastic-labs/.env
   ```

3. Cree el directorio de trabajo si no existe.

   ```bash
   sudo mkdir -p /opt/elastic-labs/work
   sudo chown "$USER":"$USER" /opt/elastic-labs/work
   chmod 700 /opt/elastic-labs/work
   ```

4. Compruebe el estado de los contenedores del laboratorio.

   ```bash
   docker compose ps
   ```

   Si `es717-lab` no está en ejecución, inicie los servicios definidos sin eliminar volúmenes:

   ```bash
   docker compose up -d
   ```

5. Cargue las variables de entorno del laboratorio en la sesión actual.

   > El archivo `.env` debe contener al menos `ELASTIC_PASSWORD=ElasticLab_2026!`. Las contraseñas adicionales de usuarios de prueba se añadirán de forma controlada.

   ```bash
   set -a
   source /opt/elastic-labs/.env
   set +a

   export ES_URL="https://localhost:9200"
   export ES_CA="/opt/elastic-labs/certs/ca/ca.crt"
   export ES_USER="elastic"
   ```

6. Espere a que Elasticsearch responda correctamente.

   ```bash
   until curl --silent --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_cluster/health?wait_for_status=yellow&timeout=5s" >/dev/null; do
     echo "Esperando a Elasticsearch..."
     sleep 5
   done

   echo "Elasticsearch está disponible."
   ```

   **Salida esperada**

   ```text
   Elasticsearch está disponible.
   ```

---

## Procedimiento paso a paso

### Paso 1. Verificar TLS y autenticación del clúster

**Objetivo:** Confirmar que la API HTTP usa HTTPS, que el certificado de la CA es válido para el laboratorio y que el usuario `elastic` puede autenticarse.

#### Instrucciones

1. Consulte la identidad autenticada mediante la API de seguridad.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_security/_authenticate?pretty" \
     | tee /opt/elastic-labs/work/01-authenticate-elastic.json
   ```

2. Consulte el estado del clúster.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_cluster/health?pretty" \
     | tee /opt/elastic-labs/work/01-cluster-health.json
   ```

3. Consulte los certificados conocidos por Elasticsearch.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_ssl/certificates?pretty" \
     | tee /opt/elastic-labs/work/01-ssl-certificates.json
   ```

4. Consulte la configuración de seguridad relevante para HTTP y transporte.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_nodes/settings?filter_path=nodes.*.settings.xpack.security*&pretty" \
     | tee /opt/elastic-labs/work/01-security-settings.json
   ```

#### Salida esperada

La respuesta de autenticación debe mostrar el usuario `elastic` y roles administrativos, similares a:

```json
{
  "username" : "elastic",
  "roles" : [
    "superuser"
  ],
  "enabled" : true,
  "authentication_realm" : {
    "name" : "reserved",
    "type" : "reserved"
  }
}
```

La salud del clúster debe ser `yellow` o `green`. En un clúster de un solo nodo, `yellow` puede ser aceptable si existen réplicas sin asignar.

#### Verificación

Ejecute:

```bash
jq -r '.username' /opt/elastic-labs/work/01-authenticate-elastic.json
jq -r '.status' /opt/elastic-labs/work/01-cluster-health.json
```

Debe obtener:

```text
elastic
yellow
```

o:

```text
elastic
green
```

---

### Paso 2. Preparar credenciales de usuarios nativos de laboratorio

**Objetivo:** Declarar contraseñas exclusivas de usuarios de prueba en el archivo protegido `.env`.

#### Instrucciones

1. Añada las contraseñas de laboratorio si aún no existen.

   ```bash
   grep -q '^LAB_INGEST_PASSWORD=' /opt/elastic-labs/.env || cat >> /opt/elastic-labs/.env <<'EOF'
   LAB_INGEST_PASSWORD="IngestLab_2026!"
   EOF

   grep -q '^LAB_ANALYST_PASSWORD=' /opt/elastic-labs/.env || cat >> /opt/elastic-labs/.env <<'EOF'
   LAB_ANALYST_PASSWORD="AnalystLab_2026!"
   EOF

   sudo chmod 0600 /opt/elastic-labs/.env
   ```

2. Recargue las variables en la sesión actual.

   ```bash
   set -a
   source /opt/elastic-labs/.env
   set +a
   ```

3. Confirme que las variables están definidas sin imprimir sus valores.

   ```bash
   test -n "$LAB_INGEST_PASSWORD" && echo "LAB_INGEST_PASSWORD definida"
   test -n "$LAB_ANALYST_PASSWORD" && echo "LAB_ANALYST_PASSWORD definida"
   ```

#### Salida esperada

```text
LAB_INGEST_PASSWORD definida
LAB_ANALYST_PASSWORD definida
```

#### Verificación

Compruebe nuevamente los permisos:

```bash
stat -c '%a %n' /opt/elastic-labs/.env
```

La salida debe indicar permisos `600`.

---

### Paso 3. Crear roles con privilegios mínimos

**Objetivo:** Definir roles separados para ingestión, análisis y monitorización operativa.

#### Instrucciones

1. Cree el rol `log_ingestor_lab`.

   Este rol solamente podrá crear documentos en data streams o índices cuyo nombre comience por `logs-ops-lab`.

   ```bash
   cat > /opt/elastic-labs/work/role-log-ingestor-lab.json <<'EOF'
   {
     "cluster": [],
     "indices": [
       {
         "names": ["logs-ops-lab*"],
         "privileges": ["create_doc"],
         "allow_restricted_indices": false
       }
     ],
     "applications": [],
     "run_as": [],
     "metadata": {
       "descripcion": "Ingestión mínima de documentos en logs-ops-lab",
       "entorno": "laboratorio"
     }
   }
   EOF

   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_security/role/log_ingestor_lab" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/role-log-ingestor-lab.json \
     | tee /opt/elastic-labs/work/03-role-log-ingestor-lab-result.json
   ```

2. Cree el rol `log_analyst_lab`.

   Este rol puede buscar, leer documentos y consultar metadatos del data stream. Además, recibe acceso de solo lectura a Discover en el espacio predeterminado de Kibana.

   ```bash
   cat > /opt/elastic-labs/work/role-log-analyst-lab.json <<'EOF'
   {
     "cluster": [],
     "indices": [
       {
         "names": ["logs-ops-lab*"],
         "privileges": ["read", "view_index_metadata"],
         "allow_restricted_indices": false
       }
     ],
     "applications": [
       {
         "application": "kibana-.kibana",
         "privileges": ["feature_discover.read"],
         "resources": ["space:default"]
       }
     ],
     "run_as": [],
     "metadata": {
       "descripcion": "Consulta de logs operativos y acceso de lectura a Discover",
       "entorno": "laboratorio"
     }
   }
   EOF

   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_security/role/log_analyst_lab" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/role-log-analyst-lab.json \
     | tee /opt/elastic-labs/work/03-role-log-analyst-lab-result.json
   ```

3. Cree el rol `log_operator_lab`.

   El operador podrá observar la salud y métricas del clúster, pero no podrá leer ni modificar documentos.

   ```bash
   cat > /opt/elastic-labs/work/role-log-operator-lab.json <<'EOF'
   {
     "cluster": ["monitor"],
     "indices": [],
     "applications": [],
     "run_as": [],
     "metadata": {
       "descripcion": "Monitorización del clúster sin acceso a documentos",
       "entorno": "laboratorio"
     }
   }
   EOF

   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_security/role/log_operator_lab" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/role-log-operator-lab.json \
     | tee /opt/elastic-labs/work/03-role-log-operator-lab-result.json
   ```

4. Consulte los roles creados.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_security/role/log_ingestor_lab,log_analyst_lab,log_operator_lab?pretty" \
     | tee /opt/elastic-labs/work/03-roles-created.json
   ```

#### Salida esperada

Cada operación `PUT` debe devolver:

```json
{
  "role": {
    "created": true
  }
}
```

Si se repite la práctica, puede aparecer:

```json
{
  "role": {
    "created": false
  }
}
```

Esto indica que el rol existía y fue actualizado correctamente.

#### Verificación

Compruebe los privilegios configurados:

```bash
jq '{
  ingestor: .log_ingestor_lab.indices,
  analyst: .log_analyst_lab.indices,
  operator: .log_operator_lab.cluster
}' /opt/elastic-labs/work/03-roles-created.json
```

Debe observar:

- `create_doc` para `log_ingestor_lab`.
- `read` y `view_index_metadata` para `log_analyst_lab`.
- `monitor` como privilegio de clúster para `log_operator_lab`.

---

### Paso 4. Crear usuarios nativos para ingestión y análisis

**Objetivo:** Crear identidades humanas o de servicio separadas de la cuenta administrativa `elastic`.

#### Instrucciones

1. Cree el usuario de ingestión `ingest_lab`.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_security/user/ingest_lab" \
     -H "Content-Type: application/json" \
     -d "{
       \"password\": \"$LAB_INGEST_PASSWORD\",
       \"roles\": [\"log_ingestor_lab\"],
       \"full_name\": \"Servicio de Ingestión del Laboratorio\",
       \"email\": \"ingest_lab@example.org\",
       \"enabled\": true
     }" \
     | tee /opt/elastic-labs/work/04-user-ingest-lab-result.json
   ```

2. Cree el usuario analista `analyst_lab`.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_security/user/analyst_lab" \
     -H "Content-Type: application/json" \
     -d "{
       \"password\": \"$LAB_ANALYST_PASSWORD\",
       \"roles\": [\"log_analyst_lab\"],
       \"full_name\": \"Analista de Logs del Laboratorio\",
       \"email\": \"analyst_lab@example.org\",
       \"enabled\": true
     }" \
     | tee /opt/elastic-labs/work/04-user-analyst-lab-result.json
   ```

3. Consulte la identidad autenticada como `analyst_lab`.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "analyst_lab:$LAB_ANALYST_PASSWORD" \
     "$ES_URL/_security/_authenticate?pretty" \
     | tee /opt/elastic-labs/work/04-authenticate-analyst.json
   ```

#### Salida esperada

Las operaciones de creación deben devolver:

```json
{
  "created": true
}
```

La autenticación del analista debe indicar:

```json
{
  "username": "analyst_lab",
  "roles": [
    "log_analyst_lab"
  ],
  "enabled": true
}
```

#### Verificación

Ejecute:

```bash
jq -r '.username, (.roles[])' /opt/elastic-labs/work/04-authenticate-analyst.json
```

La salida debe incluir:

```text
analyst_lab
log_analyst_lab
```

---

### Paso 5. Crear la política ILM para logs operativos

**Objetivo:** Definir una política de ciclo de vida con rollover en fase `hot` y eliminación automática en fase `delete`.

#### Instrucciones

1. Cree la política `logs-ops-ilm-v1`.

   La condición de rollover será:

   - Edad máxima: `1d`.
   - Tamaño máximo del shard primario: `1gb`.
   - Retención de laboratorio: eliminación después de `7d`.

   ```bash
   cat > /opt/elastic-labs/work/logs-ops-ilm-v1.json <<'EOF'
   {
     "policy": {
       "phases": {
         "hot": {
           "min_age": "0ms",
           "actions": {
             "rollover": {
               "max_age": "1d",
               "max_primary_shard_size": "1gb"
             }
           }
         },
         "delete": {
           "min_age": "7d",
           "actions": {
             "delete": {}
           }
         }
       }
     }
   }
   EOF

   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_ilm/policy/logs-ops-ilm-v1" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/logs-ops-ilm-v1.json \
     | tee /opt/elastic-labs/work/05-ilm-policy-result.json
   ```

2. Consulte la política creada.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_ilm/policy/logs-ops-ilm-v1?pretty" \
     | tee /opt/elastic-labs/work/05-ilm-policy.json
   ```

#### Salida esperada

La creación de la política debe devolver:

```json
{
  "acknowledged": true
}
```

#### Verificación

```bash
jq '.["logs-ops-ilm-v1"].policy.phases' /opt/elastic-labs/work/05-ilm-policy.json
```

Confirme que existen las fases `hot` y `delete`, con `max_age`, `max_primary_shard_size` y `min_age: 7d`.

---

### Paso 6. Crear component template e index template para el data stream

**Objetivo:** Definir mappings ECS mínimos y asociar la política ILM a los índices backing del data stream.

#### Instrucciones

1. Cree el component template con settings de ILM y mappings ECS mínimos.

   ```bash
   cat > /opt/elastic-labs/work/logs-ops-component-v1.json <<'EOF'
   {
     "template": {
       "settings": {
         "index.lifecycle.name": "logs-ops-ilm-v1",
         "index.number_of_shards": 1,
         "index.number_of_replicas": 0
       },
       "mappings": {
         "dynamic": true,
         "properties": {
           "@timestamp": {
             "type": "date"
           },
           "log": {
             "properties": {
               "level": {
                 "type": "keyword"
               }
             }
           },
           "service": {
             "properties": {
               "name": {
                 "type": "keyword"
               }
             }
           },
           "host": {
             "properties": {
               "name": {
                 "type": "keyword"
               }
             }
           },
           "event": {
             "properties": {
               "dataset": {
                 "type": "keyword"
               }
             }
           },
           "message": {
             "type": "text"
           }
         }
       }
     },
     "_meta": {
       "descripcion": "Settings ILM y campos ECS mínimos para logs operativos",
       "version_laboratorio": "v1"
     }
   }
   EOF

   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_component_template/logs-ops-component-v1" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/logs-ops-component-v1.json \
     | tee /opt/elastic-labs/work/06-component-template-result.json
   ```

2. Cree el index template para el patrón `logs-ops-lab*`.

   La prioridad `500` evita que un template genérico de menor prioridad reemplace la configuración del laboratorio.

   ```bash
   cat > /opt/elastic-labs/work/logs-ops-template-v1.json <<'EOF'
   {
     "index_patterns": ["logs-ops-lab*"],
     "data_stream": {},
     "composed_of": ["logs-ops-component-v1"],
     "priority": 500,
     "_meta": {
       "descripcion": "Template de data stream para logs operativos del laboratorio",
       "propietario": "equipo-plataforma"
     }
   }
   EOF

   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_index_template/logs-ops-template-v1" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/logs-ops-template-v1.json \
     | tee /opt/elastic-labs/work/06-index-template-result.json
   ```

3. Simule el template antes de crear el data stream.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X POST "$ES_URL/_index_template/_simulate_index/logs-ops-lab?pretty" \
     | tee /opt/elastic-labs/work/06-template-simulation.json
   ```

#### Salida esperada

Las operaciones `PUT` deben devolver:

```json
{
  "acknowledged": true
}
```

La simulación debe mostrar, dentro de `template.settings`, la política:

```json
{
  "index": {
    "lifecycle": {
      "name": "logs-ops-ilm-v1"
    }
  }
}
```

#### Verificación

```bash
jq -r '.template.settings.index.lifecycle.name' \
  /opt/elastic-labs/work/06-template-simulation.json
```

La salida esperada es:

```text
logs-ops-ilm-v1
```

---

### Paso 7. Crear el data stream `logs-ops-lab`

**Objetivo:** Crear un data stream protegido por el template y la política ILM.

#### Instrucciones

1. Cree el data stream.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X PUT "$ES_URL/_data_stream/logs-ops-lab" \
     | tee /opt/elastic-labs/work/07-data-stream-create-result.json
   ```

2. Consulte los data streams existentes.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_data_stream/logs-ops-lab?pretty" \
     | tee /opt/elastic-labs/work/07-data-stream.json
   ```

3. Consulte el índice backing creado automáticamente.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_cat/indices/.ds-logs-ops-lab-*?v"
   ```

#### Salida esperada

La creación debe devolver:

```json
{
  "acknowledged": true
}
```

La consulta del data stream debe mostrar un índice backing similar a:

```text
.ds-logs-ops-lab-2026.08.25-000001
```

La fecha concreta dependerá de la fecha de ejecución.

#### Verificación

```bash
jq -r '.data_streams[0].name' /opt/elastic-labs/work/07-data-stream.json
```

Salida esperada:

```text
logs-ops-lab
```

---

### Paso 8. Reinyectar eventos con campos ECS mínimos

**Objetivo:** Copiar una selección de eventos desde `logs-app-lab-v1` hacia el nuevo data stream, normalizándolos a campos ECS mínimos.

#### Instrucciones

1. Consulte hasta 20 documentos del índice generado en la práctica anterior.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X POST "$ES_URL/logs-app-lab-v1/_search" \
     -H "Content-Type: application/json" \
     -d '{
       "size": 20,
       "sort": [
         {
           "@timestamp": "desc"
         }
       ],
       "_source": true,
       "query": {
         "match_all": {}
       }
     }' \
     | tee /opt/elastic-labs/work/08-source-events.json
   ```

2. Genere un archivo NDJSON para Bulk API. Los documentos se transforman para garantizar los campos:

   - `@timestamp`
   - `log.level`
   - `service.name`
   - `host.name`
   - `event.dataset`
   - `message`

   ```bash
   jq -c '
     .hits.hits[]._source as $s |
     {"create": {}},
     {
       "@timestamp": ($s["@timestamp"] // (now | todateiso8601)),
       "log": {
         "level": ($s.log.level // $s["log.level"] // $s.level // "INFO")
       },
       "service": {
         "name": ($s.service.name // $s["service.name"] // "logs-app-lab")
       },
       "host": {
         "name": ($s.host.name // $s["host.name"] // "es717-lab-host")
       },
       "event": {
         "dataset": ($s.event.dataset // $s["event.dataset"] // "logs.ops.lab")
       },
       "message": ($s.message // "Evento reinyectado desde logs-app-lab-v1")
     }
   ' /opt/elastic-labs/work/08-source-events.json \
   > /opt/elastic-labs/work/08-logs-ops-bulk.ndjson
   ```

3. Si no existían eventos en `logs-app-lab-v1`, genere tres eventos controlados para continuar la práctica.

   ```bash
   if [ ! -s /opt/elastic-labs/work/08-logs-ops-bulk.ndjson ]; then
     cat > /opt/elastic-labs/work/08-logs-ops-bulk.ndjson <<'EOF'
   {"create":{}}
   {"@timestamp":"2026-08-25T10:00:00Z","log":{"level":"INFO"},"service":{"name":"checkout-api"},"host":{"name":"app-node-01"},"event":{"dataset":"logs.ops.lab"},"message":"Servicio iniciado correctamente"}
   {"create":{}}
   {"@timestamp":"2026-08-25T10:05:00Z","log":{"level":"WARN"},"service":{"name":"checkout-api"},"host":{"name":"app-node-01"},"event":{"dataset":"logs.ops.lab"},"message":"Latencia elevada detectada"}
   {"create":{}}
   {"@timestamp":"2026-08-25T10:10:00Z","log":{"level":"ERROR"},"service":{"name":"checkout-api"},"host":{"name":"app-node-02"},"event":{"dataset":"logs.ops.lab"},"message":"Error de conexión con el servicio de pagos"}
   EOF
   fi
   ```

4. Inyecte los documentos en el data stream usando la cuenta administrativa. La API Bulk utiliza la operación `create`, apropiada para data streams.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X POST "$ES_URL/logs-ops-lab/_bulk?filter_path=errors,items.*.create.status,items.*.create.error" \
     -H "Content-Type: application/x-ndjson" \
     --data-binary @/opt/elastic-labs/work/08-logs-ops-bulk.ndjson \
     | tee /opt/elastic-labs/work/08-bulk-result.json
   ```

5. Consulte los eventos reinyectados.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X POST "$ES_URL/logs-ops-lab/_search?pretty" \
     -H "Content-Type: application/json" \
     -d '{
       "size": 10,
       "sort": [
         {
           "@timestamp": "desc"
         }
       ],
       "query": {
         "match_all": {}
       }
     }' \
     | tee /opt/elastic-labs/work/08-logs-ops-search.json
   ```

#### Salida esperada

El resultado de Bulk debe indicar:

```json
{
  "errors": false
}
```

La búsqueda debe devolver documentos con estructura similar a:

```json
{
  "@timestamp": "2026-08-25T10:10:00Z",
  "log": {
    "level": "ERROR"
  },
  "service": {
    "name": "checkout-api"
  },
  "host": {
    "name": "app-node-02"
  },
  "event": {
    "dataset": "logs.ops.lab"
  },
  "message": "Error de conexión con el servicio de pagos"
}
```

#### Verificación

```bash
jq '.hits.total' /opt/elastic-labs/work/08-logs-ops-search.json
jq -r '.hits.hits[0]._source.event.dataset' /opt/elastic-labs/work/08-logs-ops-search.json
```

El total debe ser mayor que cero y el dataset debe ser:

```text
logs.ops.lab
```

---

### Paso 9. Crear y validar una API key limitada para ingestión

**Objetivo:** Crear una API key que solo pueda insertar documentos en `logs-ops-lab` y comprobar que no tenga privilegios administrativos.

#### Instrucciones

1. Cree la API key limitada. La clave expira después de 30 días.

   ```bash
   cat > /opt/elastic-labs/work/09-api-key-request.json <<'EOF'
   {
     "name": "ingest-logs-ops-lab",
     "expiration": "30d",
     "role_descriptors": {
       "ingest_logs_ops_lab": {
         "cluster": [],
         "index": [
           {
             "names": ["logs-ops-lab*"],
             "privileges": ["create_doc"]
           }
         ]
       }
     },
     "metadata": {
       "sistema": "elastic-lab",
       "entorno": "laboratorio",
       "propietario": "practica-07",
       "uso": "ingestion-logs"
     }
   }
   EOF

   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X POST "$ES_URL/_security/api_key" \
     -H "Content-Type: application/json" \
     --data-binary @/opt/elastic-labs/work/09-api-key-request.json \
     > /opt/elastic-labs/work/09-api-key-response.secret.json

   chmod 600 /opt/elastic-labs/work/09-api-key-response.secret.json
   ```

2. Extraiga la clave codificada y guárdela en un archivo secreto local con permisos `0600`.

   ```bash
   jq -r '.encoded' /opt/elastic-labs/work/09-api-key-response.secret.json \
     > /opt/elastic-labs/work/ingest-logs-ops-lab-api-key.secret

   chmod 600 /opt/elastic-labs/work/ingest-logs-ops-lab-api-key.secret

   jq 'del(.encoded)' /opt/elastic-labs/work/09-api-key-response.secret.json \
     > /opt/elastic-labs/work/09-api-key-evidence.json

   export API_KEY="$(cat /opt/elastic-labs/work/ingest-logs-ops-lab-api-key.secret)"
   ```

3. Verifique los metadatos de creación sin exponer la API key.

   ```bash
   jq '{id, name, expiration, metadata}' \
     /opt/elastic-labs/work/09-api-key-evidence.json
   ```

4. Inserte un documento usando la API key.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -X POST "$ES_URL/logs-ops-lab/_doc" \
     -H "Authorization: ApiKey $API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "@timestamp": "2026-08-25T12:00:00Z",
       "log": {
         "level": "INFO"
       },
       "service": {
         "name": "api-ingestion-test"
       },
       "host": {
         "name": "agent-lab-01"
       },
       "event": {
         "dataset": "logs.ops.lab"
       },
       "message": "Documento insertado mediante API key limitada"
     }' \
     | tee /opt/elastic-labs/work/09-api-key-index-result.json
   ```

5. Intente ejecutar una operación administrativa con la API key. Esta acción debe fallar.

   ```bash
   curl --silent --show-error \
     --cacert "$ES_CA" \
     -o /opt/elastic-labs/work/09-api-key-admin-denied.json \
     -w "HTTP_STATUS=%{http_code}\n" \
     -X GET "$ES_URL/_cluster/health" \
     -H "Authorization: ApiKey $API_KEY"
   ```

6. Consulte la respuesta de denegación.

   ```bash
   cat /opt/elastic-labs/work/09-api-key-admin-denied.json | jq .
   ```

#### Salida esperada

La inserción con API key debe devolver un resultado con `result` igual a `created`.

La solicitud a `/_cluster/health` debe devolver:

```text
HTTP_STATUS=403
```

La respuesta JSON debe contener un error de seguridad similar a:

```json
{
  "error": {
    "type": "security_exception",
    "reason": "action [cluster:monitor/health] is unauthorized for API key id [...]"
  },
  "status": 403
}
```

#### Verificación

Compruebe que la API key no tenga privilegios de clúster y que pueda crear documentos:

```bash
jq -r '.result' /opt/elastic-labs/work/09-api-key-index-result.json
grep -q 'HTTP_STATUS=403' <(
  curl --silent --cacert "$ES_CA" \
    -o /dev/null \
    -w "HTTP_STATUS=%{http_code}\n" \
    -X GET "$ES_URL/_cluster/health" \
    -H "Authorization: ApiKey $API_KEY"
) && echo "API key correctamente restringida"
```

Salida esperada:

```text
created
API key correctamente restringida
```

---

### Paso 10. Forzar rollover y comprobar la asignación de ILM

**Objetivo:** Crear un nuevo índice backing de forma controlada y verificar que ILM administra ambos índices backing.

#### Instrucciones

1. Consulte el data stream antes del rollover.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_data_stream/logs-ops-lab?pretty" \
     | tee /opt/elastic-labs/work/10-data-stream-before-rollover.json
   ```

2. Fuerce un rollover manual del data stream.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     -X POST "$ES_URL/logs-ops-lab/_rollover" \
     -H "Content-Type: application/json" \
     -d '{}' \
     | tee /opt/elastic-labs/work/10-rollover-result.json
   ```

3. Consulte de nuevo el data stream.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_data_stream/logs-ops-lab?pretty" \
     | tee /opt/elastic-labs/work/10-data-stream-after-rollover.json
   ```

4. Consulte el estado ILM de los índices backing.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/logs-ops-lab/_ilm/explain?pretty" \
     | tee /opt/elastic-labs/work/10-ilm-explain.json
   ```

5. Liste los índices backing y su política de ciclo de vida.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_cat/indices/.ds-logs-ops-lab-*?h=health,status,index,docs.count,store.size&v"
   ```

#### Salida esperada

El resultado de rollover debe contener valores similares a:

```json
{
  "old_index": ".ds-logs-ops-lab-2026.08.25-000001",
  "new_index": ".ds-logs-ops-lab-2026.08.25-000002",
  "rolled_over": true,
  "dry_run": false
}
```

En la respuesta de ILM Explain, los índices backing deben referenciar la política:

```json
{
  "policy": "logs-ops-ilm-v1",
  "phase": "hot"
}
```

#### Verificación

```bash
jq '
  .indices
  | to_entries[]
  | {
      index: .key,
      policy: .value.policy,
      phase: .value.phase,
      action: .value.action,
      step: .value.step
    }
' /opt/elastic-labs/work/10-ilm-explain.json
```

Debe observar dos índices backing o más, todos asociados a `logs-ops-ilm-v1`.

---

### Paso 11. Validar permisos de analista e ingesta

**Objetivo:** Confirmar que cada identidad solo puede ejecutar las acciones previstas por su rol.

#### Instrucciones

1. Valide que `analyst_lab` puede leer logs.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "analyst_lab:$LAB_ANALYST_PASSWORD" \
     -X POST "$ES_URL/logs-ops-lab/_search" \
     -H "Content-Type: application/json" \
     -d '{
       "size": 5,
       "query": {
         "match_all": {}
       }
     }' \
     | tee /opt/elastic-labs/work/11-analyst-search-success.json
   ```

2. Valide que `analyst_lab` no puede escribir en el data stream.

   ```bash
   curl --silent --show-error \
     --cacert "$ES_CA" \
     -u "analyst_lab:$LAB_ANALYST_PASSWORD" \
     -o /opt/elastic-labs/work/11-analyst-write-denied.json \
     -w "HTTP_STATUS=%{http_code}\n" \
     -X POST "$ES_URL/logs-ops-lab/_doc" \
     -H "Content-Type: application/json" \
     -d '{
       "@timestamp": "2026-08-25T13:00:00Z",
       "message": "Esta operación debe ser denegada"
     }'
   ```

3. Valide que `ingest_lab` puede crear un documento.

   ```bash
   curl --silent --show-error --fail \
     --cacert "$ES_CA" \
     -u "ingest_lab:$LAB_INGEST_PASSWORD" \
     -X POST "$ES_URL/logs-ops-lab/_doc" \
     -H "Content-Type: application/json" \
     -d '{
       "@timestamp": "2026-08-25T13:05:00Z",
       "log": {
         "level": "INFO"
       },
       "service": {
         "name": "native-user-ingestion"
       },
       "host": {
         "name": "agent-lab-02"
       },
       "event": {
         "dataset": "logs.ops.lab"
       },
       "message": "Documento creado por el usuario ingest_lab"
     }' \
     | tee /opt/elastic-labs/work/11-ingest-user-write-success.json
   ```

4. Valide que `ingest_lab` no puede leer documentos.

   ```bash
   curl --silent --show-error \
     --cacert "$ES_CA" \
     -u "ingest_lab:$LAB_INGEST_PASSWORD" \
     -o /opt/elastic-labs/work/11-ingest-user-read-denied.json \
     -w "HTTP_STATUS=%{http_code}\n" \
     -X POST "$ES_URL/logs-ops-lab/_search" \
     -H "Content-Type: application/json" \
     -d '{
       "query": {
         "match_all": {}
       }
     }'
   ```

5. Revise las respuestas de denegación.

   ```bash
   jq '.error.reason' /opt/elastic-labs/work/11-analyst-write-denied.json
   jq '.error.reason' /opt/elastic-labs/work/11-ingest-user-read-denied.json
   ```

#### Salida esperada

| Prueba | Resultado esperado |
|---|---|
| `analyst_lab` realiza `_search` | HTTP 200 |
| `analyst_lab` intenta indexar | HTTP 403 |
| `ingest_lab` crea documento | HTTP 201 |
| `ingest_lab` intenta `_search` | HTTP 403 |

#### Verificación

Ejecute las comprobaciones resumidas:

```bash
jq -r '.hits.total.value // .hits.total' \
  /opt/elastic-labs/work/11-analyst-search-success.json

jq -r '.result' \
  /opt/elastic-labs/work/11-ingest-user-write-success.json
```

La primera salida debe ser mayor que cero. La segunda debe ser:

```text
created
```

---

### Paso 12. Preparar la fuente de datos para Discover

**Objetivo:** Confirmar que el data stream está disponible para utilizarse en Discover durante la práctica siguiente.

#### Instrucciones

1. Acceda a Kibana en el navegador usando el endpoint definido por el laboratorio, normalmente:

   ```text
   https://localhost:5601
   ```

2. Inicie sesión como `elastic`.

3. Abra **Stack Management** → **Data Views** → **Create data view**.

4. Cree una vista de datos con los siguientes valores:

   | Campo | Valor |
   |---|---|
   | Nombre | `Logs Ops Lab` |
   | Patrón de índice | `logs-ops-lab*` |
   | Campo de tiempo | `@timestamp` |

5. Guarde la vista de datos.

6. Abra **Analytics** → **Discover** y seleccione la vista `Logs Ops Lab`.

7. Aplique un filtro KQL para confirmar que se visualizan los documentos:

   ```kql
   event.dataset: "logs.ops.lab"
   ```

8. Opcionalmente, cierre sesión e inicie como `analyst_lab` para comprobar el acceso de solo lectura a Discover.

#### Salida esperada

Discover debe mostrar eventos del data stream `logs-ops-lab`, incluidos documentos con campos:

- `@timestamp`
- `log.level`
- `service.name`
- `host.name`
- `event.dataset`
- `message`

#### Verificación

En Discover, ejecute la consulta KQL:

```kql
log.level: ERROR
```

Debe aparecer al menos el evento de error reinyectado o creado durante esta práctica.

---

## Validación y pruebas

Ejecute el siguiente bloque para realizar una validación resumida del laboratorio:

```bash
echo "=== Salud del clúster ==="
curl --silent --show-error --fail \
  --cacert "$ES_CA" \
  -u "$ES_USER:$ELASTIC_PASSWORD" \
  "$ES_URL/_cluster/health?filter_path=status,cluster_name&pretty"

echo
echo "=== Roles de laboratorio ==="
curl --silent --show-error --fail \
  --cacert "$ES_CA" \
  -u "$ES_USER:$ELASTIC_PASSWORD" \
  "$ES_URL/_security/role/log_ingestor_lab,log_analyst_lab,log_operator_lab?pretty" \
  | jq 'keys'

echo
echo "=== Usuarios nativos ==="
curl --silent --show-error --fail \
  --cacert "$ES_CA" \
  -u "$ES_USER:$ELASTIC_PASSWORD" \
  "$ES_URL/_security/user/ingest_lab,analyst_lab?pretty" \
  | jq 'to_entries[] | {usuario: .key, roles: .value.roles, enabled: .value.enabled}'

echo
echo "=== Data stream ==="
curl --silent --show-error --fail \
  --cacert "$ES_CA" \
  -u "$ES_USER:$ELASTIC_PASSWORD" \
  "$ES_URL/_data_stream/logs-ops-lab?pretty" \
  | jq '.data_streams[] | {name, generation, indices}'

echo
echo "=== Estado ILM ==="
curl --silent --show-error --fail \
  --cacert "$ES_CA" \
  -u "$ES_USER:$ELASTIC_PASSWORD" \
  "$ES_URL/logs-ops-lab/_ilm/explain?pretty" \
  | jq '.indices | to_entries[] | {index: .key, policy: .value.policy, phase: .value.phase}'

echo
echo "=== Conteo de documentos ==="
curl --silent --show-error --fail \
  --cacert "$ES_CA" \
  -u "$ES_USER:$ELASTIC_PASSWORD" \
  "$ES_URL/logs-ops-lab/_count?pretty"
```

La práctica se considera completada cuando se cumplen todas las condiciones siguientes:

- El clúster responde mediante HTTPS y autenticación.
- Existen los tres roles definidos.
- Existen los usuarios `ingest_lab` y `analyst_lab`.
- Existe la política `logs-ops-ilm-v1`.
- Existe el data stream `logs-ops-lab`.
- El data stream contiene al menos un índice backing administrado por ILM.
- Se ha realizado un rollover y existe una nueva generación del data stream.
- La API key puede crear documentos, pero no consultar salud del clúster.
- `analyst_lab` puede leer, pero no escribir.
- `ingest_lab` puede escribir, pero no leer.

## Resolución de problemas

### Problema 1. `curl` devuelve un error de certificado TLS o no puede verificar la CA

**Síntomas**

```text
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

o:

```text
curl: (60) SSL certificate problem: certificate verify failed
```

**Causa**

El comando no está usando el certificado CA del laboratorio, la ruta es incorrecta o el certificado presentado por Elasticsearch no corresponde al nombre de host utilizado.

**Solución**

1. Verifique que el certificado exista y pueda leerse.

   ```bash
   ls -l /opt/elastic-labs/certs/ca/ca.crt
   ```

2. Asegúrese de utilizar HTTPS y la opción `--cacert`.

   ```bash
   curl --cacert /opt/elastic-labs/certs/ca/ca.crt \
     -u "elastic:$ELASTIC_PASSWORD" \
     https://localhost:9200/
   ```

3. No utilice `-k` o `--insecure` como solución permanente. Esa opción desactiva la validación TLS y oculta errores reales de certificados o nombres DNS.

4. Si el certificado no contiene `localhost` como SAN, consulte la configuración de la composición del laboratorio y utilice el nombre configurado en el certificado.

### Problema 2. La escritura en el data stream devuelve `403 security_exception` o `illegal_argument_exception`

**Síntomas**

La API key o el usuario `ingest_lab` recibe un error similar a:

```json
{
  "error": {
    "type": "security_exception",
    "reason": "action [indices:data/write/index] is unauthorized"
  },
  "status": 403
}
```

O la API Bulk muestra errores por operación no permitida en un data stream.

**Causa**

Las causas más frecuentes son:

- El rol o la API key no tiene `create_doc` sobre el patrón `logs-ops-lab*`.
- Se está escribiendo en un nombre diferente de `logs-ops-lab`.
- Se usa una operación `index` en lugar de `create` mediante Bulk API.
- El data stream o su index template no existe.

**Solución**

1. Compruebe que el data stream exista.

   ```bash
   curl --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_data_stream/logs-ops-lab?pretty"
   ```

2. Compruebe los privilegios del rol de ingestión.

   ```bash
   curl --cacert "$ES_CA" \
     -u "$ES_USER:$ELASTIC_PASSWORD" \
     "$ES_URL/_security/role/log_ingestor_lab?pretty"
   ```

3. Confirme que el rol contiene:

   ```json
   {
     "names": ["logs-ops-lab*"],
     "privileges": ["create_doc"]
   }
   ```

4. Para Bulk API, utilice una línea de acción `create`:

   ```json
   {"create":{}}
   ```

5. Si se modificó el rol o la API key, repita la creación de la API key para garantizar que se apliquen los privilegios previstos.

## Limpieza

Esta práctica deja recursos que serán utilizados en actividades posteriores. No elimine el data stream, los templates, la política ILM, los usuarios ni los volúmenes Docker.

1. Elimine de la sesión actual las variables sensibles.

   ```bash
   unset API_KEY
   unset LAB_INGEST_PASSWORD
   unset LAB_ANALYST_PASSWORD
   unset ELASTIC_PASSWORD
   ```

2. Mantenga protegido el archivo que contiene la API key del laboratorio.

   ```bash
   chmod 600 /opt/elastic-labs/work/ingest-logs-ops-lab-api-key.secret
   chmod 600 /opt/elastic-labs/work/09-api-key-response.secret.json
   ```

3. Si no necesita conservar la respuesta completa que incluye la API key codificada, elimínela. La evidencia sin el secreto permanece en `09-api-key-evidence.json`.

   ```bash
   rm -f /opt/elastic-labs/work/09-api-key-response.secret.json
   ```

4. No ejecute los siguientes comandos:

   ```bash
   docker compose down -v
   docker volume rm es717-data
   ```

## Resumen

En esta práctica se verificó el uso de TLS y autenticación en Elasticsearch 7.17.29, separando la cuenta administrativa de las identidades operativas. Se implementaron roles de mínimo privilegio para ingestión, análisis y monitorización, y se validó que una API key limitada no puede realizar operaciones administrativas.

También se creó la política `logs-ops-ilm-v1`, el component template, el index template y el data stream `logs-ops-lab`. Los eventos reinyectados disponen de campos ECS mínimos y los índices backing están administrados por ILM. El rollover controlado demostró la creación de una nueva generación del data stream, que quedará disponible para análisis en Discover durante la siguiente práctica.

### Recursos opcionales

- [Elasticsearch Security](https://www.elastic.co/docs/deploy-manage/security)
- [Roles y privilegios en Elasticsearch](https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/user-roles)
- [API keys de Elasticsearch](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-create-api-key)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/index-lifecycle-management.html)
- [Data streams en Elasticsearch 7.17](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/data-streams.html)
