# Identificar un cuello de botella y validar un ajuste operativo

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 80 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |
| Tecnologías | Elasticsearch 7.17.29, Kibana 7.17.29, Bulk API, Cat APIs, Nodes Stats API, Apache JMeter 5.6.3 |

## Descripción general

En esta práctica desplegará el clúster de origen `es717-lab`, generará una carga reproducible de indexación masiva y recogerá una línea base de métricas operativas. El índice inicial tendrá una configuración deliberadamente ineficiente para un clúster de un solo nodo: demasiados shards primarios, una réplica no asignable y un intervalo de refresco frecuente.

A continuación, identificará el factor dominante a partir de evidencias de shards, segmentos, heap, presión de indexación, pools de hilos y latencia de bulk. Finalmente, aplicará un ajuste de bajo riesgo mediante un índice optimizado y comparará los resultados antes y después en un informe técnico reutilizable en la Práctica 7.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Inspeccionar salud, nodos, shards, almacenamiento, heap, segmentos y presión de indexación mediante APIs de Elasticsearch.
- [ ] Crear y ejecutar una prueba reproducible de bulk indexing con Apache JMeter.
- [ ] Establecer una línea base con tasa de documentos, latencia, rechazos y métricas del índice.
- [ ] Identificar el impacto operativo de demasiados shards y refresh frecuente en un único nodo.
- [ ] Aplicar una configuración optimizada y validar cuantitativamente la mejora.

## Prerrequisitos

### Conocimientos

- Uso básico de terminal Linux, `curl`, JSON y archivos YAML.
- Conceptos de índices, documentos, shards primarios, réplicas, refresh y Bulk API.
- Interpretación básica de estado `green`, `yellow` y `red` en Elasticsearch.
- Conocimiento inicial de Apache JMeter en modo no gráfico.

### Acceso y requisitos técnicos

- Acceso de lectura y escritura sobre `/opt/elastic-labs`.
- Docker Engine y Docker Compose plugin operativos.
- Apache JMeter 5.6.3 instalado y disponible mediante el comando `jmeter`.
- `curl` y `jq` instalados.
- Puertos `9200` y `5601` libres.
- Parámetro Linux `vm.max_map_count=262144`.
- Al menos 16 GB de RAM disponible; se recomiendan 24 GB.
- Al menos 40 GB libres en disco SSD.

## Entorno de laboratorio

### Recursos previstos

| Recurso | Requisito mínimo | Recomendado |
|---|---:|---:|
| CPU | 4 vCPU | 6 vCPU |
| Memoria RAM | 16 GB | 24 GB |
| Espacio libre SSD | 40 GB | 60 GB |
| Elasticsearch | 7.17.29 | 7.17.29 |
| Kibana | 7.17.29 | 7.17.29 |
| JMeter | 5.6.3 | 5.6.3 |

### Convenciones obligatorias

| Elemento | Valor |
|---|---|
| Directorio raíz | `/opt/elastic-labs` |
| Archivo Compose | `/opt/elastic-labs/compose.yaml` |
| Directorio de trabajo | `/opt/elastic-labs/work` |
| Red Docker | `elastic-lab-net` |
| Elasticsearch origen | `es717-lab` |
| Kibana origen | `kibana717-lab` |
| Cluster origen | `es717-lab-cluster` |
| Nodo origen | `es717-node-1` |
| Endpoint Elasticsearch | `http://localhost:9200` |
| Endpoint Kibana | `http://localhost:5601` |
| Índice de línea base | `logs-app-lab-v1` |
| Índice optimizado | `logs-app-lab-v2` |

> **Importante:** No ejecute `docker compose down -v`. Ese comando eliminaría los volúmenes persistentes requeridos para prácticas posteriores.

---

## Procedimiento paso a paso

### Paso 1. Preparar la estructura de trabajo y validar el host

**Objetivo:** Crear la estructura obligatoria del laboratorio, validar requisitos del host y proteger las credenciales del entorno.

**Instrucciones:**

1. Cree los directorios de trabajo requeridos.

   ```bash
   sudo mkdir -p /opt/elastic-labs/work/{evidence,results,jmeter}
   sudo chown -R "$USER":"$USER" /opt/elastic-labs
   cd /opt/elastic-labs
   ```

2. Verifique el valor de `vm.max_map_count`.

   ```bash
   sysctl vm.max_map_count
   ```

3. Si el resultado es menor que `262144`, aplique el valor temporal y persistente.

   ```bash
   sudo sysctl -w vm.max_map_count=262144

   echo 'vm.max_map_count=262144' | \
     sudo tee /etc/sysctl.d/99-elastic-labs.conf

   sudo sysctl --system
   ```

4. Cree el archivo de variables de entorno con las credenciales exclusivas del laboratorio.

   ```bash
   cat > /opt/elastic-labs/.env <<'EOF'
   ELASTIC_PASSWORD=ElasticLab_2026!
   EOF

   chmod 0600 /opt/elastic-labs/.env
   ```

5. Verifique Docker, Docker Compose, JMeter, espacio de disco y memoria disponible.

   ```bash
   docker --version
   docker compose version
   jmeter --version | head -n 2
   df -h /opt
   free -h
   ```

