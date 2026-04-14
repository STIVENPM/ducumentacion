# Backlog Técnico — Sistema de Aerolínea

**Proyecto:** Sistema de gestión de aerolínea  
**Documento:** backlog_tecnico.md  
**Autor:** Aprendiz SENA – ADSO  
**Fecha:** 2026-04-14  
**Herramienta sugerida:** GitHub Projects / Jira / Trello

---

## 1. Objetivo del backlog

Organizar y priorizar el trabajo técnico necesario para estabilizar, versionar y desplegar la base de datos del sistema de aerolínea. Las historias de usuario (HU) representan unidades de trabajo ejecutables, con criterios de aceptación verificables y dependencias explícitas entre ellas.

---

## 2. Historias de usuario esenciales

### HU-001 — Identificar y documentar dominios funcionales del modelo existente

| Campo | Detalle |
|---|---|
| **ID** | HU-001 |
| **Título** | Identificar y documentar dominios funcionales del modelo existente |
| **Como** | Aprendiz/desarrollador del proyecto |
| **Quiero** | Analizar el archivo `modelo_postgresql.sql` e identificar los dominios funcionales, sus entidades principales y sus relaciones |
| **Para** | Tener una base documental que guíe todas las decisiones técnicas del proyecto |
| **Prioridad** | Alta |
| **Estado** | ✅ Completada |
| **Estimación** | 2 horas |
| **Dependencias** | Ninguna |

**Criterios de aceptación:**
- Se identifican los 11 dominios funcionales del modelo (geografía, aerolínea, identidad, seguridad, clientes, aeropuerto, aeronaves, operaciones de vuelo, ventas, abordaje, pagos y facturación).
- Se documenta la entidad raíz de cada dominio.
- Se documentan las relaciones clave entre dominios.
- El documento `analisis_dominios.md` queda en la carpeta `docs/`.

---

### HU-002 — Organizar la estructura base del repositorio y definir ramas develop, qa y main

| Campo | Detalle |
|---|---|
| **ID** | HU-002 |
| **Título** | Estructura del repositorio y estrategia de ramas |
| **Como** | Aprendiz/desarrollador |
| **Quiero** | Crear la estructura de carpetas del repositorio y configurar las ramas `develop`, `qa` y `main` |
| **Para** | Garantizar que el trabajo evolucione de forma ordenada, trazable y sin mezclar cambios entre entornos |
| **Prioridad** | Alta |
| **Estado** | ✅ Completada |
| **Estimación** | 1 hora |
| **Dependencias** | HU-001 |

**Criterios de aceptación:**
- El repositorio tiene las ramas `develop`, `qa` y `main`.
- Existe la estructura de carpetas: `docs/`, `adr/`, `docker/`, `liquibase/changelogs/`.
- El archivo `README.md` describe el propósito del proyecto y cómo levantar el entorno.
- Ningún desarrollador hace commits directos a `main` o `qa`.
- Los tags en `main` siguen versionado semántico (`v1.0.0`).

---

### HU-003 — Contenerizar PostgreSQL para levantar la base de datos en entorno local

| Campo | Detalle |
|---|---|
| **ID** | HU-003 |
| **Título** | Contenedorización de PostgreSQL |
| **Como** | Desarrollador del equipo |
| **Quiero** | Levantar PostgreSQL mediante Docker Compose con un solo comando |
| **Para** | Garantizar que cualquier integrante del equipo pueda reproducir el entorno de base de datos sin configuraciones manuales |
| **Prioridad** | Alta |
| **Estado** | ✅ Completada |
| **Estimación** | 1 hora |
| **Dependencias** | HU-002 |

**Criterios de aceptación:**
- El archivo `docker/docker-compose.yml` levanta PostgreSQL 16 correctamente.
- El contenedor expone el puerto 5432.
- Las credenciales se gestionan con archivos `.env` (no se suben al repositorio).
- `pg_isready` retorna OK dentro del contenedor.
- Los datos persisten entre reinicios gracias al volumen `postgres_data`.

---

### HU-004 — Contenerizar Liquibase e integrarlo al proyecto

| Campo | Detalle |
|---|---|
| **ID** | HU-004 |
| **Título** | Contenedorización e integración de Liquibase |
| **Como** | Desarrollador del equipo |
| **Quiero** | Que Liquibase arranque automáticamente como contenedor al levantar el entorno, esperando que PostgreSQL esté saludable |
| **Para** | Automatizar la aplicación de changelogs sin intervención manual |
| **Prioridad** | Alta |
| **Estado** | ✅ Completada |
| **Estimación** | 1 hora |
| **Dependencias** | HU-003 |

**Criterios de aceptación:**
- El servicio `liquibase` en `docker-compose.yml` tiene `depends_on` con `condition: service_healthy`.
- Liquibase aplica el `changelog-master.xml` al iniciar.
- Los logs del contenedor confirman que los changesets se ejecutaron sin errores.
- El comando `docker compose down -v && docker compose up -d` produce el mismo resultado de forma reproducible.

---

### HU-005 — Separar el DDL en changelogs organizados por dominio funcional

