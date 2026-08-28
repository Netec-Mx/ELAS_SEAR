# Investigar errores y tendencias mediante Query DSL, KQL y ES\|QL.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 100 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica investigará errores, advertencias y degradaciones de servicio en el data stream `logs-acme.api-default` del clúster Elasticsearch 9.3.0. Primero verificará la calidad y cobertura mínima de los eventos disponibles; si el conjunto no contiene la variedad necesaria, agregará eventos adicionales mediante el pipeline existente `logs-acme-api-v1`.

Posteriormente utilizará Elasticsearch Query DSL para aplicar consultas `term`, `match`, `range`, `exists` y `bool`, así como agregaciones para identificar servicios afectados, códigos HTTP, tendencias temporales y duración media. Finalmente reproducirá investigaciones exploratorias en Kibana mediante KQL y construirá resúmenes con ES|QL.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Verificar que `logs-acme.api-default` contiene eventos con servicio, ambiente, severidad, códigos HTTP, duración y marcas de tiempo.
- [ ] Ejecutar investigaciones con Query DSL usando `term`, `match`, `range`, `exists` y combinaciones `bool`.
- [ ] Diferenciar el uso de campos `text` y `keyword` en búsquedas exactas y búsquedas analizadas.
- [ ] Construir agregaciones `terms`, `filter`, `date_histogram` y `avg` para detectar tendencias operativas.
- [ ] Comparar el uso de Query DSL, KQL y ES|QL, incluyendo la validación de compatibilidad frente a Elasticsearch 7.17.29.

## Prerrequisitos

### Conocimientos requeridos

- Prácticas 1 a 4 completadas.
- Uso básico de Docker Compose, `curl`, `jq` y terminal Linux.
- Conocimiento básico de JSON, ECS, filtros temporales y códigos HTTP.
- Comprensión de los conceptos de `keyword`, `text`, `@timestamp`, `event.duration` y niveles de log.

### Accesos y estado requerido

- Data stream `logs-acme.api-default` creado y poblado.
- Pipeline de ingestión `logs-acme-api-v1` disponible.
- Acceso a:
  - Elasticsearch 9.3.0: `http://localhost:9201`
  - Kibana 9.3.0: `http://localhost:5602`
  - Elasticsearch 7.17.29: `http://localhost:9200`
- Usuario de laboratorio: `elastic`.
- Archivo de credenciales: `/opt/elastic-labs/.env`.
- Directorio de trabajo: `/opt/elastic-labs/work`.

> **Importante:** No ejecute `docker compose down -v`. Esta práctica utiliza el estado persistente de las prácticas anteriores.

## Entorno de laboratorio

| Componente | Valor esperado |
|---|---|
| Directorio raíz | `/opt/elastic-labs` |
| Red Docker | `elastic-lab-net` |
| Contenedor Elasticsearch 9.3.0 | `es930-lab` |
| Contenedor Kibana 9.3.0 | `kibana930-lab` |
| Endpoint Elasticsearch 9.3.0 | `http://localhost:9201` |
| Endpoint Kibana 9.3.0 | `http://localhost:5602` |
| Data stream | `logs-acme.api-default` |
| Pipeline | `logs-acme-api-v1` |
| Volumen persistente ES 9.3.0 | `es930-data` |

### Preparación inicial

1. Abra una terminal y acceda al directorio raíz:

   ```bash
   cd /opt/elastic-labs
   ```

2. Verifique los permisos del archivo de variables de entorno:

   ```bash
   stat -c '%a %n' /opt/elastic-labs/.env
   ```

   Si no muestra permisos `600`, corríjalos:

   ```bash
   chmod 0600 /opt/elastic-labs/.env
   ```

3. Cargue las credenciales en la sesión actual:

   ```bash
   set -a
   . /opt/elastic-labs/.env
   set +a
   ```

4. Cree el directorio de evidencias de la práctica:

   ```bash
   mkdir -p /opt/elastic-labs/work/lab-05-00-01
   cd /opt/elastic-labs/work/lab-05-00-01
   ```

5. Verifique que los contenedores requeridos estén activos:

   ```bash
   docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' \
     | grep -E 'es930-lab|kibana930-lab|es717-lab|kibana717-lab'
   ```

---

## Procedimiento paso a paso

### Paso 1. Validar conectividad, salud y recursos del data stream

**Objetivo:** Confirmar que Elasticsearch 9.3.0 está disponible, que el data stream existe y que contiene documentos.

**Instrucciones**