**Resultado esperado:**

- `vm.max_map_count = 262144`.
- Docker y Docker Compose muestran versiones instaladas.
- JMeter informa versión `5.6.3` o compatible.
- El archivo `.env` tiene permisos `-rw-------`.

**Verificación:**

```bash
ls -l /opt/elastic-labs/.env
sysctl vm.max_map_count
```

La salida esperada debe incluir permisos `600` y el valor `262144`.

---

### Paso 2. Crear y desplegar Elasticsearch 7.17.29 y Kibana 7.17.29

**Objetivo:** Desplegar el clúster de origen de un solo nodo con almacenamiento persistente y autenticación básica habilitada.

**Instrucciones:**

1. Cree el archivo `/opt/elastic-labs/compose.yaml`.

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
         - xpack.security.enabled=true
         - xpack.security.http.ssl.enabled=false
         - xpack.security.transport.ssl.enabled=false
         - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
         - ES_JAVA_OPTS=-Xms1g -Xmx1g
       ports:
         - "9200:9200"
       volumes:
         - es717-data:/usr/share/elasticsearch/data
       networks:
         - elastic-lab-net

     kibana717-lab:
       image: docker.elastic.co/kibana/kibana:7.17.29
       container_name: kibana717-lab
       environment:
         - ELASTICSEARCH_HOSTS=http://es717-lab:9200
         - ELASTICSEARCH_USERNAME=elastic
         - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
         - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=elastic-lab-kibana-717-encryption-key-2026
       ports:
         - "5601:5601"
       volumes:
         - kibana717-data:/usr/share/kibana/data
       depends_on:
         - es717-lab
       networks:
         - elastic-lab-net

     es930-lab:
       image: docker.elastic.co/elasticsearch/elasticsearch:9.3.0
       container_name: es930-lab
       profiles: ["future"]
       environment:
         - node.name=es930-node-1
         - cluster.name=es930-lab-cluster
         - discovery.type=single-node
       ports:
         - "9201:9200"
       volumes:
         - es930-data:/usr/share/elasticsearch/data
       networks:
         - elastic-lab-net

     kibana930-lab:
       image: docker.elastic.co/kibana/kibana:9.3.0
       container_name: kibana930-lab
       profiles: ["future"]
       ports:
         - "5602:5601"
       volumes:
         - kibana930-data:/usr/share/kibana/data
       networks:
         - elastic-lab-net

   volumes:
     es717-data:
       name: es717-data
     es930-data:
       name: es930-data
     kibana717-data:
       name: kibana717-data
     kibana930-data:
       name: kibana930-data

   networks:
     elastic-lab-net:
       name: elastic-lab-net
   EOF
   ```

2. Inicie únicamente el clúster de origen y Kibana 7.17.29.

   ```bash
   cd /opt/elastic-labs
   docker compose up -d es717-lab kibana717-lab
   ```

3. Espere a que Elasticsearch responda.

   ```bash
   source /opt/elastic-labs/.env

   until curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200/_cluster/health | jq -e '.status' >/dev/null; do
     echo "Esperando a Elasticsearch..."
     sleep 5
   done
   ```

4. Consulte la información del clúster y la licencia.

   ```bash
   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200 | jq

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200/_license | jq '.license | {type, status, uid}'
   ```

**Resultado esperado:**

- Los contenedores `es717-lab` y `kibana717-lab` aparecen en estado `Up`.
- Elasticsearch informa versión `7.17.29`.
- El nombre del clúster es `es717-lab-cluster`.
- La licencia aparece como `basic` y `active` o equivalente.

**Verificación:**

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

curl -s -u "elastic:${ELASTIC_PASSWORD}" \
  http://localhost:9200/_cluster/health?pretty
```

El estado inicial debe ser `green`, ya que aún no existen índices de usuario con réplicas.

---

### Paso 3. Crear datos de logs y el índice subóptimo de línea base

**Objetivo:** Generar una carga NDJSON reproducible y crear un índice con una configuración deliberadamente ineficiente para un nodo único.

**Instrucciones:**

1. Cree un archivo NDJSON de 200 eventos de aplicación. Cada acción `index` no especifica `_id`, por lo que Elasticsearch generará identificadores únicos en cada repetición del bulk.

   ```bash
   cd /opt/elastic-labs/work

   : > bulk-200.ndjson

   for i in $(seq 1 200); do
     level="INFO"
     if [ $((i % 10)) -eq 0 ]; then
       level="ERROR"
     elif [ $((i % 5)) -eq 0 ]; then
       level="WARN"
     fi

     cat >> bulk-200.ndjson <<EOF
   {"index":{}}
   {"@timestamp":"2026-08-25T10:00:00.000Z","service":{"name":"checkout-api"},"host":{"name":"app-lab-01"},"log":{"level":"${level}"},"event":{"dataset":"app.logs","category":"application"},"message":"Laboratory event ${i} for performance validation","http":{"response":{"status_code":$((200 + i % 5))}},"labels":{"environment":"lab","test_run":"bulk-performance"}}
   EOF
   done

   wc -l bulk-200.ndjson
   du -h bulk-200.ndjson
   tail -n 2 bulk-200.ndjson
   ```

