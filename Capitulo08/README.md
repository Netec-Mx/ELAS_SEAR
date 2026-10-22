# Administración de Elasticsearch para gestión y análisis de logs

## Práctica 8a. Construir un dashboard operativo de logs con Lens, controles y drilldowns

### Objetivo

Al finalizar la práctica, serás capaz de:

- Crear visualizaciones con Lens: métricas, series de tiempo, barras y tablas, incluidas fórmulas.
- Crear un mapa con la geolocalización de los clientes.
- Armar un dashboard operativo con controles, filtros y drilldowns entre dashboards.
- Organizar el contenido con Kibana Spaces.

### Objetivo visual

![Dashboard operativo con controles y drilldown](img/cap08a.png)

### Duración aproximada

- 45 minutos.

### Prerrequisitos

- Práctica 7 completada: data views **Logs tienda - acceso web**, **Logs tienda - aplicación** y **Logs tienda - todas las fuentes**, y la búsqueda guardada `Incidente 2026-10-20 - Errores 5xx checkout`.
- Zona horaria de Kibana fijada en `America/Mexico_City` (Práctica 7, Paso 1).

---

### Instrucciones

**Escenario:** después del incidente del 20 de octubre, el equipo de operaciones necesita un dashboard que muestre en una sola pantalla la salud de la tienda: volumen, tasa de error, latencia, endpoints con problemas y origen de los clientes. Desde ahí debe poder saltar a un dashboard de detalle.

> **Importante:** los nombres de menús y botones de Kibana están en inglés, porque Kibana no tiene interfaz en español. Entre versiones menores de 8.x algunos botones cambian ligeramente de lugar; si no encuentras uno, busca la opción equivalente en el menú del panel (⋯) o en la barra superior.

---

### Tarea 1. Crear el dashboard y sus métricas principales

#### Paso 1. Crea el dashboard y fija el rango de tiempo

1. Ve a **Analytics > Dashboards** y haz clic en **Create dashboard**.
2. En el selector de tiempo, define el rango absoluto **Oct 20, 2026 @ 08:00:00.000 → Oct 20, 2026 @ 20:00:00.000** y haz clic en **Update**.

**Qué hace y por qué:** un dashboard es un lienzo de paneles que comparten el rango de tiempo, la consulta y los filtros. Todos los paneles se actualizan juntos cuando cambias cualquiera de ellos.

#### Paso 2. Crea la métrica "Peticiones totales"

1. Haz clic en **Create visualization** (o **Add panel > Create visualization**). Se abre **Lens**.
2. Elige el data view **Logs tienda - acceso web**.
3. En el selector de tipo de gráfica elige **Metric**.
4. Arrastra **Records** (o el contador de documentos) a **Primary metric**.
5. Haz clic en la métrica, cambia **Name** a `Peticiones totales` y cierra el panel lateral.
6. Haz clic en **Save and return**.

**Qué hace y por qué:** Lens es el editor visual de Kibana: sugiere gráficas según los campos que arrastras y genera las agregaciones (Práctica 6) por ti. La métrica debe mostrar **3,421**.

#### Paso 3. Crea la métrica "% errores 5xx" con una fórmula

1. Crea otra visualización **Metric** con el mismo data view.
2. En **Primary metric**, elige **Formula** y escribe:

```text
count(kql='http.response.status_code >= 500') / count()
```

3. En **Value format** elige **Percent** con 2 decimales. Nombre: `% errores 5xx`.
4. Opcional: en **Color by value** define un color rojo cuando el valor sea mayor que 0.02 (2 %).
5. **Save and return**.

**Qué hace y por qué:** las **fórmulas** de Lens combinan funciones (`count`, `sum`, `average`, `percentile`, `unique_count`…) con operadores y filtros KQL. Una tasa de error es más útil que un conteo de errores porque no depende del volumen. El valor del día es **1.93 %**: 66 de 3,421.

#### Paso 4. Crea la métrica "Latencia p95 (ms)"

1. Nueva visualización **Metric**, mismo data view.
2. **Formula**:

```text
percentile(event.duration, percentile=95) / 1000000
```