1. Consulte la salud del clúster 9.3.0:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_cluster/health?pretty" \
     | tee 01-cluster-health.json
   ```

2. Verifique la identidad del nodo y la versión:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/" \
     | jq '{cluster_name, name, version: .version.number}' \
     | tee 02-cluster-info.json
   ```

3. Confirme la existencia del data stream:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_data_stream/logs-acme.api-default?pretty" \
     | tee 03-data-stream.json
   ```

4. Obtenga el total aproximado de documentos:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/logs-acme.api-default/_count?pretty" \
     | tee 04-document-count.json
   ```

5. Inspeccione diez eventos recientes:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 10,
       "sort": [
         {
           "@timestamp": {
             "order": "desc"
           }
         }
       ],
       "_source": [
         "@timestamp",
         "service.name",
         "service.environment",
         "log.level",
         "http.response.status_code",
         "event.duration",
         "message",
         "error.message",
         "trace.id"
       ],
       "query": {
         "match_all": {}
       }
     }' | tee 05-recent-events.json
   ```

**Resultado esperado**

- El clúster presenta estado `yellow` o `green`. En un clúster de un solo nodo, `yellow` puede ser normal si existen réplicas no asignadas.
- El nombre de clúster es `es930-lab-cluster`.
- El nodo se identifica como `es930-node-1`.
- El data stream `logs-acme.api-default` existe.
- La respuesta `_count` muestra al menos algunos documentos.

**Verificación**

Revise los eventos almacenados:

```bash
jq '.hits.hits[]._source' 05-recent-events.json
```

Debe observar campos ECS relevantes, especialmente `@timestamp`, `service.name`, `service.environment`, `log.level`, `message` y, en eventos HTTP, `http.response.status_code`.

---

### Paso 2. Evaluar la calidad y cobertura del conjunto de datos

**Objetivo:** Determinar si los datos existentes incluyen suficientes servicios, ambientes, severidades y códigos HTTP para realizar la investigación.

**Instrucciones**

1. Consulte la distribución por servicio, ambiente, severidad y código HTTP:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 0,
       "aggs": {
         "servicios": {
           "terms": {
             "field": "service.name",
             "size": 20
           }
         },
         "ambientes": {
           "terms": {
             "field": "service.environment",
             "size": 20
           }
         },
         "severidades": {
           "terms": {
             "field": "log.level",
             "size": 20
           }
         },
         "codigos_http": {
           "terms": {
             "field": "http.response.status_code",
             "size": 20
           }
         }
       }
     }' | tee 06-data-quality-distribution.json
   ```

2. Identifique documentos que contienen un mensaje de error:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 0,
       "query": {
         "exists": {
           "field": "error.message"
         }
       }
     }' | tee 07-error-message-count.json
   ```

3. Identifique eventos que no contienen `trace.id`:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 0,
       "query": {
         "bool": {
           "must_not": [
             {
               "exists": {
                 "field": "trace.id"
               }
             }
           ]
         }
       }
     }' | tee 08-missing-trace-id-count.json
   ```

4. Revise los resultados resumidos:

   ```bash
   jq '.aggregations' 06-data-quality-distribution.json
   jq '.hits.total' 07-error-message-count.json
   jq '.hits.total' 08-missing-trace-id-count.json
   ```

**Resultado esperado**

El conjunto debería incluir, como mínimo:

- Dos o más servicios.
- Al menos los ambientes `production` y `staging` o equivalentes.
- Niveles `info`, `warn` y `error`.
- Códigos 2xx, 4xx y 5xx.
- Algunos documentos con `error.message`.
- Algunos documentos con `trace.id`.

**Verificación**

Si faltan varios de estos elementos, continúe con el Paso 3. Si el conjunto ya contiene una cobertura suficiente, omita la carga adicional y pase al Paso 4.

---

### Paso 3. Incorporar un lote adicional de eventos, si es necesario

**Objetivo:** Agregar eventos de prueba con variedad de servicios, ambientes, severidades, errores HTTP y duraciones usando el pipeline existente.

**Instrucciones**