2. El archivo debe tener exactamente 400 líneas: una línea de acción y una línea de documento por evento.

3. Cree el índice de línea base con 24 shards primarios, una réplica y `refresh_interval` de un segundo.

   ```bash
   source /opt/elastic-labs/.env

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     -X PUT http://localhost:9200/logs-app-lab-v1 \
     -H 'Content-Type: application/json' \
     -d '{
       "settings": {
         "number_of_shards": 24,
         "number_of_replicas": 1,
         "refresh_interval": "1s"
       },
       "mappings": {
         "dynamic": false,
         "properties": {
           "@timestamp": { "type": "date" },
           "service": {
             "properties": {
               "name": { "type": "keyword" }
             }
           },
           "host": {
             "properties": {
               "name": { "type": "keyword" }
             }
           },
           "log": {
             "properties": {
               "level": { "type": "keyword" }
             }
           },
           "event": {
             "properties": {
               "dataset": { "type": "keyword" },
               "category": { "type": "keyword" }
             }
           },
           "message": { "type": "text" },
           "http": {
             "properties": {
               "response": {
                 "properties": {
                   "status_code": { "type": "integer" }
                 }
               }
             }
           },
           "labels": {
             "properties": {
               "environment": { "type": "keyword" },
               "test_run": { "type": "keyword" }
             }
           }
         }
       }
     }' | jq
   ```

4. Consulte la salud y los shards del índice recién creado.

   ```bash
   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cluster/health?pretty'

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/shards/logs-app-lab-v1?v&bytes=mb'
   ```

**Resultado esperado:**

- El archivo `bulk-200.ndjson` tiene 400 líneas.
- El índice `logs-app-lab-v1` tiene 24 primarios asignados.
- El clúster pasa a estado `yellow`.
- Las 24 réplicas aparecen como `UNASSIGNED`.

**Verificación:**

```bash
curl -s -u "elastic:${ELASTIC_PASSWORD}" \
  'http://localhost:9200/_cat/indices/logs-app-lab-v1?v&h=health,status,index,pri,rep,docs.count,store.size'
```

> **Interpretación:** El estado `yellow` no significa pérdida de datos en este momento: los shards primarios están activos. Sin embargo, en un único nodo las réplicas no pueden asignarse porque Elasticsearch no coloca un primario y su réplica en el mismo nodo. La configuración no proporciona redundancia y añade complejidad operativa innecesaria.

---

### Paso 4. Recopilar la evidencia operativa previa a la carga

**Objetivo:** Capturar métricas de salud, shards, asignación, memoria, almacenamiento, pools de hilos, segmentos y presión de indexación antes de ejecutar la carga.

**Instrucciones:**

1. Cree el directorio de evidencia de línea base.

   ```bash
   cd /opt/elastic-labs/work
   mkdir -p evidence/baseline-before
   source /opt/elastic-labs/.env
   ```

2. Ejecute las consultas de diagnóstico y guarde cada respuesta.

   ```bash
   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cluster/health?pretty' \
     > evidence/baseline-before/cluster-health.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/nodes?v&h=name,roles,master,heap.percent,ram.percent,cpu,load_1m,disk.used_percent,node.role' \
     > evidence/baseline-before/cat-nodes.txt

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/allocation?v&bytes=mb' \
     > evidence/baseline-before/cat-allocation.txt

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/shards/logs-app-lab-v1?v&bytes=mb' \
     > evidence/baseline-before/cat-shards-v1.txt

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_nodes/stats/jvm,fs,indices,thread_pool,indexing_pressure?pretty' \
     > evidence/baseline-before/nodes-stats.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/logs-app-lab-v1/_stats?pretty' \
     > evidence/baseline-before/index-stats-v1.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/thread_pool?v&h=node_name,name,active,queue,rejected,completed' \
     > evidence/baseline-before/thread-pools.txt

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_nodes/hot_threads?threads=3&type=cpu' \
     > evidence/baseline-before/hot-threads.txt
   ```

3. Revise los datos más relevantes.

   ```bash
   cat evidence/baseline-before/cat-nodes.txt
   cat evidence/baseline-before/cat-allocation.txt
   cat evidence/baseline-before/cat-shards-v1.txt

   jq '
     .nodes[] |
     {
       node: .name,
       heap_used_percent: .jvm.mem.heap_used_percent,
       gc_young_collections: .jvm.gc.collectors.young.collection_count,
       gc_old_collections: .jvm.gc.collectors.old.collection_count,
       fs_available_bytes: .fs.total.available_in_bytes,
       indexing_pressure_current_bytes: .indexing_pressure.memory.current.combined_coordinating_and_primary_in_bytes
     }
   ' evidence/baseline-before/nodes-stats.json
   ```

**Resultado esperado:**

- Las evidencias quedan almacenadas bajo `work/evidence/baseline-before`.
- El estado es `yellow` por réplicas no asignadas.
- `cat-shards-v1.txt` muestra 24 primarios iniciados y 24 réplicas no asignadas.
- Los contadores de `rejected` normalmente comienzan en cero.

