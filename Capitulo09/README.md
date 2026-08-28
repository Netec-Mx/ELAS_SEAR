# Ruta de actualización desde 7.17 hacia 8.19 y posteriormente hacia 9.x, utilizando 9.3 como versión objetivo

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 65 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica se ensaya una migración paralela de datos desde Elasticsearch 7.17.29 hacia Elasticsearch 8.19.0 y, posteriormente, hacia Elasticsearch 9.3.0. La ruta utiliza snapshots verificables para los datos de Elasticsearch y exportación/importación controlada para los objetos guardados de Kibana.

El resultado será un runbook técnico con evidencias de estado, snapshots, restauraciones, validaciones funcionales, controles de seguridad y criterios de rollback. No se realiza una actualización binaria in-place: los clústeres de destino se crean en paralelo para reducir el riesgo y facilitar la recuperación.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Ejecutar la ruta compatible `7.17.29 → 8.19.0 → 9.3.0` mediante snapshots y restauraciones controladas.
- [ ] Crear, verificar y documentar los snapshots `preupgrade-717-001` y `preupgrade-819-001`.
- [ ] Validar índices, data streams, ILM, seguridad, consultas DSL, KQL y ES|QL después de cada transición.
- [ ] Diferenciar la migración de datos de Elasticsearch de la migración de objetos guardados de Kibana.
- [ ] Elaborar un runbook con decisiones operativas, criterios de aceptación y rollback.

## Prerrequisitos

### Conocimientos requeridos

- Prácticas 6, 7 y 8 completadas.
- Uso básico de Docker Compose, `curl`, `jq` y APIs REST de Elasticsearch.
- Conocimientos de snapshots, repositorios `fs`, ILM, data streams y control de acceso basado en roles.
- Comprensión de que un snapshot no equivale a una reversión binaria de una versión mayor.

### Acceso y estado inicial requeridos

- Acceso con privilegios `sudo` al host Linux.
- Directorio de trabajo obligatorio: `/opt/elastic-labs`.
- Docker Engine y Docker Compose plugin operativos.
- Al menos 30 GB adicionales disponibles antes de descargar las imágenes 8.19.0 y 9.3.0.
- Datos disponibles en:
  - `logs-app-lab-v1`
  - `logs-ops-lab`
- Política ILM disponible: `logs-ops-ilm-v1`.
- Roles y consultas guardadas de prácticas anteriores documentados.
- Puertos disponibles: `9200`, `9201`, `5601`, `5602` y `5603`.

> **Importante:** No ejecute `docker compose down -v`. La opción `-v` eliminaría volúmenes y destruiría el estado necesario para la continuidad del laboratorio.

## Entorno de laboratorio

### Recursos recomendados

| Recurso | Mínimo | Recomendado |
|---|---:|---:|
| CPU | 4 vCPU | 6 vCPU |
| RAM | 16 GB | 24 GB |
| Espacio SSD libre | 40 GB | 60 GB |
| Sistema operativo | Ubuntu Server/Desktop compatible | Ubuntu 24.04 LTS |
| `vm.max_map_count` | `262144` | `262144` |

### Componentes y puertos

| Componente | Contenedor | Versión | Puerto host |
|---|---|---:|---:|
| Elasticsearch origen | `es717-lab` | 7.17.29 | `9200` |
| Kibana origen | `kibana717-lab` | 7.17.29 | `5601` |
| Elasticsearch intermedio | `es819-lab` | 8.19.0 | `9202` |
| Kibana intermedio | `kibana819-lab` | 8.19.0 | `5603` |
| Elasticsearch destino | `es930-lab` | 9.3.0 | `9201` |
| Kibana destino | `kibana930-lab` | 9.3.0 | `5602` |

### Clústeres, red y volúmenes

| Elemento | Valor |
|---|---|
| Red Docker | `elastic-lab-net` |
| Clúster 7.17 | `es717-lab-cluster` |
| Nodo 7.17 | `es717-node-1` |
| Clúster 9.3 | `es930-lab-cluster` |
| Nodo 9.3 | `es930-node-1` |
| Volumen Elasticsearch 7.17 | `es717-data` |
| Volumen Elasticsearch 9.3 | `es930-data` |
| Volumen Kibana 7.17 | `kibana717-data` |
| Volumen Kibana 9.3 | `kibana930-data` |
| Directorio de evidencias | `/opt/elastic-labs/work` |
| Snapshots de 7.17 | `/opt/elastic-labs/snapshots/717` |
| Snapshots de 8.19 | `/opt/elastic-labs/snapshots/819` |

> **Nota de seguridad:** El laboratorio usa autenticación básica por HTTP local para simplificar las validaciones de migración. En producción, habilite TLS HTTP y TLS de transporte, gestione certificados válidos y no reutilice la contraseña del laboratorio.

---

## Procedimiento paso a paso

### Paso 1. Preparar el directorio, credenciales y Docker Compose

**Objetivo:** Preparar una configuración reproducible para los tres entornos Elasticsearch y sus instancias Kibana asociadas.

**Instrucciones:**

1. Cree la estructura obligatoria de directorios y configure el parámetro del kernel requerido por Elasticsearch.

   ```bash
   sudo mkdir -p /opt/elastic-labs/{work,snapshots/717,snapshots/819}
   sudo chown -R "$USER":"$USER" /opt/elastic-labs

   echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-elastic-labs.conf
   sudo sysctl --system
   ```

2. Confirme el valor aplicado y el espacio disponible.

   ```bash
   sysctl vm.max_map_count
   df -h /opt
   docker version --format '{{.Server.Version}}'
   docker compose version
   ```

3. Cree el archivo de variables con permisos restrictivos.

   ```bash
   cat > /opt/elastic-labs/.env <<'EOF'
   ELASTIC_PASSWORD=ElasticLab_2026!
   KIBANA_SYSTEM_PASSWORD=KibanaSystemLab_2026!
   EOF

   chmod 0600 /opt/elastic-labs/.env
   ```