1. Verifique que el pipeline existe y guarde su definición como evidencia:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/_ingest/pipeline/logs-acme-api-v1?pretty" \
     | tee 09-pipeline-definition.json
   ```

2. Cree un archivo NDJSON con eventos de ejemplo. Las marcas temporales se distribuyen durante los últimos 55 minutos.

   ```bash
   T0=$(date -u -d '55 minutes ago' +'%Y-%m-%dT%H:%M:%SZ')
   T1=$(date -u -d '45 minutes ago' +'%Y-%m-%dT%H:%M:%SZ')
   T2=$(date -u -d '35 minutes ago' +'%Y-%m-%dT%H:%M:%SZ')
   T3=$(date -u -d '25 minutes ago' +'%Y-%m-%dT%H:%M:%SZ')
   T4=$(date -u -d '15 minutes ago' +'%Y-%m-%dT%H:%M:%SZ')
   T5=$(date -u -d '5 minutes ago' +'%Y-%m-%dT%H:%M:%SZ')

   cat > additional-events.ndjson <<EOF
   {"create":{}}
   {"@timestamp":"$T0","service":{"name":"payments-api","environment":"production"},"log":{"level":"info"},"http":{"response":{"status_code":200}},"event":{"duration":120000000},"message":"Payment accepted","trace":{"id":"trace-pay-001"},"url":{"path":"/payments"}}
   {"create":{}}
   {"@timestamp":"$T1","service":{"name":"payments-api","environment":"production"},"log":{"level":"warn"},"http":{"response":{"status_code":429}},"event":{"duration":850000000},"message":"Rate limit reached","trace":{"id":"trace-pay-002"},"url":{"path":"/payments"}}
   {"create":{}}
   {"@timestamp":"$T2","service":{"name":"payments-api","environment":"production"},"log":{"level":"error"},"http":{"response":{"status_code":500}},"event":{"duration":3200000000},"message":"Database connection timeout while processing payment","error":{"type":"DatabaseTimeout","message":"database connection timeout"},"trace":{"id":"trace-pay-003"},"url":{"path":"/payments"}}
   {"create":{}}
   {"@timestamp":"$T2","service":{"name":"payments-api","environment":"production"},"log":{"level":"error"},"http":{"response":{"status_code":503}},"event":{"duration":4100000000},"message":"Payment dependency unavailable","error":{"type":"DependencyUnavailable","message":"connection timeout to ledger service"},"trace":{"id":"trace-pay-004"},"url":{"path":"/payments"}}
   {"create":{}}
   {"@timestamp":"$T3","service":{"name":"orders-api","environment":"production"},"log":{"level":"error"},"http":{"response":{"status_code":502}},"event":{"duration":2800000000},"message":"Database connection refused","error":{"type":"DatabaseConnectionError","message":"database connection refused"},"trace":{"id":"trace-order-001"},"url":{"path":"/orders"}}
   {"create":{}}
   {"@timestamp":"$T3","service":{"name":"orders-api","environment":"production"},"log":{"level":"error"},"http":{"response":{"status_code":500}},"event":{"duration":3600000000},"message":"Order processing failed","error":{"type":"ProcessingError","message":"database connection timeout"},"trace":{"id":"trace-order-002"},"url":{"path":"/orders"}}
   {"create":{}}
   {"@timestamp":"$T4","service":{"name":"catalog-api","environment":"staging"},"log":{"level":"warn"},"http":{"response":{"status_code":404}},"event":{"duration":180000000},"message":"Product not found","trace":{"id":"trace-cat-001"},"url":{"path":"/catalog/items/999"}}
   {"create":{}}
   {"@timestamp":"$T4","service":{"name":"catalog-api","environment":"production"},"log":{"level":"error"},"http":{"response":{"status_code":504}},"event":{"duration":5200000000},"message":"Upstream catalog timeout","error":{"type":"GatewayTimeout","message":"connection timeout to inventory service"},"trace":{"id":"trace-cat-002"},"url":{"path":"/catalog"}}
   {"create":{}}
   {"@timestamp":"$T5","service":{"name":"payments-api","environment":"production"},"log":{"level":"error"},"http":{"response":{"status_code":500}},"event":{"duration":3900000000},"message":"Database connection timeout while processing payment","error":{"type":"DatabaseTimeout","message":"database connection timeout"},"trace":{"id":"trace-pay-005"},"url":{"path":"/payments"}}
   {"create":{}}
   {"@timestamp":"$T5","service":{"name":"orders-api","environment":"production"},"log":{"level":"info"},"http":{"response":{"status_code":200}},"event":{"duration":220000000},"message":"Order created","url":{"path":"/orders"}}
   {"create":{}}
   {"@timestamp":"$T5","service":{"name":"orders-api","environment":"production"},"log":{"level":"error"},"http":{"response":{"status_code":500}},"event":{"duration":90000000},"message":"Health endpoint failed","error":{"type":"HealthCheckError","message":"database connection timeout"},"url":{"path":"/health"}}
   EOF
   ```

3. Cargue los eventos a través del pipeline:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/x-ndjson' \
     -X POST \
     "http://localhost:9201/logs-acme.api-default/_bulk?pipeline=logs-acme-api-v1" \
     --data-binary @additional-events.ndjson \
     | tee 10-bulk-load-response.json
   ```

