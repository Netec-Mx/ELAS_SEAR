# Administración de Elasticsearch para gestión y análisis de logs

## Práctica 7. Investigar un incidente con Discover, KQL y ES|QL, y documentar la causa raíz

### Objetivo

Al finalizar la práctica, serás capaz de:

- Aplicar una metodología de análisis de logs que va del síntoma a la causa raíz.
- Usar Kibana Discover con data views, KQL, filtros, rango de tiempo, estadísticas de campos y búsquedas guardadas.
- Aplicar patrones de investigación: errores, latencias, top N, picos de volumen y correlación por `trace.id` y `host.name`.
- Consultar logs con ES|QL usando `FROM`, `WHERE`, `EVAL`, `STATS ... BY`, `DISSECT`, `GROK`, `SORT`, `KEEP` y `LIMIT`.
- Analizar patrones de logs y eventos de seguridad: autenticaciones fallidas, IPs y usuarios inusuales.
- Documentar un incidente con línea de tiempo, evidencia y causa raíz.

### Objetivo visual

![Metodología de investigación: del síntoma a la causa raíz](img/cap07.png)

### Duración aproximada

- 95 minutos.

### Prerrequisitos

- Prácticas 5a y 5b completadas, con estos data streams cargados:
  - `logs-weblab.access-default` (3,421 documentos)
  - `logs-weblab.app-default` (2,458 documentos)
  - `logs-syslog.lab-default` (663 documentos)
- Licencia de prueba (trial) activa, solo para el paso opcional de *log rate analysis*.

---

### Instrucciones

**Escenario:** el martes **20 de octubre de 2026**, a las 14:15, el área comercial reporta: *"Varios clientes dicen que no pudieron pagar su carrito después de la comida"*. Eres el analista de guardia. Debes confirmar el problema, delimitar su alcance, encontrar la causa raíz con evidencia y revisar si hubo otros eventos relevantes ese día.

**Metodología (subtema 7.1).** Sigue estas fases en orden; cada tarea de la práctica corresponde a una:

| Fase | Pregunta | Herramienta principal |
|---|---|---|
| 1. Síntoma | ¿Qué ve el usuario? ¿Se confirma en los datos? | Discover + KQL |
| 2. Alcance | ¿Qué endpoint, servidor y periodo? ¿A cuántos afecta? | Estadísticas de campos, ES\|QL `STATS` |
| 3. Línea de tiempo | ¿Cuándo empezó y terminó? ¿Qué cambió justo antes? | Histograma, ES\|QL `BUCKET` |
| 4. Correlación | ¿Qué dicen las otras fuentes del mismo momento, host o traza? | `trace.id`, `host.name` |
| 5. Causa raíz | ¿Qué explica **todos** los hechos? | Evidencia combinada |
| 6. Documentación | ¿Qué pasó, por qué, cómo se resolvió y cómo evitarlo? | Reporte de incidente |

> **Importante:** formula hipótesis y busca evidencia que las **refute**, no solo que las confirme. El error más común es quedarse con la primera explicación plausible.

---

### Tarea 1. Preparar Kibana para la investigación

#### Paso 1. Fija la zona horaria de Kibana

1. Abre el menú ☰ y ve a **Stack Management > Advanced Settings**.
2. Busca `dateFormat:tz`, cambia `Browser` por `America/Mexico_City` y haz clic en **Save changes**.

**Qué hace y por qué:** Elasticsearch guarda las fechas en UTC y Kibana las muestra por defecto en la zona del navegador. Fijar la zona evita confusiones cuando varios analistas, o un reporte programado, ven los mismos datos. Todos los horarios de esta práctica están en hora de la Ciudad de México.

#### Paso 2. Crea los data views

1. Ve a **Stack Management > Data Views** y haz clic en **Create data view**.
2. Crea estos tres data views. En todos, el **Timestamp field** es `@timestamp`.

