# Analizar la arquitectura, los roles de nodos y el estado de un deployment de Elasticsearch.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 60 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica se desplegarán dos clústeres aislados de Elasticsearch mediante Docker Compose: Elasticsearch 7.17.29 y Elasticsearch 9.3.0, cada uno con su instancia de Kibana. Se inspeccionarán los componentes de la arquitectura distribuida, los roles de nodo, la salud del clúster, la asignación de *shards* y el comportamiento de las réplicas en un diseño de nodo único.

El clúster Elasticsearch 9.3.0 quedará como entorno principal para las prácticas posteriores de generación, consulta y análisis de datos. Elasticsearch 7.17.29 permanecerá disponible para comparaciones y para la práctica de actualización.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Levantar y validar los deployments de Elasticsearch 7.17.29 y 9.3.0 con Kibana mediante Docker Compose.
- [ ] Identificar clústeres, nodos, índices, *shards* primarios, réplicas, alias, *data streams* y Kibana.
- [ ] Consultar la salud, los roles de nodo, la configuración de descubrimiento y la asignación de *shards* mediante APIs de Elasticsearch.
- [ ] Explicar por qué un clúster de un solo nodo presenta estado `yellow` cuando existen réplicas configuradas.
- [ ] Relacionar el diseño de laboratorio con patrones de producción que separan roles de maestro, datos, ingesta y coordinación.

## Requisitos previos

### Conocimientos

- Uso básico de terminal Linux y edición de archivos YAML.
- Conceptos básicos de Docker, Docker Compose, JSON y APIs REST.
- Comprensión inicial de los conceptos de clúster, nodo, índice, documento, *shard* primario y réplica.
- Uso básico de `curl` y `jq`.

### Acceso y software

| Requisito | Valor esperado |
|---|---|
| Sistema operativo | Ubuntu Desktop 24.04.2 LTS o Ubuntu Server 22.04.5 LTS |
| Docker Engine | 27.5.1 o compatible |
| Docker Compose plugin | 2.32.4 o compatible |
| Memoria disponible | 16 GB mínimo; 24 GB recomendado |
| CPU disponible | 4 vCPU mínimo; 6 vCPU recomendado |
| Disco SSD libre | 40 GB mínimo; 60 GB recomendado |
| Puertos disponibles | 9200, 9201, 5601 y 5602 |
| Directorio de trabajo | `/opt/elastic-labs` |
| Parámetro del kernel | `vm.max_map_count=262144` |
| Acceso a Internet | Necesario para descargar imágenes desde Elastic Docker Registry |

> **Importante:** Esta práctica utiliza credenciales exclusivas de laboratorio. No reutilice `ElasticLab_2026!` fuera del entorno de formación.

---

## Entorno de laboratorio

La práctica crea dos stacks aislados en la misma red Docker:

| Componente | Versión | Contenedor | Puerto de host | Clúster |
|---|---:|---|---:|---|
| Elasticsearch | 7.17.29 | `es717-lab` | `9200` | `es717-lab-cluster` |
| Kibana | 7.17.29 | `kibana717-lab` | `5601` | Asociado a `es717-lab-cluster` |
| Elasticsearch | 9.3.0 | `es930-lab` | `9201` | `es930-lab-cluster` |
| Kibana | 9.3.0 | `kibana930-lab` | `5602` | Asociado a `es930-lab-cluster` |

Los volúmenes persistentes obligatorios son:

- `es717-data`
- `es930-data`
- `kibana717-data`
- `kibana930-data`

La red Docker obligatoria es `elastic-lab-net`.

> **Continuidad del curso:** No ejecute `docker compose down -v`. La opción `-v` eliminaría los volúmenes persistentes y destruiría el estado que se reutilizará en prácticas posteriores.

---

## Procedimiento paso a paso

### Paso 1. Preparar el directorio de trabajo y validar el host

**Objetivo:** Crear la estructura de directorios obligatoria y confirmar que Docker, puertos y parámetros del kernel cumplen los requisitos.

**Instrucciones:**

1. Abra una terminal y cree el directorio raíz del laboratorio con el directorio de evidencias.

   ```bash
   sudo mkdir -p /opt/elastic-labs/work
   sudo chown -R "$USER":"$USER" /opt/elastic-labs
   cd /opt/elastic-labs
   ```

2. Verifique las versiones de Docker y Docker Compose.

   ```bash
   docker --version
   docker compose version
   ```