4. Valide que la respuesta no contiene errores:

   ```bash
   jq '.errors' 10-bulk-load-response.json
   ```

**Resultado esperado**

La respuesta muestra:

```json
false
```

para el atributo `errors`. Los documentos nuevos quedan indexados en el data stream y pasan por `logs-acme-api-v1`.

**Verificación**

Ejecute nuevamente la consulta de distribución del Paso 2 y confirme que existen códigos HTTP 200, 404, 429, 500, 502, 503 y 504.

> Si el pipeline exige campos de entrada adicionales definidos en prácticas anteriores, adapte el archivo NDJSON a su contrato de entrada y ejecute primero una prueba con `POST /_ingest/pipeline/logs-acme-api-v1/_simulate`.

---

### Paso 4. Investigar errores 5xx mediante Query DSL

**Objetivo:** Localizar errores HTTP 5xx recientes en producción utilizando filtros estructurados y ordenar los resultados por fecha.

**Instrucciones**

1. Ejecute una búsqueda de errores 5xx ocurridos en las últimas dos horas:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 50,
       "track_total_hits": true,
       "sort": [
         {
           "@timestamp": {
             "order": "desc"
           }
         }
       ],
       "_source": [
         "@timestamp",
         "service.name",
         "service.environment",
         "log.level",
         "http.response.status_code",
         "event.duration",
         "message",
         "error.type",
         "error.message",
         "trace.id",
         "url.path"
       ],
       "query": {
         "bool": {
           "filter": [
             {
               "term": {
                 "service.environment": "production"
               }
             },
             {
               "range": {
                 "@timestamp": {
                   "gte": "now-2h",
                   "lte": "now"
                 }
               }
             },
             {
               "range": {
                 "http.response.status_code": {
                   "gte": 500,
                   "lt": 600
                 }
               }
             }
           ]
         }
       }
     }' | tee 11-production-5xx.json
   ```

2. Muestre un resumen legible de los resultados:

   ```bash
   jq -r '.hits.hits[]._source |
     [
       ."@timestamp",
       .service.name,
       .log.level,
       .http.response.status_code,
       .event.duration,
       .message
     ] | @tsv' 11-production-5xx.json
   ```

3. Localice eventos con `error.message` usando `exists`:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 20,
       "query": {
         "exists": {
           "field": "error.message"
         }
       },
       "sort": [
         {
           "@timestamp": {
             "order": "desc"
           }
         }
       ]
     }' | tee 12-events-with-error-message.json
   ```

**Resultado esperado**

- Los resultados de `11-production-5xx.json` incluyen documentos con códigos entre 500 y 599.
- No se calcula relevancia para los filtros estructurados; por ello, los valores `_score` pueden ser `0.0`.
- `12-events-with-error-message.json` contiene documentos enriquecidos con descripciones de error.

**Verificación**

Confirme el total exacto de 5xx:

```bash
jq '.hits.total' 11-production-5xx.json
```

Explique en su informe por qué `filter` es apropiado en este caso: servicio, ambiente, rango temporal y código HTTP son condiciones objetivas, no criterios de relevancia textual.

---

### Paso 5. Combinar `bool`, `term`, `match`, `range`, `exists` y exclusiones

**Objetivo:** Investigar fallos de conexión a base de datos en `orders-api`, excluyendo tráfico de salud.

**Instrucciones**