| Name | Index pattern |
|---|---|
| Logs tienda - acceso web | `logs-weblab.access-*` |
| Logs tienda - aplicación | `logs-weblab.app-*` |
| Logs tienda - todas las fuentes | `logs-weblab.*,logs-syslog.*` |

**Qué hace y por qué:** un **data view** indica a Kibana qué índices o data streams consultar y cuál es el campo de tiempo. Un data view por fuente da listas de campos más limpias; uno combinado permite ver en una sola línea de tiempo eventos web, de aplicación y de sistema, lo que es indispensable para correlacionar.

#### Paso 3. Abre Discover con el rango de tiempo del incidente

1. Ve a **Analytics > Discover** y elige el data view **Logs tienda - acceso web**.
2. En el selector de tiempo, haz clic en **Show dates** y define un rango **absoluto**: de `Oct 20, 2026 @ 08:00:00.000` a `Oct 20, 2026 @ 20:00:00.000`.
3. Haz clic en **Update**.

**Qué hace y por qué:** los logs del laboratorio son de una fecha fija. Con el rango por defecto (*Last 15 minutes*) Discover no mostraría nada. Revisa el **histograma**: se ve un pico de volumen hacia las 18:00. Lo analizarás en la Tarea 3.

---

### Tarea 2. Fase 1 y 2: confirmar el síntoma y delimitar el alcance con KQL

**KQL** (Kibana Query Language) es el lenguaje de la barra de búsqueda. Su sintaxis es `campo : valor`, con operadores `and`, `or`, `not`, rangos (`>=`) y comodines (`*`).

#### Paso 4. Busca errores del servidor

Escribe en la barra de búsqueda y presiona `Enter`:

```text
http.response.status_code >= 500
```

**Qué hace y por qué:** filtra las respuestas 5xx, que son fallas del servicio y no del cliente. El contador debe mostrar **66 hits**. En el histograma, los errores se concentran entre las **13:38 y las 14:10**. El síntoma reportado se confirma con datos.

#### Paso 5. Revisa las estadísticas de los campos

1. En la barra lateral izquierda, haz clic en el campo `url.path` y observa los **valores principales** (*Top values*).
2. Repite con `nginx.access.upstream_host`, `host.name` y `http.response.status_code`.

**Qué hace y por qué:** las estadísticas muestran la distribución de valores **en los documentos filtrados**. Resultados:

- `url.path`: `/api/checkout` concentra 63 de los 66 errores.
- `nginx.access.upstream_host`: el **100 %** de los errores pasa por `srv-app02`.
- `host.name` (servidor web): `srv-web01` y `srv-web02` reparten los errores de forma similar, así que el problema **no** está en la capa web.
- `http.response.status_code`: 40 respuestas 503 y 23 respuestas 504.

Este es el patrón *top N*: en segundos reduce el alcance de "la tienda falla" a "el checkout falla cuando lo atiende `srv-app02`".

#### Paso 6. Pon a prueba la hipótesis con una consulta que la refute

```text
url.path : "/api/checkout" and not nginx.access.upstream_host : "srv-app02"
```

Ahora cambia el rango de tiempo a **13:38 – 14:11** y agrega la columna `http.response.status_code`: en la lista de campos, pasa el cursor sobre el campo y haz clic en el botón **+** (*Add field as column*).

**Qué hace y por qué:** si el problema fuera de la base de datos o de toda la aplicación, `srv-app01` también fallaría. Todas sus peticiones de checkout en la ventana del incidente respondieron **200**. La hipótesis "el problema está en `srv-app02`" sobrevive a un intento de refutarla.

#### Paso 7. Usa filtros y guarda la búsqueda

1. Regresa el rango a **08:00 – 20:00** y escribe la consulta `http.response.status_code >= 500`.
2. En la barra lateral, en el campo `url.path`, haz clic en la lupa con **+** junto a `/api/checkout` para agregarlo como **filtro**.
3. Haz clic en **Save** (arriba a la derecha) y guarda la búsqueda como `Incidente 2026-10-20 - Errores 5xx checkout`.

