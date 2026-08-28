# Validar versión, licenciamiento, conectividad y salud de deployments 7.17 y 9.3

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 45 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica se construirá una línea base operacional para los deployments Elasticsearch 7.17.29 y 9.3.0 creados en la práctica anterior. Se validarán versiones, builds, distribución, licencia, salud de clúster, APIs disponibles y conectividad desde el host, Kibana y la red Docker.

El resultado será un conjunto de evidencias reproducibles y una matriz de compatibilidad inicial. Esta información permitirá distinguir entre conectividad básica y compatibilidad funcional, especialmente antes de crear recursos de datos o plantear una migración entre generaciones.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Consultar y registrar la versión, build, distribución y características básicas de Elasticsearch 7.17.29 y 9.3.0.
- [ ] Comparar el estado de licencia y las capacidades reportadas por las APIs `_license` y `_xpack`.
- [ ] Verificar conectividad HTTP autenticada desde `curl`, Kibana Dev Tools y un contenedor conectado a `elastic-lab-net`.
- [ ] Confirmar que cada instancia Kibana está conectada exclusivamente al deployment Elasticsearch de su misma generación.
- [ ] Elaborar un inventario de endpoints y una matriz inicial de compatibilidad para APIs, clientes y agentes.

## Requisitos previos

### Conocimientos necesarios

- Comprensión básica de JSON, códigos HTTP y cabeceras HTTP.
- Uso básico de `curl`, `jq` y comandos Docker.
- Conocimiento de los conceptos de clúster, nodo, licencia, API REST y autenticación básica.
- Comprensión de que una respuesta HTTP `200` confirma conectividad y autorización para una API, pero no garantiza compatibilidad funcional completa.

### Accesos y estado requerido

- La práctica 1 debe estar completada.
- Deben existir y estar en ejecución los contenedores:
  - `es717-lab`
  - `kibana717-lab`
  - `es930-lab`
  - `kibana930-lab`
- Debe disponer de acceso local a:
  - Elasticsearch 7.17.29: `http://localhost:9200`
  - Kibana 7.17.29: `http://localhost:5601`
  - Elasticsearch 9.3.0: `http://localhost:9201`
  - Kibana 9.3.0: `http://localhost:5602`
- Deben estar instalados `curl`, `jq`, Docker Engine y Docker Compose.
- El archivo `/opt/elastic-labs/.env` debe contener las credenciales de laboratorio y tener permisos `0600`.

> **Importante:** No ejecute `docker compose down -v`. La opción `-v` eliminaría los volúmenes persistentes requeridos para la continuidad del curso.

## Entorno de laboratorio

### Componentes validados

| Componente | Nombre fijo | Versión objetivo | Acceso |
|---|---|---:|---|
| Elasticsearch legado | `es717-lab` | 7.17.29 | `http://localhost:9200` |
| Kibana legado | `kibana717-lab` | 7.17.29 | `http://localhost:5601` |
| Elasticsearch actual | `es930-lab` | 9.3.0 | `http://localhost:9201` |
| Kibana actual | `kibana930-lab` | 9.3.0 | `http://localhost:5602` |
| Red Docker | `elastic-lab-net` | No aplica | Resolución DNS interna |
| Directorio raíz | `/opt/elastic-labs` | No aplica | Archivos del laboratorio |
| Directorio de evidencias | `/opt/elastic-labs/work` | No aplica | JSON, cabeceras y matriz |

### Variables y convenciones

| Variable | Valor esperado |
|---|---|
| Usuario | `elastic` |
| Contraseña de laboratorio | Declarada en `/opt/elastic-labs/.env` |
| Clúster 7.17 | `es717-lab-cluster` |
| Nodo 7.17 | `es717-node-1` |
| Clúster 9.3 | `es930-lab-cluster` |
| Nodo 9.3 | `es930-node-1` |
| Tipo de descubrimiento | `single-node` |

### Preparación inicial

Ejecute los siguientes comandos desde una terminal Linux:

```bash
cd /opt/elastic-labs

sudo chmod 600 .env
sudo mkdir -p /opt/elastic-labs/work/lab-02-00-01
sudo chmod 700 /opt/elastic-labs/work/lab-02-00-01

set -a
source /opt/elastic-labs/.env
set +a

docker compose ps
docker network inspect elastic-lab-net --format '{{.Name}}'
```

Compruebe que la variable de contraseña existe sin imprimir su valor:

```bash
test -n "${ELASTIC_PASSWORD}" && echo "ELASTIC_PASSWORD cargada correctamente"
```

Si el nombre de variable utilizado en su archivo `.env` es distinto, ajuste los comandos de esta práctica. No incluya la contraseña literal en los archivos de evidencia ni en capturas de pantalla.

---

## Procedimiento paso a paso

### Paso 1. Confirmar el estado de los contenedores y los puertos publicados

**Objetivo:** Verificar que los cuatro componentes requeridos están en ejecución, usan los nombres esperados y publican los puertos locales correctos.

**Instrucciones:**