1. Ejecute la consulta compuesta:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 50,
       "track_total_hits": true,
       "sort": [
         {
           "@timestamp": {
             "order": "desc"
           }
         }
       ],
       "_source": [
         "@timestamp",
         "service.name",
         "service.environment",
         "log.level",
         "http.response.status_code",
         "message",
         "error.message",
         "trace.id",
         "url.path"
       ],
       "query": {
         "bool": {
           "filter": [
             {
               "term": {
                 "service.name": "orders-api"
               }
             },
             {
               "term": {
                 "service.environment": "production"
               }
             },
             {
               "term": {
                 "log.level": "error"
               }
             },
             {
               "range": {
                 "@timestamp": {
                   "gte": "now-2h",
                   "lte": "now"
                 }
               }
             },
             {
               "exists": {
                 "field": "trace.id"
               }
             }
           ],
           "must": [
             {
               "match": {
                 "error.message": {
                   "query": "database connection",
                   "operator": "and"
                 }
               }
             }
           ],
           "must_not": [
             {
               "term": {
                 "url.path.keyword": "/health"
               }
             }
           ]
         }
       }
     }' | tee 13-orders-database-investigation.json
   ```

2. Ejecute una variante progresiva sin la exclusión `/health`:

   ```bash
   sed 's/,"must_not":\[{"term":{"url.path.keyword":"\/health"}}\]//' \
     13-orders-database-investigation.json >/dev/null
   ```

   En lugar de modificar la evidencia anterior, vuelva a ejecutar la consulta desde Dev Tools eliminando temporalmente el bloque `must_not`. Compare ambos resultados.

3. Consulte el mapping de los campos relevantes:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/logs-acme.api-default/_mapping/field/message,error.message,url.path?pretty" \
     | tee 14-field-mappings.json
   ```

4. Pruebe una búsqueda analizada con `match` sobre `message`:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 10,
       "query": {
         "match": {
           "message": "database connection timeout"
         }
       }
     }' | tee 15-match-message.json
   ```

5. Si el mapping muestra el subcampo `message.keyword`, pruebe una coincidencia exacta:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 10,
       "query": {
         "term": {
           "message.keyword": "Database connection timeout while processing payment"
         }
       }
     }' | tee 16-term-message-keyword.json
   ```

**Resultado esperado**

- La consulta compuesta devuelve errores de `orders-api` de producción con `trace.id`.
- Los eventos de `/health` quedan excluidos.
- `match` localiza mensajes que contienen términos analizados.
- `term` sobre `message.keyword`, si está disponible, exige coincidencia exacta del valor completo.

**Verificación**

Documente esta diferencia:

| Tipo de búsqueda | Campo recomendado | Razón |
|---|---|---|
| Valor exacto estructurado | `service.name`, `log.level`, `url.path.keyword` | Son categorías o valores no analizados. |
| Texto de error | `message`, `error.message` con `match` | El texto se analiza en términos. |
| Mensaje completo exacto | `message.keyword` con `term` | Requiere igualdad exacta y depende de que el subcampo exista. |

---

### Paso 6. Construir agregaciones para identificar tendencia e impacto

**Objetivo:** Determinar qué servicio genera más errores, en qué intervalo aumentan los 5xx y cuál es la duración media por código HTTP.

**Instrucciones**

1. Ejecute una agregación por servicio y código HTTP para los eventos 5xx de producción:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 0,
       "query": {
         "bool": {
           "filter": [
             {
               "term": {
                 "service.environment": "production"
               }
             },
             {
               "range": {
                 "@timestamp": {
                   "gte": "now-2h",
                   "lte": "now"
                 }
               }
             }
           ]
         }
       },
       "aggs": {
         "por_servicio": {
           "terms": {
             "field": "service.name",
             "size": 10
           },
           "aggs": {
             "errores_5xx": {
               "filter": {
                 "range": {
                   "http.response.status_code": {
                     "gte": 500,
                     "lt": 600
                   }
                 }
               },
               "aggs": {
                 "por_codigo": {
                   "terms": {
                     "field": "http.response.status_code",
                     "size": 10
                   }
                 }
               }
             }
           }
         }
       }
     }' | tee 17-errors-by-service-and-status.json
   ```

2. Cree un histograma temporal de errores 5xx en intervalos de diez minutos:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 0,
       "query": {
         "bool": {
           "filter": [
             {
               "range": {
                 "@timestamp": {
                   "gte": "now-2h",
                   "lte": "now"
                 }
               }
             },
             {
               "range": {
                 "http.response.status_code": {
                   "gte": 500,
                   "lt": 600
                 }
               }
             }
           ]
         }
       },
       "aggs": {
         "tendencia_5xx": {
           "date_histogram": {
             "field": "@timestamp",
             "fixed_interval": "10m",
             "min_doc_count": 0
           },
           "aggs": {
             "por_servicio": {
               "terms": {
                 "field": "service.name",
                 "size": 10
               }
             }
           }
         }
       }
     }' | tee 18-5xx-time-trend.json
   ```