3. Verifique el valor de `vm.max_map_count`.

   ```bash
   sysctl vm.max_map_count
   ```

4. Si el valor es inferior a `262144`, configúrelo temporalmente.

   ```bash
   sudo sysctl -w vm.max_map_count=262144
   ```

5. Para conservar la configuración después de reiniciar el host Linux, cree un archivo de configuración del sistema.

   ```bash
   echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-elastic-labs.conf
   sudo sysctl --system
   ```

6. Verifique que los puertos requeridos no estén ocupados.

   ```bash
   ss -ltn | grep -E ':(9200|9201|5601|5602)\s' || true
   ```

7. Cree los archivos iniciales que se utilizarán durante el laboratorio.

   ```bash
   touch /opt/elastic-labs/work/comandos-ejecutados.txt
   touch /opt/elastic-labs/work/observaciones-arquitectura.md
   ```

**Salida esperada:**

- Docker y Docker Compose muestran una versión instalada.
- `vm.max_map_count` muestra `262144`.
- El comando `ss` no muestra procesos utilizando los puertos requeridos, o solamente muestra servicios que usted ya haya identificado y detenido.
- Existe la ruta `/opt/elastic-labs/work`.

**Verificación:**

```bash
ls -ld /opt/elastic-labs /opt/elastic-labs/work
sysctl vm.max_map_count
```

El resultado debe confirmar que el usuario actual puede escribir en el directorio del laboratorio y que el parámetro del kernel tiene el valor requerido.

---

### Paso 2. Crear el archivo de credenciales y el archivo Docker Compose

**Objetivo:** Definir las credenciales de laboratorio con permisos restrictivos y crear la configuración reproducible de los dos deployments.

**Instrucciones:**

1. Cree el archivo `/opt/elastic-labs/.env`.

   ```bash
   cat > /opt/elastic-labs/.env <<'EOF'
   ELASTIC_PASSWORD=ElasticLab_2026!
   EOF
   ```

2. Restrinja los permisos del archivo de credenciales.

   ```bash
   chmod 0600 /opt/elastic-labs/.env
   ls -l /opt/elastic-labs/.env
   ```

3. Cree el archivo obligatorio `/opt/elastic-labs/compose.yaml`.

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
         - bootstrap.memory_lock=true
         - ES_JAVA_OPTS=-Xms1g -Xmx1g
         - xpack.security.enabled=true
         - xpack.security.http.ssl.enabled=false
         - xpack.security.transport.ssl.enabled=false
         - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
       ulimits:
         memlock:
           soft: -1
           hard: -1
       ports:
         - "9200:9200"
       volumes:
         - es717-data:/usr/share/elasticsearch/data
       networks:
         - elastic-lab-net
       healthcheck:
         test: ["CMD-SHELL", "curl -fsu elastic:$${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health >/dev/null"]
         interval: 15s
         timeout: 10s
         retries: 20

     kibana717-lab:
       image: docker.elastic.co/kibana/kibana:7.17.29
       container_name: kibana717-lab
       depends_on:
         es717-lab:
           condition: service_healthy
       environment:
         - ELASTICSEARCH_HOSTS=http://es717-lab:9200
         - ELASTICSEARCH_USERNAME=kibana_system
         - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
         - SERVER_NAME=kibana717-lab
       entrypoint: ["/bin/sh", "-ec"]
       command:
         - |
           until curl -fsu elastic:"$${ELASTIC_PASSWORD}" http://es717-lab:9200/_cluster/health >/dev/null; do sleep 3; done
           curl -fsS -X POST -u elastic:"$${ELASTIC_PASSWORD}" \
             -H 'Content-Type: application/json' \
             http://es717-lab:9200/_security/user/kibana_system/_password \
             -d "{\"password\":\"$${ELASTIC_PASSWORD}\"}"
           exec /usr/local/bin/kibana-docker
       ports:
         - "5601:5601"
       volumes:
         - kibana717-data:/usr/share/kibana/data
       networks:
         - elastic-lab-net

     es930-lab:
       image: docker.elastic.co/elasticsearch/elasticsearch:9.3.0
       container_name: es930-lab
       environment:
         - node.name=es930-node-1
         - cluster.name=es930-lab-cluster
         - discovery.type=single-node
         - bootstrap.memory_lock=true
         - ES_JAVA_OPTS=-Xms1g -Xmx1g
         - xpack.security.enabled=true
         - xpack.security.http.ssl.enabled=false
         - xpack.security.transport.ssl.enabled=false
         - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
       ulimits:
         memlock:
           soft: -1
           hard: -1
       ports:
         - "9201:9200"
       volumes:
         - es930-data:/usr/share/elasticsearch/data
       networks:
         - elastic-lab-net
       healthcheck:
         test: ["CMD-SHELL", "curl -fsu elastic:$${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health >/dev/null"]
         interval: 15s
         timeout: 10s
         retries: 20

     kibana930-lab:
       image: docker.elastic.co/kibana/kibana:9.3.0
       container_name: kibana930-lab
       depends_on:
         es930-lab:
           condition: service_healthy
       environment:
         - ELASTICSEARCH_HOSTS=http://es930-lab:9200
         - ELASTICSEARCH_USERNAME=kibana_system
         - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
         - SERVER_NAME=kibana930-lab
       entrypoint: ["/bin/sh", "-ec"]
       command:
         - |
           until curl -fsu elastic:"$${ELASTIC_PASSWORD}" http://es930-lab:9200/_cluster/health >/dev/null; do sleep 3; done
           curl -fsS -X POST -u elastic:"$${ELASTIC_PASSWORD}" \
             -H 'Content-Type: application/json' \
             http://es930-lab:9200/_security/user/kibana_system/_password \
             -d "{\"password\":\"$${ELASTIC_PASSWORD}\"}"
           exec /usr/local/bin/kibana-docker
       ports:
         - "5602:5601"
       volumes:
         - kibana930-data:/usr/share/kibana/data
       networks:
         - elastic-lab-net

   networks:
     elastic-lab-net:
       name: elastic-lab-net
       driver: bridge

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