| Campo | Detalle |
|---|---|
| **ID** | HU-005 |
| **Título** | Changelogs por dominio funcional en Liquibase |
| **Como** | Aprendiz/desarrollador |
| **Quiero** | Separar el DDL del `modelo_postgresql.sql` en 12 changelogs, uno por dominio funcional |
| **Para** | Versionar el DDL de forma ordenada, aplicar cambios incrementales y poder hacer rollback por dominio |
| **Prioridad** | Alta |
| **Estado** | 🔄 En progreso |
| **Estimación** | 4 horas |
| **Dependencias** | HU-004 |

**Criterios de aceptación:**
- Existen 12 archivos de changelog, uno por dominio (`01-geografia`, `02-aerolinea`, ..., `12-facturacion`).
- Cada changeset tiene su bloque `<rollback>`.
- Los IDs de changeset son únicos y descriptivos.
- El `changelog-master.xml` orquesta todos los dominios en el orden correcto.
- Aplicar `liquibase update` desde cero crea las 52 tablas del modelo base sin errores.
- Se puede aplicar solo un dominio usando `--contexts=nombre-dominio`.

---

### HU-006 — Diseñar e implementar estrategia de roles y permisos diferenciados

| Campo | Detalle |
|---|---|
| **ID** | HU-006 |
| **Título** | Roles y permisos diferenciados |
| **Como** | Administrador del sistema |
| **Quiero** | Que el sistema tenga roles funcionales bien definidos, con permisos asignados tanto en las tablas del modelo como en PostgreSQL |
| **Para** | Aplicar el principio de mínimo privilegio y garantizar que cada actor solo acceda a lo que necesita |
| **Prioridad** | Media |
| **Estado** | 📋 Pendiente |
| **Estimación** | 3 horas |
| **Dependencias** | HU-005 |

**Criterios de aceptación:**
- Las tablas `security_role` y `security_permission` están pobladas con los 7 roles y 10 permisos definidos en ADR-002.
- Existen roles nativos de PostgreSQL (`app_sales_agent`, `app_flight_ops`, etc.) con `GRANT` aplicados.
- El rol `app_data_analyst` solo tiene `SELECT` sobre todas las tablas.
- `REVOKE DELETE ON payment FROM PUBLIC` está aplicado.
- Existe la tabla `audit_log` con triggers en tablas sensibles (`payment`, `ticket`, `reservation`).

---

### HU-007 — Construir plan de datos de prueba con orden de carga por dependencias

| Campo | Detalle |
|---|---|
| **ID** | HU-007 |
| **Título** | Plan y scripts de datos de prueba |
| **Como** | Desarrollador/tester |
| **Quiero** | Tener scripts SQL organizados en 6 capas que poblen la base de datos con datos coherentes y verificables |
| **Para** | Poder probar todos los flujos del sistema: reserva, vuelo, abordaje, pagos y reclamaciones |
| **Prioridad** | Media |
| **Estado** | ✅ Completada |
| **Estimación** | 3 horas |
| **Dependencias** | HU-005 |

**Criterios de aceptación:**
- Existen 6 scripts de inserción bajo `liquibase/changelogs/13-datos-prueba/`.
- Los scripts tienen contexto Liquibase `test-data` (no se ejecutan en producción).
- Cada capa tiene su script de validación con resultados esperados documentados.
- El vuelo `RC101` existe con delay de 95 minutos (prerequisito para reclamaciones).
- Las reclamaciones `CLM-2026-0001` y `CLM-2026-0002` se insertan correctamente en Capa 6.
- El rollback por capas invierte la inserción sin errores.

---

### HU-008 — Documentar seguimiento técnico y decisiones arquitectónicas

| Campo | Detalle |
|---|---|
| **ID** | HU-008 |
| **Título** | Documentación técnica y seguimiento de ADR |
| **Como** | Aprendiz/desarrollador |
| **Quiero** | Mantener documentación actualizada de todos los ADR, decisiones tomadas y seguimiento del proyecto |
| **Para** | Garantizar trazabilidad, facilitar la revisión técnica y demostrar criterio en las decisiones |
| **Prioridad** | Media |
| **Estado** | ✅ Completada |
| **Estimación** | 2 horas |
| **Dependencias** | HU-001 |

**Criterios de aceptación:**
- Los 5 ADR están documentados en la carpeta `adr/` con estructura completa (título, contexto, problema, decisión, justificación, consecuencias).
- ADR-001 propone el dominio de incidencias y reclamaciones.
- ADR-002 define los roles y permisos del sistema.
- ADR-003 justifica el uso de Liquibase.
- ADR-004 define la estrategia de ramas `develop/qa/main`.
- ADR-005 propone la contenedorización con Docker Compose.
- El archivo `docs/seguimientos.md` registra hitos, decisiones y cambios relevantes.

---

### HU-009 — Implementar dominio de incidencias y reclamaciones (ADR-001)