**Verificación:**

```bash
find /opt/elastic-labs/work/evidence/baseline-before -type f | sort
```

Debe listar al menos ocho archivos de evidencia.

---

### Paso 5. Ejecutar la carga de línea base con Apache JMeter

**Objetivo:** Medir rendimiento de bulk indexing sobre el índice subóptimo usando una carga reproducible.

**Instrucciones:**

1. Cree el plan de prueba JMeter. La prueba ejecutará 16 hilos, 25 iteraciones por hilo y un bulk de 200 documentos por solicitud.

   ```bash
   cat > /opt/elastic-labs/work/jmeter/bulk-load.jmx <<'EOF'
   <?xml version="1.0" encoding="UTF-8"?>
   <jmeterTestPlan version="1.2" properties="5.0" jmeter="5.6.3">
     <hashTree>
       <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Bulk indexing performance test" enabled="true">
         <stringProp name="TestPlan.comments">Carga reproducible para Elasticsearch Bulk API</stringProp>
         <boolProp name="TestPlan.functional_mode">false</boolProp>
         <boolProp name="TestPlan.tearDown_on_shutdown">true</boolProp>
       </TestPlan>
       <hashTree>
         <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Bulk clients" enabled="true">
           <stringProp name="ThreadGroup.on_sample_error">CONTINUE</stringProp>
           <elementProp name="ThreadGroup.main_controller" elementType="LoopController" guiclass="LoopControlPanel" testclass="LoopController" testname="Loop Controller" enabled="true">
             <boolProp name="LoopController.continue_forever">false</boolProp>
             <stringProp name="LoopController.loops">${__P(loops,25)}</stringProp>
           </elementProp>
           <stringProp name="ThreadGroup.num_threads">${__P(threads,16)}</stringProp>
           <stringProp name="ThreadGroup.ramp_time">5</stringProp>
           <boolProp name="ThreadGroup.scheduler">false</boolProp>
         </ThreadGroup>
         <hashTree>
           <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="POST _bulk" enabled="true">
             <stringProp name="HTTPSampler.domain">localhost</stringProp>
             <stringProp name="HTTPSampler.port">9200</stringProp>
             <stringProp name="HTTPSampler.protocol">http</stringProp>
             <stringProp name="HTTPSampler.path">/${__P(index,logs-app-lab-v1)}/_bulk</stringProp>
             <stringProp name="HTTPSampler.method">POST</stringProp>
             <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
             <boolProp name="HTTPSampler.auto_redirects">false</boolProp>
             <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
             <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
             <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
               <collectionProp name="Arguments.arguments">
                 <elementProp name="" elementType="HTTPArgument">
                   <boolProp name="HTTPArgument.always_encode">false</boolProp>
                   <stringProp name="Argument.value">${__FileToString(${__P(bulkfile)})}</stringProp>
                   <stringProp name="Argument.metadata">=</stringProp>
                 </elementProp>
               </collectionProp>
             </elementProp>
           </HTTPSamplerProxy>
           <hashTree>
             <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="Headers" enabled="true">
               <collectionProp name="HeaderManager.headers">
                 <elementProp name="Content-Type" elementType="Header">
                   <stringProp name="Header.name">Content-Type</stringProp>
                   <stringProp name="Header.value">application/x-ndjson</stringProp>
                 </elementProp>
                 <elementProp name="Authorization" elementType="Header">
                   <stringProp name="Header.name">Authorization</stringProp>
                   <stringProp name="Header.value">${__P(auth)}</stringProp>
                 </elementProp>
               </collectionProp>
             </HeaderManager>
             <hashTree/>
             <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Bulk sin errores parciales" enabled="true">
               <collectionProp name="Asserion.test_strings">
                 <stringProp name="49586">"errors":false</stringProp>
               </collectionProp>
               <stringProp name="Assertion.custom_message">La respuesta Bulk contiene errores parciales</stringProp>
               <stringProp name="Assertion.test_field">Assertion.response_data</stringProp>
               <boolProp name="Assertion.assume_success">false</boolProp>
               <intProp name="Assertion.test_type">2</intProp>
             </ResponseAssertion>
             <hashTree/>
           </hashTree>
         </hashTree>
       </hashTree>
     </hashTree>
   </jmeterTestPlan>
   EOF
   ```

2. Calcule el encabezado HTTP Basic sin escribir la contraseña directamente en el plan de prueba.

   ```bash
   source /opt/elastic-labs/.env
   AUTH_HEADER="Basic $(printf 'elastic:%s' "${ELASTIC_PASSWORD}" | base64 -w 0)"
   ```

3. Ejecute la carga de línea base y guarde el resultado CSV.

   ```bash
   cd /opt/elastic-labs/work

   jmeter -n \
     -t jmeter/bulk-load.jmx \
     -l results/baseline-v1.jtl \
     -Jthreads=16 \
     -Jloops=25 \
     -Jindex=logs-app-lab-v1 \
     -Jbulkfile=/opt/elastic-labs/work/bulk-200.ndjson \
     -Jauth="${AUTH_HEADER}"
   ```