4. Valide la sintaxis y la interpolación de variables del archivo Compose.

   ```bash
   docker compose --env-file /opt/elastic-labs/.env config > /opt/elastic-labs/work/compose-validado.yaml
   ```

5. Revise que no se hayan publicado puertos distintos a los requeridos.

   ```bash
   grep -nE '9200:9200|9201:9200|5601:5601|5602:5601' /opt/elastic-labs/work/compose-validado.yaml
   ```

**Salida esperada:**

- El archivo `.env` tiene permisos `-rw-------`.
- `docker compose config` finaliza sin errores de sintaxis.
- El archivo validado contiene los cuatro servicios, la red `elastic-lab-net` y los cuatro volúmenes persistentes.

**Verificación:**

```bash
cd /opt/elastic-labs
docker compose --env-file .env config --services
```

Debe mostrar:

```text
es717-lab
kibana717-lab
es930-lab
kibana930-lab
```

---

### Paso 3. Iniciar los dos deployments y validar la conectividad básica

**Objetivo:** Descargar las imágenes, iniciar los contenedores y confirmar que ambos clústeres responden por HTTP autenticado.

**Instrucciones:**

1. Sitúese en el directorio de laboratorio.

   ```bash
   cd /opt/elastic-labs
   ```

2. Inicie los servicios en segundo plano.

   ```bash
   docker compose --env-file .env up -d
   ```

3. Consulte el estado de los servicios.

   ```bash
   docker compose ps
   ```

4. Espere hasta que Elasticsearch muestre estado `healthy`. Si Kibana tarda más tiempo en iniciar, espere uno o dos minutos adicionales.

   ```bash
   watch -n 5 'docker compose ps'
   ```

   Para salir de `watch`, pulse `Ctrl+C`.

5. Cargue la variable de contraseña en la sesión actual.

   ```bash
   set -a
   . /opt/elastic-labs/.env
   set +a
   ```

6. Consulte la información principal de Elasticsearch 7.17.29.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200/ | jq .
   ```

7. Consulte la información principal de Elasticsearch 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9201/ | jq .
   ```