3. Calcule duración media por código HTTP. `event.duration` se expresa en nanosegundos:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 0,
       "aggs": {
         "por_codigo_http": {
           "terms": {
             "field": "http.response.status_code",
             "size": 20
           },
           "aggs": {
             "duracion_media_ns": {
               "avg": {
                 "field": "event.duration"
               }
             }
           }
         }
       }
     }' | tee 19-average-duration-by-status.json
   ```

4. Presente los resultados en formato legible:

   ```bash
   jq '.aggregations.por_servicio.buckets' \
     17-errors-by-service-and-status.json

   jq '.aggregations.tendencia_5xx.buckets[] |
     {intervalo: .key_as_string, eventos_5xx: .doc_count}' \
     18-5xx-time-trend.json

   jq '.aggregations.por_codigo_http.buckets[] |
     {
       codigo_http: .key,
       eventos: .doc_count,
       duracion_media_ns: .duracion_media_ns.value,
       duracion_media_ms: (.duracion_media_ns.value / 1000000)
     }' 19-average-duration-by-status.json
   ```

**Resultado esperado**

- La agregación por servicio permite identificar el servicio con mayor cantidad de errores 5xx.
- El `date_histogram` muestra los intervalos con mayor volumen de errores.
- Los códigos 5xx normalmente presentan una duración media superior a rutas exitosas, según los eventos disponibles.

**Verificación**

Responda en su informe técnico:

1. ¿Qué servicio tiene el mayor número de errores 5xx?
2. ¿En qué intervalo de diez minutos se concentra el mayor número de 5xx?
3. ¿Qué código HTTP tiene la mayor duración media?
4. ¿La duración se interpretó correctamente en nanosegundos y milisegundos?

---

### Paso 7. Reproducir filtros exploratorios con KQL en Kibana

**Objetivo:** Usar KQL en Discover para explorar los mismos datos sin escribir una solicitud REST completa.

**Instrucciones**

1. Abra Kibana 9.3.0 en el navegador:

   ```text
   http://localhost:5602
   ```

2. Inicie sesión con:

   - Usuario: `elastic`
   - Contraseña: la declarada en `/opt/elastic-labs/.env`

3. Acceda a **Discover**.

4. Seleccione o cree una vista de datos que cubra:

   ```text
   logs-acme.api-default
   ```

5. Ajuste el selector temporal a **Últimas 2 horas**. Si los eventos se cargaron con fechas generadas manualmente, seleccione un rango absoluto que los incluya.

6. Ejecute el siguiente filtro KQL:

   ```kql
   service.environment : "production" and
   http.response.status_code >= 500 and
   http.response.status_code < 600
   ```

7. Agregue las columnas:

   - `@timestamp`
   - `service.name`
   - `service.environment`
   - `log.level`
   - `http.response.status_code`
   - `event.duration`
   - `error.message`
   - `trace.id`
   - `url.path`

8. Ejecute una investigación específica de errores de conexión:

   ```kql
   service.name : "orders-api" and
   service.environment : "production" and
   log.level : "error" and
   error.message : "database connection" and
   trace.id : *
   ```

9. Excluya los eventos de salud:

   ```kql
   service.name : "orders-api" and
   service.environment : "production" and
   log.level : "error" and
   error.message : "database connection" and
   trace.id : * and
   not url.path : "/health"
   ```

10. Guarde la búsqueda con el nombre:

   ```text
   Lab 05 - Orders API - Errores de base de datos
   ```

**Resultado esperado**

Discover muestra los eventos coincidentes y permite ordenar visualmente por `@timestamp`. La exclusión `not url.path : "/health"` reduce el ruido asociado a comprobaciones de salud.

**Verificación**

Capture una evidencia visual o registre en el informe:

- Filtro KQL utilizado.
- Rango temporal seleccionado.
- Número de documentos encontrados.
- Servicio, código HTTP y mensaje de error predominante.

---

### Paso 8. Consultar tendencias con ES|QL y contrastar compatibilidad

**Objetivo:** Crear resúmenes analíticos con ES|QL y documentar la diferencia de compatibilidad respecto a Elasticsearch 7.17.29.

**Instrucciones**

1. En Kibana 9.3.0, abra **Dev Tools**.

2. Ejecute una consulta ES|QL para resumir errores 5xx por servicio y código HTTP:

   ```esql
   FROM logs-acme.api-default
   | WHERE service.environment == "production"
     AND http.response.status_code >= 500
     AND http.response.status_code < 600
   | STATS eventos = COUNT(*),
           duracion_media_ns = AVG(event.duration)
     BY service.name, http.response.status_code
   | SORT eventos DESC
   ```

3. Ejecute una consulta ES|QL de tendencia temporal:

   ```esql
   FROM logs-acme.api-default
   | WHERE http.response.status_code >= 500
     AND http.response.status_code < 600
   | STATS errores_5xx = COUNT(*)
     BY intervalo = DATE_TRUNC(10 minutes, @timestamp),
        servicio = service.name
   | SORT intervalo ASC, errores_5xx DESC
   ```

4. Ejecute una consulta para identificar errores con duración superior a dos segundos. Recuerde que dos segundos equivalen a `2000000000` nanosegundos:

   ```esql
   FROM logs-acme.api-default
   | WHERE event.duration > 2000000000
   | KEEP @timestamp, service.name, service.environment,
          http.response.status_code, event.duration, error.message
   | SORT event.duration DESC
   | LIMIT 20
   ```

5. Guarde los resultados relevantes mediante capturas de pantalla o copie las consultas en el informe.

6. Verifique la versión del clúster heredado 7.17.29:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9200/" \
     | jq '{cluster_name, version: .version.number}' \
     | tee 20-es717-version.json
   ```