3. **Value format**: **Number**, 0 decimales, sufijo ` ms`. Nombre: `Latencia p95`.
4. **Save and return**.

**Qué hace y por qué:** `event.duration` está en nanosegundos (ECS); dividir entre 1,000,000 lo convierte a milisegundos. Como viste en la Práctica 6, el p95 representa mejor la experiencia del usuario que el promedio.

---

### Tarea 2. Series de tiempo, barras, tabla y mapa

#### Paso 5. Crea la serie de tiempo por clase de respuesta

1. Nueva visualización con el data view **Logs tienda - acceso web**; tipo **Bar vertical stacked**.
2. Arrastra `@timestamp` al **Horizontal axis**, con el intervalo en **Customize time interval > 5 minutes**.
3. En **Vertical axis**, deja **Count of records**.
4. En **Breakdown**, elige la función **Filters** y crea tres filtros KQL con su etiqueta:
   - `http.response.status_code < 400` → `2xx/3xx`
   - `http.response.status_code >= 400 and http.response.status_code < 500` → `4xx`
   - `http.response.status_code >= 500` → `5xx`
5. Asigna los colores verde, amarillo y rojo en **Color**. Título del panel: `Peticiones por clase de respuesta`.
6. **Save and return**.

**Qué hace y por qué:** genera un `date_histogram` con una subagregación `filters` (Práctica 6). Se distinguen a simple vista el incidente (barras rojas entre 13:38 y 14:10), el ataque a `/api/login` (barras amarillas de 13:45 a 13:56) y la campaña de las 18:00 (pico solo verde).

#### Paso 6. Crea la tabla de endpoints con problemas

1. Nueva visualización, tipo **Table**.
2. **Rows**: `url.path` (**Top values**, 10, ordenado por la métrica de errores) y luego `nginx.access.upstream_host` (**Top values**, 3).
3. **Metrics**:
   - **Count of records** → nombre `Peticiones`.
   - **Formula** `count(kql='http.response.status_code >= 500')` → `Errores 5xx`.
   - **Formula** `count(kql='http.response.status_code >= 500') / count()` en formato **Percent** → `Tasa de error`.
   - **Formula** `percentile(event.duration, percentile=95) / 1000000` → `p95 ms`.
4. En la columna `Tasa de error`, activa **Color by value** (color de celda) con un umbral rojo a partir de 0.05.
5. Título: `Endpoints por servidor de aplicación`. **Save and return**.

**Qué hace y por qué:** la tabla resume en una vista lo que en la Práctica 7 encontraste con varias consultas: `/api/checkout` en `srv-app02` aparece en rojo, con una tasa de error alta y un p95 cercano a 3,000 ms. Las tablas con colores por umbral son el formato más eficiente para un top N operativo.

#### Paso 7. Crea la gráfica de errores de aplicación por servicio

1. Nueva visualización con el data view **Logs tienda - aplicación**; tipo **Bar horizontal**.
2. **Vertical axis**: `service.name` (**Top values**, 6). **Horizontal axis**: **Count of records**. **Breakdown**: `log.level`.
3. Título: `Eventos de aplicación por servicio y severidad`. **Save and return**.

**Qué hace y por qué:** un dashboard operativo combina fuentes: el log web muestra el **efecto** (503) y el log de aplicación, la **causa** (los ERROR de `checkout-service`). Cada panel puede usar su propio data view.

#### Paso 8. Crea un mapa del origen de las peticiones

1. En el dashboard, haz clic en **Add panel** (o **Add from library / Create visualization**) y elige **Maps**.
2. Haz clic en **Add layer > Clusters** (en algunas versiones, **Clusters and grids**).
3. Elige el data view **Logs tienda - acceso web**; campo geoespacial `source.geo.location`.
4. Haz clic en **Add and continue**; en **Metrics**, deja **Count**. Haz clic en **Keep changes**.
5. Agrega otra capa **Documents** con el mismo data view y en **Filtering** escribe `http.response.status_code : (401 or 429)` para marcar los intentos de inicio de sesión rechazados. Elige un color rojo.
6. Guarda el mapa como `Origen de peticiones` con **Add to dashboard** activado.