**Qué hace y por qué:** los **filtros** (las "píldoras" debajo de la barra) pueden activarse, desactivarse, invertirse (*Exclude results*) y fijarse entre aplicaciones (*Pin across all apps*), sin reescribir la consulta. La **búsqueda guardada** conserva la consulta, los filtros, las columnas y el orden; se puede agregar a un dashboard o exportar a CSV (Práctica 8b).

---

### Tarea 3. Fase 3: construir la línea de tiempo con ES|QL

**ES|QL** es un lenguaje de consulta por tuberías: cada comando recibe una tabla y produce otra. En Discover se activa con el selector de lenguaje (**ES|QL** o **Try ES|QL** en la barra superior). En Dev Tools se usa con `POST _query`.

#### Paso 8. Cambia Discover a modo ES|QL y obtén el top de errores

En Discover, cambia a **ES|QL** y ejecuta:

```esql
FROM logs-weblab.access-default
| WHERE http.response.status_code >= 500
| STATS errores = COUNT(*) BY url.path, nginx.access.upstream_host
| SORT errores DESC
| LIMIT 10
```

**Qué hace y por qué:** `FROM` elige la fuente, `WHERE` filtra, `STATS ... BY` agrega por grupos, `SORT` ordena y `LIMIT` limita. El resultado confirma en una tabla lo visto en el Paso 5: `/api/checkout` + `srv-app02` = 63. En Discover, el selector de tiempo se aplica automáticamente a la consulta ES|QL.

#### Paso 9. Calcula la tasa de error por intervalos de 5 minutos

```esql
FROM logs-weblab.access-default
| WHERE url.path == "/api/checkout"
| EVAL es_error = CASE(http.response.status_code >= 500, 1, 0)
| STATS total = COUNT(*), errores = SUM(es_error) BY intervalo = BUCKET(@timestamp, 5 minutes)
| EVAL tasa_pct = ROUND(errores * 100.0 / total, 1)
| WHERE total > 0
| SORT intervalo
```

**Qué hace y por qué:** `EVAL` crea columnas calculadas; `CASE` convierte la condición en 1 o 0 para poder sumarla; `BUCKET` agrupa el tiempo en intervalos fijos. La tasa pasa de **0 %** a **60 %** en el intervalo de las **13:35**, se mantiene entre **30 % y 65 %** hasta el de las **14:05**, baja a 14 % en el de las **14:10** (último error a las 14:10:02) y vuelve a **0 %** desde las **14:15**. Un pico en números absolutos puede deberse solo a más tráfico; una tasa (errores ÷ total) separa "hay más peticiones" de "las peticiones fallan".

#### Paso 10. Compara la latencia por servidor de aplicación

```esql
FROM logs-weblab.access-default
| WHERE url.path == "/api/checkout"
| EVAL ms = event.duration / 1000000
| STATS peticiones = COUNT(*), media_ms = ROUND(AVG(ms)), p95_ms = PERCENTILE(ms, 95), max_ms = MAX(ms) BY nginx.access.upstream_host
```

**Qué hace y por qué:** convierte `event.duration` (nanosegundos) a milisegundos. En `srv-app02` el p95 es de unos **3,055 ms**, un valor sospechosamente "redondo" que coincide con un **timeout**. En `srv-app01` el p95 es de unos 665 ms. Un valor repetido en la latencia casi siempre indica un límite configurado (timeout), no lentitud natural.

#### Paso 11. Identifica qué cambió justo antes del inicio

Cambia el rango a **13:00 – 14:30** y ejecuta:

```esql
FROM logs-weblab.app-default
| WHERE service.name == "checkout-service"
  AND (message LIKE "Iniciando*" OR message LIKE "Configuracion cargada*" OR message LIKE "Rollback*" OR message LIKE "Recibida senal*")
| KEEP @timestamp, host.name, user.name, message
| SORT @timestamp
```