1. Sitúese en el directorio de trabajo y cargue las variables de entorno si aún no lo ha hecho.

   ```bash
   cd /opt/elastic-labs
   set -a
   source .env
   set +a
   ```

2. Consulte el estado de los servicios Compose.

   ```bash
   docker compose ps
   ```

3. Obtenga una vista resumida de los contenedores y sus puertos.

   ```bash
   docker ps \
     --filter "name=es717-lab" \
     --filter "name=kibana717-lab" \
     --filter "name=es930-lab" \
     --filter "name=kibana930-lab" \
     --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
   ```

4. Verifique que los contenedores Elasticsearch están conectados a la red Docker requerida.

   ```bash
   docker network inspect elastic-lab-net \
     --format '{{range $id, $c := .Containers}}{{println $c.Name}}{{end}}' \
     | sort
   ```

5. Guarde la evidencia del estado de servicios.

   ```bash
   docker compose ps > work/lab-02-00-01/01-compose-ps.txt
   docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}" \
     > work/lab-02-00-01/01-docker-ps.txt
   ```

**Resultado esperado:**

- Los cuatro contenedores aparecen con estado `Up` o equivalente.
- `es717-lab` publica el puerto `9200`.
- `kibana717-lab` publica el puerto `5601`.
- `es930-lab` publica el puerto `9201`.
- `kibana930-lab` publica el puerto `5602`.
- Los contenedores Elasticsearch y Kibana aparecen en `elastic-lab-net`.

**Verificación:**

Ejecute:

```bash
docker compose ps --status running
```

Debe observar los cuatro servicios requeridos. Si falta alguno o tiene estado distinto de ejecución, resuelva el problema antes de continuar.

---

### Paso 2. Validar conectividad HTTP, cabeceras y API raíz desde el host

**Objetivo:** Comprobar que ambos endpoints Elasticsearch responden desde el host y registrar versión, número de build, distribución y nombre de clúster.

**Instrucciones:**

1. Consulte las cabeceras HTTP del deployment 7.17.29.

   ```bash
   curl -sS -D work/lab-02-00-01/02-es717-headers.txt \
     -o /dev/null \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     http://localhost:9200/
   ```

2. Consulte la API raíz de Elasticsearch 7.17.29 y guarde la respuesta JSON.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     http://localhost:9200/ \
     | tee work/lab-02-00-01/02-es717-root.json \
     | jq '{
         name,
         cluster_name,
         cluster_uuid,
         version: {
           number,
           build_flavor,
           build_type,
           build_hash,
           build_date,
           lucene_version,
           minimum_wire_compatibility_version,
           minimum_index_compatibility_version
         },
         tagline
       }'
   ```

3. Consulte las cabeceras HTTP del deployment 9.3.0.

   ```bash
   curl -sS -D work/lab-02-00-01/02-es930-headers.txt \
     -o /dev/null \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     http://localhost:9201/
   ```

4. Consulte la API raíz de Elasticsearch 9.3.0 y guarde la respuesta JSON.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     http://localhost:9201/ \
     | tee work/lab-02-00-01/02-es930-root.json \
     | jq '{
         name,
         cluster_name,
         cluster_uuid,
         version: {
           number,
           build_flavor,
           build_type,
           build_hash,
           build_date,
           lucene_version,
           minimum_wire_compatibility_version,
           minimum_index_compatibility_version
         },
         tagline
       }'
   ```

5. Extraiga una comparación resumida sin asumir que todos los campos existen en ambas generaciones.

   ```bash
   jq -r '
     [
       .name,
       .cluster_name,
       .version.number,
       (.version.build_flavor // "campo_no_disponible"),
       .version.build_type,
       .version.build_hash,
       .version.build_date,
       .version.lucene_version,
       .version.minimum_wire_compatibility_version,
       .version.minimum_index_compatibility_version
     ] | @tsv
   ' work/lab-02-00-01/02-es717-root.json \
     > work/lab-02-00-01/02-versiones.tsv

   jq -r '
     [
       .name,
       .cluster_name,
       .version.number,
       (.version.build_flavor // "campo_no_disponible"),
       .version.build_type,
       .version.build_hash,
       .version.build_date,
       .version.lucene_version,
       .version.minimum_wire_compatibility_version,
       .version.minimum_index_compatibility_version
     ] | @tsv
   ' work/lab-02-00-01/02-es930-root.json \
     >> work/lab-02-00-01/02-versiones.tsv

   column -t -s $'\t' work/lab-02-00-01/02-versiones.tsv
   ```

**Resultado esperado:**

- Ambas solicitudes devuelven HTTP `200 OK`.
- La cabecera `X-Elastic-Product: Elasticsearch` puede estar presente en la respuesta, según la versión y configuración.
- El endpoint en `9200` informa versión `7.17.29`, nombre de nodo `es717-node-1` y clúster `es717-lab-cluster`.
- El endpoint en `9201` informa versión `9.3.0`, nombre de nodo `es930-node-1` y clúster `es930-lab-cluster`.
- Algunos campos del objeto `version`, por ejemplo `build_flavor`, pueden no estar presentes o cambiar de comportamiento entre generaciones.