*Si no aparecen puntos, revisa en Discover si los documentos tienen `source.geo.location`. Sin acceso a Internet, la base GeoIP no se descarga (Práctica 5a, Paso 2) y el mapa queda vacío; continúa con el resto de la práctica.*

**Qué hace y por qué:** el procesador `geoip` de la Práctica 5a agregó coordenadas a cada IP pública. La capa de clusters muestra de dónde vienen los clientes (México en su mayoría) y la capa de documentos resalta el origen del ataque de fuerza bruta.

#### Paso 9. Agrega la búsqueda guardada y organiza el dashboard

1. Haz clic en **Add from library** y elige la búsqueda guardada `Incidente 2026-10-20 - Errores 5xx checkout`.
2. Organiza los paneles: las tres métricas arriba, la serie de tiempo debajo a lo ancho, después la tabla y la gráfica de aplicación lado a lado, y al final el mapa y la búsqueda guardada.
3. Haz clic en **Save**, nombra el dashboard `Operación tienda - Logs web` y activa **Store time with dashboard**.

**Qué hace y por qué:** un buen dashboard operativo va de lo general (¿estamos bien?) a lo particular (¿qué está fallando y dónde?). Guardar el tiempo con el dashboard hace que siempre abra en el día del incidente; en un dashboard real usarías un rango relativo, como *Last 24 hours*.

---

### Tarea 3. Controles y filtros interactivos

#### Paso 10. Agrega controles

1. Con el dashboard en modo edición, abre el menú **Controls** de la barra superior (en algunas versiones: **Add panel > Controls**) y elige **Add control**.
2. Crea estos controles con el data view **Logs tienda - acceso web**:
   - **Options list** sobre `host.name`, etiqueta `Servidor web`.
   - **Options list** sobre `nginx.access.upstream_host`, etiqueta `Servidor de aplicación`.
   - **Options list** sobre `url.path`, etiqueta `Endpoint`.
   - **Range slider** sobre `http.response.status_code`, etiqueta `Código HTTP`.
3. Opcional: agrega un **Time slider** para recorrer el día por intervalos.
4. Guarda el dashboard.

**Qué hace y por qué:** los **controles** permiten filtrar sin escribir KQL, lo que es ideal para usuarios no técnicos o para el NOC. Por defecto están **encadenados**: al elegir un servidor web, la lista de endpoints solo muestra los que ese servidor atendió.

#### Paso 11. Prueba los controles y los filtros

1. En **Servidor de aplicación** elige `srv-app02`, y en **Código HTTP** ajusta el rango a 500–599.
2. Observa cómo cambian todos los paneles. Luego limpia los controles.
3. Haz clic en una barra roja de la serie de tiempo: Kibana agrega un filtro y ajusta el rango de tiempo.
4. Regresa con el botón **Back** del navegador o restablece el rango y elimina el filtro.

**Qué hace y por qué:** los paneles de Lens son interactivos: hacer clic en un elemento crea un filtro para todo el dashboard (*filter on click*). Así se pasa de "algo está mal" a "esto es lo que está mal" en pocos clics.

---

### Tarea 4. Drilldowns entre dashboards

#### Paso 12. Crea el dashboard de detalle

1. Ve a **Analytics > Dashboards > Create dashboard**.
2. Agrega con **Add from library** la búsqueda guardada `Incidente 2026-10-20 - Errores 5xx checkout`.
3. Crea una visualización **Table** con el data view **Logs tienda - todas las fuentes**: **Rows** `trace.id` (Top values, 20) y `host.name`; **Metrics** **Count of records**.
4. Agrega un panel **Text** (Markdown) con instrucciones para el analista, por ejemplo: `Copie un trace.id y búsquelo en Discover con el data view "Logs tienda - todas las fuentes".`
5. Guarda como `Detalle de errores web`.

**Qué hace y por qué:** separar el resumen del detalle mantiene el dashboard principal ligero y legible; el detalle solo se abre cuando se necesita.

#### Paso 13. Configura el drilldown