Después amplía el rango a **07:00 – 20:00** y vuelve a ejecutar la consulta.

**Qué hace y por qué:** la pregunta más útil de cualquier investigación es *"¿qué cambió?"*. La consulta muestra los eventos de ciclo de vida del servicio:

| Hora | Host | Evento |
|---|---|---|
| 07:58 | srv-app01 y srv-app02 | `Iniciando checkout-service version=2.4.0`, `db.pool.maxSize=50` |
| **13:34:50** | srv-app02 | `Recibida senal de apagado` (usuario `deploy-bot`) |
| **13:35:02** | srv-app02 | `Iniciando checkout-service version=2.4.1` |
| **13:35:03** | srv-app02 | `Configuracion cargada: db.pool.maxSize=5` |
| 14:08:10 | srv-app02 | `Rollback solicitado a version=2.4.0 ticket=CHG-4471` |
| 14:08:41 | srv-app02 | `Iniciando ... version=2.4.0`, `db.pool.maxSize=50` |

Tres minutos antes del primer error se desplegó la versión 2.4.1 **solo en `srv-app02`**, con un pool de conexiones a la base de datos 10 veces menor.

#### Paso 12. Extrae datos del mensaje con `DISSECT`

```esql
FROM logs-weblab.app-default
| WHERE message LIKE "*Connection is not available*"
| DISSECT message "HikariPool-1 - Connection is not available, request timed out after %{timeout_ms}ms. (total=%{total}, active=%{active}, idle=%{idle}, waiting=%{waiting})"
| EVAL waiting = TO_INTEGER(waiting), total = TO_INTEGER(total)
| STATS eventos = COUNT(*), primero = MIN(@timestamp), ultimo = MAX(@timestamp),
        pool_total = MAX(total), max_en_espera = MAX(waiting) BY host.name
```

**Qué hace y por qué:** `DISSECT` extrae campos de un texto en el momento de la consulta, sin haberlos definido al ingerir. El resultado muestra **63** eventos, todos en `srv-app02`, entre las **13:38:48** y las **14:09:58**, con un pool de **5** conexiones (todas activas) y hasta 30 peticiones en espera. Esto confirma el mecanismo: las peticiones esperan una conexión, se agota el timeout de 3,000 ms y el servicio responde 503/504.

---

### Tarea 4. Fase 4: correlacionar fuentes por `trace.id` y `host.name`

#### Paso 13. Sigue una petición fallida de punta a punta

En ES|QL:

```esql
FROM logs-weblab.* METADATA _index
| WHERE trace.id == "664595a620d2a6692d4bc1330b677eae"
| KEEP @timestamp, _index, host.name, log.level, http.response.status_code, message
| SORT @timestamp
```

**Qué hace y por qué:** `METADATA _index` agrega el nombre del índice de origen. La misma traza aparece en tres eventos:

1. `logs-weblab.app` – `WARN` en `srv-app02`: *Connection is not available…*
2. `logs-weblab.app` – `ERROR` en `srv-app02`: *Fallo al confirmar pedido id=PED-…* con el stack trace.
3. `logs-weblab.access` – `srv-web02` responde **503** al cliente.

Así se demuestra que el 503 que vio el cliente lo causó el agotamiento del pool. Para hacer lo mismo en KQL, usa el data view **Logs tienda - todas las fuentes** con la consulta `trace.id : 664595a620d2a6692d4bc1330b677eae`.

#### Paso 14. Correlaciona por host con los logs del sistema

Usa el data view **Logs tienda - todas las fuentes**, el rango **13:30 – 14:15** y la consulta KQL:

```text
host.name : "srv-app02" and (process.name : (sshd or sudo or systemd) or log.level : (WARN or ERROR))
```

Agrega las columnas `host.name`, `process.name`, `user.name` y `message`.