4. Cree el archivo `/opt/elastic-labs/compose.yaml`.

   ```bash
   cat > /opt/elastic-labs/compose.yaml <<'EOF'
   services:
     es717-lab:
       image: docker.elastic.co/elasticsearch/elasticsearch:7.17.29
       container_name: es717-lab
       environment:
         - node.name=es717-node-1
         - cluster.name=es717-lab-cluster
         - discovery.type=single-node
         - ES_JAVA_OPTS=-Xms1g -Xmx1g
         - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
         - xpack.security.enabled=true
         - xpack.security.http.ssl.enabled=false
         - xpack.security.transport.ssl.enabled=false
         - path.repo=/usr/share/elasticsearch/snapshots
       ports:
         - "9200:9200"
       volumes:
         - es717-data:/usr/share/elasticsearch/data
         - /opt/elastic-labs/snapshots:/usr/share/elasticsearch/snapshots
       networks:
         - elastic-lab-net

     kibana717-lab:
       image: docker.elastic.co/kibana/kibana:7.17.29
       container_name: kibana717-lab
       depends_on:
         - es717-lab
       environment:
         - ELASTICSEARCH_HOSTS=http://es717-lab:9200
         - ELASTICSEARCH_USERNAME=kibana_system
         - ELASTICSEARCH_PASSWORD=${KIBANA_SYSTEM_PASSWORD}
       ports:
         - "5601:5601"
       volumes:
         - kibana717-data:/usr/share/kibana/data
       networks:
         - elastic-lab-net

     es819-lab:
       image: docker.elastic.co/elasticsearch/elasticsearch:8.19.0
       container_name: es819-lab
       profiles: ["upgrade819"]
       environment:
         - node.name=es819-node-1
         - cluster.name=es819-lab-cluster
         - discovery.type=single-node
         - ES_JAVA_OPTS=-Xms1g -Xmx1g
         - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
         - xpack.security.enabled=true
         - xpack.security.http.ssl.enabled=false
         - xpack.security.transport.ssl.enabled=false
         - path.repo=/usr/share/elasticsearch/snapshots
       ports:
         - "9202:9200"
       volumes:
         - /opt/elastic-labs/snapshots:/usr/share/elasticsearch/snapshots
       networks:
         - elastic-lab-net

     kibana819-lab:
       image: docker.elastic.co/kibana/kibana:8.19.0
       container_name: kibana819-lab
       profiles: ["upgrade819"]
       depends_on:
         - es819-lab
       environment:
         - ELASTICSEARCH_HOSTS=http://es819-lab:9200
         - ELASTICSEARCH_USERNAME=kibana_system
         - ELASTICSEARCH_PASSWORD=${KIBANA_SYSTEM_PASSWORD}
       ports:
         - "5603:5601"
       networks:
         - elastic-lab-net

     es930-lab:
       image: docker.elastic.co/elasticsearch/elasticsearch:9.3.0
       container_name: es930-lab
       environment:
         - node.name=es930-node-1
         - cluster.name=es930-lab-cluster
         - discovery.type=single-node
         - ES_JAVA_OPTS=-Xms1g -Xmx1g
         - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
         - xpack.security.enabled=true
         - xpack.security.http.ssl.enabled=false
         - xpack.security.transport.ssl.enabled=false
         - path.repo=/usr/share/elasticsearch/snapshots
       ports:
         - "9201:9200"
       volumes:
         - es930-data:/usr/share/elasticsearch/data
         - /opt/elastic-labs/snapshots:/usr/share/elasticsearch/snapshots
       networks:
         - elastic-lab-net

     kibana930-lab:
       image: docker.elastic.co/kibana/kibana:9.3.0
       container_name: kibana930-lab
       depends_on:
         - es930-lab
       environment:
         - ELASTICSEARCH_HOSTS=http://es930-lab:9200
         - ELASTICSEARCH_USERNAME=kibana_system
         - ELASTICSEARCH_PASSWORD=${KIBANA_SYSTEM_PASSWORD}
       ports:
         - "5602:5601"
       volumes:
         - kibana930-data:/usr/share/kibana/data
       networks:
         - elastic-lab-net

   networks:
     elastic-lab-net:
       name: elastic-lab-net

   volumes:
     es717-data:
       name: es717-data
     es930-data:
       name: es930-data
     kibana717-data:
       name: kibana717-data
     kibana930-data:
       name: kibana930-data
   EOF
   ```

5. Descargue las imágenes y arranque únicamente el entorno de origen 7.17.

   ```bash
   cd /opt/elastic-labs
   docker compose pull
   docker compose up -d es717-lab kibana717-lab
   ```

6. Espere a que Elasticsearch 7.17 responda.

   ```bash
   source /opt/elastic-labs/.env

   until curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200/_cluster/health | jq -e '.status' >/dev/null; do
     echo "Esperando Elasticsearch 7.17..."
     sleep 5
   done
   ```

7. Establezca la contraseña del usuario de servicio de Kibana si todavía no está configurada.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X POST "http://localhost:9200/_security/user/kibana_system/_password" \
     -H 'Content-Type: application/json' \
     -d "{\"password\":\"${KIBANA_SYSTEM_PASSWORD}\"}" | jq
   ```

8. Reinicie Kibana 7.17 para que vuelva a autenticarse.

   ```bash
   docker compose restart kibana717-lab
   ```

**Resultado esperado:**

- `vm.max_map_count` muestra `262144`.
- El archivo `.env` tiene permisos `-rw-------`.
- Los contenedores `es717-lab` y `kibana717-lab` están en estado `Up`.
- Elasticsearch 7.17 responde en `http://localhost:9200`.

**Verificación:**

```bash
curl -s -u "elastic:${ELASTIC_PASSWORD}" \
  http://localhost:9200 | jq '{cluster_name, name, version: .version.number}'

docker compose ps
```

La salida debe incluir:

```json
{
  "cluster_name": "es717-lab-cluster",
  "name": "es717-node-1",
  "version": "7.17.29"
}
```

---

### Paso 2. Crear la línea base y revisar la preparación en Elasticsearch 7.17

**Objetivo:** Confirmar que el entorno origen es estable, inventariar los datos de laboratorio y recopilar evidencias previas a la actualización.

**Instrucciones:**

1. Defina funciones de acceso para reducir errores durante la práctica.

   ```bash
   cd /opt/elastic-labs
   source .env

   api717() {
     curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
       -H 'Content-Type: application/json' "$@"
   }

   api819() {
     curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
       -H 'Content-Type: application/json' "$@"
   }

   api930() {
     curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
       -H 'Content-Type: application/json' "$@"
   }
   ```