1. Abre `Operación tienda - Logs web` en modo edición.
2. En el panel `Endpoints por servidor de aplicación`, abre el menú del panel (⋯) y elige **Create drilldown**.
3. Elige **Go to dashboard**, nombre `Ver detalle de errores`, destino `Detalle de errores web`, y deja activadas **Use filters and query from origin dashboard** y **Use date range from origin dashboard**.
4. Haz clic en **Create drilldown** y guarda el dashboard.
5. Sal del modo edición, haz clic en la celda `/api/checkout` de la tabla y elige **Ver detalle de errores**.

**Qué hace y por qué:** el **drilldown** abre el dashboard destino con el filtro de la celda en la que hiciste clic (`url.path: /api/checkout`), el rango de tiempo y la consulta del origen. Reproduce el flujo natural de investigación: resumen → detalle → evento individual. También existen drilldowns de **URL**, que abren un sistema externo, como un ticket o un runbook, con valores del evento.

---

### Tarea 5. Organizar con Kibana Spaces

#### Paso 14. Crea el espacio "Operaciones"

1. Ve a **Stack Management > Spaces** y haz clic en **Create space**.
2. Nombre: `Operaciones`; identificador: `operaciones`; elige un color y un avatar.
3. En **Features**, oculta las aplicaciones que el equipo no usa, por ejemplo, **Security** y **Machine Learning**.
4. Haz clic en **Create space**.

**Qué hace y por qué:** un **space** es un contenedor aislado de objetos de Kibana (dashboards, data views, búsquedas, reglas). Permite separar equipos o ambientes y, junto con los roles de la Práctica 9, controlar quién ve qué. Ocultar funciones simplifica el menú, pero **no** es un control de seguridad: los permisos se dan con roles.

#### Paso 15. Copia el dashboard al espacio

1. Ve a **Stack Management > Saved Objects**.
2. Busca `Operación tienda - Logs web`, abre su menú (⋯) y elige **Copy to spaces**.
3. Selecciona **Operaciones**, deja activada la opción de copiar los objetos relacionados y haz clic en **Copy**.
4. Cambia de espacio con el selector de la esquina superior izquierda y abre el dashboard en **Operaciones**.

**Qué hace y por qué:** al copiar, Kibana incluye las **dependencias**: visualizaciones, data views, la búsqueda guardada y el mapa. Para mover contenido entre clústeres (por ejemplo, de pruebas a producción) se usa **Export** e **Import** de saved objects en formato NDJSON.

---

### Resultado esperado

- Dashboard `Operación tienda - Logs web` con: tres métricas (3,421 peticiones, 1.93 % de errores 5xx, latencia p95), la serie de tiempo por clase de respuesta, la tabla de endpoints, la gráfica de la aplicación, el mapa, la búsqueda guardada y cuatro controles.
- Dashboard `Detalle de errores web`, al que se llega con el drilldown `Ver detalle de errores`.
- Space `Operaciones` con una copia del dashboard y sus dependencias.

---

## Práctica 8b. Generar reportes CSV y PDF del dashboard y configurar una alerta por umbral de errores

### Objetivo

Al finalizar la práctica, serás capaz de:

- Exportar a CSV una búsqueda de Discover y generar reportes PDF/PNG de un dashboard.
- Compartir enlaces y programar la generación de reportes en Windows.
- Configurar conectores y una regla de alerta por umbral de errores, y comprobar que se dispara.

### Objetivo visual

![Reportes y alertas a partir de los logs](img/cap08b.png)

### Duración aproximada

- 30 minutos.

### Prerrequisitos

- Práctica 8a completada.
- Licencia de prueba activa (Práctica 2, Paso 20): los reportes PDF/PNG de dashboards no están incluidos en la licencia Basic.
- Llaves de cifrado configuradas en `kibana.yml` (Práctica 2, Paso 18): las reglas de alerta las necesitan.
- **Filebeat en ejecución** (Práctica 5a, Paso 11), porque la alerta se probará con eventos nuevos.

---

### Instrucciones

**Escenario:** la dirección quiere recibir cada mañana un PDF del dashboard y el área de auditoría pide la lista de errores del incidente en CSV. Además, el reporte del incidente (Práctica 7) pedía una **alerta automática** para que el próximo problema en checkout no lo detecten los clientes.

---

### Tarea 1. Reportes CSV, PDF y enlaces

#### Paso 1. Exporta la búsqueda guardada a CSV