**Verificación:**

Ejecute:

```bash
jq -r '.version.number + " | " + .cluster_name + " | " + .name' \
  work/lab-02-00-01/02-es717-root.json \
  work/lab-02-00-01/02-es930-root.json
```

La salida debe contener exactamente las combinaciones de versión, clúster y nodo definidas para el laboratorio.

> **Interpretación:** La API raíz confirma la identidad del servidor que responde. No confirma por sí sola que todos los clientes, agentes, consultas, snapshots o funcionalidades de administración sean compatibles con otra generación.

---

### Paso 3. Validar salud del clúster y topología de nodo único

**Objetivo:** Confirmar que ambos clústeres tienen salud operativa y que su topología corresponde a un deployment de un solo nodo con `discovery.type=single-node`.

**Instrucciones:**

1. Consulte el estado de salud del clúster 7.17.29.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     "http://localhost:9200/_cluster/health?pretty" \
     | tee work/lab-02-00-01/03-es717-cluster-health.json \
     | jq '{
         cluster_name,
         status,
         number_of_nodes,
         number_of_data_nodes,
         active_primary_shards,
         active_shards,
         relocating_shards,
         initializing_shards,
         unassigned_shards,
         timed_out
       }'
   ```

2. Consulte el estado de salud del clúster 9.3.0.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     "http://localhost:9201/_cluster/health?pretty" \
     | tee work/lab-02-00-01/03-es930-cluster-health.json \
     | jq '{
         cluster_name,
         status,
         number_of_nodes,
         number_of_data_nodes,
         active_primary_shards,
         active_shards,
         relocating_shards,
         initializing_shards,
         unassigned_shards,
         timed_out
       }'
   ```

3. Consulte la configuración efectiva de descubrimiento en cada nodo.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_nodes/settings?filter_path=nodes.*.name,nodes.*.settings.discovery.type" \
     | tee work/lab-02-00-01/03-es717-node-settings.json \
     | jq .

   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_nodes/settings?filter_path=nodes.*.name,nodes.*.settings.discovery.type" \
     | tee work/lab-02-00-01/03-es930-node-settings.json \
     | jq .
   ```

4. Obtenga una vista legible de salud para evidencias rápidas.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cat/health?v" \
     | tee work/lab-02-00-01/03-es717-cat-health.txt

   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/health?v" \
     | tee work/lab-02-00-01/03-es930-cat-health.txt
   ```

**Resultado esperado:**

- Cada clúster reporta `number_of_nodes: 1`.
- Cada clúster reporta `number_of_data_nodes: 1` si el nodo tiene rol de datos.
- El estado puede ser `green` si no hay réplicas no asignadas.
- En un deployment de un solo nodo es frecuente observar estado `yellow` si existen índices configurados con réplicas, porque no hay un segundo nodo donde asignarlas.
- `discovery.type` debe indicar `single-node`.

**Verificación:**

Compruebe que no existen shards en inicialización o reubicación:

```bash
jq '{cluster_name, status, number_of_nodes, initializing_shards, relocating_shards, unassigned_shards}' \
  work/lab-02-00-01/03-es717-cluster-health.json \
  work/lab-02-00-01/03-es930-cluster-health.json
```

Registre si el estado es `yellow`. En este laboratorio, `yellow` no implica necesariamente una interrupción; debe justificarse con el número de réplicas no asignadas y la topología de nodo único.

---

### Paso 4. Consultar y comparar licencia y capacidades X-Pack

**Objetivo:** Identificar el tipo, estado y fecha de expiración de la licencia en ambos deployments, y observar diferencias de disponibilidad de APIs entre generaciones.

**Instrucciones:**

1. Consulte la License API en Elasticsearch 7.17.29.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     "http://localhost:9200/_license?pretty" \
     | tee work/lab-02-00-01/04-es717-license.json \
     | jq '.license | {
         uid,
         type,
         status,
         issued_to,
         issuer,
         start_date_in_millis,
         expiry_date_in_millis,
         max_nodes,
         max_resource_units
       }'
   ```

2. Consulte la License API en Elasticsearch 9.3.0.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     "http://localhost:9201/_license?pretty" \
     | tee work/lab-02-00-01/04-es930-license.json \
     | jq '.license | {
         uid,
         type,
         status,
         issued_to,
         issuer,
         start_date_in_millis,
         expiry_date_in_millis,
         max_nodes,
         max_resource_units
       }'
   ```

3. Consulte la API `_xpack` en Elasticsearch 7.17.29.

   ```bash
   curl -sS \
     -o work/lab-02-00-01/04-es717-xpack.json \
     -w "HTTP %{http_code}\n" \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     "http://localhost:9200/_xpack"
   ```

