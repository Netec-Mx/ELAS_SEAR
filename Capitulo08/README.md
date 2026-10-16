- **Dependencias:** Laboratorio 4 (`weblogs` con 5006 documentos, con `team` y `tier`) y Laboratorio 5 (data view `weblogs`). Si el data view no existe, se crea en la Parte A.

## Objetivo

Construir un dashboard de operación sobre los logs del curso y configurar una alerta que se dispare con tráfico nuevo. Al terminar, el participante podrá:

- Explorar los logs en Discover con KQL y guardar una búsqueda.

- Crear visualizaciones en Lens (métrica, serie de tiempo, barras y tabla) y reunirlas en un dashboard con un control de filtro.

- Enviar documentos con la hora de llegada mediante un ingest pipeline para simular tráfico en vivo.

- Crear un conector y una regla de alerta de tipo Elasticsearch query, provocar la alerta y verificar su notificación y su recuperación.

- Ubicar en Kibana las piezas por las que entran datos de Beats, Elastic Agent y APM.

## Requisitos

- `GET weblogs/_count` devuelve 5006.

- Acceso a Kibana con el usuario `elastic` (o un usuario con acceso a Stack Management y Dashboards).

- Las fechas de los datos están en UTC. Kibana muestra las horas en la zona horaria del navegador: para Ciudad de México (UTC-6) hay una diferencia de 6 horas.

## Distribución del tiempo

| **Parte** | **Contenido**                          | **Tiempo** |
|-----------|----------------------------------------|------------|
| Parte A   | Discover y data view                   | 6 min      |
| Parte B   | Dashboard con Lens                     | 17 min     |
| Parte C   | Tráfico en vivo con un ingest pipeline | 7 min      |
| Parte D   | Conector y regla de alerta             | 15 min     |
| Parte E   | Recorrido por Integraciones y Fleet    | 5 min      |
| Parte F   | Limpieza y cierre                      | 4 min      |

## Convenciones

- Los bloques con `GET`, `PUT` o `POST` se ejecutan en Dev Tools. Las demás partes se hacen en la interfaz de Kibana; los nombres de menús pueden variar un poco según la versión 9.x.

- Cuando una pantalla ofrece varias opciones, solo se indican las que importan para el laboratorio. El resto se deja con su valor por defecto.

- El rango de tiempo del dashboard se fija en términos absolutos porque los datos son de octubre de 2026 y el selector relativo ("últimos 15 minutos") no los mostraría.

## Parte A: Discover y data view (6 min)

### Paso A1. Comprobar el data view

1.  En Kibana, abrir **Stack Management** y luego **Data Views**.

2.  Si existe el data view `weblogs` (del laboratorio 5), continuar con el paso A2.

3.  Si no existe, seleccionar **Create data view** con el nombre `weblogs`, el patrón de índice `weblogs` y el campo de marca de tiempo `@timestamp`. Guardar.

**Qué hace:** un data view define qué índices se consultan y qué campo es el reloj. Discover, Lens y las reglas usan data views, no nombres de índice directos.

### Paso A2. Explorar y guardar una búsqueda

1.  Abrir **Discover**, seleccionar el data view `weblogs` y fijar el rango de tiempo absoluto: del 12 de octubre de 2026 a las 00:00 al 15 de octubre de 2026 a las 00:00.

2.  Comprobar el número de documentos. **Resultado esperado:** 4976. Son los 5006 documentos menos los 30 que no tienen `@timestamp` (los `parse_failure`) y por eso quedan fuera de cualquier rango de tiempo.

3.  En la barra de búsqueda, escribir la consulta KQL `service : "payments" and status >= 500` y ejecutarla. **Resultado esperado:** 40 documentos.

4.  En el panel de campos, abrir `detail` y revisar los valores más frecuentes.

5.  Guardar la sesión con el nombre `Errores 5xx de payments`.

**Qué hace:** KQL (Kibana Query Language) es el lenguaje de búsqueda de la barra. `service : "payments"` busca un valor y los operadores `and`, `or`, `not` y las comparaciones (`>=`) combinan condiciones. Equivale a una consulta `bool` con filtros.

## Parte B: dashboard con Lens (17 min)

Se construye un dashboard con cuatro paneles y un control. Los valores esperados permiten comprobar cada panel.