| Campo | Detalle |
|---|---|
| **ID** | HU-009 |
| **Título** | DDL del dominio de incidencias y reclamaciones |
| **Como** | Desarrollador |
| **Quiero** | Crear las 4 tablas del dominio de incidencias (`incident_type`, `claim_status`, `claim`, `claim_compensation`) con su changelog propio |
| **Para** | Ampliar el modelo base con capacidad de registrar y gestionar reclamaciones de pasajeros |
| **Prioridad** | Media |
| **Estado** | 📋 Pendiente |
| **Estimación** | 2 horas |
| **Dependencias** | HU-005 |

**Criterios de aceptación:**
- Existe el changelog `liquibase/changelogs/incidencias/01-create-tables.xml`.
- Las 4 tablas se crean sin errores al aplicar el changeset.
- La tabla `claim` tiene FK opcionales hacia `flight_segment`, `ticket` y `baggage`.
- El CHECK en `claim_compensation` valida que `compensation_type` sea uno de los valores permitidos.
- El rollback del changeset elimina las 4 tablas en orden inverso.
- El `changelog-master.xml` incluye el nuevo changelog después del dominio 12.

---

### HU-010 — Datos de prueba para el dominio de incidencias (ADR-001)

| Campo | Detalle |
|---|---|
| **ID** | HU-010 |
| **Título** | Datos de prueba — dominio de reclamaciones |
| **Como** | Tester/desarrollador |
| **Quiero** | Insertar datos de prueba coherentes en las tablas del dominio de incidencias, vinculados al vuelo con delay y al equipaje perdido |
| **Para** | Validar el flujo completo de una reclamación desde su apertura hasta la compensación |
| **Prioridad** | Baja |
| **Estado** | 📋 Pendiente |
| **Estimación** | 1 hora |
| **Dependencias** | HU-007, HU-009 |

**Criterios de aceptación:**
- Existen 2 reclamaciones en la tabla `claim` (una por delay, una por equipaje perdido).
- La reclamación `CLM-2026-0001` tiene una compensación de 5000 millas en `claim_compensation`.
- La consulta de validación de Capa 6 retorna 2 filas con los datos esperados.
- Los datos de reclamaciones pueden revertirse con el rollback de Capa 6.

---

## 3. Tablero de estado del backlog

| ID | Historia | Prioridad | Estado | Estimación | Dependencias |
|---|---|---|---|---|---|
| HU-001 | Análisis de dominios | Alta | ✅ Completada | 2h | — |
| HU-002 | Estructura del repositorio y ramas | Alta | ✅ Completada | 1h | HU-001 |
| HU-003 | Contenedorización PostgreSQL | Alta | ✅ Completada | 1h | HU-002 |
| HU-004 | Contenedorización Liquibase | Alta | ✅ Completada | 1h | HU-003 |
| HU-005 | Changelogs por dominio | Alta | 🔄 En progreso | 4h | HU-004 |
| HU-006 | Roles y permisos diferenciados | Media | 📋 Pendiente | 3h | HU-005 |
| HU-007 | Plan de datos de prueba | Media | ✅ Completada | 3h | HU-005 |
| HU-008 | Documentación y ADR | Media | ✅ Completada | 2h | HU-001 |
| HU-009 | DDL dominio incidencias | Media | 📋 Pendiente | 2h | HU-005 |
| HU-010 | Datos de prueba – incidencias | Baja | 📋 Pendiente | 1h | HU-007, HU-009 |

**Total estimado:** 20 horas  
**Completadas:** 6 HU (HU-001, 002, 003, 004, 007, 008)  
**En progreso:** 1 HU (HU-005)  
**Pendientes:** 3 HU (HU-006, 009, 010)

---

## 4. Mapa de dependencias entre HU

```
HU-001 (Análisis dominios)
  └── HU-002 (Estructura repositorio + ramas)
        └── HU-003 (PostgreSQL en contenedor)
              └── HU-004 (Liquibase en contenedor)
                    └── HU-005 (Changelogs por dominio)
                          ├── HU-006 (Roles y permisos)
                          ├── HU-007 (Plan datos prueba)
                          │     └── HU-010 (Datos incidencias)
                          └── HU-009 (DDL dominio incidencias)
                                └── HU-010 (Datos incidencias)

HU-001
  └── HU-008 (Documentación y ADR) — puede ejecutarse en paralelo
```

---

## 5. Convenciones del backlog

### Estados

| Ícono | Estado | Descripción |
|---|---|---|
| ✅ | Completada | Todos los criterios de aceptación verificados |
| 🔄 | En progreso | Trabajo activo en curso |
| 📋 | Pendiente | No iniciada, esperando dependencias o priorización |
| ⏸️ | Bloqueada | Bloqueada por dependencia externa no resuelta |

### Prioridades

- **Alta:** Bloquea el inicio de otras HU; debe completarse primero.
- **Media:** Necesaria para la entrega final; puede ejecutarse una vez las HU de Alta están resueltas.
- **Baja:** Deseable; puede moverse al componente desescolarizado si el tiempo no alcanza.

### Nomenclatura de ramas por HU

```
feature/HU-005-changelogs-por-dominio
feature/HU-006-roles-permisos
feature/HU-009-ddl-incidencias
fix/HU-005-ruta-rota-changelog
docs/HU-008-adr-actualizacion
```