4. Calcule métricas de rendimiento desde el archivo JTL. Cada solicitud exitosa representa 200 documentos.

   ```bash
   awk -F, '
   NR == 1 { next }
   {
     if (NR == 2 || $1 < min) min=$1
     if ($1 > max) max=$1
     total++
     if ($8 == "true") ok++
     else fail++
     elapsed_sum += $2
     if ($2 > elapsed_max) elapsed_max=$2
   }
   END {
     duration=(max-min)/1000
     if (duration <= 0) duration=1
     docs=ok*200
     printf "Solicitudes totales: %d\n", total
     printf "Solicitudes exitosas: %d\n", ok
     printf "Solicitudes fallidas: %d\n", fail
     printf "Documentos indexados estimados: %d\n", docs
     printf "Duracion observada: %.2f s\n", duration
     printf "Tasa estimada: %.2f documentos/s\n", docs/duration
     printf "Latencia media bulk: %.2f ms\n", elapsed_sum/total
     printf "Latencia maxima bulk: %d ms\n", elapsed_max
   }' results/baseline-v1.jtl | tee results/baseline-v1-summary.txt
   ```

5. Capture las métricas posteriores a la carga.

   ```bash
   mkdir -p evidence/baseline-after

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cluster/health?pretty' \
     > evidence/baseline-after/cluster-health.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/logs-app-lab-v1/_stats?pretty' \
     > evidence/baseline-after/index-stats-v1.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_nodes/stats/jvm,fs,indices,thread_pool,indexing_pressure?pretty' \
     > evidence/baseline-after/nodes-stats.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/thread_pool?v&h=node_name,name,active,queue,rejected,completed' \
     > evidence/baseline-after/thread-pools.txt

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/shards/logs-app-lab-v1?v&bytes=mb' \
     > evidence/baseline-after/cat-shards-v1.txt
   ```

**Resultado esperado:**

- JMeter genera `results/baseline-v1.jtl`.
- Se ejecutan hasta 400 solicitudes bulk: `16 × 25`.
- Si no hay fallos, se indexan aproximadamente 80 000 documentos.
- El índice acumula segmentos distribuidos entre 24 primarios.
- El clúster continúa en estado `yellow` por las réplicas no asignadas.

**Verificación:**

```bash
cat /opt/elastic-labs/work/results/baseline-v1-summary.txt

curl -s -u "elastic:${ELASTIC_PASSWORD}" \
  'http://localhost:9200/logs-app-lab-v1/_count' | jq
```

El conteo debe ser cercano a `80000`. Puede variar si se produjo alguna solicitud fallida.

---

### Paso 6. Diagnosticar el cuello de botella y aplicar el ajuste operativo

**Objetivo:** Relacionar la evidencia obtenida con una causa dominante y crear un índice optimizado para el mismo patrón de carga.

**Instrucciones:**

1. Compare el número de shards, el estado de las réplicas y las métricas de segmentos.

   ```bash
   cd /opt/elastic-labs/work

   echo "===== Shards de linea base ====="
   cat evidence/baseline-after/cat-shards-v1.txt

   echo "===== Estadisticas de indexacion y segmentos ====="
   jq '
     ._all.primaries |
     {
       documentos: .docs.count,
       tiempo_indexacion_ms: .indexing.index_time_in_millis,
       operaciones_indexacion: .indexing.index_total,
       refresh_total: .refresh.total,
       refresh_tiempo_ms: .refresh.total_time_in_millis,
       segmentos: .segments.count,
       almacenamiento_bytes: .store.size_in_bytes
     }
   ' evidence/baseline-after/index-stats-v1.json

   echo "===== Pools de hilos ====="
   cat evidence/baseline-after/thread-pools.txt
   ```

2. Registre el diagnóstico técnico inicial:

   - `logs-app-lab-v1` usa 24 shards primarios para un único nodo y un volumen moderado de datos.
   - Las 24 réplicas permanecen sin asignar y mantienen el clúster en estado `yellow`.
   - El `refresh_interval` de `1s` obliga a publicar segmentos con alta frecuencia durante una carga sostenida.
   - Un número elevado de shards pequeños incrementa metadatos, archivos, segmentos y trabajo de coordinación.
   - Los contadores `rejected` o los errores HTTP `429`, si existen, indican que la tasa del cliente supera temporalmente la capacidad del nodo.

3. Cree el índice optimizado. El ajuste combina un shard primario, cero réplicas para un único nodo y un intervalo de refresh de 30 segundos.

   ```bash
   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     -X PUT http://localhost:9200/logs-app-lab-v2 \
     -H 'Content-Type: application/json' \
     -d '{
       "settings": {
         "number_of_shards": 1,
         "number_of_replicas": 0,
         "refresh_interval": "30s"
       },
       "mappings": {
         "dynamic": false,
         "properties": {
           "@timestamp": { "type": "date" },
           "service": {
             "properties": {
               "name": { "type": "keyword" }
             }
           },
           "host": {
             "properties": {
               "name": { "type": "keyword" }
             }
           },
           "log": {
             "properties": {
               "level": { "type": "keyword" }
             }
           },
           "event": {
             "properties": {
               "dataset": { "type": "keyword" },
               "category": { "type": "keyword" }
             }
           },
           "message": { "type": "text" },
           "http": {
             "properties": {
               "response": {
                 "properties": {
                   "status_code": { "type": "integer" }
                 }
               }
             }
           },
           "labels": {
             "properties": {
               "environment": { "type": "keyword" },
               "test_run": { "type": "keyword" }
             }
           }
         }
       }
     }' | jq
   ```