**Qué hace y por qué:** en una sola línea de tiempo ves que a las **13:34:40** el usuario `deploy-bot` entró por SSH a `srv-app02` y reinició `checkout-service` con `sudo`; minutos después empiezan los WARN y ERROR, y a las 14:08 se repite la secuencia para el rollback. El log del sistema corrobora el log de la aplicación, una evidencia independiente del despliegue.

---

### Tarea 5. Descartar falsas alarmas: picos de volumen y patrones

#### Paso 15. Analiza el pico de las 18:00

Regresa al data view **Logs tienda - acceso web**, rango **08:00 – 20:00**, y ejecuta en ES|QL:

```esql
FROM logs-weblab.access-default
| EVAL es_error = CASE(http.response.status_code >= 500, 1, 0)
| STATS peticiones = COUNT(*), errores = SUM(es_error) BY intervalo = BUCKET(@timestamp, 30 minutes)
| SORT peticiones DESC
| LIMIT 5
```

**Qué hace y por qué:** los dos intervalos con más peticiones son el de las **13:30** (483 peticiones y 46 errores) y el de las **18:00** (447 peticiones y **0 errores**), contra 100 a 140 en un intervalo normal. El de las 13:30 coincide con el incidente: lo inflan los reintentos de los clientes y el ataque a `/api/login` que verás en la Tarea 6. El de las 18:00, al filtrarlo en Discover, es casi todo tráfico a `/api/catalog/items` con respuesta 200: una campaña comercial, no un incidente. Un pico de volumen sin aumento en la tasa de error ni en la latencia no es una falla.

#### Paso 16. Revisa los patrones de mensajes

1. Cambia Discover de nuevo a **KQL** (selector de lenguaje) y elige el data view **Logs tienda - aplicación**.
2. Con el rango **08:00 – 20:00**, abre la pestaña **Patterns** (junto a **Documents** y **Field statistics**).
3. Si no aparece la pestaña, haz clic en el campo `message` de la barra lateral y elige **Run pattern analysis**.

**Qué hace y por qué:** el análisis de patrones agrupa mensajes con la misma estructura y reemplaza las partes variables, como los IDs, por comodines. En lugar de leer 2,458 líneas, ves unas pocas docenas de patrones con su frecuencia, por ejemplo, `Login fallido usuario= ip= motivo=credenciales_invalidas` o `HikariPool-1 Connection is not available...`. Los patrones raros o nuevos merecen atención.

#### Paso 17. (Opcional, requiere trial) Ejecuta log rate analysis

1. Ve a **Machine Learning > AIOps Labs > Log Rate Analysis** y elige el data view **Logs tienda - acceso web**.
2. Con el rango **08:00 – 20:00**, filtra con `http.response.status_code >= 500` y haz clic en la barra del histograma de las **13:40**.
3. Haz clic en **Run analysis**.

**Qué hace y por qué:** la herramienta compara el periodo del pico con una referencia y lista los campos y valores **estadísticamente asociados** al cambio (`nginx.access.upstream_host: srv-app02`, `url.path: /api/checkout`, `http.response.status_code: 503`). Llega automáticamente a la misma conclusión que construiste a mano.

---

### Tarea 6. Análisis de logs de acceso y seguridad

Durante la revisión aparecieron cientos de respuestas 401 en `/api/login`. Un buen analista no ignora un hallazgo paralelo.

#### Paso 18. Identifica IPs con autenticaciones fallidas

```esql
FROM logs-weblab.access-default
| WHERE url.path == "/api/login" AND http.response.status_code IN (401, 429)
| STATS intentos = COUNT(*), primero = MIN(@timestamp), ultimo = MAX(@timestamp),
        agentes = VALUES(user_agent.original) BY source.ip
| SORT intentos DESC
| LIMIT 5
```

**Qué hace y por qué:** la IP **89.160.20.156** concentra **247** respuestas (222 con 401 y 25 con 429) entre las 13:45 y las 13:56, con el agente `python-requests/2.31.0`: un ataque automatizado de fuerza bruta. Los 429 indican que el servicio la bloqueó temporalmente. Revisa con `source.geo.country_iso_code` desde qué país se originó.