2. Recopile la línea base operativa del clúster 7.17.

   ```bash
   api717 "http://localhost:9200/_cluster/health?pretty" \
     | tee work/717-cluster-health-pre.json

   api717 "http://localhost:9200/_cat/nodes?v" \
     | tee work/717-nodes-pre.txt

   api717 "http://localhost:9200/_cat/indices?v&expand_wildcards=all" \
     | tee work/717-indices-pre.txt

   api717 "http://localhost:9200/_cat/shards?v" \
     | tee work/717-shards-pre.txt

   api717 "http://localhost:9200/_cat/allocation?v" \
     | tee work/717-allocation-pre.txt

   api717 "http://localhost:9200/_nodes/stats/jvm,fs,indices?pretty" \
     | tee work/717-node-stats-pre.json
   ```

3. Verifique explícitamente que no existan shards no asignados.

   ```bash
   api717 "http://localhost:9200/_cat/shards?h=index,shard,prirep,state,unassigned.reason" \
     | awk '$4 == "UNASSIGNED" {print}'
   ```

4. Revise los índices y data streams de laboratorio.

   ```bash
   api717 "http://localhost:9200/_cat/indices/logs-app-lab-v1?v"
   api717 "http://localhost:9200/_data_stream/logs-ops-lab?pretty" \
     | tee work/717-data-stream-pre.json

   api717 "http://localhost:9200/logs-app-lab-v1/_count?pretty" \
     | tee work/717-count-app-pre.json

   api717 "http://localhost:9200/logs-ops-lab/_count?pretty" \
     | tee work/717-count-ops-pre.json
   ```

5. Revise la política ILM y el estado del ciclo de vida del data stream.

   ```bash
   api717 "http://localhost:9200/_ilm/policy/logs-ops-ilm-v1?pretty" \
     | tee work/717-ilm-policy.json

   api717 "http://localhost:9200/logs-ops-lab/_ilm/explain?pretty" \
     | tee work/717-ilm-explain.json
   ```

6. Revise la versión de creación de los índices de laboratorio. Los índices creados en 6.x o versiones anteriores requieren reindexación antes de una actualización mayor.

   ```bash
   api717 "http://localhost:9200/logs-app-lab-v1/_settings?filter_path=*.settings.index.version.created&pretty" \
     | tee work/717-created-version-app.json

   api717 "http://localhost:9200/.ds-logs-ops-lab-*/_settings?expand_wildcards=all&filter_path=*.settings.index.version.created&pretty" \
     | tee work/717-created-version-ops.json
   ```

7. Consulte avisos de deprecación disponibles en 7.17.

   ```bash
   api717 "http://localhost:9200/_migration/deprecations?pretty" \
     | tee work/717-deprecations.json
   ```

8. Abra Kibana 7.17 en `http://localhost:5601`, autentíquese con `elastic` y la contraseña de laboratorio, y revise **Stack Management → Upgrade Assistant**. Registre los avisos detectados en:

   ```bash
   nano work/717-upgrade-assistant-notes.md
   ```

   Incluya como mínimo:
   - Avisos de índices.
   - Avisos de configuración.
   - Plantillas o pipelines obsoletos.
   - Riesgos para la transición a 8.19.

**Resultado esperado:**

- La salud del clúster es `green`.
- No se muestran líneas con estado `UNASSIGNED`.
- Existen datos en `logs-app-lab-v1` y/o en `logs-ops-lab`.
- La política `logs-ops-ilm-v1` está disponible.
- No existen índices de laboratorio creados originalmente en 6.x o anteriores.

**Verificación:**

```bash
jq -r '.status' work/717-cluster-health-pre.json
jq -r '.count' work/717-count-app-pre.json
jq -r '.count' work/717-count-ops-pre.json
```

Documente los conteos obtenidos. Se utilizarán como referencia de integridad después de cada restauración.

> **Punto de decisión:** Si la salud no es `green`, existen shards no asignados o se detectan índices incompatibles, no continúe. Corrija el problema, reindexe los índices heredados si corresponde y repita la línea base.

---

### Paso 3. Exportar objetos de Kibana y crear el snapshot de 7.17

**Objetivo:** Separar explícitamente la migración de datos de Elasticsearch de la migración de objetos guardados de Kibana, y generar un snapshot verificable antes de la transición a 8.19.

**Instrucciones:**