8. Guarde evidencias de las respuestas.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200/ | jq . \
     > /opt/elastic-labs/work/es717-root.json

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9201/ | jq . \
     > /opt/elastic-labs/work/es930-root.json
   ```

9. Abra Kibana en un navegador:

   - Kibana 7.17.29: <http://localhost:5601>
   - Kibana 9.3.0: <http://localhost:5602>

   Inicie sesión con:

   ```text
   Usuario: elastic
   Contraseña: ElasticLab_2026!
   ```

**Salida esperada:**

La respuesta de cada endpoint raíz contiene, entre otros campos:

```json
{
  "name": "es717-node-1",
  "cluster_name": "es717-lab-cluster",
  "version": {
    "number": "7.17.29"
  }
}
```

Y para el segundo clúster:

```json
{
  "name": "es930-node-1",
  "cluster_name": "es930-lab-cluster",
  "version": {
    "number": "9.3.0"
  }
}
```

**Verificación:**

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Deben aparecer los contenedores `es717-lab`, `kibana717-lab`, `es930-lab` y `kibana930-lab`. Los contenedores de Elasticsearch deben indicar estado `healthy` después de completar su inicialización.

---

### Paso 4. Inspeccionar identidad, salud y roles de los nodos

**Objetivo:** Obtener evidencia directa de la identidad de ambos clústeres, de sus nodos y de sus roles efectivos.

**Instrucciones:**

1. Consulte la salud de Elasticsearch 7.17.29 y guarde la evidencia.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cluster/health?pretty" \
     | tee /opt/elastic-labs/work/es717-cluster-health.json
   ```

2. Consulte la salud de Elasticsearch 9.3.0 y guarde la evidencia.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cluster/health?pretty" \
     | tee /opt/elastic-labs/work/es930-cluster-health.json
   ```

3. Consulte el resumen de nodos del clúster 7.17.29.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cat/nodes?v"
   ```

4. Consulte el resumen de nodos del clúster 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/nodes?v"
   ```

5. Guarde la salida de las Cat APIs en archivos de evidencia.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cat/nodes?v" \
     > /opt/elastic-labs/work/es717-cat-nodes.txt

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/nodes?v" \
     > /opt/elastic-labs/work/es930-cat-nodes.txt
   ```

6. Consulte la Nodes API de cada clúster. La primera petición obtiene la respuesta completa requerida para la práctica; la segunda aplica un filtro para facilitar su interpretación.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_nodes?pretty" \
     > /opt/elastic-labs/work/es717-nodes-completo.json

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_nodes?pretty" \
     > /opt/elastic-labs/work/es930-nodes-completo.json
   ```

7. Extraiga los roles y configuraciones relevantes de descubrimiento.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_nodes/settings,process?filter_path=nodes.*.name,nodes.*.roles,nodes.*.settings.discovery,nodes.*.settings.cluster,nodes.*.process.id&pretty"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_nodes/settings,process?filter_path=nodes.*.name,nodes.*.roles,nodes.*.settings.discovery,nodes.*.settings.cluster,nodes.*.process.id&pretty"
   ```

8. Documente en `/opt/elastic-labs/work/observaciones-arquitectura.md` los roles observados. Utilice una tabla como la siguiente.

   ```bash
   cat >> /opt/elastic-labs/work/observaciones-arquitectura.md <<'EOF'

   ## Roles observados

   | Clúster | Nodo | Roles observados | Interpretación |
   |---|---|---|---|
   | es717-lab-cluster | es717-node-1 | Completar según API | Nodo único con múltiples responsabilidades |
   | es930-lab-cluster | es930-node-1 | Completar según API | Nodo único con múltiples responsabilidades |

   EOF
   ```

**Salida esperada:**

- La salud indica un único nodo (`number_of_nodes: 1`).
- El nombre de cada clúster coincide con el configurado en Compose.
- Los nodos muestran roles combinados, normalmente relacionados con maestro, datos, ingesta y otras capacidades habilitadas por defecto.
- La configuración de descubrimiento incluye `single-node`.

Ejemplo conceptual de una fila de Cat Nodes:

```text
ip         heap.percent ram.percent cpu load_1m node.role master name
172.20.x.x           35          80   2    0.30 dimrstw      * es930-node-1
```

La representación abreviada de roles puede variar según la versión. La Nodes API es la fuente más adecuada para interpretar los nombres completos de los roles.

**Verificación:**

Confirme los nombres de clúster y nodo con los siguientes comandos:

```bash
jq -r '.cluster_name, .name, .version.number' /opt/elastic-labs/work/es717-root.json
jq -r '.cluster_name, .name, .version.number' /opt/elastic-labs/work/es930-root.json
```

Debe identificar:

```text
es717-lab-cluster
es717-node-1
7.17.29
```

y:

```text
es930-lab-cluster
es930-node-1
9.3.0
```

---

### Paso 5. Crear índices, alias y un data stream para observar la arquitectura

**Objetivo:** Crear recursos controlados que permitan distinguir entre índice, *shard* primario, réplica, alias y *data stream*.

**Instrucciones:**