4. Revise las categorías principales disponibles en la respuesta 7.17.29.

   ```bash
   jq 'keys' work/lab-02-00-01/04-es717-xpack.json
   jq '{
     build,
     license,
     features: (
       .features
       | with_entries(.value |= {
           available,
           enabled
         })
     )
   }' work/lab-02-00-01/04-es717-xpack.json
   ```

5. Consulte `_xpack` en Elasticsearch 9.3.0 y registre tanto la respuesta como el código HTTP.

   ```bash
   curl -sS \
     -D work/lab-02-00-01/04-es930-xpack-headers.txt \
     -o work/lab-02-00-01/04-es930-xpack-body.json \
     -w "HTTP %{http_code}\n" \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     "http://localhost:9201/_xpack"
   ```

6. Si la respuesta de 9.3.0 es JSON, muestre su contenido.

   ```bash
   jq . work/lab-02-00-01/04-es930-xpack-body.json
   ```

7. Genere una tabla de comparación de licencia.

   ```bash
   {
     echo -e "deployment\tversion\tlicense_type\tlicense_status\tissued_to"
     printf "es717\t"
     jq -r '[.license.type, .license.status, .license.issued_to] | @tsv' \
       work/lab-02-00-01/04-es717-license.json
     printf "es930\t"
     jq -r '[.license.type, .license.status, .license.issued_to] | @tsv' \
       work/lab-02-00-01/04-es930-license.json
   } > work/lab-02-00-01/04-license-comparison.tsv

   column -t -s $'\t' work/lab-02-00-01/04-license-comparison.tsv
   ```

**Resultado esperado:**

- Las dos llamadas a `GET /_license` devuelven HTTP `200`.
- Cada respuesta contiene un objeto `license` con campos como `type` y `status`.
- En un entorno de laboratorio, la licencia suele ser `basic` y su estado suele ser `active`, aunque debe registrar el valor realmente observado.
- Elasticsearch 7.17.29 normalmente expone información X-Pack mediante `GET /_xpack`.
- En generaciones modernas, `GET /_xpack` puede devolver un error HTTP `404` debido a cambios o retirada de la API. Este resultado es una diferencia de compatibilidad que debe registrarse, no corregirse forzando una API obsoleta.

**Verificación:**

Confirme que ambas licencias tienen estado activo:

```bash
jq -r '.license.status' \
  work/lab-02-00-01/04-es717-license.json \
  work/lab-02-00-01/04-es930-license.json
```

Documente el código HTTP obtenido para `_xpack` en 9.3.0. Si no es `200`, clasifique esta API como “requiere validación previa” o “no recomendada para automatización multi-versión” en la matriz final.

> **Interpretación operativa:** La disponibilidad de una función depende de la versión, licencia, configuración y privilegios del usuario. No debe inferirse que una funcionalidad está habilitada únicamente porque existe una API o porque otro producto compatible implementa un endpoint de nombre similar.

---

### Paso 5. Inspeccionar HTTP, roles de nodo e índices existentes

**Objetivo:** Identificar los datos HTTP expuestos por cada nodo, los roles asignados y el estado de los índices actuales.

**Instrucciones:**

1. Consulte los metadatos HTTP del nodo 7.17.29.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     "http://localhost:9200/_nodes/http?pretty" \
     | tee work/lab-02-00-01/05-es717-nodes-http.json \
     | jq '{
         cluster_name,
         nodes: [
           .nodes[]
           | {
               name,
               roles,
               version,
               http: {
                 bound_address: .http.bound_address,
                 publish_address: .http.publish_address,
                 max_content_length_in_bytes: .http.max_content_length_in_bytes
               }
             }
         ]
       }'
   ```

2. Consulte los metadatos HTTP del nodo 9.3.0.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     "http://localhost:9201/_nodes/http?pretty" \
     | tee work/lab-02-00-01/05-es930-nodes-http.json \
     | jq '{
         cluster_name,
         nodes: [
           .nodes[]
           | {
               name,
               roles,
               version,
               http: {
                 bound_address: .http.bound_address,
                 publish_address: .http.publish_address,
                 max_content_length_in_bytes: .http.max_content_length_in_bytes
               }
             }
         ]
       }'
   ```

3. Liste los índices del clúster 7.17.29.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cat/indices?v&expand_wildcards=all" \
     | tee work/lab-02-00-01/05-es717-cat-indices.txt
   ```

4. Liste los índices del clúster 9.3.0.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/indices?v&expand_wildcards=all" \
     | tee work/lab-02-00-01/05-es930-cat-indices.txt
   ```