1. Exporte los objetos guardados relevantes desde Kibana 7.17. La exportación incluye búsquedas, visualizaciones, dashboards y patrones de índice, junto con sus referencias.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X POST "http://localhost:5601/api/saved_objects/_export" \
     -H 'kbn-xsrf: true' \
     -H 'Content-Type: application/json' \
     -d '{
       "type": ["dashboard", "visualization", "search", "index-pattern"],
       "includeReferencesDeep": true
     }' \
     -o work/kibana-717-saved-objects.ndjson
   ```

2. Compruebe que el archivo no esté vacío y conserve una evidencia de su contenido sin modificarlo.

   ```bash
   wc -l work/kibana-717-saved-objects.ndjson
   head -n 3 work/kibana-717-saved-objects.ndjson \
     | tee work/kibana-717-saved-objects-preview.txt
   ```

3. Exporte la configuración que deberá recrearse en el entorno de destino. Los snapshots creados en esta práctica no restaurarán el estado global.

   ```bash
   api717 "http://localhost:9200/_index_template/logs-ops-lab*?pretty" \
     | tee work/717-index-templates.json

   api717 "http://localhost:9200/_component_template?pretty" \
     | tee work/717-component-templates.json

   api717 "http://localhost:9200/_security/role?pretty" \
     | tee work/717-roles.json
   ```

4. Registre el repositorio de snapshots de 7.17. El contenedor ve el directorio del host como `/usr/share/elasticsearch/snapshots`.

   ```bash
   api717 -X PUT "http://localhost:9200/_snapshot/fs_lab_repo" \
     -d '{
       "type": "fs",
       "settings": {
         "location": "/usr/share/elasticsearch/snapshots/717",
         "compress": true
       }
     }' | tee work/717-repository-create.json
   ```

5. Verifique el repositorio.

   ```bash
   api717 -X POST "http://localhost:9200/_snapshot/fs_lab_repo/_verify?pretty" \
     | tee work/717-repository-verify.json
   ```

6. Cree el snapshot de preactualización. Se incluyen exclusivamente los índices y data streams de laboratorio; el estado global se excluye deliberadamente.

   ```bash
   api717 -X PUT \
     "http://localhost:9200/_snapshot/fs_lab_repo/preupgrade-717-001?wait_for_completion=true" \
     -d '{
       "indices": "logs-app-lab-v1,logs-ops-lab",
       "ignore_unavailable": false,
       "include_global_state": false,
       "partial": false
     }' | tee work/717-snapshot-preupgrade.json
   ```

7. Consulte el snapshot y registre la lista exacta de índices incluidos.

   ```bash
   api717 "http://localhost:9200/_snapshot/fs_lab_repo/preupgrade-717-001?pretty" \
     | tee work/717-snapshot-detail.json

   jq -r '.snapshots[0] | {snapshot, state, start_time, end_time, indices}' \
     work/717-snapshot-detail.json
   ```

**Resultado esperado:**

- Existe el archivo `work/kibana-717-saved-objects.ndjson`.
- El repositorio `fs_lab_repo` responde correctamente a `_verify`.
- El snapshot `preupgrade-717-001` tiene estado `SUCCESS`.
- La lista de índices incluye `logs-app-lab-v1` y los backing indices correspondientes a `logs-ops-lab`.

**Verificación:**

```bash
jq -r '.snapshots[0].state' work/717-snapshot-detail.json
find /opt/elastic-labs/snapshots/717 -maxdepth 2 -type f | head
```

El estado debe ser:

```text
SUCCESS
```

> **Decisión de rollback para esta etapa:** Si la migración a 8.19 falla, no intente restaurar un snapshot generado por 8.19 sobre 7.17. Mantenga operativo el clúster 7.17 y restaure, si es necesario, `preupgrade-717-001` únicamente en un entorno compatible.

---

### Paso 4. Desplegar Elasticsearch 8.19, restaurar el snapshot y migrar objetos de Kibana

**Objetivo:** Restaurar los datos de laboratorio en la versión intermedia 8.19.0, recrear configuraciones necesarias y migrar objetos guardados de Kibana de forma controlada.

**Instrucciones:**

1. Arranque Elasticsearch y Kibana 8.19 mediante el perfil temporal.

   ```bash
   cd /opt/elastic-labs
   docker compose --profile upgrade819 up -d es819-lab kibana819-lab
   ```

2. Espere a que Elasticsearch 8.19 esté disponible.

   ```bash
   until curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9202/_cluster/health | jq -e '.status' >/dev/null; do
     echo "Esperando Elasticsearch 8.19..."
     sleep 5
   done
   ```

3. Configure la contraseña del usuario de servicio de Kibana 8.19 y reinicie Kibana.

   ```bash
   api819 -X POST \
     "http://localhost:9202/_security/user/kibana_system/_password" \
     -d "{\"password\":\"${KIBANA_SYSTEM_PASSWORD}\"}" | jq

   docker compose restart kibana819-lab
   ```

4. Registre el mismo repositorio lógico, ahora apuntando al directorio de snapshots de 7.17.

   ```bash
   api819 -X PUT "http://localhost:9202/_snapshot/fs_lab_repo" \
     -d '{
       "type": "fs",
       "settings": {
         "location": "/usr/share/elasticsearch/snapshots/717",
         "compress": true
       }
     }' | tee work/819-repository-create.json

   api819 -X POST "http://localhost:9202/_snapshot/fs_lab_repo/_verify?pretty" \
     | tee work/819-repository-verify.json
   ```

5. Inspeccione el snapshot antes de restaurarlo.

   ```bash
   api819 "http://localhost:9202/_snapshot/fs_lab_repo/preupgrade-717-001?pretty" \
     | tee work/819-visible-717-snapshot.json
   ```

6. Restaure exclusivamente los índices y el data stream de laboratorio. Se ajusta el número de réplicas a cero porque el clúster tiene un único nodo.

   ```bash
   api819 -X POST \
     "http://localhost:9202/_snapshot/fs_lab_repo/preupgrade-717-001/_restore?wait_for_completion=true" \
     -d '{
       "indices": "logs-app-lab-v1,logs-ops-lab",
       "ignore_unavailable": false,
       "include_global_state": false,
       "include_aliases": true,
       "index_settings": {
         "index.number_of_replicas": 0
       }
     }' | tee work/819-restore-717.json
   ```

7. Reinstale o valide la política ILM. Extraiga únicamente el bloque `policy` capturado desde 7.17.

   ```bash
   jq '."logs-ops-ilm-v1".policy' work/717-ilm-policy.json \
     > work/logs-ops-ilm-v1-policy-body.json

   api819 -X PUT \
     "http://localhost:9202/_ilm/policy/logs-ops-ilm-v1" \
     --data-binary @work/logs-ops-ilm-v1-policy-body.json \
     | tee work/819-ilm-policy-put.json
   ```

8. Si las prácticas previas utilizaron plantillas componibles específicas, recréelas desde los archivos exportados. Revise los nombres disponibles:

   ```bash
   jq -r '.index_templates[].name' work/717-index-templates.json
   jq -r '.component_templates[].name' work/717-component-templates.json | head -n 20
   ```

   Para cada plantilla de laboratorio, aplique el cuerpo correspondiente. Sustituya `NOMBRE_PLANTILLA` por el nombre real documentado:

   ```bash
   jq '.index_templates[] | select(.name == "NOMBRE_PLANTILLA") | .index_template' \
     work/717-index-templates.json > work/template-body.json

   api819 -X PUT \
     "http://localhost:9202/_index_template/NOMBRE_PLANTILLA" \
     --data-binary @work/template-body.json
   ```

9. Cree un rol de mínimo privilegio de laboratorio para validar la autorización en 8.19.

   ```bash
   cat > work/role-logs-ops-reader.json <<'EOF'
   {
     "cluster": ["monitor"],
     "indices": [
       {
         "names": ["logs-app-lab-v1", "logs-ops-lab", ".ds-logs-ops-lab-*"],
         "privileges": ["read", "view_index_metadata"]
       }
     ],
     "applications": [],
     "run_as": [],
     "metadata": {
       "lab": "elastic-labs",
       "purpose": "validacion-lectura-post-upgrade"
     }
   }
   EOF

   api819 -X PUT \
     "http://localhost:9202/_security/role/logs_ops_lab_reader" \
     --data-binary @work/role-logs-ops-reader.json \
     | tee work/819-role-create.json
   ```

10. Importe los objetos guardados de Kibana 7.17 en Kibana 8.19.

    ```bash
    curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
      -X POST "http://localhost:5603/api/saved_objects/_import?overwrite=true" \
      -H 'kbn-xsrf: true' \
      --form file=@work/kibana-717-saved-objects.ndjson \
      | tee work/819-kibana-import-result.json
    ```

11. Abra `http://localhost:5603` y compruebe en **Discover** que puede seleccionar el data view asociado a `logs-ops-lab` o crear uno con el patrón:

    ```text
    logs-ops-lab*
    ```

