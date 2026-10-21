# Administración de Elasticsearch para gestión y análisis de logs

## Práctica 2. Instalar y configurar Elasticsearch y Kibana 8.x en Windows

### Objetivo

Al finalizar la práctica, serás capaz de:

- Preparar un equipo con Windows 11 para ejecutar Elasticsearch y Kibana 8.19.
- Instalar y configurar un clúster de un nodo con la seguridad activada por defecto (TLS y autenticación).
- Instalar Elasticsearch como servicio de Windows y conectar Kibana mediante un enrollment token.
- Verificar la arquitectura del clúster (roles de nodo, shards primarios y réplicas) y diagnosticar por qué un índice queda en estado `yellow`.

### Objetivo visual

![Arquitectura del laboratorio en Windows 11](img/cap02.png)

### Duración aproximada

- 45 minutos (más el tiempo de descarga de los paquetes).

### Prerrequisitos

- Equipo con **Windows 11 de 64 bits**, mínimo **8 GB de RAM** (se recomiendan 16 GB) y **20 GB libres** en disco.
- Cuenta con **privilegios de administrador** local.
- Acceso a Internet para descargar los paquetes, o los archivos `.zip` entregados por el instructor (ambiente sin Internet).
- Paquete de laboratorio `labs.zip` entregado por el instructor (contiene los logs de ejemplo, configuraciones y scripts del curso).
- Un navegador web actualizado (Edge, Chrome o Firefox).

> **Importante:** en todo el curso se usa la versión **8.19.x**. Usa la **misma versión** para Elasticsearch, Kibana, Filebeat, Logstash y Metricbeat. En esta guía el valor de ejemplo es `8.19.21`; si el instructor indica otro parche, sustitúyelo en la variable `$VER`.

---

### Instrucciones

### Tarea 1. Preparar el sistema operativo

Elasticsearch es una aplicación Java que usa mucha memoria, muchos archivos abiertos y red. En Linux esto obliga a ajustar `vm.max_map_count`, `ulimits` y la swap (subtema 2.4). En Windows esos parámetros no existen o se administran de otra forma; esta tarea aplica los equivalentes de Windows y deja listo el directorio de trabajo.

#### Paso 1. Abre PowerShell como administrador y verifica los recursos