4. Verifique que el índice optimizado tiene una configuración adecuada para el laboratorio.

   ```bash
   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/logs-app-lab-v2/_settings?pretty'

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cluster/health?pretty'
   ```

5. Capture un punto de evidencia previo a la segunda prueba.

   ```bash
   mkdir -p evidence/optimized-before

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/logs-app-lab-v2/_stats?pretty' \
     > evidence/optimized-before/index-stats-v2.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_nodes/stats/jvm,fs,indices,thread_pool,indexing_pressure?pretty' \
     > evidence/optimized-before/nodes-stats.json
   ```

**Resultado esperado:**

- `logs-app-lab-v2` tiene un shard primario y cero réplicas.
- El estado general del clúster pasa a `green`, porque no existen réplicas pendientes en `v2`.
- El refresh más espaciado reduce el trabajo de publicación de segmentos durante la carga.

**Verificación:**

```bash
curl -s -u "elastic:${ELASTIC_PASSWORD}" \
  'http://localhost:9200/_cat/indices/logs-app-lab-v*?v&h=health,status,index,pri,rep,docs.count,store.size'
```

Debe observar `v1` en amarillo y `v2` inicialmente en verde.

---

### Paso 7. Ejecutar la carga optimizada y documentar la comparación

**Objetivo:** Repetir exactamente la misma carga sobre el índice optimizado y construir el informe técnico de rendimiento.

**Instrucciones:**

1. Ejecute JMeter con los mismos valores de hilos, iteraciones y archivo bulk, cambiando únicamente el índice de destino.

   ```bash
   cd /opt/elastic-labs/work
   source /opt/elastic-labs/.env

   AUTH_HEADER="Basic $(printf 'elastic:%s' "${ELASTIC_PASSWORD}" | base64 -w 0)"

   jmeter -n \
     -t jmeter/bulk-load.jmx \
     -l results/optimized-v2.jtl \
     -Jthreads=16 \
     -Jloops=25 \
     -Jindex=logs-app-lab-v2 \
     -Jbulkfile=/opt/elastic-labs/work/bulk-200.ndjson \
     -Jauth="${AUTH_HEADER}"
   ```

2. Calcule el resumen de la prueba optimizada.

   ```bash
   awk -F, '
   NR == 1 { next }
   {
     if (NR == 2 || $1 < min) min=$1
     if ($1 > max) max=$1
     total++
     if ($8 == "true") ok++
     else fail++
     elapsed_sum += $2
     if ($2 > elapsed_max) elapsed_max=$2
   }
   END {
     duration=(max-min)/1000
     if (duration <= 0) duration=1
     docs=ok*200
     printf "Solicitudes totales: %d\n", total
     printf "Solicitudes exitosas: %d\n", ok
     printf "Solicitudes fallidas: %d\n", fail
     printf "Documentos indexados estimados: %d\n", docs
     printf "Duracion observada: %.2f s\n", duration
     printf "Tasa estimada: %.2f documentos/s\n", docs/duration
     printf "Latencia media bulk: %.2f ms\n", elapsed_sum/total
     printf "Latencia maxima bulk: %d ms\n", elapsed_max
   }' results/optimized-v2.jtl | tee results/optimized-v2-summary.txt
   ```

3. Capture las evidencias posteriores a la prueba optimizada.

   ```bash
   mkdir -p evidence/optimized-after

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cluster/health?pretty' \
     > evidence/optimized-after/cluster-health.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/logs-app-lab-v2/_stats?pretty' \
     > evidence/optimized-after/index-stats-v2.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_nodes/stats/jvm,fs,indices,thread_pool,indexing_pressure?pretty' \
     > evidence/optimized-after/nodes-stats.json

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/thread_pool?v&h=node_name,name,active,queue,rejected,completed' \
     > evidence/optimized-after/thread-pools.txt

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/shards/logs-app-lab-v2?v&bytes=mb' \
     > evidence/optimized-after/cat-shards-v2.txt
   ```

4. Obtenga las métricas de índice del escenario optimizado.

   ```bash
   jq '
     ._all.primaries |
     {
       documentos: .docs.count,
       tiempo_indexacion_ms: .indexing.index_time_in_millis,
       operaciones_indexacion: .indexing.index_total,
       refresh_total: .refresh.total,
       refresh_tiempo_ms: .refresh.total_time_in_millis,
       segmentos: .segments.count,
       almacenamiento_bytes: .store.size_in_bytes
     }
   ' evidence/optimized-after/index-stats-v2.json
   ```