1. Ve a **Discover** y abre (**Open**) la búsqueda `Incidente 2026-10-20 - Errores 5xx checkout`.
2. Verifica el rango **Oct 20, 2026 08:00 – 20:00**.
3. Haz clic en **Share** (o en el ícono de compartir) > **Export** > **CSV** > **Generate CSV**.
4. Cuando aparezca la notificación, haz clic en **Download report**, o ve a **Stack Management > Reporting** y descárgalo desde ahí.

**Qué hace y por qué:** el CSV contiene las columnas de la búsqueda guardada para los 63 documentos que cumplen la consulta y el filtro. Kibana genera los reportes en **segundo plano** y los guarda en un índice del sistema, así que no bloquean al usuario y se pueden descargar después. El tamaño máximo se controla con `xpack.reporting.csv.maxSizeBytes` en `kibana.yml`.

#### Paso 2. Genera un PDF del dashboard

1. Abre el dashboard `Operación tienda - Logs web` (fuera del modo edición).
2. Haz clic en **Share** > **Export** > **PDF**, activa **For printing** (u **Optimize for printing**) y haz clic en **Generate PDF**.
3. Repite con **PNG**.
4. Descarga ambos desde **Stack Management > Reporting**.

**Qué hace y por qué:** Kibana usa un navegador Chromium sin interfaz (*headless*), incluido en la distribución de Windows, para renderizar el dashboard y convertirlo a PDF o PNG. La primera generación puede tardar más mientras se inicia Chromium. *Si el reporte falla, revisa en la consola de Kibana los mensajes de `reporting`; la causa más común es poca memoria disponible.*

#### Paso 3. Comparte un enlace

1. En el dashboard, haz clic en **Share > Link**.
2. Activa **Short URL** y copia el enlace.
3. Ábrelo en una ventana privada del navegador.

**Qué hace y por qué:** el enlace incluye el estado actual: rango de tiempo, filtros y valores de los controles. Quien lo abra debe autenticarse y tener permiso sobre el dashboard y los datos: compartir el enlace no comparte el acceso.

---

### Tarea 2. Programar reportes

#### Paso 4. Copia la URL de generación del PDF

1. En el dashboard, haz clic en **Share > Export > PDF**.
2. Haz clic en **Copy POST URL** (en algunas versiones está dentro de **Advanced options**).
3. Pégala en un archivo de texto temporal.

**Qué hace y por qué:** la **POST URL** es la llamada a la API de reporting de Kibana que genera exactamente ese reporte. Cualquier programador de tareas puede invocarla para automatizar el reporte.

#### Paso 5. Crea una API key para el script

En **Dev Tools**:

```http
POST _security/api_key
{
  "name": "reportes-programados-dashboard",
  "expiration": "30d",
  "metadata": { "proposito": "Practica 8b - reporte PDF diario" }
}
```

Copia el valor del campo **`encoded`** de la respuesta.

**Qué hace y por qué:** una **API key** permite que el script se autentique sin guardar la contraseña de `elastic`; además, se puede revocar en cualquier momento y caduca sola. Kibana acepta API keys de Elasticsearch en el encabezado `Authorization: ApiKey <encoded>`. En la Práctica 9 verás cómo limitar sus privilegios.

#### Paso 6. Configura y prueba el script

1. Abre el script:

```powershell
notepad C:\labs\scripts\generar_reporte.ps1
```

2. Sustituye `PEGAR_AQUI_LA_POST_URL` por la POST URL y `PEGAR_AQUI_EL_VALOR_ENCODED` por la API key. Conserva las comillas simples y guarda el archivo.
3. Ejecútalo:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\labs\scripts\generar_reporte.ps1
```

**Qué hace y por qué:** el script hace la petición POST con el encabezado `kbn-xsrf` (obligatorio en las APIs de Kibana que modifican datos) y registra la respuesta en `C:\labs\reportes\reportes_programados.log`. La respuesta incluye un `path` para descargar el reporte. En **Stack Management > Reporting** aparece un nuevo PDF. Las comillas simples son importantes porque la POST URL contiene caracteres como `$` y `%` que PowerShell interpretaría.

#### Paso 7. Programa la tarea en Windows

En PowerShell como administrador:

```powershell
schtasks /Create /TN "Reporte diario dashboard logs" /SC DAILY /ST 07:00 `
  /TR "powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\labs\scripts\generar_reporte.ps1"

schtasks /Run /TN "Reporte diario dashboard logs"
Get-Content C:\labs\reportes\reportes_programados.log -Tail 2
```

