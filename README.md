# Elasticsearch 7.17 y 9.3

Curso práctico orientado a la administración y operación de Elasticsearch en la nube para almacenar, consultar, proteger, optimizar y mantener logs de aplicaciones. Incluye compatibilidad entre las versiones 7.17 y 9.3, data streams, ingest pipelines, consultas, seguridad, ciclo de vida, respaldo, migración y troubleshooting.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [Analizar la arquitectura, los roles de nodos y el estado de un deployment de Elasticsearch.](Capitulo01/README.md#analizar-la-arquitectura-los-roles-de-nodos-y-el-estado-de-un-deployment-de-elasticsearch)
  - Descripción: Actividad para analizar un deployment de Elasticsearch, identificar los roles de nodos y revisar su estado, disponibilidad, distribución de datos y responsabilidades operativas.
  - Duración estimada: 60 min

### Capítulo 2

- [Validar versión, licenciamiento, conectividad y salud de deployments 7.17 y 9.3](Capitulo02/README.md#validar-versión-licenciamiento-conectividad-y-salud-de-deployments-717-y-93)
  - Descripción: Actividad para validar versión, licenciamiento, conectividad y salud de deployments Elasticsearch 7.17 y 9.3, considerando compatibilidades, restricciones y requisitos de actualización.
  - Duración estimada: 45 min

### Capítulo 3

- [Crear templates, mappings y un data stream para logs de aplicaciones.](Capitulo03/README.md#crear-templates-mappings-y-un-data-stream-para-logs-de-aplicaciones)
  - Descripción: Actividad para crear templates, mappings y un data stream destinado a logs de aplicaciones, considerando la estructura de almacenamiento, los tipos de campos y la prevención de mapping explosion.
  - Duración estimada: 70 min

### Capítulo 4

- [Crear, probar y depurar un pipeline para logs JSON y no estructurados](Capitulo04/README.md#crear-probar-y-depurar-un-pipeline-para-logs-json-y-no-estructurados)
  - Descripción: Actividad para crear, probar y depurar un ingest pipeline que transforme logs JSON y no estructurados mediante procesadores de ingestión, simulación de pipelines y normalización con Elastic Common Schema — ECS.
  - Duración estimada: 110 min

### Capítulo 5

- [Investigar errores y tendencias mediante Query DSL, KQL y ES|QL.](Capitulo05/README.md#investigar-errores-y-tendencias-mediante-query-dsl-kql-y-esql)
  - Descripción: Actividad para investigar errores y tendencias con Query DSL en Elasticsearch 7.17 y 9.3, KQL en Kibana 7.17 y 9.3, y ES|QL exclusivamente en el entorno 9.3.
  - Duración estimada: 100 min

### Capítulo 6

- [Identificar un cuello de botella y validar un ajuste operativo](Capitulo06/README.md#identificar-un-cuello-de-botella-y-validar-un-ajuste-operativo)
  - Descripción: Actividad para identificar un cuello de botella a partir del estado del clúster, nodos, shards, almacenamiento, memoria y presión de indexación, y validar un ajuste operativo de capacidad u optimización.
  - Duración estimada: 80 min

### Capítulo 7

- [Configurar roles, privilegios y una política de ciclo de vida para logs.](Capitulo07/README.md#configurar-roles-privilegios-y-una-política-de-ciclo-de-vida-para-logs)
  - Descripción: Actividad para configurar roles y privilegios de acceso, junto con una política de ciclo de vida para logs basada en ILM, data tiers, rollover y retención.
  - Duración estimada: 75 min

### Capítulo 8

- [Investigar un incidente con Discover y consultas guardadas](Capitulo08/README.md#investigar-un-incidente-con-discover-y-consultas-guardadas)
  - Descripción: Actividad para investigar un incidente con Kibana Discover, aplicando data views, filtros, campos, análisis temporal y consultas guardadas vinculadas con la operación.
  - Duración estimada: 50 min

### Capítulo 9

- [Ruta de actualización desde 7.17 hacia 8.19 y posteriormente hacia 9.x, utilizando 9.3 como versión objetivo](Capitulo09/README.md#ruta-de-actualización-desde-717-hacia-819-y-posteriormente-hacia-9x-utilizando-93-como-versión-objetivo)
  - Descripción: Actividad para recorrer la ruta de actualización desde Elasticsearch 7.17 hacia 8.19 y posteriormente hacia 9.x, utilizando 9.3 como versión objetivo y considerando las operaciones de mantenimiento y recuperación del capítulo.
  - Duración estimada: 65 min

### Capítulo 10

- [4 Laboratorio final: Diagnosticar y documentar un incidente en una plataforma de logs](Capitulo10/README.md#4-laboratorio-final-diagnosticar-y-documentar-un-incidente-en-una-plataforma-de-logs)
  - Descripción: Actividad final para diagnosticar y documentar un incidente en una plataforma de logs, aplicando la metodología de diagnóstico, la revisión de fallos de indexación, búsqueda, shards y capacidad, y el checklist operativo de escalamiento y documentación.
  - Duración estimada: 60 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.