7. Registre la validación de compatibilidad:

   - Elasticsearch 7.17.29 no debe asumirse compatible con la experiencia y sintaxis de ES|QL disponible en Kibana/Elasticsearch 9.3.0.
   - Antes de adoptar una consulta ES|QL en un entorno heredado, valide la documentación y las capacidades reales de la versión objetivo.
   - Para el clúster 7.17.29, utilice Query DSL o KQL cuando sea necesario mantener compatibilidad con la plataforma heredada.

**Resultado esperado**

- ES|QL devuelve una tabla resumida sin necesidad de definir explícitamente una estructura JSON de agregaciones.
- La consulta temporal identifica intervalos y servicios asociados a aumentos de errores.
- La verificación de versión confirma que Elasticsearch 7.17.29 pertenece a una generación distinta y requiere validación específica antes de usar ES|QL.

**Verificación**

Complete la siguiente comparación en el informe:

| Necesidad | Lenguaje recomendado | Justificación |
|---|---|---|
| Automatización REST, alertas y control exacto de la consulta | Query DSL | Permite cuerpos JSON completos, filtros, ordenamiento y agregaciones detalladas. |
| Exploración rápida en Discover | KQL | Es legible, interactivo y está integrado en la interfaz de Kibana. |
| Resumen tabular, agrupaciones y exploración analítica | ES\|QL | Reduce la complejidad de agregaciones para análisis tabulares, sujeto a compatibilidad de versión. |
| Entorno Elasticsearch 7.17.29 | Query DSL o KQL | La disponibilidad de ES\|QL debe verificarse antes de adoptarla. |

---

## Validación y pruebas

Realice la siguiente validación final antes de cerrar la práctica.

1. Confirme que el data stream contiene documentos:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/logs-acme.api-default/_count" | jq
   ```

2. Confirme que existen eventos 5xx:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 0,
       "query": {
         "range": {
           "http.response.status_code": {
             "gte": 500,
             "lt": 600
           }
         }
       }
     }' | jq '.hits.total'
   ```

3. Confirme que existen documentos con `error.message`:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 0,
       "query": {
         "exists": {
           "field": "error.message"
         }
       }
     }' | jq '.hits.total'
   ```

4. Verifique que se generaron evidencias locales:

   ```bash
   find /opt/elastic-labs/work/lab-05-00-01 \
     -maxdepth 1 -type f -printf '%f\n' | sort
   ```

5. Elabore un informe breve llamado `informe-investigacion.md`:

   ```bash
   cat > informe-investigacion.md <<'EOF'
   # Informe de investigación - Lab 05-00-01

   ## Evidencia
   - Data stream investigado: logs-acme.api-default
   - Periodo analizado:
   - Total de eventos 5xx:
   - Archivos de evidencia:

   ## Hipótesis
   - Hipótesis inicial:
   - Servicios potencialmente afectados:

   ## Diagnóstico
   - Servicio con mayor volumen de 5xx:
   - Código HTTP predominante:
   - Intervalo temporal de mayor concentración:
   - Duración media relevante:
   - Mensajes de error observados:

   ## Consultas utilizadas
   - Query DSL:
   - KQL:
   - ES|QL:

   ## Mitigación y recomendación operativa
   - Acción inmediata:
   - Equipo o nivel de escalamiento:
   - Acción preventiva:
   - Consideración de compatibilidad para Elasticsearch 7.17.29:
   EOF
   ```

6. Complete el informe con sus resultados reales. Incluya como mínimo una consulta Query DSL, una expresión KQL, una consulta ES|QL y una recomendación operativa basada en evidencia.

Criterios de aprobación:

- Se verificó el estado del clúster y la existencia del data stream.
- Se ejecutaron consultas con `term`, `match`, `range`, `exists` y `bool`.
- Se demostró una exclusión mediante `must_not`.
- Se generaron agregaciones por servicio, código HTTP y tiempo.
- Se ejecutaron filtros KQL en Discover.
- Se ejecutaron consultas ES|QL en Kibana 9.3.0.
- Se documentó que la compatibilidad de ES|QL debe validarse antes de utilizarlo con Elasticsearch 7.17.29.

## Solución de problemas

### Problema 1: La búsqueda devuelve cero documentos aunque el data stream existe

**Síntomas**

- `_count` devuelve `0`.
- Discover no muestra eventos.
- Las búsquedas con `now-2h` no devuelven coincidencias.

**Causa probable**

Los eventos pueden tener marcas temporales fuera del rango seleccionado, el data stream puede no haberse poblado en prácticas anteriores o la carga opcional no se realizó correctamente.

**Solución**

1. Inspeccione el rango temporal real de los documentos:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     -H 'Content-Type: application/json' \
     "http://localhost:9201/logs-acme.api-default/_search" \
     -d '{
       "size": 1,
       "sort": [
         {
           "@timestamp": {
             "order": "asc"
           }
         }
       ]
     }' | jq '.hits.hits[0]._source["@timestamp"]'
   ```