1. Cree un índice de demostración en Elasticsearch 7.17.29 con dos *shards* primarios y una réplica por primario.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X PUT "http://localhost:9200/arquitectura-717" \
     -H 'Content-Type: application/json' \
     -d '{
       "settings": {
         "number_of_shards": 2,
         "number_of_replicas": 1
       },
       "mappings": {
         "properties": {
           "@timestamp": { "type": "date" },
           "mensaje": { "type": "keyword" },
           "origen": { "type": "keyword" }
         }
       }
     }' | jq .
   ```

2. Cree un índice de demostración equivalente en Elasticsearch 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X PUT "http://localhost:9201/arquitectura-930" \
     -H 'Content-Type: application/json' \
     -d '{
       "settings": {
         "number_of_shards": 2,
         "number_of_replicas": 1
       },
       "mappings": {
         "properties": {
           "@timestamp": { "type": "date" },
           "mensaje": { "type": "keyword" },
           "origen": { "type": "keyword" }
         }
       }
     }' | jq .
   ```

3. Cree un alias de lectura para el índice de Elasticsearch 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X POST "http://localhost:9201/_aliases" \
     -H 'Content-Type: application/json' \
     -d '{
       "actions": [
         {
           "add": {
             "index": "arquitectura-930",
             "alias": "arquitectura-actual"
           }
         }
       ]
     }' | jq .
   ```

4. Cree una plantilla de índice que habilite un *data stream* de logs en Elasticsearch 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X PUT "http://localhost:9201/_index_template/logs-lab-template" \
     -H 'Content-Type: application/json' \
     -d '{
       "index_patterns": ["logs-lab-*"],
       "priority": 500,
       "data_stream": {},
       "template": {
         "settings": {
           "number_of_shards": 1,
           "number_of_replicas": 1
         },
         "mappings": {
           "properties": {
             "@timestamp": { "type": "date" },
             "message": { "type": "match_only_text" },
             "service": {
               "properties": {
                 "name": { "type": "keyword" }
               }
             },
             "log": {
               "properties": {
                 "level": { "type": "keyword" }
               }
             }
           }
         }
       }
     }' | jq .
   ```

5. Cree el *data stream*.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X PUT "http://localhost:9201/_data_stream/logs-lab-aplicacion" \
     | jq .
   ```

6. Indexe un documento en el índice de demostración de Elasticsearch 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X POST "http://localhost:9201/arquitectura-930/_doc/1?refresh=true" \
     -H 'Content-Type: application/json' \
     -d '{
       "@timestamp": "2026-08-25T10:00:00Z",
       "mensaje": "Documento de prueba para arquitectura",
       "origen": "laboratorio"
     }' | jq .
   ```

7. Indexe un evento en el *data stream*. Debe incluir el campo obligatorio `@timestamp`.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X POST "http://localhost:9201/logs-lab-aplicacion/_doc?refresh=true" \
     -H 'Content-Type: application/json' \
     -d '{
       "@timestamp": "2026-08-25T10:05:00Z",
       "message": "Servicio iniciado correctamente",
       "service": {
         "name": "api-laboratorio"
       },
       "log": {
         "level": "INFO"
       }
     }' | jq .
   ```

8. Consulte el alias creado.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_alias/arquitectura-actual?pretty"
   ```

9. Consulte el *data stream* y su índice de respaldo.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_data_stream/logs-lab-aplicacion?pretty" \
     | tee /opt/elastic-labs/work/es930-data-stream.json
   ```

10. Liste los índices visibles de ambos clústeres.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cat/indices?v"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/indices?v"
   ```

**Salida esperada:**

- La creación de índices, alias, plantilla y *data stream* devuelve `"acknowledged": true`.
- El alias `arquitectura-actual` apunta a `arquitectura-930`.
- El *data stream* `logs-lab-aplicacion` muestra al menos un índice de respaldo con un nombre similar a:

```text
.ds-logs-lab-aplicacion-2026.08.25-000001
```

- Los índices `arquitectura-717` y `arquitectura-930` tienen dos *shards* primarios y una réplica configurada.

**Verificación:**

Compruebe que la búsqueda a través del alias devuelve el documento indexado:

```bash
curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
  "http://localhost:9201/arquitectura-actual/_search?pretty" \
  | jq '.hits.hits[]._source'
```

Debe aparecer el documento con `mensaje: "Documento de prueba para arquitectura"`.

---

### Paso 6. Analizar la asignación de shards y explicar el estado yellow