1. Haz clic derecho en el botón **Inicio** y elige **Terminal (Administrador)** (o **Windows PowerShell (Administrador)**).
2. Ejecuta:

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, OSArchitecture
"{0:N1} GB de RAM" -f ((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB)
Get-PSDrive C | Select-Object @{n='Libre(GB)';e={[math]::Round($_.Free/1GB,1)}}
curl.exe --version
```

**Qué hace y por qué:** confirma que el equipo es de 64 bits, que tiene memoria y disco suficientes, y que `curl.exe` está disponible (viene incluido en Windows 11). Con 8 GB de RAM asignaremos 2 GB al heap de Elasticsearch para dejar memoria a Kibana, Logstash y al sistema operativo.

#### Paso 2. Crea las carpetas de trabajo y copia el paquete de laboratorio

```powershell
New-Item -ItemType Directory -Force -Path C:\elastic, C:\elastic\data, C:\elastic\logs, C:\elastic\descargas
# Ajusta la ruta de origen si labs.zip quedó en otra carpeta
Expand-Archive -Path "$env:USERPROFILE\Downloads\labs.zip" -DestinationPath C:\ -Force
Get-ChildItem C:\labs
```

**Qué hace y por qué:** separa el software (`C:\elastic`), los datos del nodo (`C:\elastic\data`) y sus logs (`C:\elastic\logs`). Separar datos y binarios facilita las actualizaciones, ya que se reemplaza la carpeta del programa sin tocar los datos. Las rutas cortas evitan el límite de 260 caracteres de Windows, que afecta sobre todo a Kibana. `C:\labs` contiene los logs de ejemplo (`logs`), los archivos NDJSON (`datos`), las configuraciones (`config`) y los scripts (`scripts`).

#### Paso 3. Permite la ejecución de los scripts del curso

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force
Get-ChildItem -Recurse C:\labs | Unblock-File
```

**Qué hace y por qué:** Windows 11 bloquea por defecto los scripts de PowerShell y marca como "descargados de Internet" los archivos de un `.zip`. `RemoteSigned` permite ejecutar scripts locales, y `Unblock-File` quita la marca de origen a los scripts del curso.

#### Paso 4. Aplica los equivalentes de Windows de la preparación del sistema operativo

```powershell
# Sincronización de hora (NTP): los timestamps de los logs dependen de ella
w32tm /query /status
w32tm /resync

# (Opcional, recomendado) Habilitar rutas largas para Kibana
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
  -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

# (Opcional, según la política de tu organización) excluir C:\elastic del antivirus
Add-MpPreference -ExclusionPath "C:\elastic"
```

**Qué hace y por qué:**

| Parámetro en Linux (2.4) | Equivalente en Windows | Acción en este laboratorio |
|---|---|---|
| `vm.max_map_count` | No aplica; Windows no limita el número de mapeos de memoria. | Ninguna. |
| `ulimits` (archivos y procesos) | No aplica; Windows administra los handles de forma dinámica. | Ninguna. |
| Swap desactivada o `memory_lock` | Archivo de paginación; `bootstrap.memory_lock: true` usa `VirtualLock`. | Se mantiene el valor por defecto en el laboratorio; en producción se recomienda bloquear la memoria. |
| Firewall (`firewalld`, `ufw`) | Firewall de Windows Defender. | Solo se abre si otro equipo debe conectarse (Paso 5). |
| NTP (`chronyd`) | Servicio Hora de Windows (`w32tm`). | Verificar y resincronizar. |

Si `w32tm /resync` responde que el servicio no está iniciado, ejecuta antes `Start-Service w32time`. Un reloj desfasado provoca que los eventos aparezcan "en el futuro" o fuera del rango de tiempo en Kibana, que los certificados TLS se consideren inválidos y que las alertas se evalúen sobre ventanas de tiempo incorrectas.

#### Paso 5. (Opcional) Abre los puertos en el firewall

Haz este paso solo si otros equipos de la red se conectarán a tu clúster, por ejemplo, el instructor.

```powershell
New-NetFirewallRule -DisplayName "Elasticsearch HTTP 9200" -Direction Inbound -Protocol TCP -LocalPort 9200 -Action Allow -Profile Private
New-NetFirewallRule -DisplayName "Kibana 5601" -Direction Inbound -Protocol TCP -LocalPort 5601 -Action Allow -Profile Private
```

**Qué hace y por qué:** el puerto **9200** es la capa HTTP (API REST) y el **5601** es Kibana. El puerto **9300** (capa de transporte, comunicación entre nodos) solo se abre en clústeres de varios nodos. Limitar las reglas al perfil `Private` evita exponer el servicio en redes públicas.

---

### Tarea 2. Instalar y configurar Elasticsearch

Instalarás Elasticsearch desde el paquete `.zip` para Windows, ajustarás su configuración básica y harás el primer arranque, en el que la seguridad se configura automáticamente.

#### Paso 6. Descarga el paquete y verifica su integridad

```powershell
$VER = "8.19.21"
cd C:\elastic\descargas
curl.exe -L -O "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-$VER-windows-x86_64.zip"
curl.exe -L -O "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-$VER-windows-x86_64.zip.sha512"

$esperado = (Get-Content ".\elasticsearch-$VER-windows-x86_64.zip.sha512").Split(" ")[0]
$real = (Get-FileHash ".\elasticsearch-$VER-windows-x86_64.zip" -Algorithm SHA512).Hash.ToLower()
if ($esperado -eq $real) { "Checksum OK" } else { "ERROR: el archivo esta danado o fue alterado" }
```

*Si trabajas en una red sin acceso a Internet, copia el `.zip` y el `.sha512` entregados por el instructor a `C:\elastic\descargas` y ejecuta solo la verificación.*

**Qué hace y por qué:** descarga la distribución oficial (incluye su propio JDK, por lo que no necesitas instalar Java) y compara la huella SHA-512. En entornos on premise sin Internet es habitual mover los paquetes en medios internos; el checksum garantiza que el archivo llegó íntegro y sin alteraciones.

#### Paso 7. Extrae Elasticsearch

```powershell
tar -xf ".\elasticsearch-$VER-windows-x86_64.zip" -C C:\elastic
Get-ChildItem C:\elastic -Directory -Filter "elasticsearch-$VER*" | Rename-Item -NewName "elasticsearch"
Get-ChildItem C:\elastic\elasticsearch
```

**Qué hace y por qué:** `tar` (incluido en Windows 11) extrae archivos `.zip` mucho más rápido que `Expand-Archive`. Renombrar la carpeta a `elasticsearch`, sin el número de versión, permite que las rutas de las prácticas y scripts no cambien al actualizar. Observa las carpetas `bin` (ejecutables `.bat`), `config` (configuración), `jdk` (Java incluido), `lib`, `modules` y `plugins`.

#### Paso 8. Configura `elasticsearch.yml`

1. Abre el archivo de configuración:

```powershell
notepad C:\elastic\elasticsearch\config\elasticsearch.yml
```

2. Agrega al final las siguientes líneas y guarda el archivo:

```yaml
cluster.name: lab-logs
node.name: nodo-01
path.data: C:\elastic\data
path.logs: C:\elastic\logs
```

**Qué hace y por qué:**

- `cluster.name`: los nodos solo se unen a un clúster con el mismo nombre. Un nombre propio evita que un nodo se una por error a otro clúster de la red.
- `node.name`: nombre legible del nodo en las APIs (`_cat/nodes`) y en Stack Monitoring.
- `path.data` y `path.logs`: separan los datos y los logs del directorio del programa, para que una actualización no los borre.

No configures todavía la seguridad, la red ni el descubrimiento: en el primer arranque, Elasticsearch 8.x lo hace automáticamente (*security auto-configuration*).

#### Paso 9. Ajusta el tamaño del heap de la JVM

```powershell
@"
-Xms2g
-Xmx2g
"@ | Set-Content -Encoding ascii C:\elastic\elasticsearch\config\jvm.options.d\heap.options
Get-Content C:\elastic\elasticsearch\config\jvm.options.d\heap.options
```

**Qué hace y por qué:** fija el heap mínimo (`-Xms`) y el máximo (`-Xmx`) en el mismo valor para evitar redimensionamientos en tiempo de ejecución. La regla general (subtema 2.3) es asignar **hasta el 50 % de la RAM** al heap y **nunca más de unos 31 GB**, para conservar los punteros comprimidos de la JVM. La otra mitad de la RAM la usa el sistema operativo como caché de archivos, algo esencial para Lucene. No se edita `jvm.options` directamente: los archivos en `jvm.options.d` sobreviven a las actualizaciones.

#### Paso 10. Inicia Elasticsearch por primera vez en consola

```powershell
cd C:\elastic\elasticsearch
.\bin\elasticsearch.bat
```

Espera a que aparezca un recuadro similar al siguiente (puede tardar 1 o 2 minutos):

```text
✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ️  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  xY7*q2...

ℹ️  HTTP CA certificate SHA-256 fingerprint:
  3b1c...

ℹ️  Configure Kibana to use this cluster:
• Run Kibana and click the configuration link in the terminal when Kibana starts.
• Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjE5...
```

Copia la contraseña, la huella (fingerprint) y el **enrollment token** a un archivo de texto temporal. **No cierres esta ventana.**

**Qué hace y por qué:** en el primer arranque, Elasticsearch 8.x genera una autoridad certificadora (CA) propia, certificados para las capas HTTP y transporte, la contraseña del superusuario `elastic` y un token de inscripción para Kibana. Es la "seguridad por defecto" del subtema 9.1: desde el primer minuto, el clúster no acepta conexiones sin TLS ni sin credenciales.

> **Importante:** el enrollment token caduca en 30 minutos. Si caduca antes de configurar Kibana, genera otro con `C:\elastic\elasticsearch\bin\elasticsearch-create-enrollment-token.bat -s kibana`.

#### Paso 11. Define la contraseña del usuario `elastic`

Abre una **segunda** ventana de PowerShell como administrador y ejecuta:

```powershell
cd C:\elastic\elasticsearch
.\bin\elasticsearch-reset-password.bat -u elastic -i
```

Responde `y` a la confirmación y escribe dos veces la contraseña que usarás durante el curso.

**Qué hace y por qué:** la contraseña generada es aleatoria y difícil de recordar. Con `-i` (interactivo) defines una contraseña propia. En todo el curso se usa el superusuario estándar **`elastic`**. En la Práctica 9 crearás usuarios de mínimo privilegio, que son la práctica recomendada en producción.

#### Paso 12. Revisa los cambios que hizo la configuración automática

```powershell
Get-Content C:\elastic\elasticsearch\config\elasticsearch.yml | Select-Object -Last 30
Get-ChildItem C:\elastic\elasticsearch\config\certs
```

**Qué hace y por qué:** verás un bloque `BEGIN SECURITY AUTO CONFIGURATION` con `xpack.security.enabled: true`, `xpack.security.http.ssl` (certificado `http.p12`), `xpack.security.transport.ssl` (certificado `transport.p12`), `cluster.initial_master_nodes` (el nombre de tu equipo) y `http.host: 0.0.0.0`. En la carpeta `certs` está `http_ca.crt`, el certificado de la CA que usarán curl, Filebeat y Logstash para confiar en el clúster.

#### Paso 13. Verifica la conexión con curl

En la segunda ventana de PowerShell, carga el script de entorno del curso y prueba la API:

```powershell
. C:\labs\scripts\entorno.ps1
es GET "/?pretty"
es GET "/_cluster/health?pretty"
```

El script solicita la contraseña de `elastic` una sola vez por ventana. Equivale a ejecutar:

```powershell
curl.exe --cacert C:\elastic\elasticsearch\config\certs\http_ca.crt --ssl-no-revoke -u "elastic:$env:ES_PWD" "https://localhost:9200/?pretty"
```

**Qué hace y por qué:**

- `--cacert` indica a curl que confíe en la CA generada por Elasticsearch, en lugar de desactivar la validación con `-k`.
- `--ssl-no-revoke` es necesario en Windows: el curl de Windows usa Schannel, que intenta consultar listas de revocación (CRL) y falla con una CA privada que no publica ninguna.
- `-u` envía las credenciales con autenticación básica.

La primera respuesta muestra `cluster_name: lab-logs`, el número de versión `8.19.x` y el lema `"You Know, for Search"`. La segunda muestra `"status" : "green"`.

---

### Tarea 3. Instalar Elasticsearch como servicio de Windows

Ejecutar Elasticsearch en una consola sirve para el primer arranque, pero en un servidor debe iniciarse sin sesión de usuario y reiniciarse con el equipo.

#### Paso 14. Detén la instancia en consola e instala el servicio

1. En la primera ventana (donde corre Elasticsearch), presiona `Ctrl + C` y confirma con `S` o `Y` si se solicita.
2. En la ventana de administrador ejecuta:

```powershell
cd C:\elastic\elasticsearch
.\bin\elasticsearch-service.bat install
Set-Service -Name "elasticsearch-service-x64" -StartupType Automatic
Start-Service -Name "elasticsearch-service-x64"
Get-Service -Name "elasticsearch-service-x64"
```

**Qué hace y por qué:** `elasticsearch-service.bat` registra el servicio `elasticsearch-service-x64`. Al instalarlo, lee las opciones de la JVM, incluido el heap del Paso 9; por eso el heap se ajusta **antes** de instalar el servicio. Si después necesitas cambiar el heap, usa `elasticsearch-service.bat manager` o reinstala el servicio. El tipo de inicio `Automatic` hace que el servicio arranque con Windows.

#### Paso 15. Comprueba que el servicio responde

Espera unos 30 segundos y ejecuta:

```powershell
es GET "/_cat/nodes?v&h=name,node.role,master,heap.max,ram.percent,cpu"
Get-Content C:\elastic\logs\lab-logs.log -Tail 20
```

**Qué hace y por qué:** `_cat/nodes` muestra el nodo `nodo-01` con el heap máximo de 2 GB. El log del clúster se llama `<cluster.name>.log` y está en `path.logs`; es el primer lugar donde se revisa un problema de arranque.

---

### Tarea 4. Instalar y conectar Kibana

#### Paso 16. Descarga y extrae Kibana

```powershell
$VER = "8.19.21"
cd C:\elastic\descargas
curl.exe -L -O "https://artifacts.elastic.co/downloads/kibana/kibana-$VER-windows-x86_64.zip"
curl.exe -L -O "https://artifacts.elastic.co/downloads/kibana/kibana-$VER-windows-x86_64.zip.sha512"
$esperado = (Get-Content ".\kibana-$VER-windows-x86_64.zip.sha512").Split(" ")[0]
$real = (Get-FileHash ".\kibana-$VER-windows-x86_64.zip" -Algorithm SHA512).Hash.ToLower()
if ($esperado -eq $real) { "Checksum OK" } else { "ERROR en checksum" }

tar -xf ".\kibana-$VER-windows-x86_64.zip" -C C:\elastic
Get-ChildItem C:\elastic -Directory -Filter "kibana-$VER*" | Rename-Item -NewName "kibana"
```

**Qué hace y por qué:** el `.zip` crea una carpeta con el número de versión (por ejemplo `kibana-8.19.21` o `kibana-8.19.21-windows-x86_64`); el filtro con comodín la renombra a `kibana` sin importar la variante. Kibana contiene decenas de miles de archivos pequeños (Node.js). `tar` los extrae en pocos minutos; `Expand-Archive` puede tardar mucho más o fallar con rutas largas.

#### Paso 17. Genera las llaves de cifrado de Kibana

```powershell
cd C:\elastic\kibana
.\bin\kibana-encryption-keys.bat generate -q
```

La salida tiene tres líneas similares a estas:

```yaml
xpack.encryptedSavedObjects.encryptionKey: 4f1d...
xpack.reporting.encryptionKey: 9a7e...
xpack.security.encryptionKey: c2b0...
```

**Qué hace y por qué:** Kibana cifra objetos sensibles (credenciales de conectores, reglas de alerta, reportes y sesiones). Sin `xpack.encryptedSavedObjects.encryptionKey` **no se pueden crear reglas de alerta** (Práctica 8b), y sin una llave fija los reportes y las sesiones se invalidan en cada reinicio.

#### Paso 18. Configura `kibana.yml`

```powershell
notepad C:\elastic\kibana\config\kibana.yml
```

Agrega al final y guarda (pega tus tres llaves del paso anterior):

```yaml
server.port: 5601
server.host: "localhost"
server.name: "kibana-lab"
xpack.encryptedSavedObjects.encryptionKey: "<llave_1>"
xpack.reporting.encryptionKey: "<llave_2>"
xpack.security.encryptionKey: "<llave_3>"
```

**Qué hace y por qué:** `server.host: "localhost"` limita Kibana al equipo local. Para que otros equipos accedan se usa la IP o `0.0.0.0` y se abre el puerto en el firewall (Paso 5). La conexión con Elasticsearch **no** se configura aquí: la escribirá el proceso de inscripción en el paso siguiente.

#### Paso 19. Inicia Kibana e inscríbelo en el clúster

1. Inicia Kibana en una ventana de PowerShell (no requiere administrador):

```powershell
cd C:\elastic\kibana
.\bin\kibana.bat
```

2. Espera el mensaje `Kibana has not been configured. Go to http://localhost:5601/?code=XXXXXX to get started.` (2 a 4 minutos).
3. Abre ese enlace en el navegador, pega el **enrollment token** del Paso 10 y haz clic en **Configure Elastic**.
4. Si se solicita un código de verificación, consúltalo en la consola de Kibana o con `C:\elastic\kibana\bin\kibana-verification-code.bat`.
5. Inicia sesión con el usuario **`elastic`** y la contraseña del Paso 11.

*Si aparece el mensaje de que el token es inválido o caducó, genera uno nuevo con `C:\elastic\elasticsearch\bin\elasticsearch-create-enrollment-token.bat -s kibana`.*

**Qué hace y por qué:** la inscripción (*enrollment*) intercambia de forma segura la CA del clúster y crea un **service account token** para Kibana (`elastic/kibana`). Kibana deja de usar un usuario con contraseña para conectarse, lo que es más seguro. Revisa el final de `kibana.yml`: se agregaron `elasticsearch.hosts`, `elasticsearch.serviceAccountToken` y `elasticsearch.ssl.certificateAuthorities`.

> **Importante:** Kibana se ejecuta en una ventana de consola. Al inicio de cada sesión del curso, inicia `C:\elastic\kibana\bin\kibana.bat` y deja la ventana abierta. Elasticsearch ya arranca solo como servicio.

#### Paso 20. Activa la licencia de prueba (trial) de 30 días

1. En Kibana abre el menú ☰ y ve a **Stack Management > License Management**.
2. Haz clic en **Start a 30-day trial** y confirma.

También puedes hacerlo en **Dev Tools** (menú ☰ > **Management > Dev Tools**):

```http
POST _license/start_trial?acknowledge=true
GET _license
```

**Qué hace y por qué:** la licencia gratuita *Basic* incluye casi todo lo que usa el curso. Algunas funciones de las prácticas requieren un nivel superior: reportes PDF/PNG de dashboards (8b), seguridad a nivel de documento y de campo y auditoría (9), análisis de tasa de logs (7.5) y Watcher. La licencia de prueba las habilita durante 30 días, suficientes para el curso. Al vencer, el clúster vuelve a *Basic* sin perder datos.

---

### Tarea 5. Explorar la arquitectura: roles, shards y réplicas

Con el clúster en marcha, comprobarás en la práctica los conceptos de los subtemas 2.1 y 2.2.

#### Paso 21. Identifica los roles del nodo

En **Dev Tools** ejecuta:

```http
GET _cat/nodes?v&h=name,node.role,master,version
GET _nodes/_local?filter_path=nodes.*.roles
```

**Qué hace y por qué:** el nodo tiene todos los roles por defecto, abreviados como `cdfhilmrstw`. Cada letra corresponde a un rol: **c**old, **d**ata, **f**rozen, **h**ot, **i**ngest, m**l**, **m**aster, **r**emote_cluster_client, content (**s**), **t**ransform y **w**arm. El asterisco en la columna `master` indica que es el nodo maestro elegido. En producción se separan los roles: nodos maestros dedicados (3 para tener quórum), nodos de datos por nivel (hot, warm, cold) y, opcionalmente, nodos de ingesta y de coordinación (sin roles).

#### Paso 22. Crea un índice con réplicas y observa el estado `yellow`

```http
PUT demo-arquitectura
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}

GET _cluster/health/demo-arquitectura
GET _cat/shards/demo-arquitectura?v&h=index,shard,prirep,state,node,unassigned.reason
```

**Qué hace y por qué:** el índice tiene 3 shards primarios (`p`) y 1 réplica de cada uno (`r`), en total 6 shards. Los 3 primarios quedan `STARTED` y las 3 réplicas `UNASSIGNED`, por lo que el estado del índice es **yellow**: todos los datos están disponibles, pero no hay tolerancia a fallos. Elasticsearch **nunca** coloca una réplica en el mismo nodo que su primario, porque si el nodo fallara se perderían ambas copias.

#### Paso 23. Pregunta al clúster por qué no asigna la réplica

```http
GET _cluster/allocation/explain
{
  "index": "demo-arquitectura",
  "shard": 0,
  "primary": false
}
```

**Qué hace y por qué:** la API *allocation explain* devuelve la decisión de cada *decider*. Busca en la respuesta `"decider": "same_shard"` y el texto *"a copy of this shard is already allocated to this node"*. Es la herramienta principal para diagnosticar shards sin asignar, y la usarás de nuevo en la Práctica 11.

#### Paso 24. Ajusta las réplicas para un clúster de un nodo

```http
PUT demo-arquitectura/_settings
{
  "index": { "number_of_replicas": 0 }
}

GET _cluster/health/demo-arquitectura
GET _cat/shards/demo-arquitectura?v
DELETE demo-arquitectura
```

**Qué hace y por qué:** `number_of_replicas` es dinámico y se cambia en caliente; `number_of_shards`, en cambio, se fija al crear el índice. Con 0 réplicas el estado pasa a **green**. Por esa razón, en este curso de un solo nodo los índices se crean con `number_of_replicas: 0`. En producción, con 2 o más nodos de datos, se usa al menos 1 réplica.

#### Paso 25. Calcula el dimensionamiento para un caso on premise

Completa la tabla con el siguiente escenario: se reciben **40 GB de logs al día** (tamaño ya indexado), se retienen **30 días**, se usa **1 réplica** y se considera un **15 %** adicional de sobrecarga.

| Concepto | Fórmula | Resultado |
|---|---|---|
| Datos primarios | 40 GB × 30 días | |
| Con réplica | Datos primarios × (1 + 1) | |
| Con sobrecarga | Con réplica × 1.15 | |
| Disco total con margen de watermark (85 %) | Con sobrecarga ÷ 0.85 | |
| Por nodo con 3 nodos de datos | Disco total ÷ 3 | |

**Qué hace y por qué:** relaciona el volumen de logs, la retención y la replicación con el disco necesario (subtema 2.3). El divisor 0.85 refleja el *low disk watermark*: al superar el 85 % de uso, Elasticsearch deja de asignar nuevos shards a ese nodo.

---

### Resultado esperado

- El servicio `elasticsearch-service-x64` aparece con estado **Running** y tipo de inicio **Automatic**.
- `es GET "/?pretty"` responde con una salida similar a esta:

```json
{
  "name" : "nodo-01",
  "cluster_name" : "lab-logs",
  "version" : {
    "number" : "8.19.21",
    "build_flavor" : "default",
    ...
  },
  "tagline" : "You Know, for Search"
}
```

- `GET _cluster/health` muestra `"status" : "green"` y `"number_of_nodes" : 1`.
- Kibana abre en `http://localhost:5601`, permite iniciar sesión con `elastic`, y **License Management** muestra la licencia **Trial** activa.
- En la Tarea 5 observaste el estado `yellow` con réplicas `UNASSIGNED`, el decider `same_shard` y el regreso a `green` con 0 réplicas.
- Tabla de dimensionamiento: 1,200 GB, 2,400 GB, 2,760 GB, ≈ 3,247 GB, ≈ 1,082 GB por nodo.

> **Importante:** guarda en un lugar seguro la contraseña de `elastic`. En todas las prácticas siguientes, al abrir una nueva ventana de PowerShell, ejecuta `. C:\labs\scripts\entorno.ps1`.

### Referencias

- Elastic. *Install Elasticsearch with .zip on Windows [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/zip-windows.html
- Elastic. *Install Kibana on Windows [8.19].* https://www.elastic.co/guide/en/kibana/8.19/windows.html
- Elastic. *Important system configuration [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/system-config.html
- Elastic. *Node roles [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/modules-node.html
- Elastic. *Cluster allocation explain API [8.19].* https://www.elastic.co/guide/en/elasticsearch/reference/8.19/cluster-allocation-explain.html