| **Panel** | **Tipo**                                          | **Valor esperado (rango del 12 al 15 de octubre)**                      |
|-----------|---------------------------------------------------|-------------------------------------------------------------------------|
| 1         | Métrica: total de peticiones                      | 4976                                                                    |
| 2         | Serie de tiempo por hora, desglosada por servicio | Pico de payments el 13 de octubre a las 14:00 UTC (08:00 en UTC-6)      |
| 3         | Barras: errores 5xx por servicio                  | payments 40, checkout 25, search 16, catalog 14 (total 95)              |
| 4         | Tabla: tiempo de respuesta por servicio           | Promedio: payments ≈ 404, search ≈ 185, catalog ≈ 113, checkout ≈ 94 ms |

### Paso B1. Crear el dashboard

1.  Abrir **Dashboards** y seleccionar **Create dashboard**.

2.  Fijar el rango de tiempo absoluto del 12 de octubre de 2026 a las 00:00 al 15 de octubre de 2026 a las 00:00.

3.  Seleccionar **Create visualization**. Se abre Lens. Elegir el data view `weblogs`.

### Paso B2. Panel 1: total de peticiones

1.  En Lens, abrir el selector de tipo de gráfico y elegir **Metric**.

2.  Arrastrar el campo **Records** al área de la métrica. Records es el conteo de documentos.

3.  Cambiar el nombre de la métrica a `Peticiones`.

4.  Seleccionar **Save and return**. **Resultado esperado:** 4976.

### Paso B3. Panel 2: peticiones por hora y servicio

1.  Repetir **Add** \> **Create visualization** y elegir el tipo **Bar vertical stacked** (o **Area stacked**).

2.  Arrastrar `@timestamp` al eje horizontal. Lens usa un `date_histogram`; en las opciones del eje, fijar el intervalo en **1 hour**.

3.  Arrastrar **Records** al eje vertical.

4.  Arrastrar `service` a **Break down by** (los 4 valores más frecuentes).

5.  Guardar con el título `Peticiones por hora y servicio`.

**Qué hace:** cada barra es una hora. El desglose por servicio equivale a una agregación `terms` anidada dentro del `date_histogram`, la misma estructura del laboratorio 5. **Resultado esperado:** una barra de `payments` visiblemente más alta el 13 de octubre a las 14:00 UTC. Kibana la muestra a las 08:00 si el navegador está en UTC-6.

### Paso B4. Panel 3: errores 5xx por servicio

1.  Crear otra visualización de tipo **Bar horizontal**.

2.  Arrastrar `service` al eje vertical y **Records** al horizontal.

3.  En la configuración de la capa, agregar el filtro KQL `status >= 500`. El filtro por capa deja el resto del dashboard intacto.

4.  Ordenar por Records, descendente. Guardar como `Errores 5xx por servicio`.

**Resultado esperado:** payments 40, checkout 25, search 16, catalog 14. Pueden contrastarse con el resultado de ES\|QL del laboratorio 5.

### Paso B5. Panel 4: tiempo de respuesta por servicio

1.  Crear una visualización de tipo **Table**.

2.  En **Rows**, agregar `service`.

3.  En **Metrics**, agregar tres: **Records** (con el nombre `Peticiones`), **Average** de `response_time_ms` y **Percentile** de `response_time_ms` con el valor 95.

4.  Guardar como `Tiempo de respuesta por servicio`.

**Resultado esperado:** promedios de alrededor de 404 ms (payments), 185 ms (search), 113 ms (catalog) y 94 ms (checkout). El percentil 95 de payments (alrededor de 1400 ms) es más de cinco veces su mediana (alrededor de 270 ms): el promedio esconde la cola larga de las peticiones lentas.

### Paso B6. Control de filtro y guardado

1.  En el dashboard, abrir el menú **Add panel** y elegir **Controls** \> **Options list**. Configurarlo con el data view `weblogs` y el campo `service`.

2.  Probar el control: seleccionar `payments` y observar cómo los cuatro paneles se filtran. Limpiar la selección.

3.  Seleccionar **Save**. Nombre: `Operación weblogs`. Activar **Store time with dashboard**, para que al abrirlo se aplique el rango absoluto.

Un dashboard bien diseñado responde una pregunta por panel, ordena de lo general (arriba) a lo específico (abajo), usa el mismo color para el mismo servicio en todos los paneles y limita la cantidad de paneles. Con más de 8 o 10 se vuelve difícil de leer.

## Parte C: tráfico en vivo con un ingest pipeline (7 min)