**Objetivo:** Inspeccionar la ubicación de *shards* y demostrar por qué las réplicas permanecen sin asignar en un clúster de un solo nodo.

**Instrucciones:**

1. Liste los *shards* del índice de demostración de Elasticsearch 7.17.29.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cat/shards/arquitectura-717?v"
   ```

2. Liste los *shards* del índice, alias y *data stream* de Elasticsearch 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/shards/arquitectura-930?v"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/shards/logs-lab-aplicacion?v"
   ```

3. Guarde una vista completa de la distribución de *shards* del clúster 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/shards?v" \
     > /opt/elastic-labs/work/es930-cat-shards.txt
   ```

4. Solicite una explicación de asignación para la réplica no asignada del índice `arquitectura-930`.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -X POST "http://localhost:9201/_cluster/allocation/explain?pretty" \
     -H 'Content-Type: application/json' \
     -d '{
       "index": "arquitectura-930",
       "shard": 0,
       "primary": false
     }' | tee /opt/elastic-labs/work/es930-allocation-explain.json
   ```

5. Consulte de nuevo la salud del clúster 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cluster/health?pretty" \
     | jq '{
       cluster_name,
       status,
       number_of_nodes,
       active_primary_shards,
       active_shards,
       unassigned_shards,
       active_shards_percent_as_number
     }'
   ```

6. Registre la interpretación técnica en el archivo de observaciones.

   ```bash
   cat >> /opt/elastic-labs/work/observaciones-arquitectura.md <<'EOF'

   ## Análisis de shards y réplicas

   El laboratorio usa `discovery.type=single-node`; por tanto, cada clúster contiene un único nodo.
   Los shards primarios pueden asignarse a ese nodo y atender escrituras y búsquedas.
   Las réplicas no pueden asignarse al mismo nodo que su shard primario debido a las reglas de
   asignación de Elasticsearch. Si se permitiera esa colocación, la pérdida del nodo eliminaría
   simultáneamente el primario y su réplica, por lo que no aportaría alta disponibilidad.

   Un estado `yellow` es esperado cuando existen réplicas sin asignar pero todos los shards
   primarios están activos. Un estado `red` implicaría que al menos un shard primario no está
   disponible y, por tanto, una parte de los datos no puede atender operaciones normales.

   EOF
   ```

**Salida esperada:**

Para los índices creados con una réplica, la Cat Shards API debe mostrar:

- Filas con `p` y estado `STARTED` para los *shards* primarios.
- Filas con `r` y estado `UNASSIGNED` para las réplicas.

Ejemplo conceptual:

```text
index             shard prirep state      docs store ip         node
arquitectura-930  0     p      STARTED       1  ... 172.20.x.x es930-node-1
arquitectura-930  0     r      UNASSIGNED
arquitectura-930  1     p      STARTED       0  ... 172.20.x.x es930-node-1
arquitectura-930  1     r      UNASSIGNED
```

La respuesta de `/_cluster/allocation/explain` debe incluir una decisión de asignación negativa relacionada con la imposibilidad de ubicar una réplica en el mismo nodo que contiene el primario, normalmente identificada como una regla equivalente a `same_shard`.

**Verificación:**

Ejecute la siguiente consulta y compruebe que existen *shards* no asignados:

```bash
curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
  "http://localhost:9201/_cluster/health?pretty" \
  | jq '{status, active_primary_shards, unassigned_shards}'
```

El resultado esperado es normalmente `yellow` y un valor de `unassigned_shards` mayor que cero después de crear índices con réplicas.

> **Nota:** Puede haber índices internos de Kibana u otros índices del sistema que también influyan en el número de *shards*. El análisis debe centrarse especialmente en `arquitectura-717`, `arquitectura-930` y el índice de respaldo del *data stream*.

---

### Paso 7. Relacionar el diseño de laboratorio con patrones de producción

**Objetivo:** Interpretar las evidencias técnicas y contrastar el deployment de nodo único con una arquitectura distribuida de producción.

**Instrucciones:**

1. Consulte la configuración efectiva relacionada con el clúster y el descubrimiento en Elasticsearch 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_nodes/settings?filter_path=nodes.*.name,nodes.*.settings.cluster.name,nodes.*.settings.discovery.type&pretty"
   ```