**Resultado esperado:**

- Elasticsearch 8.19 responde en el puerto `9202`.
- El snapshot de 7.17 es visible y se restaura sin errores.
- Existen `logs-app-lab-v1` y `logs-ops-lab`.
- La política `logs-ops-ilm-v1` existe en 8.19.
- Los objetos guardados se importan con `success: true` o con conflictos conocidos y documentados.

**Verificación:**

```bash
api819 "http://localhost:9202/_cluster/health?pretty"
api819 "http://localhost:9202/_data_stream/logs-ops-lab?pretty"
api819 "http://localhost:9202/logs-app-lab-v1/_count?pretty"
api819 "http://localhost:9202/logs-ops-lab/_count?pretty"
api819 "http://localhost:9202/_ilm/policy/logs-ops-ilm-v1?pretty"
```

Compare los conteos con los archivos:

```bash
jq -r '.count' work/717-count-app-pre.json
jq -r '.count' work/717-count-ops-pre.json
```

---

### Paso 5. Validar 8.19 y crear el snapshot previo a 9.3

**Objetivo:** Verificar que 8.19 es una plataforma intermedia estable y crear el snapshot compatible que se restaurará en Elasticsearch 9.3.

**Instrucciones:**

1. Consulte la salud, los shards y los avisos de deprecación en 8.19.

   ```bash
   api819 "http://localhost:9202/_cluster/health?pretty" \
     | tee work/819-cluster-health-pre.json

   api819 "http://localhost:9202/_cat/shards?v" \
     | tee work/819-shards-pre.txt

   api819 "http://localhost:9202/_migration/deprecations?pretty" \
     | tee work/819-deprecations.json
   ```

2. Ejecute una consulta Query DSL equivalente a una consulta operativa de logs.

   ```bash
   cat > work/query-errors-dsl.json <<'EOF'
   {
     "size": 0,
     "query": {
       "bool": {
         "filter": [
           {
             "range": {
               "@timestamp": {
                 "gte": "now-7d"
               }
             }
           }
         ],
         "should": [
           { "match": { "log.level": "ERROR" } },
           { "match": { "message": "error" } }
         ],
         "minimum_should_match": 1
       }
     },
     "aggs": {
       "por_servicio": {
         "terms": {
           "field": "service.name.keyword",
           "size": 10
         }
       }
     }
   }
   EOF

   api819 -X POST \
     "http://localhost:9202/logs-ops-lab/_search?pretty" \
     --data-binary @work/query-errors-dsl.json \
     | tee work/819-query-errors-dsl.json
   ```

3. En Kibana 8.19, en **Discover**, ejecute una consulta KQL equivalente. Si el campo existe en sus datos, use:

   ```text
   log.level : "ERROR" or message : *error*
   ```

   Guarde una captura o registre el número de resultados en:

   ```bash
   nano work/819-kql-validation.md
   ```

4. En **Dev Tools** de Kibana 8.19, ejecute una consulta ES|QL:

   ```esql
   FROM logs-ops-lab
   | STATS eventos = COUNT(*) BY service.name, log.level
   | SORT eventos DESC
   | LIMIT 20
   ```

5. Cree un API key de lectura mínima para probar las autorizaciones posteriores a la migración. No almacene el secreto completo en el informe.

   ```bash
   api819 -X POST "http://localhost:9202/_security/api_key" \
     -d '{
       "name": "lab-819-read-validation",
       "expiration": "1d",
       "role_descriptors": {
         "lab_logs_reader": {
           "cluster": ["monitor"],
           "index": [
             {
               "names": ["logs-app-lab-v1", "logs-ops-lab", ".ds-logs-ops-lab-*"],
               "privileges": ["read", "view_index_metadata"]
             }
           ]
         }
       }
     }' | tee work/819-api-key-response.json
   ```

6. Pruebe el API key y registre sólo su identificador.

   ```bash
   API_KEY_819=$(jq -r '.encoded' work/819-api-key-response.json)

   curl -sS \
     -H "Authorization: ApiKey ${API_KEY_819}" \
     "http://localhost:9202/logs-ops-lab/_count?pretty" \
     | tee work/819-api-key-count-test.json

   jq '{id, name, expiration}' work/819-api-key-response.json \
     | tee work/819-api-key-evidence.json
   ```

7. Registre el repositorio de snapshots de 8.19.

   ```bash
   api819 -X PUT "http://localhost:9202/_snapshot/fs_lab_repo_819" \
     -d '{
       "type": "fs",
       "settings": {
         "location": "/usr/share/elasticsearch/snapshots/819",
         "compress": true
       }
     }' | tee work/819-repository-819-create.json
   ```

8. Cree el snapshot de preactualización para 9.3.

   ```bash
   api819 -X PUT \
     "http://localhost:9202/_snapshot/fs_lab_repo_819/preupgrade-819-001?wait_for_completion=true" \
     -d '{
       "indices": "logs-app-lab-v1,logs-ops-lab",
       "ignore_unavailable": false,
       "include_global_state": false,
       "partial": false
     }' | tee work/819-snapshot-preupgrade.json

   api819 "http://localhost:9202/_snapshot/fs_lab_repo_819/preupgrade-819-001?pretty" \
     | tee work/819-snapshot-detail.json
   ```