5. Obtenga una salida estructurada alternativa para facilitar una comparación posterior.

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cat/indices?format=json&expand_wildcards=all" \
     | jq '.' \
     > work/lab-02-00-01/05-es717-indices.json

   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cat/indices?format=json&expand_wildcards=all" \
     | jq '.' \
     > work/lab-02-00-01/05-es930-indices.json
   ```

**Resultado esperado:**

- Cada respuesta `_nodes/http` contiene un único nodo.
- Los roles pueden diferir entre versiones debido a cambios de roles predeterminados o a la configuración declarada en Compose.
- Las direcciones publicadas dentro de Docker pueden utilizar nombres de contenedor, direcciones privadas de la red Docker o ambas.
- `GET /_cat/indices?v` devuelve una tabla; puede contener índices del sistema, índices creados en la práctica anterior o no contener índices de usuario.
- Es normal que existan índices internos cuyo nombre comience por punto, por ejemplo `.kibana*` o índices de características del sistema.

**Verificación:**

Valide el número de nodos devuelto por la API:

```bash
jq '.nodes | length' work/lab-02-00-01/05-es717-nodes-http.json
jq '.nodes | length' work/lab-02-00-01/05-es930-nodes-http.json
```

Ambos comandos deben devolver `1`.

> **Nota de seguridad:** Los índices de sistema no deben modificarse manualmente durante esta práctica. El objetivo es inventariarlos, no administrar recursos internos de Kibana o Elasticsearch.

---

### Paso 6. Probar conectividad desde la red Docker

**Objetivo:** Confirmar que los nombres DNS internos de los contenedores se resuelven en `elastic-lab-net` y que la autenticación funciona desde un cliente ubicado en la misma red.

**Instrucciones:**

1. Verifique que la red existe.

   ```bash
   docker network inspect elastic-lab-net --format '{{.Name}}'
   ```

2. Ejecute una consulta al endpoint interno de Elasticsearch 7.17.29 desde un contenedor temporal.

   ```bash
   docker run --rm \
     --network elastic-lab-net \
     -e ELASTIC_PASSWORD="${ELASTIC_PASSWORD}" \
     curlimages/curl:8.12.1 \
     -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     http://es717-lab:9200/
   ```

3. Ejecute una consulta al endpoint interno de Elasticsearch 9.3.0 desde un contenedor temporal.

   ```bash
   docker run --rm \
     --network elastic-lab-net \
     -e ELASTIC_PASSWORD="${ELASTIC_PASSWORD}" \
     curlimages/curl:8.12.1 \
     -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     -H "Accept: application/json" \
     http://es930-lab:9200/
   ```

4. Guarde una evidencia resumida de ambas pruebas usando `jq`.

   ```bash
   docker run --rm \
     --network elastic-lab-net \
     -e ELASTIC_PASSWORD="${ELASTIC_PASSWORD}" \
     curlimages/curl:8.12.1 \
     -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     http://es717-lab:9200/ \
     | jq '{name, cluster_name, version: .version.number}' \
     > work/lab-02-00-01/06-docker-to-es717.json

   docker run --rm \
     --network elastic-lab-net \
     -e ELASTIC_PASSWORD="${ELASTIC_PASSWORD}" \
     curlimages/curl:8.12.1 \
     -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     http://es930-lab:9200/ \
     | jq '{name, cluster_name, version: .version.number}' \
     > work/lab-02-00-01/06-docker-to-es930.json
   ```

**Resultado esperado:**

- El contenedor temporal puede resolver `es717-lab` y `es930-lab`.
- Ambas respuestas contienen el nombre de nodo, nombre de clúster y versión correspondiente.
- El acceso interno usa el puerto de contenedor `9200`; no usa los puertos publicados del host `9200` y `9201`.

**Verificación:**

```bash
cat work/lab-02-00-01/06-docker-to-es717.json
cat work/lab-02-00-01/06-docker-to-es930.json
```

Confirme que el nombre de clúster de cada respuesta corresponde al deployment esperado. Si una prueba devuelve información del clúster incorrecto, detenga la práctica y revise la configuración de red y de hosts de Kibana.

> **Concepto aplicado:** La conectividad interna de Docker comprueba DNS y segmentación de red del laboratorio. Esta validación es útil para inventariar clientes potenciales, como Kibana, Beats, Elastic Agent, Logstash o aplicaciones que se ejecuten en la misma red.

---

### Paso 7. Verificar Kibana y la asociación correcta con Elasticsearch

**Objetivo:** Confirmar la versión exacta de cada interfaz Kibana y verificar que cada instancia se comunica exclusivamente con Elasticsearch de su misma generación.

**Instrucciones:**

1. Abra Kibana 7.17.29 en un navegador:

   ```text
   http://localhost:5601
   ```

2. Inicie sesión con el usuario `elastic` y la contraseña definida en `.env`.

3. En Kibana 7.17.29, abra **Dev Tools** y ejecute:

   ```http
   GET /
   ```

4. Registre los valores de:

   - `name`
   - `cluster_name`
   - `version.number`
   - `version.build_hash`

5. En la misma interfaz, identifique la versión de Kibana desde el pie de página, la pantalla **Stack Management** o la página **About**, según la interfaz disponible.

6. Abra Kibana 9.3.0 en una segunda pestaña:

   ```text
   http://localhost:5602
   ```

7. Inicie sesión y abra **Dev Tools**.

8. Ejecute la misma consulta:

   ```http
   GET /
   ```

9. Registre los valores devueltos y la versión mostrada por Kibana.

10. Guarde una evidencia manual en el siguiente archivo. Sustituya los marcadores por los valores observados.

   ```bash
   cat > work/lab-02-00-01/07-kibana-versiones.md <<'EOF'
   # Validación de Kibana y Elasticsearch

   | Interfaz | URL Kibana | Versión Kibana observada | Clúster devuelto por Dev Tools | Nodo devuelto | Versión Elasticsearch |
   |---|---|---|---|---|---|
   | Kibana 7 | http://localhost:5601 | COMPLETAR | COMPLETAR | COMPLETAR | COMPLETAR |
   | Kibana 9 | http://localhost:5602 | COMPLETAR | COMPLETAR | COMPLETAR | COMPLETAR |

   ## Criterio de asociación correcta

   - Kibana 7 debe devolver `es717-lab-cluster` y Elasticsearch `7.17.29`.
   - Kibana 9 debe devolver `es930-lab-cluster` y Elasticsearch `9.3.0`.
   EOF
   ```

11. Edite el archivo con el editor disponible, por ejemplo:

   ```bash
   nano work/lab-02-00-01/07-kibana-versiones.md
   ```

**Resultado esperado:**

- Kibana en `5601` muestra versión 7.17.29 o la versión exacta configurada para la generación 7.
- Su consola Dev Tools devuelve información de `es717-lab-cluster`.
- Kibana en `5602` muestra versión 9.3.0 o la versión exacta configurada para la generación 9.
- Su consola Dev Tools devuelve información de `es930-lab-cluster`.
- No debe existir una asociación cruzada entre Kibana 7.17.29 y Elasticsearch 9.3.0.

**Verificación:**

Revise el archivo creado:

```bash
cat work/lab-02-00-01/07-kibana-versiones.md
```

Debe contener las dos asociaciones correctas.

> **Práctica no recomendada:** No configure Kibana 7.17.29 contra Elasticsearch 9.3.0 para “probar” compatibilidad. Kibana y Elasticsearch deben mantenerse en versiones compatibles, normalmente de la misma versión principal y, para operación predecible, de la misma versión exacta. Una conexión aparente no equivale a soporte oficial.

---

### Paso 8. Construir el inventario de endpoints y matriz de compatibilidad

**Objetivo:** Transformar las observaciones técnicas en un inventario operacional que distinga APIs comunes, elementos que requieren pruebas previas y prácticas no recomendadas.

**Instrucciones:**

1. Cree un inventario de endpoints validados.

   ```bash
   cat > work/lab-02-00-01/08-inventario-endpoints.md <<'EOF'
   # Inventario de endpoints y clientes potenciales

   | Recurso | 7.17.29 | 9.3.0 | Finalidad | Evidencia |
   |---|---|---|---|---|
   | `GET /` | Validado | Validado | Identidad, versión y build | `02-es717-root.json`, `02-es930-root.json` |
   | `GET /_cluster/health` | Validado | Validado | Salud de clúster y shards | `03-*-cluster-health.json` |
   | `GET /_license` | Validado | Validado | Tipo y estado de licencia | `04-*-license.json` |
   | `GET /_xpack` | Completar | Completar | Información histórica de características | `04-*-xpack*` |
   | `GET /_nodes/http` | Validado | Validado | HTTP, nodo y roles | `05-*-nodes-http.json` |
   | `GET /_cat/indices?v` | Validado | Validado | Inventario de índices | `05-*-cat-indices.txt` |
   | Kibana Dev Tools | Validado | Validado | Cliente interactivo y diagnóstico | `07-kibana-versiones.md` |
   | Cliente Docker temporal | Validado | Validado | DNS y conectividad interna | `06-docker-to-*.json` |

   ## Clientes y agentes potenciales

   | Cliente o agente | Riesgo principal | Validación requerida antes de uso |
   |---|---|---|
   | Kibana | Compatibilidad estricta con Elasticsearch | Misma generación y versión compatible |
   | Elastic Agent / Beats | Integraciones, data streams, TLS y versión | Prueba de ingesta y plantillas |
   | Logstash | Plugins, pipelines, autenticación y salida Elasticsearch | Prueba de pipeline y manejo de errores |
   | Aplicación con cliente Elasticsearch | API REST, versión de cliente, serialización y autenticación | Pruebas de indexación, búsqueda y errores |
   | Servicio compatible de terceros | Compatibilidad parcial de API y diferencias de producto | Matriz funcional, seguridad, snapshots y soporte |
   EOF
   ```

2. Cree la matriz de compatibilidad inicial.

   ```bash
   cat > work/lab-02-00-01/08-matriz-compatibilidad.md <<'EOF'
   # Matriz inicial de compatibilidad 7.17.29 y 9.3.0

   | Área o función | Clasificación | Justificación y acción requerida |
   |---|---|---|
   | `GET /` | API común | Validada en ambos deployments; usar para identificación de versión y build. |
   | `GET /_cluster/health` | API común | Validada en ambos deployments; interpretar `yellow` según réplicas y topología. |
   | `GET /_license` | API común | Validada en ambos deployments; revisar permisos y estado de licencia. |
   | `GET /_nodes/http` | API común | Validada en ambos deployments; los campos y roles pueden variar. |
   | `GET /_cat/indices?v` | API común | Validada en ambos deployments; adecuada para inspección humana, no como contrato estable de automatización. |
   | `GET /_xpack` | Requiere validación previa | Puede estar disponible en 7.17 y cambiar o no existir en generaciones modernas. No usar sin control de versión. |
   | Kibana 7.17.29 con Elasticsearch 7.17.29 | Compatible en el laboratorio | Asociación esperada y validada mediante Dev Tools. |
   | Kibana 9.3.0 con Elasticsearch 9.3.0 | Compatible en el laboratorio | Asociación esperada y validada mediante Dev Tools. |
   | Kibana 7.17.29 con Elasticsearch 9.3.0 | No recomendado | Incompatibilidad entre generaciones; no configurar ni usar en producción. |
   | Restauración de snapshots entre generaciones | Requiere validación previa | Depende de compatibilidad de índices, versiones, repositorio y ruta de actualización soportada. |
   | Clientes de aplicación existentes | Requiere validación previa | Un `200` en una API no garantiza soporte para todas las consultas, opciones y errores. |
   | Servicio compatible de terceros | Requiere validación previa | Confirmar Query DSL, seguridad, ILM, snapshots, agentes, dashboards y soporte contractual. |
   EOF
   ```

3. Complete la fila correspondiente a `_xpack` con el resultado real observado en el paso 4. Puede editar el archivo con:

   ```bash
   nano work/lab-02-00-01/08-inventario-endpoints.md
   nano work/lab-02-00-01/08-matriz-compatibilidad.md
   ```

4. Cree una lista breve de riesgos operacionales basada en la lección sobre modelos de servicio.

   ```bash
   cat > work/lab-02-00-01/08-riesgos-operacionales.md <<'EOF'
   # Riesgos operacionales identificados

   1. La compatibilidad de una API REST aislada no garantiza equivalencia funcional entre productos o generaciones.
   2. Un servicio compatible puede diferir en licenciamiento, Query DSL, seguridad, plugins, snapshots, dashboards y calendario de versiones.
   3. En un modelo self-managed, el equipo es responsable de versiones, TLS, backups, capacidad, salud, actualizaciones y procedimientos de recuperación.
   4. La asociación cruzada de Kibana y Elasticsearch de versiones no compatibles es una práctica no recomendada.
   5. Un clúster de un solo nodo no ofrece alta disponibilidad; las réplicas pueden quedar sin asignar.
   EOF
   ```

**Resultado esperado:**

- Existen tres documentos de trabajo:
  - `08-inventario-endpoints.md`
  - `08-matriz-compatibilidad.md`
  - `08-riesgos-operacionales.md`
- La matriz separa explícitamente APIs comunes, funcionalidades sujetas a validación y prácticas no recomendadas.
- La documentación indica que una plataforma compatible no equivale automáticamente a Elasticsearch oficial ni garantiza compatibilidad total con Elastic Stack.

**Verificación:**

```bash
ls -lh work/lab-02-00-01/08-*.md
grep -n "No recomendado\|Requiere validación previa\|API común" \
  work/lab-02-00-01/08-matriz-compatibilidad.md