2. Consulte las estadísticas resumidas del clúster 9.3.0.

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cluster/stats?filter_path=cluster_name,status,nodes.count,nodes.roles,indices.count,indices.shards&pretty"
   ```

3. Complete la siguiente comparación en el archivo de observaciones.

   ```bash
   cat >> /opt/elastic-labs/work/observaciones-arquitectura.md <<'EOF'

   ## Comparación con patrones de producción

   | Aspecto | Laboratorio de un nodo | Patrón de producción recomendado |
   |---|---|---|
   | Elección de maestro | El mismo nodo participa y opera el clúster | Varios nodos master-eligible estables |
   | Almacenamiento | El mismo nodo almacena todos los primarios | Datos distribuidos en varios nodos de datos |
   | Réplicas | No pueden asignarse en otro nodo | Réplicas ubicadas en nodos o zonas distintas |
   | Ingesta | Comparte recursos con búsqueda y almacenamiento | Nodos ingest dedicados si los pipelines son costosos |
   | Coordinación | El nodo único recibe y procesa solicitudes | Nodos coordinadores dedicados bajo alta carga |
   | Alta disponibilidad | No existe tolerancia a la caída del nodo | Redundancia de nodos, zonas, snapshots y pruebas de restauración |
   | Escalado | Vertical y limitado al host Docker | Horizontal mediante incorporación de nodos y capacidad |

   EOF
   ```

4. Responda las siguientes preguntas en el mismo archivo:

   ```bash
   cat >> /opt/elastic-labs/work/observaciones-arquitectura.md <<'EOF'

   ## Preguntas de análisis

   1. ¿Qué responsabilidades concentra el nodo es930-node-1?
   Respuesta: completar.

   2. ¿Por qué el nodo maestro no debe interpretarse como el único nodo que almacena todos los datos?
   Respuesta: completar.

   3. ¿Por qué una réplica en el mismo nodo que su primario no proporciona alta disponibilidad?
   Respuesta: completar.

   4. ¿Qué carga podría justificar nodos ingest dedicados?
   Respuesta: completar.

   5. ¿Qué riesgos operativos existirían si este deployment de laboratorio se utilizara en producción?
   Respuesta: completar.

   EOF
   ```

5. Revise visualmente en Kibana 9.3.0 el acceso al entorno principal:

   - Abra <http://localhost:5602>.
   - Inicie sesión con el usuario `elastic`.
   - Acceda a **Management** o **Stack Management**.
   - Explore las secciones de índices, *data streams* y estado del clúster disponibles en la interfaz.
   - Confirme que aparecen `arquitectura-930` y `logs-lab-aplicacion`.

**Salida esperada:**

El análisis debe establecer que el nodo único concentra funciones que, en producción, pueden separarse para reducir contención de recursos y mejorar la disponibilidad. También debe indicar que el estado `yellow` del laboratorio no representa pérdida de datos mientras los primarios estén activos, pero sí evidencia ausencia de tolerancia a fallos.

**Verificación:**

Revise que el documento de observaciones exista y contenga las tres secciones solicitadas:

```bash
grep -nE '^## (Roles observados|Análisis de shards y réplicas|Comparación con patrones de producción)' \
  /opt/elastic-labs/work/observaciones-arquitectura.md
```

---

## Validación y pruebas

Ejecute la siguiente lista de comprobación final. Cada comando debe devolver información válida sin errores de autenticación, conexión o asignación de primarios.

1. Verificar los cuatro contenedores:

   ```bash
   cd /opt/elastic-labs
   docker compose ps
   ```

2. Verificar los nombres y versiones de ambos clústeres:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" http://localhost:9200/ \
     | jq '{name, cluster_name, version: .version.number}'

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" http://localhost:9201/ \
     | jq '{name, cluster_name, version: .version.number}'
   ```

3. Verificar que cada clúster tiene un único nodo:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cluster/health" | jq '{cluster_name, number_of_nodes, status}'

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cluster/health" | jq '{cluster_name, number_of_nodes, status}'
   ```

4. Verificar el índice, el alias y el *data stream* del clúster 9.3.0:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/indices/arquitectura-930?v"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_alias/arquitectura-actual?pretty"

   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_data_stream/logs-lab-aplicacion?pretty"
   ```