**Qué hace y por qué:** el Programador de tareas de Windows ejecuta el script cada día a las 07:00; `schtasks /Run` lo ejecuta de inmediato para probarlo. En Elastic 8.x, las opciones nativas para programar reportes son **Watcher** (con una acción `webhook` o `email` con adjunto de reporting, que requiere licencia Gold o superior) o un programador externo como este.

> **Importante:** como el dashboard guarda un rango absoluto, el reporte siempre mostrará el 20 de octubre. En un reporte real, guarda el dashboard con un rango relativo (por ejemplo, *Yesterday*) antes de copiar la POST URL. Al terminar el curso, elimina la tarea con `schtasks /Delete /TN "Reporte diario dashboard logs" /F`.

---

### Tarea 3. Alerta por umbral de errores

#### Paso 8. Crea el índice de destino de las alertas

En **Dev Tools**:

```http
PUT alertas-lab-logs
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "regla":      { "type": "keyword" },
      "conteo":     { "type": "long" },
      "mensaje":    { "type": "text" }
    }
  }
}
```

**Qué hace y por qué:** el conector **Index** escribe un documento por cada notificación de alerta. Así queda un historial consultable de alertas que puede graficarse o auditarse, sin depender de un servidor de correo.

#### Paso 9. Crea los conectores

1. Ve a **Stack Management > Connectors** y haz clic en **Create connector**.
2. Elige **Index**: nombre `Indice de alertas`, índice `alertas-lab-logs`, activa **Define time field for each document**. Haz clic en **Save**.
3. Crea otro conector **Server log** con el nombre `Log de Kibana`.

**Qué hace y por qué:** un **conector** define **cómo** se notifica (índice, log del servidor, correo, Slack, Microsoft Teams, webhook, ServiceNow, etc.) y se reutiliza en muchas reglas. Index y Server log están incluidos en la licencia Basic y no requieren servicios externos, por eso se usan en el laboratorio. En producción se agregaría, por ejemplo, un conector de correo con el servidor SMTP corporativo.

#### Paso 10. Crea la regla de alerta

1. Ve a **Stack Management > Rules** (o **Alerts and Insights > Rules**) y haz clic en **Create rule**.
2. Elige el tipo **Elasticsearch query**.
3. Configura:
   - **Name**: `Errores de checkout - mas de 5 en 1 minuto`
   - **Query type**: **KQL or Lucene**
   - **Data view**: **Logs tienda - aplicación**
   - **Query**: `log.level : "ERROR" and service.name : "checkout-service"`
   - **When**: **count** **IS ABOVE** `5` **FOR THE LAST** `1 minute`
   - **Check every**: `1 minute`
4. En **Actions**, agrega el conector `Indice de alertas`. En **Action frequency** elige **On status changes**, y en **Document to index** escribe:

```json
{
  "@timestamp": "{{date}}",
  "regla": "{{rule.name}}",
  "conteo": "{{context.value}}",
  "mensaje": "{{context.message}}"
}
```

5. Agrega una segunda acción con el conector `Log de Kibana`, con el mensaje `ALERTA: {{rule.name}} - {{context.value}} errores en el ultimo minuto`.
6. Opcional: agrega acciones de **Recovered** para registrar cuándo se resuelve.
7. Haz clic en **Save**.

**Qué hace y por qué:** cada minuto, la regla cuenta los ERROR de checkout del último minuto. Si hay más de 5, la alerta pasa a **Active** y ejecuta las acciones; cuando el conteo baja, pasa a **Recovered**. Las variables entre `{{ }}` (Mustache) insertan datos de la ejecución. **On status changes** evita notificar cada minuto mientras la alerta siga activa, porque el exceso de notificaciones (*alert fatigue*) es la principal razón por la que los equipos dejan de atender las alertas.