5. Cree el informe técnico requerido.

   ```bash
   cat > baseline-performance.md <<'EOF'
   # Informe técnico: validación de ajuste operativo

   ## Contexto

   - Clúster: `es717-lab-cluster`
   - Nodo: `es717-node-1`
   - Versión: Elasticsearch 7.17.29
   - Carga: Apache JMeter, 16 hilos, 25 iteraciones, 200 documentos por Bulk.
   - Volumen teórico por prueba: 80 000 documentos.

   ## Hipótesis

   La configuración inicial con 24 shards primarios y `refresh_interval=1s` genera sobrecarga de coordinación, administración de segmentos y refresh en un clúster de un solo nodo. La réplica configurada no aporta resiliencia porque no puede asignarse en el único nodo y deja el clúster en estado `yellow`.

   ## Evidencia de línea base

   Adjunte o transcriba aquí los valores de:
   - `results/baseline-v1-summary.txt`
   - `evidence/baseline-after/cat-shards-v1.txt`
   - `evidence/baseline-after/thread-pools.txt`
   - `evidence/baseline-after/index-stats-v1.json`

   ## Diagnóstico

   Factor dominante identificado: exceso de shards primarios pequeños y refresh frecuente durante bulk indexing.

   Señales observadas:
   - Número de shards primarios: 24 en `logs-app-lab-v1`.
   - Réplicas no asignadas: 24.
   - Estado del clúster: `yellow`.
   - Número de segmentos y tiempo de refresh: registrar valores observados.
   - Rechazos de pools o errores JMeter: registrar valores observados.

   ## Mitigación aplicada

   Se creó `logs-app-lab-v2` con:

   ```json
   {
     "number_of_shards": 1,
     "number_of_replicas": 0,
     "refresh_interval": "30s"
   }
   ```

   El tamaño del lote se mantuvo constante en 200 documentos para preservar la comparabilidad.

   ## Comparación cuantitativa

   | Métrica | Línea base: v1 | Optimizado: v2 | Interpretación |
   |---|---:|---:|---|
   | Shards primarios | 24 | 1 | Menor sobrecarga administrativa |
   | Réplicas | 1 no asignada | 0 | Estado verde en un solo nodo |
   | Refresh interval | 1 s | 30 s | Menor frecuencia de publicación |
   | Documentos/s | Completar | Completar | Comparar rendimiento |
   | Latencia media bulk | Completar | Completar | Comparar latencia |
   | Latencia máxima bulk | Completar | Completar | Evaluar picos |
   | Solicitudes fallidas | Completar | Completar | Validar estabilidad |
   | Rechazos thread pool | Completar | Completar | Validar saturación |
   | Segmentos | Completar | Completar | Evaluar coste de búsqueda y merge |

   ## Conclusión

   Complete una conclusión basada en los valores observados. Indique si el índice optimizado mejoró la tasa de indexación, redujo la latencia o disminuyó los errores. Si la mejora fue limitada, explique qué recurso adicional podría ser el siguiente factor limitante: CPU, disco, heap, tamaño del heap, tasa de clientes o tamaño de lotes.
   EOF
   ```

6. Edite el informe e incorpore los valores reales obtenidos.

   ```bash
   nano /opt/elastic-labs/work/baseline-performance.md
   ```

**Resultado esperado:**

- La prueba `v2` tiene el mismo volumen teórico que la prueba `v1`.
- El índice `v2` se mantiene `green`.
- La tasa de documentos por segundo normalmente mejora o la latencia media se reduce.
- El número de segmentos y el trabajo de refresh suelen ser inferiores o más eficientes en `v2`.
- El informe `baseline-performance.md` documenta hipótesis, evidencia, diagnóstico, mitigación y resultados.

**Verificación:**

```bash
cd /opt/elastic-labs/work

cat results/baseline-v1-summary.txt
cat results/optimized-v2-summary.txt

curl -s -u "elastic:${ELASTIC_PASSWORD}" \
  'http://localhost:9200/_cat/indices/logs-app-lab-v*?v&h=health,status,index,pri,rep,docs.count,store.size'
```

---

## Validación y pruebas

Ejecute las siguientes comprobaciones finales.

1. Valide la salud del clúster.

   ```bash
   source /opt/elastic-labs/.env

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cluster/health?pretty'
   ```

   Criterio esperado:

   - El estado global puede aparecer `yellow` mientras exista `logs-app-lab-v1`.
   - `logs-app-lab-v2` debe tener todos sus shards asignados.

2. Valide la configuración de ambos índices.

   ```bash
   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/logs-app-lab-v1,logs-app-lab-v2/_settings?pretty'
   ```

   Criterio esperado:

   | Índice | Primarios | Réplicas | Refresh |
   |---|---:|---:|---|
   | `logs-app-lab-v1` | 24 | 1 | 1 s |
   | `logs-app-lab-v2` | 1 | 0 | 30 s |

3. Valide el volumen de documentos en ambos índices.

   ```bash
   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/logs-app-lab-v1/_count' | jq

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/logs-app-lab-v2/_count' | jq
   ```