Las reglas de alerta evalúan una ventana de tiempo reciente, y los datos de `weblogs` son de octubre de 2026. Para probar una alerta se necesitan documentos con la hora actual. En lugar de escribir la hora a mano, un pipeline la asigna al llegar.

### Paso C1. Pipeline, template e índice

``` console
PUT _ingest/pipeline/live-pipeline
{
  "description": "Asigna @timestamp con la hora de llegada del documento",
  "processors": [
    {
      "set": {
        "field": "@timestamp",
        "value": "{{{_ingest.timestamp}}}"
      }
    }
  ]
}

PUT _index_template/weblogs-live-template
{
  "index_patterns": ["weblogs-live*"],
  "priority": 150,
  "composed_of": ["weblogs-settings", "weblogs-mappings"],
  "template": {
    "settings": { "index.default_pipeline": "live-pipeline" }
  }
}

PUT weblogs-live
```

**Qué hace:** define el pipeline, un template que lo asocia a los índices `weblogs-live*` (con el mapping de los laboratorios anteriores) y crea el índice.

- `set`: asigna un valor a un campo.

- `{{{_ingest.timestamp}}}`: variable de metadatos del pipeline con la hora en que Elasticsearch procesa el documento. Las tres llaves evitan que se escapen caracteres.

- `index.default_pipeline`: todo documento que llegue a ese índice pasa por el pipeline, como en el laboratorio 4.

### Paso C2. Tráfico normal

``` console
POST weblogs-live/_bulk?refresh=wait_for
{ "index": {} }
{ "service": "checkout", "team": "ventas", "tier": "critical", "method": "GET", "url": "/api/cart", "status": 200, "response_time_ms": 45.2, "message": "GET /api/cart 200 45.2ms ok" }
{ "index": {} }
{ "service": "checkout", "team": "ventas", "tier": "critical", "method": "POST", "url": "/api/cart", "status": 201, "response_time_ms": 78.9, "message": "POST /api/cart 201 78.9ms ok" }
{ "index": {} }
{ "service": "catalog", "team": "catalogo", "tier": "standard", "method": "GET", "url": "/api/products", "status": 200, "response_time_ms": 92.4, "message": "GET /api/products 200 92.4ms cache hit" }
{ "index": {} }
{ "service": "catalog", "team": "catalogo", "tier": "standard", "method": "GET", "url": "/api/products/17", "status": 200, "response_time_ms": 88.0, "message": "GET /api/products/17 200 88.0ms ok" }
{ "index": {} }
{ "service": "search", "team": "busqueda", "tier": "standard", "method": "GET", "url": "/api/search?q=mouse", "status": 200, "response_time_ms": 131.6, "message": "GET /api/search?q=mouse 200 131.6ms ok" }
{ "index": {} }
{ "service": "payments", "team": "pagos", "tier": "critical", "method": "POST", "url": "/api/pay", "status": 200, "response_time_ms": 240.3, "message": "POST /api/pay 200 240.3ms ok" }
```

**Qué hace:** indexa 6 peticiones normales sin `@timestamp`. **Resultado esperado:** `"errors": false`.

``` console
GET weblogs-live/_search?size=1&filter_path=hits.total.value,hits.hits._source.@timestamp
```

**Resultado esperado:** `total.value` de 6 y un `@timestamp` con la fecha y hora actuales (UTC).

### Paso C3. Data view para los datos en vivo

En **Stack Management** \> **Data Views**, crear el data view `weblogs-live` con el patrón `weblogs-live` y el campo de tiempo `@timestamp`. La regla de la parte D lo usará.

## Parte D: conector y regla de alerta (15 min)

### Paso D1. Crear un conector de tipo Index

1.  Abrir **Stack Management** \> **Connectors** (en algunas versiones, dentro de **Rules**) y seleccionar **Create connector**.

2.  Elegir el tipo **Index**. Nombre: `Alertas curso`. Índice: `alertas-curso`. Activar **Refresh index**.

3.  Guardar y usar la pestaña **Test** para ejecutar el conector con el documento `{ "mensaje": "prueba" }`.

``` console
GET alertas-curso/_search
```

**Qué hace:** un conector es el canal de salida de una alerta (correo, Slack, un webhook, un índice). El tipo Index escribe cada notificación como un documento, sin necesitar un servicio externo, por lo que sirve para el laboratorio. **Resultado esperado:** un documento con `mensaje: prueba`.