#### Paso 19. Identifica usuarios inusuales

```esql
FROM logs-weblab.app-default
| WHERE message LIKE "Login fallido*"
| STATS intentos = COUNT(*), ips = COUNT_DISTINCT(source.ip) BY user.name
| SORT intentos DESC
```

**Qué hace y por qué:** los usuarios atacados (`test`, `ventas`, `admin`, `admin1`, `root`, `soporte`, `administrador`) son nombres genéricos que **no** pertenecen a clientes: es un ataque de diccionario. Los clientes reales tienen 1 o 2 fallos aislados. Comparar contra lo "normal" es la base para detectar usuarios inusuales.

#### Paso 20. Correlaciona con SSH: ¿el atacante tuvo éxito?

```esql
FROM logs-syslog.lab-default
| WHERE source.ip == "89.160.20.156"
| STATS eventos = COUNT(*), usuarios = VALUES(user.name) BY event.outcome, host.name
```

Luego revisa la secuencia completa en KQL con el data view **Logs tienda - todas las fuentes** y el rango **13:49 – 14:06**:

```text
host.name : "srv-web01" and (source.ip : "89.160.20.156" or user.name : "soporte")
```

**Qué hace y por qué:** la misma IP atacó también SSH en `srv-web01`: **145** contraseñas fallidas. A las **13:58:41** hubo un **inicio de sesión exitoso** del usuario `soporte`, y a las 13:59:30 y 13:59:52 ese usuario intentó, sin éxito, ejecutar `/bin/bash` y leer `/etc/shadow` como root (`user NOT in sudoers`). Es un **compromiso de cuenta** que debe escalarse de inmediato al equipo de seguridad, aunque no tenga relación con la falla del checkout.

#### Paso 21. Usa `GROK` en ES|QL para un análisis rápido

```esql
FROM logs-syslog.lab-default
| WHERE message LIKE "Invalid user*"
| GROK message "Invalid user %{USERNAME:usuario_probado} from %{IP:ip_origen}"
| STATS intentos = COUNT(*) BY usuario_probado
| SORT intentos DESC
```

**Qué hace y por qué:** `GROK` funciona como `DISSECT`, pero con expresiones regulares y patrones con nombre; es útil cuando el formato varía. El resultado muestra los usuarios inexistentes que probó el atacante (`admin`, `test`, `oracle`, `ubuntu`, `postgres`…).

#### Paso 22. Ejecuta ES|QL desde Dev Tools

En **Dev Tools**:

```http
POST _query?format=txt
{
  "query": """
    FROM logs-syslog.lab-default
    | WHERE @timestamp >= "2026-10-20T19:00:00Z" AND @timestamp < "2026-10-20T21:00:00Z"
    | WHERE process.name == "sshd" AND event.outcome IS NOT NULL
    | EVAL fallo = CASE(event.outcome == "failure", 1, 0), exito = CASE(event.outcome == "success", 1, 0)
    | STATS fallidos = SUM(fallo), exitosos = SUM(exito) BY source.ip
    | SORT fallidos DESC
  """
}
```

**Qué hace y por qué:** la API `_query` ejecuta ES|QL desde scripts y herramientas externas. `format=txt` devuelve una tabla legible; también acepta `csv` y `json`. Fuera de Discover no hay selector de tiempo, así que el rango se indica en `WHERE` y **en UTC**: 13:00–15:00 en la Ciudad de México equivalen a 19:00–21:00 UTC.

---

### Tarea 7. Fase 6: documentar la causa raíz

#### Paso 23. Completa el reporte de incidente

Crea un documento con la siguiente estructura y llénalo con tu evidencia. Para cada afirmación, incluye la consulta KQL o ES|QL que la respalda.

