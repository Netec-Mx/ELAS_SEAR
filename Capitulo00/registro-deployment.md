# Registro del deployment · rutanorte-logs

> Archivo de soporte de la Actividad 0. Complétalo mientras creas el deployment y consérvalo durante todo el curso: lo usarás en los Labs 1, 2 y 9.
> **No anotes contraseñas en este archivo.** Guárdalas en tu gestor de contraseñas.

| Dato | Dónde se obtiene | Valor |
|---|---|---|
| Nombre del deployment | Paso 0.2 | |
| ID del deployment | Consola ECH › Deployment › *Overview* | |
| Proveedor cloud | Paso 0.2 | |
| Región | Paso 0.2 | |
| Perfil de hardware | Paso 0.2 | |
| Versión del stack | Paso 0.2 y `GET /` | |
| Número de zonas | Consola ECH › *Manage* › *Edit* | |
| Tamaño por zona (RAM) | Consola ECH › *Manage* › *Edit* | |
| Endpoint de Elasticsearch | Consola ECH › *Overview* › *Copy endpoint* | |
| Endpoint de Kibana | Consola ECH › *Overview* › *Copy endpoint* | |
| Cloud ID | Consola ECH › *Overview* | |
| Contraseña de `elastic` guardada en | Paso 0.2 (gestor de contraseñas, archivo .csv descargado) | |
| Usuario de laboratorio | Paso 0.8 | `rn_admin` |
| Fecha de creación | | |
| Fecha en que vence la prueba (si aplica) | Consola ECH › *Billing* o banner superior | |

## Verificación (Paso 0.9)

| Comprobación | Valor esperado | Valor obtenido |
|---|---|---|
| `GET /` › `version.number` | 9.3.x (o la 9.x elegida) | |
| `GET _cluster/health` › `status` | `green` | |
| `GET _cluster/health` › `number_of_nodes` | 2 o más | |
| `GET _security/_authenticate` › `username` | `rn_admin` | |