### Paso D2. Crear la regla

1.  Abrir **Stack Management** \> **Rules** y seleccionar **Create rule**.

2.  Elegir el tipo **Elasticsearch query**. Nombre: `Errores 5xx en weblogs-live`. Intervalo de comprobación: **1 minute**.

3.  En la definición de la consulta, elegir **KQL**, el data view `weblogs-live` y la consulta `status >= 500`.

4.  Condición: cuando el **count** de documentos **is above** `4` durante los últimos **5 minutes**.

5.  En las acciones, seleccionar el conector `Alertas curso`. Ejecutar la acción cuando se cumple la consulta (grupo **Query matched**) y con frecuencia **On status changes**.

6.  En el cuerpo de la acción (documentos), escribir:

``` console
[{ "regla": "{{rule.name}}", "estado": "{{alert.actionGroupName}}", "valor": "{{context.value}}", "condicion": "{{context.conditions}}", "fecha": "{{context.date}}" }]
```

7.  Agregar una segunda acción para el grupo **Recovered**, con el conector `Alertas curso` y el mismo cuerpo. El estado aparecerá como Recovered.

8.  Guardar la regla.

- **Intervalo de comprobación** (1 minuto): cada cuánto se evalúa la regla. Conviene que sea menor que la ventana para no dejar huecos.

- **Ventana de tiempo** (5 minutos): cuánto hacia atrás mira la consulta en cada ejecución.

- **Umbral** (más de 4): una sola petición con error no genera alerta; evita ruido por fallos aislados.

- **On status changes**: notifica cuando la alerta se activa o se recupera, no en cada ejecución. Reduce la fatiga de alertas.

- `{{rule.name}}`, `{{context.value}}`: variables de plantilla (Mustache) que Kibana reemplaza al enviar la notificación.

### Paso D3. Provocar la alerta

``` console
POST weblogs-live/_bulk?refresh=wait_for
{ "index": {} }
{ "service": "payments", "team": "pagos", "tier": "critical", "method": "POST", "url": "/api/pay", "status": 500, "response_time_ms": 1830.5, "detail": "upstream timeout", "message": "POST /api/pay 500 1830.5ms upstream timeout" }
{ "index": {} }
{ "service": "payments", "team": "pagos", "tier": "critical", "method": "POST", "url": "/api/pay", "status": 500, "response_time_ms": 2214.0, "detail": "upstream timeout", "message": "POST /api/pay 500 2214.0ms upstream timeout" }
{ "index": {} }
{ "service": "payments", "team": "pagos", "tier": "critical", "method": "POST", "url": "/api/pay", "status": 503, "response_time_ms": 512.7, "detail": "db connection refused", "message": "POST /api/pay 503 512.7ms db connection refused" }
{ "index": {} }
{ "service": "payments", "team": "pagos", "tier": "critical", "method": "POST", "url": "/api/pay", "status": 500, "response_time_ms": 1990.1, "detail": "upstream timeout", "message": "POST /api/pay 500 1990.1ms upstream timeout" }
{ "index": {} }
{ "service": "payments", "team": "pagos", "tier": "critical", "method": "POST", "url": "/api/pay", "status": 503, "response_time_ms": 604.3, "detail": "db connection refused", "message": "POST /api/pay 503 604.3ms db connection refused" }
{ "index": {} }
{ "service": "payments", "team": "pagos", "tier": "critical", "method": "POST", "url": "/api/pay", "status": 500, "response_time_ms": 2450.8, "detail": "upstream timeout", "message": "POST /api/pay 500 2450.8ms upstream timeout" }
```

**Qué hace:** simula una falla de pagos: 6 errores 5xx, más que el umbral de 4.

### Paso D4. Verificar la alerta

1.  Esperar hasta 2 minutos (la regla se evalúa cada minuto). Abrir **Stack Management** \> **Rules** y seleccionar la regla.

2.  En la página de la regla, revisar el estado (**Active** cuando la alerta está en curso), la pestaña de alertas y el historial de ejecuciones.

3.  Verificar la notificación en Dev Tools con el comando siguiente.

``` console
GET alertas-curso/_search?filter_path=hits.total.value,hits.hits._source
```

**Resultado esperado:** además del documento de prueba, un documento con `regla` igual a `Errores 5xx en weblogs-live`, el estado `Query matched` y un `valor` de 6.

### Paso D5. Recuperación