```

Debe encontrar las tres clasificaciones solicitadas.

---

## Validación y pruebas finales

Ejecute el siguiente bloque para realizar una validación resumida. El bloque no modifica la configuración ni crea índices.

```bash
cd /opt/elastic-labs
set -a
source .env
set +a

echo "=== Versiones y clusters ==="
for endpoint in http://localhost:9200 http://localhost:9201; do
  curl -sS \
    -u "elastic:${ELASTIC_PASSWORD}" \
    "${endpoint}/" \
    | jq -r '"endpoint='"${endpoint}"' cluster=\(.cluster_name) node=\(.name) version=\(.version.number)"'
done

echo
echo "=== Salud ==="
for endpoint in http://localhost:9200 http://localhost:9201; do
  curl -sS \
    -u "elastic:${ELASTIC_PASSWORD}" \
    "${endpoint}/_cluster/health" \
    | jq -r '"cluster=\(.cluster_name) status=\(.status) nodes=\(.number_of_nodes) unassigned=\(.unassigned_shards)"'
done

echo
echo "=== Licencias ==="
for endpoint in http://localhost:9200 http://localhost:9201; do
  curl -sS \
    -u "elastic:${ELASTIC_PASSWORD}" \
    "${endpoint}/_license" \
    | jq -r '"type=\(.license.type) status=\(.license.status) issued_to=\(.license.issued_to)"'