5. Verificar que las réplicas no están asignadas debido al diseño de un nodo:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/shards/arquitectura-930?v"
   ```

6. Verificar que se generaron las evidencias principales:

   ```bash
   find /opt/elastic-labs/work -maxdepth 1 -type f -printf '%f\n' | sort
   ```

Evidencias mínimas esperadas:

```text
compose-validado.yaml
es717-cat-nodes.txt
es717-cluster-health.json
es717-nodes-completo.json
es717-root.json
es930-allocation-explain.json
es930-cat-nodes.txt
es930-cat-shards.txt
es930-cluster-health.json
es930-data-stream.json
es930-nodes-completo.json
es930-root.json
observaciones-arquitectura.md
```

---

## Resolución de problemas

### Problema 1: Elasticsearch no inicia y los logs muestran un error relacionado con `vm.max_map_count`

**Síntomas:**

- El contenedor `es717-lab` o `es930-lab` se reinicia continuamente.
- `docker compose ps` muestra el contenedor como `Restarting` o no saludable.
- Los logs contienen mensajes parecidos a `max virtual memory areas vm.max_map_count ... is too low`.

**Causa:**

El host Linux no tiene configurado el valor mínimo requerido por Elasticsearch para `vm.max_map_count`. Elasticsearch utiliza mapas de memoria para trabajar con segmentos de Lucene y no inicia si el límite del host es insuficiente.

**Solución:**

1. Configure el valor requerido en el host:

   ```bash
   sudo sysctl -w vm.max_map_count=262144
   ```

2. Persista la configuración:

   ```bash
   echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-elastic-labs.conf
   sudo sysctl --system
   ```

3. Reinicie solamente los servicios afectados, sin eliminar volúmenes:

   ```bash
   cd /opt/elastic-labs
   docker compose restart es717-lab es930-lab
   docker compose ps
   ```

4. Revise los logs si el problema persiste:

   ```bash
   docker logs es930-lab --tail 100
   ```

### Problema 2: El clúster permanece en estado `yellow` después de crear los índices de demostración

**Síntomas:**

- `GET /_cluster/health` devuelve `"status": "yellow"`.
- `GET /_cat/shards?v` muestra réplicas con estado `UNASSIGNED`.
- La API `/_cluster/allocation/explain` indica que no es posible asignar una réplica al nodo actual.

**Causa:**

El laboratorio está configurado intencionalmente con `discovery.type=single-node`. Elasticsearch no asigna una réplica en el mismo nodo que contiene el *shard* primario, porque esa copia no protegería los datos ante la caída del nodo. El estado `yellow` es esperado mientras los *shards* primarios estén activos.

**Solución:**

No intente forzar la asignación de réplicas en este laboratorio. Documente el comportamiento como evidencia de la limitación de alta disponibilidad del diseño de nodo único.

Si necesita que el entorno de laboratorio muestre estado `green` de forma temporal para un índice no crítico, reduzca las réplicas a cero:

```bash
curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
  -X PUT "http://localhost:9201/arquitectura-930/_settings" \
  -H 'Content-Type: application/json' \
  -d '{
    "index": {
      "number_of_replicas": 0
    }
  }' | jq .
```

> No aplique esta modificación si necesita conservar la evidencia de réplicas no asignadas para la evaluación de esta práctica.

---

## Limpieza

Esta práctica establece el estado inicial compartido para prácticas posteriores. Por ese motivo, **no elimine los volúmenes ni ejecute `docker compose down -v`**.

Si necesita liberar memoria y CPU temporalmente, detenga los contenedores conservando los datos:

```bash
cd /opt/elastic-labs
docker compose stop
```

Para reanudar el entorno más adelante:

```bash
cd /opt/elastic-labs
docker compose --env-file .env up -d
```

Para comprobar que los volúmenes persistentes se conservan:

```bash
docker volume ls | grep -E 'es717-data|es930-data|kibana717-data|kibana930-data'
```

---

## Resumen

En esta práctica se desplegaron y validaron dos clústeres Elasticsearch aislados: `es717-lab-cluster` con Elasticsearch 7.17.29 y `es930-lab-cluster` con Elasticsearch 9.3.0. Se utilizaron las APIs `GET /`, `GET /_cluster/health`, `GET /_cat/nodes?v`, `GET /_nodes`, `GET /_cat/shards?v` y `POST /_cluster/allocation/explain` para obtener evidencia directa de la arquitectura en ejecución.

También se crearon índices, un alias y un *data stream* en el clúster 9.3.0. El estado `yellow` observado al configurar réplicas es esperado en un deployment de nodo único: los *shards* primarios están disponibles, pero Elasticsearch no puede ubicar las réplicas en un nodo diferente. En producción, la alta disponibilidad requiere varios nodos, distribución de *shards*, réplicas asignables, capacidad suficiente y una estrategia validada de recuperación mediante *snapshots*.