9. Exporte los objetos guardados ya migrados por Kibana 8.19. Este archivo será el origen controlado para Kibana 9.3.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X POST "http://localhost:5603/api/saved_objects/_export" \
     -H 'kbn-xsrf: true' \
     -H 'Content-Type: application/json' \
     -d '{
       "type": ["dashboard", "visualization", "search", "index-pattern"],
       "includeReferencesDeep": true
     }' \
     -o work/kibana-819-saved-objects.ndjson
   ```

**Resultado esperado:**

- La salud de 8.19 es `green`.
- Las consultas DSL, KQL y ES|QL devuelven resultados coherentes con los datos restaurados.
- El API key permite leer los índices autorizados.
- `preupgrade-819-001` tiene estado `SUCCESS`.
- Existe el archivo `work/kibana-819-saved-objects.ndjson`.

**Verificación:**

```bash
jq -r '.snapshots[0].state' work/819-snapshot-detail.json
jq -r '.count' work/819-api-key-count-test.json
wc -l work/kibana-819-saved-objects.ndjson
```

> **Punto de decisión:** No avance a 9.3 si existen errores funcionales relevantes en 8.19, deprecaciones críticas sin resolver, problemas de autenticación, ILM inoperante o diferencias injustificadas en los conteos de documentos.

---

### Paso 6. Restaurar en Elasticsearch 9.3 y validar la plataforma objetivo

**Objetivo:** Restaurar el snapshot compatible de 8.19 en Elasticsearch 9.3 y validar datos, ILM, seguridad y consultas operativas en la plataforma objetivo.

**Instrucciones:**

1. Arranque el entorno Elasticsearch y Kibana 9.3.

   ```bash
   cd /opt/elastic-labs
   docker compose up -d es930-lab kibana930-lab
   ```

2. Espere a que Elasticsearch 9.3 responda.

   ```bash
   until curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9201/_cluster/health | jq -e '.status' >/dev/null; do
     echo "Esperando Elasticsearch 9.3..."
     sleep 5
   done
   ```

3. Configure la contraseña de `kibana_system` y reinicie Kibana 9.3.

   ```bash
   api930 -X POST \
     "http://localhost:9201/_security/user/kibana_system/_password" \
     -d "{\"password\":\"${KIBANA_SYSTEM_PASSWORD}\"}" | jq

   docker compose restart kibana930-lab
   ```

4. Registre el repositorio que contiene el snapshot creado desde 8.19.

   ```bash
   api930 -X PUT "http://localhost:9201/_snapshot/fs_lab_repo_819" \
     -d '{
       "type": "fs",
       "settings": {
         "location": "/usr/share/elasticsearch/snapshots/819",
         "compress": true
       }
     }' | tee work/930-repository-create.json

   api930 -X POST "http://localhost:9201/_snapshot/fs_lab_repo_819/_verify?pretty" \
     | tee work/930-repository-verify.json
   ```

5. Compruebe que el snapshot `preupgrade-819-001` sea visible.

   ```bash
   api930 "http://localhost:9201/_snapshot/fs_lab_repo_819/preupgrade-819-001?pretty" \
     | tee work/930-visible-819-snapshot.json
   ```

6. Restaure los datos de laboratorio y ajuste las réplicas para un nodo único.

   ```bash
   api930 -X POST \
     "http://localhost:9201/_snapshot/fs_lab_repo_819/preupgrade-819-001/_restore?wait_for_completion=true" \
     -d '{
       "indices": "logs-app-lab-v1,logs-ops-lab",
       "ignore_unavailable": false,
       "include_global_state": false,
       "include_aliases": true,
       "index_settings": {
         "index.number_of_replicas": 0
       }
     }' | tee work/930-restore-819.json
   ```

7. Reinstale la política ILM en 9.3.

   ```bash
   api930 -X PUT \
     "http://localhost:9201/_ilm/policy/logs-ops-ilm-v1" \
     --data-binary @work/logs-ops-ilm-v1-policy-body.json \
     | tee work/930-ilm-policy-put.json
   ```

8. Reinstale las plantillas de laboratorio requeridas, si fueron utilizadas en los ejercicios anteriores, aplicando el mismo procedimiento validado en 8.19.

9. Cree el rol de lectura mínima en 9.3.

   ```bash
   api930 -X PUT \
     "http://localhost:9201/_security/role/logs_ops_lab_reader" \
     --data-binary @work/role-logs-ops-reader.json \
     | tee work/930-role-create.json
   ```

10. Cree y pruebe un API key de lectura en 9.3.

    ```bash
    api930 -X POST "http://localhost:9201/_security/api_key" \
      -d '{
        "name": "lab-930-read-validation",
        "expiration": "1d",
        "role_descriptors": {
          "lab_logs_reader": {
            "cluster": ["monitor"],
            "index": [
              {
                "names": ["logs-app-lab-v1", "logs-ops-lab", ".ds-logs-ops-lab-*"],
                "privileges": ["read", "view_index_metadata"]
              }
            ]
          }
        }
      }' | tee work/930-api-key-response.json

    API_KEY_930=$(jq -r '.encoded' work/930-api-key-response.json)

    curl -sS \
      -H "Authorization: ApiKey ${API_KEY_930}" \
      "http://localhost:9201/logs-ops-lab/_count?pretty" \
      | tee work/930-api-key-count-test.json
    ```

11. Importe los objetos guardados exportados desde Kibana 8.19.

    ```bash
    curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
      -X POST "http://localhost:5602/api/saved_objects/_import?overwrite=true" \
      -H 'kbn-xsrf: true' \
      --form file=@work/kibana-819-saved-objects.ndjson \
      | tee work/930-kibana-import-result.json
    ```

12. Abra Kibana 9.3 en `http://localhost:5602`. En **Discover**, cree o seleccione el data view para:

    ```text
    logs-ops-lab*
    ```

13. Ejecute la siguiente consulta KQL:

    ```text
    log.level : "ERROR" or message : *error*
    ```

14. En **Dev Tools**, ejecute la consulta ES|QL equivalente:

    ```esql
    FROM logs-ops-lab
    | STATS eventos = COUNT(*) BY service.name, log.level
    | SORT eventos DESC
    | LIMIT 20
    ```

**Resultado esperado:**

- Elasticsearch 9.3 responde en `http://localhost:9201`.
- La restauración finaliza correctamente.
- La salud del clúster es `green`.
- `logs-app-lab-v1`, `logs-ops-lab` y la política `logs-ops-ilm-v1` están disponibles.
- El API key permite consultar los datos autorizados.
- Kibana 9.3 contiene los objetos migrados y puede ejecutar consultas sobre los datos restaurados.