#### Paso 11. Provoca la alerta con errores simulados

Verifica que Filebeat siga en ejecución y, en PowerShell, ejecuta:

```powershell
C:\labs\scripts\generar_errores.ps1 -Cantidad 20 -IntervaloSeg 2
```

**Qué hace y por qué:** el script agrega 20 líneas `ERROR [checkout-service]` con la hora **actual** a `C:\labs\logs\app\app.log`, una cada 2 segundos. Filebeat las envía al data stream y el pipeline `weblab-app` las procesa igual que los logs originales. En unos 40 segundos se acumulan más de 5 errores en un minuto.

#### Paso 12. Comprueba que la alerta se disparó

1. Espera 1 o 2 minutos. En **Stack Management > Rules**, abre la regla: en la pestaña **Alerts** debe aparecer una alerta **Active**.
2. En la ventana de consola de Kibana busca la línea `ALERTA: Errores de checkout...`.
3. En **Dev Tools**:

```http
GET alertas-lab-logs/_search
{ "sort": [ { "@timestamp": "desc" } ] }
```

4. Unos minutos después de que el script termine, la alerta pasa a **Recovered**.

**Qué hace y por qué:** confirma la cadena completa: log → Filebeat → pipeline → data stream → regla → conectores. Probar una alerta con un evento simulado es una buena práctica: una alerta que nunca se ha visto disparar no es confiable.

#### Paso 13. (Opcional) Crea la misma regla con ES|QL

Crea otra regla **Elasticsearch query**, con **Query type: ES|QL**:

```esql
FROM logs-weblab.app-default
| WHERE log.level == "ERROR"
| STATS errores = COUNT(*) BY service.name
| WHERE errores > 5
```

Con **Time field** `@timestamp`, **For the last** `1 minute` y las mismas acciones.

**Qué hace y por qué:** con ES|QL, la regla genera una alerta cuando la consulta devuelve filas. El `BY service.name` hace que la regla cubra **todos** los servicios y no solo checkout. Otra alternativa es la regla **Log threshold** de Observability, pensada específicamente para conteos de logs agrupados por campo.

#### Paso 14. Silencia la regla al terminar

En la lista de reglas, abre el menú de la regla y elige **Snooze** (por ejemplo, 1 día) o **Disable**.

**Qué hace y por qué:** *snooze* silencia temporalmente las notificaciones, sin dejar de evaluar la regla, durante un mantenimiento o una prueba. *Disable* detiene la evaluación. Así evitas notificaciones al repetir ejercicios en prácticas posteriores.

---

### Resultado esperado

- En **Stack Management > Reporting** aparecen un CSV (63 filas), un PDF y un PNG del dashboard, y un PDF más generado por el script programado.
- La tarea programada `Reporte diario dashboard logs` existe en Windows y su bitácora registra la respuesta de Kibana.
- La regla `Errores de checkout - mas de 5 en 1 minuto` pasó a **Active** al ejecutar el script y después a **Recovered**.
- El índice `alertas-lab-logs` contiene al menos un documento similar a:

```json
{
  "@timestamp": "2026-10-21T16:42:10.512Z",
  "regla": "Errores de checkout - mas de 5 en 1 minuto",
  "conteo": 9,
  "mensaje": "Document count is 9 in the last 1m in Logs tienda - aplicación data view. Alert when greater than 5."
}
```

### Referencias

- Elastic. *Lens [8.19].* https://www.elastic.co/guide/en/kibana/8.19/lens.html
- Elastic. *Dashboards, controls y drilldowns [8.19].* https://www.elastic.co/guide/en/kibana/8.19/dashboard.html
- Elastic. *Kibana Spaces [8.19].* https://www.elastic.co/guide/en/kibana/8.19/xpack-spaces.html
- Elastic. *Reporting and sharing [8.19].* https://www.elastic.co/guide/en/kibana/8.19/reporting-getting-started.html
- Elastic. *Alerting y Elasticsearch query rule [8.19].* https://www.elastic.co/guide/en/kibana/8.19/rule-type-es-query.html
- Elastic. *Connectors [8.19].* https://www.elastic.co/guide/en/kibana/8.19/action-types.html