2. Repita la búsqueda con un rango absoluto que cubra la fecha encontrada.
3. Si no existen documentos, ejecute el Paso 3 y confirme que `10-bulk-load-response.json` contiene `"errors": false`.
4. En Discover, ajuste el selector temporal al mismo rango absoluto.

### Problema 2: Error al consultar `message.keyword` o resultados inesperados con `term` sobre texto

**Síntomas**

- Elasticsearch devuelve un error de campo no existente para `message.keyword`.
- La consulta `term` sobre `message` no devuelve el mensaje esperado.
- `match` devuelve resultados, pero `term` no.

**Causa probable**

`message` o `error.message` está mapeado como campo `text`, puede no tener un subcampo `.keyword`, o el valor indexado no coincide exactamente con la cadena buscada.

**Solución**

1. Consulte el mapping real:

   ```bash
   curl -sS -u "elastic:${ELASTIC_PASSWORD}" \
     "http://localhost:9201/logs-acme.api-default/_mapping/field/message,error.message?pretty"
   ```

2. Use `match` para búsquedas de texto analizado:

   ```json
   {
     "query": {
       "match": {
         "message": "database connection timeout"
       }
     }
   }
   ```

3. Use `term` únicamente sobre campos exactos, por ejemplo `service.name`, `log.level`, `service.environment`, `http.response.status_code` o un subcampo `.keyword` que exista.
4. Si necesita igualdad exacta y no existe `.keyword`, revise el template y el mapping antes de modificar el esquema; no cambie mappings de producción únicamente para esta práctica.

## Limpieza

Esta práctica genera evidencias que deben conservarse para el informe y para prácticas posteriores. No elimine volúmenes Docker ni ejecute `docker compose down -v`.

1. Revise los archivos generados:

   ```bash
   ls -lh /opt/elastic-labs/work/lab-05-00-01
   ```

2. Si desea detener temporalmente los servicios al terminar la sesión, hágalo sin eliminar volúmenes:

   ```bash
   cd /opt/elastic-labs
   docker compose stop es930-lab kibana930-lab
   ```

3. Para continuar el laboratorio posteriormente:

   ```bash
   cd /opt/elastic-labs
   docker compose start es930-lab kibana930-lab
   ```

4. Mantenga los archivos de evidencia, especialmente:

   - `11-production-5xx.json`
   - `13-orders-database-investigation.json`
   - `17-errors-by-service-and-status.json`
   - `18-5xx-time-trend.json`
   - `19-average-duration-by-status.json`
   - `informe-investigacion.md`

## Resumen

En esta práctica aplicó Query DSL para investigar logs estructurados mediante coincidencias exactas, búsqueda de texto, rangos temporales, validación de existencia de campos y lógica booleana. Utilizó agregaciones para identificar servicios con mayor volumen de errores, tendencias 5xx y duración media por código HTTP.

También reprodujo filtros en Kibana con KQL y creó resúmenes analíticos con ES|QL en Elasticsearch 9.3.0. La conclusión operativa clave es que Query DSL, KQL y ES|QL son complementarios: la elección depende de la interfaz, el tipo de análisis, la necesidad de automatización y la compatibilidad de la versión de Elasticsearch.