4. Valide que JMeter no registró fallos inesperados.

   ```bash
   awk -F, 'NR > 1 && $8 != "true" { failures++ } END { print "Fallos:", failures+0 }' \
     /opt/elastic-labs/work/results/baseline-v1.jtl

   awk -F, 'NR > 1 && $8 != "true" { failures++ } END { print "Fallos:", failures+0 }' \
     /opt/elastic-labs/work/results/optimized-v2.jtl
   ```

5. Valide la existencia de las evidencias y del informe.

   ```bash
   test -f /opt/elastic-labs/work/baseline-performance.md && echo "Informe presente"

   find /opt/elastic-labs/work/evidence -type f | sort
   ```

La práctica se considera completada cuando el informe contiene datos reales de ambas ejecuciones y una conclusión técnica justificada por las métricas recopiladas.

## Solución de problemas

### Problema 1: Elasticsearch no inicia o muestra error relacionado con `vm.max_map_count`

**Síntomas:**

- El contenedor `es717-lab` termina inmediatamente.
- `docker logs es717-lab` muestra un error de bootstrap checks.
- El log contiene referencias a `vm.max_map_count`.

**Causa:**

El host Linux no tiene configurado `vm.max_map_count=262144`, requisito necesario para Elasticsearch.

**Corrección:**

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elastic-labs.conf
sudo sysctl --system

cd /opt/elastic-labs
docker compose up -d es717-lab
docker logs --tail 50 es717-lab
```

Verifique nuevamente:

```bash
sysctl vm.max_map_count
```

---

### Problema 2: JMeter registra respuestas `401`, `429` o fallos por aserción Bulk

**Síntomas:**

- El archivo `.jtl` contiene solicitudes fallidas.
- JMeter informa `401 Unauthorized`, `429 Too Many Requests` o errores parciales de Bulk.
- La aserción indica que la respuesta no contiene `"errors":false`.

**Causa:**

- Un error `401` normalmente indica credenciales o encabezado `Authorization` incorrecto.
- Un error `429` indica que la carga excede temporalmente la capacidad de indexación del nodo.
- Los errores parciales pueden surgir por saturación, falta de recursos o un archivo NDJSON mal formado.

**Corrección:**

1. Valide credenciales y endpoint.

   ```bash
   source /opt/elastic-labs/.env

   curl -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200/_cluster/health?pretty
   ```

2. Valide que el NDJSON contiene 400 líneas y termina con salto de línea.

   ```bash
   wc -l /opt/elastic-labs/work/bulk-200.ndjson
   tail -c 1 /opt/elastic-labs/work/bulk-200.ndjson | od -An -t x1
   ```

3. Si aparecen `429`, reduzca la presión del cliente y repita la ejecución con menos hilos, conservando la misma configuración en ambos escenarios comparados.

   ```bash
   jmeter -n \
     -t /opt/elastic-labs/work/jmeter/bulk-load.jmx \
     -l /opt/elastic-labs/work/results/retry-v2.jtl \
     -Jthreads=8 \
     -Jloops=25 \
     -Jindex=logs-app-lab-v2 \
     -Jbulkfile=/opt/elastic-labs/work/bulk-200.ndjson \
     -Jauth="${AUTH_HEADER}"
   ```

4. Consulte pools de hilos y presión de indexación.

   ```bash
   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_cat/thread_pool?v&h=node_name,name,active,queue,rejected,completed'

   curl -s -u "elastic:${ELASTIC_PASSWORD}" \
     'http://localhost:9200/_nodes/stats/indexing_pressure?pretty'
   ```

## Limpieza

Esta práctica crea evidencias e índices que serán reutilizados posteriormente. No elimine volúmenes ni ejecute `docker compose down -v`.

Si necesita detener temporalmente los servicios para liberar memoria, use:

```bash
cd /opt/elastic-labs
docker compose stop es717-lab kibana717-lab
```

Para reanudar el laboratorio manteniendo datos, índices, evidencias y configuración:

```bash
cd /opt/elastic-labs
docker compose start es717-lab kibana717-lab
```

Conserve especialmente:

```text
/opt/elastic-labs/work/baseline-performance.md
/opt/elastic-labs/work/bulk-200.ndjson
/opt/elastic-labs/work/evidence/
/opt/elastic-labs/work/results/
```

## Resumen

En esta práctica desplegó Elasticsearch 7.17.29 y Kibana 7.17.29 en un entorno persistente de un solo nodo. Creó un índice de línea base con 24 shards primarios, una réplica no asignable y refresh frecuente; posteriormente midió su comportamiento bajo carga bulk con Apache JMeter.

El diagnóstico se fundamentó en evidencia obtenida desde `_cluster/health`, `_cat/nodes`, `_cat/allocation`, `_cat/shards`, `_nodes/stats`, `_nodes/hot_threads`, `_stats` y `_cat/thread_pool`. Finalmente, aplicó un ajuste operativo de bajo riesgo mediante un índice con un shard primario, cero réplicas y `refresh_interval` de 30 segundos, comparando rendimiento, latencia, fallos y métricas de segmentos.

Los índices y evidencias generados constituyen el estado compartido para la siguiente práctica, donde se aplicarán políticas ILM y un data stream gobernado para logs.