| Sección | Contenido esperado |
|---|---|
| Título | INC-2026-10-20-01: Fallas en el pago (checkout) de la tienda en línea |
| Resumen | Entre las 13:38 y las 14:10 falló con 503/504 el 44 % de las peticiones de checkout (63 de 144), con picos de 65 % en intervalos de 5 minutos. |
| Impacto | 63 peticiones de checkout fallidas (clientes que no pudieron pagar), durante ~32 minutos. Solo afectó a las peticiones atendidas por `srv-app02`. |
| Detección | Reporte del área comercial a las 14:15. **No hubo alerta automática** (acción de mejora). |
| Línea de tiempo | 13:34:40 SSH de `deploy-bot` a srv-app02 → 13:35:02 despliegue de v2.4.1 → 13:35:03 `db.pool.maxSize=5` → 13:38:48 primer timeout del pool → 13:38:51 primer 503 → 14:08:10 rollback (CHG-4471) → 14:08:41 `maxSize=50` → 14:10:02 último error. |
| Causa raíz | La versión 2.4.1 se desplegó en `srv-app02` con `db.pool.maxSize=5` en lugar de 50. Bajo carga normal, las 5 conexiones se agotaron y las peticiones superaron el timeout de 3,000 ms. |
| Evidencia | Pasos 5, 6, 9, 10, 11, 12, 13 y 14 (consultas y resultados). |
| Hipótesis descartadas | Falla de la base de datos (srv-app01 funcionó), problema en la capa web (ambos servidores web afectados por igual), aumento de tráfico (la tasa de error, no solo el volumen, subió; el pico de las 18:00 no tuvo errores). |
| Hallazgo paralelo | Fuerza bruta desde 89.160.20.156 contra `/api/login` y SSH; acceso exitoso con `soporte` en srv-web01 a las 13:58:41 e intentos de escalar privilegios. **Escalado a seguridad.** |
| Acciones | Validar la configuración del pool en el pipeline de despliegue; despliegue gradual (canary) con monitoreo; alerta por tasa de error en checkout (Práctica 8b); bloquear la IP, rotar la credencial de `soporte`, exigir llaves SSH y revisar la actividad del usuario. |

**Qué hace y por qué:** un incidente no se cierra cuando el servicio se recupera, sino cuando se entiende **por qué** ocurrió y se definen acciones para que no se repita. Vincular cada afirmación a una consulta reproducible hace que el reporte sea verificable por otros y reutilizable como alerta o dashboard.

---

### Resultado esperado

- Tres data views creados y la búsqueda guardada `Incidente 2026-10-20 - Errores 5xx checkout`.
- Hechos clave identificados:

| Hecho | Valor |
|---|---|
| Respuestas 5xx del día | 66 |
| Errores de checkout por `srv-app02` | 63 (40 × 503, 23 × 504) |
| Ventana del incidente | 13:38:51 – 14:10:02 |
| Cambio previo | Despliegue de v2.4.1 a las 13:35:02 con `db.pool.maxSize=5` |
| Resolución | Rollback a v2.4.0 a las 14:08 (CHG-4471) |
| Picos de volumen (30 min) | 13:30: 483 peticiones con 46 errores (incidente + ataque); 18:00: 447 peticiones con 0 errores (campaña) |
| Ataque a `/api/login` | 89.160.20.156: 222 × 401 y 25 × 429 |
| Ataque SSH | 145 contraseñas fallidas; acceso exitoso de `soporte` a las 13:58:41 |

- Reporte de incidente completo con línea de tiempo, causa raíz, evidencia, hipótesis descartadas y acciones.

### Referencias

- Elastic. *Discover [8.19].* https://www.elastic.co/guide/en/kibana/8.19/discover.html
- Elastic. *Kibana Query Language (KQL) [8.19].* https://www.elastic.co/guide/en/kibana/8.19/kuery-query.html
- Elastic. *ES|QL reference [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/esql-language.html
- Elastic. *Log pattern analysis y Log rate analysis [8.19].* https://www.elastic.co/guide/en/kibana/8.19/xpack-ml-aiops.html