Sin nuevos errores, la alerta se recupera cuando los 6 errores salen de la ventana de 5 minutos. Mientras tanto se avanza con la Parte E. Después, volver a consultar `alertas-curso`: debe haber un documento con el estado `Recovered`. Si la regla nunca se recupera, revisar que no se hayan enviado más errores y que el reloj del navegador y del servidor coincidan en la hora UTC.

## Parte E: recorrido por Integraciones y Fleet (5 min)

El laboratorio no instala agentes. Se reconocen las piezas para decidir cómo entrarían los datos en un caso real.

1.  Abrir **Integrations**. Buscar `nginx`, `system` y `apm`. Para cada una, anotar qué tipo de datos recoge (logs, métricas, trazas) y qué activos instala (data streams, dashboards, pipelines).

2.  Abrir **Fleet** \> **Agent policies**. Anotar qué política existe y qué integraciones contiene. Esta política se asigna a los agentes que se enrolen.

3.  En **Fleet** \> **Settings**, localizar los Fleet Server hosts y los outputs. Así se sabe a dónde conectan los agentes.

4.  Anotar en la libreta qué integración usarías para recoger los logs de acceso de un servidor Nginx y por qué es preferible a escribir un grok propio.

Contexto: Elastic Agent es el agente único para logs, métricas y seguridad, administrado desde Fleet. Beats (Filebeat, Metricbeat) siguen soportados. Los datos de aplicaciones se envían con APM (Elastic Distributions of OpenTelemetry o agentes APM) y se ven en Observability.

## Parte F: limpieza y cierre (4 min)

### Paso F1. Retirar la regla y el conector

1.  Abrir **Stack Management** \> **Rules**, seleccionar la regla `Errores 5xx en weblogs-live` y eliminarla. Una regla activa sigue ejecutándose cada minuto.

2.  Abrir **Connectors** y eliminar `Alertas curso`.

``` console
DELETE alertas-curso
```

**Qué hace:** elimina el índice donde el conector escribía las notificaciones.

## Resumen de valores esperados

| **Comprobación**                             | **Resultado**                                   |
|----------------------------------------------|-------------------------------------------------|
| Discover, 12 a 15 de octubre (UTC)           | 4976 documentos                                 |
| KQL `service : "payments" and status >= 500` | 40                                              |
| Panel 1: total de peticiones                 | 4976                                            |
| Panel 3: errores 5xx por servicio            | payments 40, checkout 25, search 16, catalog 14 |
| `weblogs-live` tras la Parte C               | 6 documentos con `@timestamp` actual            |
| Alerta tras provocar la falla                | `valor` de 6 en `alertas-curso`                 |
| Alerta tras 5 minutos sin errores            | Documento con estado Recovered                  |

## Estado final

| **Elemento**                                                                                     | **Se conserva**  | **Lo reutiliza**                         |
|--------------------------------------------------------------------------------------------------|------------------|------------------------------------------|
| Dashboard `Operación weblogs` y búsqueda `Errores 5xx de payments`                               | Sí               | Laboratorio 10 (proyecto final)          |
| Data views `weblogs` y `weblogs-live`, `live-pipeline`, `weblogs-live-template` y `weblogs-live` | Sí               | Laboratorio 9 (runbook) y proyecto final |
| Regla, conector `Alertas curso` e índice `alertas-curso`                                         | No (se eliminan) | \-                                       |

## Preguntas de discusión

- ¿Por qué la regla usa una ventana de 5 minutos con un intervalo de 1 minuto, y no una ventana de 1 minuto?

- ¿Qué ocurriría con el umbral de 4 si el tráfico se multiplicara por 100? ¿Convendría una tasa de error en lugar de un conteo?

- ¿Qué diferencia hay entre alertar por un síntoma (errores 5xx) y por una causa (CPU alta)? ¿Cuál despierta a alguien de noche?

- Si el dashboard lo ven gerentes y también el equipo de guardia, ¿cuántos dashboards harías?

- ¿Qué cambia en el diseño de las alertas con 200 servicios en lugar de 4?

- ¿Por qué conviene que los datos entren con Elastic Agent y una integración, en vez de un script propio?

## Resultado final

Al terminar el laboratorio, el entorno debe quedar como se muestra en la imagen: lo que se conserva, cómo verificarlo y lo que se elimina.

![Resultado final del laboratorio 8: Construir dashboards y configurar alertas](imagenes/lab08_resultado_final.png)