**Verificación:**

```bash
api930 "http://localhost:9201/_cluster/health?pretty" \
  | tee work/930-cluster-health-final.json

api930 "http://localhost:9201/_cat/indices?v&expand_wildcards=all" \
  | tee work/930-indices-final.txt

api930 "http://localhost:9201/_data_stream/logs-ops-lab?pretty" \
  | tee work/930-data-stream-final.json

api930 "http://localhost:9201/logs-app-lab-v1/_count?pretty" \
  | tee work/930-count-app-final.json

api930 "http://localhost:9201/logs-ops-lab/_count?pretty" \
  | tee work/930-count-ops-final.json

api930 "http://localhost:9201/logs-ops-lab/_ilm/explain?pretty" \
  | tee work/930-ilm-explain-final.json
```

---

### Paso 7. Elaborar el runbook de actualización y rollback

**Objetivo:** Consolidar las evidencias técnicas en un procedimiento reutilizable para la transición controlada a la plataforma 9.3.

**Instrucciones:**

1. Cree el documento de runbook.

   ```bash
   cat > work/runbook-upgrade-717-819-930.md <<'EOF'
   # Runbook de actualización Elasticsearch 7.17.29 → 8.19.0 → 9.3.0

   ## Alcance
   Migración paralela de índices y data streams de laboratorio mediante snapshots.
   Los objetos de Kibana se migran por exportación e importación controlada.

   ## Punto de partida
   - Clúster origen: es717-lab-cluster
   - Versión origen: 7.17.29
   - Snapshot origen: preupgrade-717-001
   - Clúster intermedio: es819-lab-cluster
   - Snapshot intermedio: preupgrade-819-001
   - Clúster destino: es930-lab-cluster

   ## Controles previos obligatorios
   1. Estado del clúster green.
   2. Sin shards UNASSIGNED.
   3. Espacio en disco suficiente para imágenes, volúmenes y snapshots.
   4. Índices incompatibles reindexados antes del salto mayor.
   5. Política ILM, templates, roles y consultas críticas inventariados.
   6. Snapshot SUCCESS verificado y visible desde el clúster de destino.
   7. Objetos guardados de Kibana exportados.

   ## Ruta de ejecución
   1. Validar 7.17 y crear preupgrade-717-001.
   2. Restaurar datos en 8.19.
   3. Recrear ILM, templates y controles de seguridad en 8.19.
   4. Importar objetos Kibana 7.17 en Kibana 8.19.
   5. Validar consultas DSL, KQL y ES|QL.
   6. Crear preupgrade-819-001.
   7. Restaurar datos en 9.3.
   8. Recrear ILM, templates, roles y API keys en 9.3.
   9. Importar objetos Kibana 8.19 en Kibana 9.3.
   10. Ejecutar validación funcional y operativa final.

   ## Criterios de aceptación
   - Estado green en cada plataforma.
   - Conteos de documentos equivalentes entre origen y destino.
   - Data stream logs-ops-lab visible y consultable.
   - Política logs-ops-ilm-v1 existente y sin errores de explain.
   - Consultas DSL, KQL y ES|QL ejecutadas correctamente.
   - API key de mínimo privilegio validada.
   - Objetos guardados importados o conflictos documentados y resueltos.

   ## Criterios de rollback
   - Snapshot no SUCCESS.
   - Conteos de documentos incoherentes.
   - Error crítico de seguridad, ILM o acceso a datos.
   - Falla de una consulta operativa crítica.
   - Error de compatibilidad no resuelto en 8.19 o 9.3.

   ## Procedimiento de rollback
   1. Detener la promoción del nuevo entorno.
   2. Mantener el clúster anterior intacto y disponible.
   3. No restaurar snapshots de una versión posterior sobre una versión mayor anterior.
   4. Restaurar el último snapshot compatible en un clúster de recuperación compatible.
   5. Registrar causa, evidencias, impacto y acción correctiva antes de reintentar.
   EOF
   ```

2. Añada los conteos y resultados reales obtenidos durante la práctica.

   ```bash
   {
     echo
     echo "## Evidencias de conteo"
     echo
     echo "### Elasticsearch 7.17"
     jq '{indice: "logs-app-lab-v1", count}' work/717-count-app-pre.json
     jq '{indice: "logs-ops-lab", count}' work/717-count-ops-pre.json
     echo
     echo "### Elasticsearch 9.3"
     jq '{indice: "logs-app-lab-v1", count}' work/930-count-app-final.json
     jq '{indice: "logs-ops-lab", count}' work/930-count-ops-final.json
   } >> work/runbook-upgrade-717-819-930.md
   ```

3. Revise el documento y confirme que contiene snapshots, validaciones, decisiones y rollback.

   ```bash
   less work/runbook-upgrade-717-819-930.md
   ```

**Resultado esperado:**

- Existe un runbook reutilizable en `work/runbook-upgrade-717-819-930.md`.
- El documento diferencia claramente:
  - Datos Elasticsearch: snapshots y restauración.
  - Objetos Kibana: exportación e importación.
  - Configuración de seguridad, ILM y templates: recreación o validación explícita.

**Verificación:**

```bash
grep -E "preupgrade-717-001|preupgrade-819-001|rollback|Kibana|ILM" \
  work/runbook-upgrade-717-819-930.md
```

---

## Validación y pruebas

Ejecute la siguiente comparación final de conteos:

```bash
printf "%-25s %-12s %-12s\n" "Índice o data stream" "7.17" "9.3"
printf "%-25s %-12s %-12s\n" "logs-app-lab-v1" \
  "$(jq -r '.count' work/717-count-app-pre.json)" \
  "$(jq -r '.count' work/930-count-app-final.json)"
printf "%-25s %-12s %-12s\n" "logs-ops-lab" \
  "$(jq -r '.count' work/717-count-ops-pre.json)" \
  "$(jq -r '.count' work/930-count-ops-final.json)"
```

Valide los criterios técnicos finales:

| Validación | Comando o prueba | Resultado esperado |
|---|---|---|
| Salud 9.3 | `jq -r '.status' work/930-cluster-health-final.json` | `green` |
| Shards | `api930 "http://localhost:9201/_cat/shards?h=state" \| sort \| uniq -c` | Sin `UNASSIGNED` |
| Snapshot 7.17 | `jq -r '.snapshots[0].state' work/717-snapshot-detail.json` | `SUCCESS` |
| Snapshot 8.19 | `jq -r '.snapshots[0].state' work/819-snapshot-detail.json` | `SUCCESS` |
| Data stream | `api930 "http://localhost:9201/_data_stream/logs-ops-lab"` | Visible |
| ILM | `api930 "http://localhost:9201/_ilm/policy/logs-ops-ilm-v1"` | Política disponible |
| Seguridad | API key en `work/930-api-key-count-test.json` | Conteo permitido |
| Kibana 9.3 | Discover, KQL y ES\|QL | Consultas ejecutadas |
| Objetos guardados | `work/930-kibana-import-result.json` | Importación correcta o conflictos documentados |

Ejecute una prueba DSL final en 9.3:

```bash
api930 -X POST \
  "http://localhost:9201/logs-ops-lab/_search?pretty" \
  --data-binary @work/query-errors-dsl.json \
  | tee work/930-query-errors-dsl.json
```

La migración se considera aceptada si los conteos son coherentes, los índices y data streams están disponibles, ILM funciona, la seguridad autoriza sólo las operaciones previstas y las consultas operativas se ejecutan en Kibana 9.3.

## Resolución de problemas

### Problema 1: El repositorio `fs_lab_repo` no se verifica o muestra `repository_exception`

**Síntomas:**

- La llamada `POST /_snapshot/fs_lab_repo/_verify` falla.
- Elasticsearch informa que la ubicación no es accesible o que `path.repo` no permite el directorio.
- El snapshot no puede crearse o no es visible desde 8.19 o 9.3.

**Causa:**

El directorio del host no está montado en el contenedor, tiene permisos insuficientes o la configuración `path.repo` no coincide con la ruta interna del contenedor.

**Solución:**

1. Compruebe la existencia y permisos de los directorios en el host:

   ```bash
   ls -ld /opt/elastic-labs/snapshots \
          /opt/elastic-labs/snapshots/717 \
          /opt/elastic-labs/snapshots/819
   ```

2. Ajuste permisos para el entorno de laboratorio:

   ```bash
   chmod -R 0777 /opt/elastic-labs/snapshots
   ```

3. Confirme el montaje y `path.repo` dentro del contenedor:

   ```bash
   docker exec es717-lab ls -ld /usr/share/elasticsearch/snapshots/717
   docker exec es717-lab \
     curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200/_nodes/settings?filter_path=nodes.*.settings.path.repo
   ```

4. Reinicie el contenedor afectado y vuelva a registrar el repositorio:

   ```bash
   cd /opt/elastic-labs
   docker compose restart es717-lab
   ```

### Problema 2: La restauración termina, pero el clúster queda `yellow` o el data stream no aparece

**Síntomas:**

- La restauración responde sin error, pero `/_cluster/health` muestra `yellow`.
- Los backing indices de `.ds-logs-ops-lab-*` existen, pero `GET /_data_stream/logs-ops-lab` falla.
- ILM o la plantilla del data stream no están presentes.

**Causa:**

En un clúster de un solo nodo pueden existir réplicas configuradas en `1`. Además, el snapshot se restauró con `include_global_state: false`, por lo que políticas ILM, templates y otros elementos de configuración deben recrearse por separado.

**Solución:**

1. Ajuste las réplicas de los índices restaurados a cero:

   ```bash
   api930 -X PUT \
     "http://localhost:9201/logs-app-lab-v1,.ds-logs-ops-lab-*/_settings" \
     -d '{
       "index.number_of_replicas": 0
     }'
   ```

2. Reinstale la política ILM:

   ```bash
   api930 -X PUT \
     "http://localhost:9201/_ilm/policy/logs-ops-ilm-v1" \
     --data-binary @work/logs-ops-ilm-v1-policy-body.json
   ```

3. Reinstale las plantillas componibles del data stream desde los archivos exportados.

4. Compruebe el estado de ILM y la salud:

   ```bash
   api930 "http://localhost:9201/logs-ops-lab/_ilm/explain?pretty"
   api930 "http://localhost:9201/_cluster/health?pretty"
   ```

## Limpieza

La práctica deja los clústeres y volúmenes persistentes disponibles para la Práctica 10. No elimine los volúmenes.

1. Detenga únicamente el entorno temporal 8.19 después de confirmar que el snapshot `preupgrade-819-001` es `SUCCESS`.

   ```bash
   cd /opt/elastic-labs
   docker compose --profile upgrade819 stop es819-lab kibana819-lab
   ```

2. Mantenga en ejecución el entorno objetivo 9.3 para la siguiente práctica.

   ```bash
   docker compose ps
   ```

3. Opcionalmente, archive las evidencias.

   ```bash
   tar -czf /opt/elastic-labs/work/lab-09-evidencias.tar.gz \
     -C /opt/elastic-labs/work \
     .
   ```

4. Si necesita detener todos los servicios al finalizar la sesión, use:

   ```bash
   docker compose stop
   ```

> **No use:** `docker compose down -v`  
> Esta operación eliminaría los volúmenes persistentes `es717-data`, `es930-data`, `kibana717-data` y `kibana930-data`.

## Resumen

En esta práctica se ejecutó una ruta de actualización compatible desde Elasticsearch 7.17.29 hacia 8.19.0 y finalmente hacia 9.3.0. Se verificó la estabilidad del entorno origen, se crearon snapshots antes de cada salto mayor, se restauraron datos de laboratorio en clústeres paralelos y se validaron índices, data streams, ILM, roles, API keys y consultas operativas.

El punto clave es la separación de responsabilidades: los snapshots migran datos de Elasticsearch, mientras que los objetos guardados de Kibana requieren exportación e importación controlada. El clúster 9.3 y el runbook generado constituyen la base operativa para la Práctica 10.

### Recursos opcionales

- Elasticsearch Snapshot and Restore: `https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html`
- Elasticsearch Upgrade Guide: `https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html`
- Kibana Saved Objects API: `https://www.elastic.co/docs/api/doc/kibana/`
- Elasticsearch ILM: `https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html`