done

echo
echo "=== Evidencias generadas ==="
find work/lab-02-00-01 -maxdepth 1 -type f -printf '%f\n' | sort
```

### Criterios de aceptación

La práctica se considera completada cuando se cumplen todos los criterios:

- [ ] Los cuatro contenedores obligatorios están en ejecución.
- [ ] Elasticsearch 7.17.29 responde por `localhost:9200`.
- [ ] Elasticsearch 9.3.0 responde por `localhost:9201`.
- [ ] Se registraron las respuestas de `GET /`, `GET /_cluster/health`, `GET /_license`, `GET /_xpack`, `GET /_nodes/http` y `GET /_cat/indices?v`.
- [ ] Se identificó y documentó el comportamiento real de `_xpack` en ambas generaciones.
- [ ] Se comprobó conectividad desde un cliente temporal en `elastic-lab-net`.
- [ ] Kibana 7 está asociado a `es717-lab-cluster`.
- [ ] Kibana 9 está asociado a `es930-lab-cluster`.
- [ ] Se creó una matriz que clasifica APIs comunes, validaciones previas y prácticas no recomendadas.
- [ ] No se ejecutó `docker compose down -v` ni se eliminaron volúmenes persistentes.

## Resolución de problemas

### Problema 1: `curl` devuelve HTTP 401 o `missing authentication credentials`

**Síntomas:**

- La solicitud a `http://localhost:9200/` o `http://localhost:9201/` devuelve `401 Unauthorized`.
- Aparece el mensaje `missing authentication credentials for REST request`.
- Los comandos con `-u "elastic:${ELASTIC_PASSWORD}"` no funcionan.

