<a href="https://netec.com">
  <img src="https://static.wixstatic.com/media/96edd2_934dddb3230b4b16b1d1c56c51143738~mv2.png" alt="Logo de Netec" width="300"/>
</a>

# Administración de Elasticsearch para gestión de logs en la nube

## Plataforma de laboratorios

Te damos la bienvenida a la **plataforma de laboratorios** del curso **Administración de Elasticsearch para gestión de logs en la nube**. Aquí aplicarás técnicas de administración, ingestión, consulta, seguridad, mantenimiento y diagnóstico de una plataforma de logs mediante prácticas guiadas con Elasticsearch y Kibana.

## Lista de laboratorios

Cada uno de estos laboratorios está diseñado para ofrecerte una experiencia práctica. Haz clic en los enlaces para comenzar y sigue el orden de los capítulos.



## Capítulo 1: Arquitectura y operación distribuida

### [Práctica 1: Analizar la arquitectura, los roles de nodos y el estado de un deployment de Elasticsearch 9.3](Capitulo01/README.md)

- **Descripción**: Despliega entornos aislados de Elasticsearch y Kibana con Docker Compose. Inspecciona roles de nodos, salud del clúster, asignación de shards y réplicas para interpretar la arquitectura y sus implicaciones de disponibilidad.
- ⏱️ **Duración estimada**: 55 min.

## Capítulo 2: Versiones, servicios cloud y compatibilidad

### [Práctica 2: Validar versión, licenciamiento, conectividad y salud del deployment 9.3](Capitulo02/README.md)

- **Descripción**: Construye una línea base de versiones, licencias, conectividad y salud de los deployments. Verifica el acceso desde terminal, Kibana y la red Docker, y documenta endpoints y compatibilidad de componentes.
- ⏱️ **Duración estimada**: 40 min.

## Capítulo 3: Índices, mappings, templates y data streams

### [Práctica 3: Crear templates, mappings y un data stream para logs de aplicaciones](Capitulo03/README.md)

- **Descripción**: Crea component templates, mappings explícitos y reglas dinámicas controladas para almacenar logs con campos ECS. Configura un data stream, indexa eventos y comprueba sus índices de respaldo y distribución de shards.
- ⏱️ **Duración estimada**: 65 min.

## Capítulo 4: Ingest pipelines y normalización

### [Práctica 4: Crear, probar y depurar un pipeline para logs JSON y no estructurados](Capitulo04/README.md)

- **Descripción**: Implementa un ingest pipeline para procesar logs JSON y texto no estructurado. Normaliza fechas, nombres y tipos de campos hacia ECS, conserva el evento original y utiliza simulaciones y rutas de fallo para depurar la ingestión.
- ⏱️ **Duración estimada**: 100 min.

## Capítulo 5: Consultas para investigación de logs

### [Práctica 5: Investigar errores y tendencias mediante Query DSL, KQL y ES\|QL](Capitulo05/README.md)

- **Descripción**: Investiga errores y degradaciones de servicio mediante filtros, consultas y agregaciones. Compara Query DSL, KQL y ES|QL para identificar servicios afectados, códigos HTTP y tendencias temporales.
- ⏱️ **Duración estimada**: 95 min.

## Capítulo 6: Rendimiento y capacidad

### [Práctica 6: Identificar un cuello de botella y validar un ajuste operativo](Capitulo06/README.md)

- **Descripción**: Genera una carga reproducible de indexación masiva con JMeter y recopila métricas de rendimiento. Identifica el impacto de shards y refresh, aplica una configuración optimizada y compara los resultados antes y después.
- ⏱️ **Duración estimada**: 75 min.

## Capítulo 7: Seguridad y ciclo de vida

### [Práctica 7: Configurar roles, privilegios y una política de ciclo de vida para logs](Capitulo07/README.md)

- **Descripción**: Protege el acceso mediante TLS, usuarios, roles de mínimo privilegio y una API key restringida. Configura una política ILM y un data stream con rollover y retención, y verifica los permisos efectivos.
- ⏱️ **Duración estimada**: 65 min.

## Capítulo 8: Operación mediante Kibana

### [Práctica 8: Investigar un incidente con Discover y consultas guardadas](Capitulo08/README.md)

- **Descripción**: Investiga un incidente controlado con Kibana Discover. Delimita la ventana temporal, aplica filtros y consultas KQL, identifica errores y timeouts, y conserva consultas guardadas y evidencias para el informe operativo.
- ⏱️ **Duración estimada**: 50 min.

## Capítulo 9: Mantenimiento, snapshots y migraciones

### [Práctica 9: Ruta de actualización desde 7.17 hacia 8.19 y posteriormente hacia 9.3 como versión objetivo](Capitulo09/README.md)

- **Descripción**: Ensaya una migración paralela desde Elasticsearch 7.17 hacia 8.19 y 9.3 mediante snapshots y restauraciones. Transfiere objetos guardados de Kibana, valida datos y funciones, y documenta un runbook con criterios de aceptación y rollback.
- ⏱️ **Duración estimada**: 65 min.

## Capítulo 10: Troubleshooting y proyecto final

### [Práctica 10: Laboratorio final: Diagnosticar y documentar un incidente en una plataforma de logs](Capitulo10/README.md)

- **Descripción**: Diagnostica un incidente en la plataforma de logs utilizando Kibana y APIs operativas. Recopila evidencias, distingue problemas de ingestión, permisos, shards y capacidad, aplica una mitigación reversible y elabora un informe técnico con cronología y acciones preventivas.
- ⏱️ **Duración estimada**: 55 min.

---

## 📬 Contacto y más información

Si tienes alguna pregunta o necesitas más detalles, no dudes en [contactarnos](mailto:soporte@netec.com). También puedes encontrar más recursos en nuestra [página](https://netec.com).

---

¡Gracias por visitar nuestra plataforma! Revisa todos los laboratorios y comienza tu experiencia de aprendizaje.