**Causa probable:**

La variable `ELASTIC_PASSWORD` no fue cargada desde `/opt/elastic-labs/.env`, el archivo contiene un nombre de variable diferente o la contraseña no coincide con la configurada en los contenedores.

**Solución:**

1. Verifique permisos y existencia del archivo:

   ```bash
   ls -l /opt/elastic-labs/.env
   ```

2. Cargue las variables nuevamente:

   ```bash
   cd /opt/elastic-labs
   set -a
   source .env
   set +a
   ```

3. Compruebe que la variable existe sin mostrar el secreto:

   ```bash
   test -n "${ELASTIC_PASSWORD}" && echo "Variable cargada"
   ```

4. Revise el nombre de variable definido en `.env`. Si el archivo usa otro nombre, adapte los comandos o exporte temporalmente la variable correcta.

5. No cambie credenciales ni recree volúmenes durante esta práctica sin autorización del instructor.

### Problema 2: El clúster aparece en estado `yellow` y existen shards sin asignar

**Síntomas:**

- `GET /_cluster/health` devuelve `"status": "yellow"`.
- El campo `unassigned_shards` es mayor que cero.
- `_cat/health?v` muestra shards no asignados.

**Causa probable:**

El laboratorio usa un clúster de un solo nodo. Si algún índice tiene al menos una réplica configurada, Elasticsearch no puede asignar esa réplica al mismo nodo donde se encuentra el shard primario.

**Solución:**

1. Confirme la topología de un solo nodo:

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     http://localhost:9200/_cluster/health \
     | jq '{cluster_name, number_of_nodes, status, unassigned_shards}'
   ```

2. Identifique los shards no asignados:

   ```bash
   curl -sS \
     -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/_cat/shards?v"
   ```

3. Registre el estado como una limitación esperada de disponibilidad en un entorno de laboratorio de nodo único.

4. No reduzca réplicas de índices de sistema ni cambie configuraciones globales durante esta práctica. La finalidad es diagnosticar y documentar la condición, no alterar la línea base.

## Limpieza

No elimine contenedores, redes ni volúmenes persistentes. La siguiente práctica reutilizará los deployments y las evidencias generadas.

Puede cerrar las pestañas de Kibana y eliminar únicamente contenedores temporales si alguno quedó detenido:

```bash
docker container prune -f
```

Compruebe que los servicios principales continúan en ejecución:

```bash
cd /opt/elastic-labs
docker compose ps
```

Conserve el directorio de evidencias:

```text
/opt/elastic-labs/work/lab-02-00-01
```

## Resumen

En esta práctica se validó la identidad técnica, conectividad, salud, licencia y exposición HTTP de Elasticsearch 7.17.29 y 9.3.0. También se comprobó que cada Kibana está asociado al deployment de su misma generación y se documentaron diferencias de comportamiento, particularmente en APIs que pueden cambiar o desaparecer entre versiones.

La línea base obtenida permite tomar decisiones informadas antes de incorporar datos, agentes, clientes de aplicación, snapshots o procedimientos de actualización. El hallazgo principal es que conectividad HTTP satisfactoria no equivale a compatibilidad completa: versiones, licencias, APIs, seguridad, clientes, snapshots y operaciones deben evaluarse explícitamente.

### Recursos sugeridos

- Documentación de Elasticsearch: API raíz, Cluster Health API, License API y Nodes Info API.
- Documentación de compatibilidad entre Elasticsearch y Kibana.
- Documentación de Elastic Cloud Hosted y modelo de responsabilidad compartida.
- Documentación de migración y compatibilidad de snapshots antes de cualquier actualización entre versiones principales.
