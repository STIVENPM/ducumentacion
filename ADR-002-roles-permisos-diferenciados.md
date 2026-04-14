# ADR-002: Diseño de manejo de roles con permisos diferenciados para seguridad y trazabilidad

- **Estado:** Propuesto
- **Fecha:** 2025-04-14
- **Autor:** Aprendiz SENA – ADSO

---

## Contexto

El modelo base ya incluye las tablas `security_role`, `security_permission`, `user_role` y `role_permission`, lo que indica que el diseño contempla un sistema RBAC (Role-Based Access Control). Sin embargo, el modelo solo define la estructura relacional: no establece qué roles existirán, qué permisos se les asignarán, ni cómo se aplicarán esos permisos a nivel de base de datos (a nivel de PostgreSQL, no solo en la capa de aplicación).

En un sistema de aerolínea, distintos actores operan sobre el modelo: agentes de ventas, operadores de vuelo, administradores de sistema, personal de abordaje, analistas de datos, entre otros. Cada actor debe tener acceso restringido únicamente a los datos y operaciones que le corresponden.

## Problema

El modelo no define:
- Qué roles concretos operarán en el sistema.
- Qué permisos tiene cada rol (lectura, escritura, ejecución sobre qué tablas).
- Cómo esos roles se materializan en PostgreSQL (roles de base de datos, no solo filas en tablas).
- Cómo garantizar trazabilidad de acciones sensibles (pagos, cancelaciones, modificaciones de tickets).

Sin esta definición, cualquier conexión a la base tiene acceso irrestricto, lo que representa un riesgo de seguridad grave.

## Decisión

Se definen **roles funcionales** del sistema con sus permisos diferenciados, aplicados en dos niveles:

### Nivel 1: Datos de referencia en tablas del modelo

Se poblarán las tablas `security_role` y `security_permission` con los siguientes registros base:

| role_code | role_name | Descripción |
|---|---|---|
| `SYS_ADMIN` | Administrador del sistema | Acceso total; gestión de usuarios y roles |
| `SALES_AGENT` | Agente de ventas | Crear reservas, emitir tickets, registrar pagos |
| `FLIGHT_OPS` | Operaciones de vuelo | Gestión de vuelos, segmentos, delays |
| `BOARDING_STAFF` | Personal de abordaje | Check-in, boarding pass, validación de embarque |
| `CUSTOMER_SVC` | Servicio al cliente | Consulta de reservas, gestión de reclamaciones |
| `DATA_ANALYST` | Analista de datos | Solo lectura sobre todas las tablas |
| `MAINTENANCE_TECH` | Técnico de mantenimiento | Escritura en eventos de mantenimiento de aeronaves |

Permisos asignados por rol (muestra representativa):

| permission_code | Descripción | Roles con acceso |
|---|---|---|
| `RESERVATION_CREATE` | Crear reservas | SALES_AGENT, SYS_ADMIN |
| `TICKET_ISSUE` | Emitir tickets | SALES_AGENT, SYS_ADMIN |
| `PAYMENT_PROCESS` | Procesar pagos | SALES_AGENT, SYS_ADMIN |
| `FLIGHT_UPDATE` | Actualizar estado de vuelo | FLIGHT_OPS, SYS_ADMIN |
| `DELAY_REGISTER` | Registrar delays | FLIGHT_OPS, SYS_ADMIN |
| `CHECKIN_PROCESS` | Procesar check-in | BOARDING_STAFF, SYS_ADMIN |
| `BOARDING_VALIDATE` | Validar embarque | BOARDING_STAFF, SYS_ADMIN |
| `CLAIM_MANAGE` | Gestionar reclamaciones | CUSTOMER_SVC, SYS_ADMIN |
| `REPORT_READ` | Leer reportes y datos | DATA_ANALYST, SYS_ADMIN |
| `MAINTENANCE_WRITE` | Registrar mantenimiento | MAINTENANCE_TECH, SYS_ADMIN |

### Nivel 2: Roles nativos de PostgreSQL

Se crean roles de base de datos que reflejan los roles funcionales, aplicando `GRANT` y `REVOKE` directamente sobre las tablas:

```sql
-- Crear roles en PostgreSQL
CREATE ROLE app_sales_agent LOGIN PASSWORD 'changeme';
CREATE ROLE app_flight_ops LOGIN PASSWORD 'changeme';
CREATE ROLE app_boarding LOGIN PASSWORD 'changeme';
CREATE ROLE app_customer_svc LOGIN PASSWORD 'changeme';
CREATE ROLE app_data_analyst LOGIN PASSWORD 'changeme';
CREATE ROLE app_maintenance LOGIN PASSWORD 'changeme';

-- Ejemplo de permisos para SALES_AGENT
GRANT SELECT, INSERT, UPDATE ON reservation TO app_sales_agent;
GRANT SELECT, INSERT ON ticket TO app_sales_agent;
GRANT SELECT, INSERT ON payment TO app_sales_agent;
GRANT SELECT ON flight, flight_segment TO app_sales_agent;

-- Ejemplo de permisos para DATA_ANALYST (solo lectura)
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_data_analyst;

-- Revocar permisos peligrosos a todos
REVOKE DELETE ON payment FROM PUBLIC;
REVOKE TRUNCATE ON ALL TABLES IN SCHEMA public FROM PUBLIC;
```

### Nivel 3: Trazabilidad con auditoría

Se propone una tabla de auditoría para registrar acciones sensibles:

```sql
CREATE TABLE audit_log (
    audit_log_id    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_account_id uuid REFERENCES user_account(user_account_id),
    table_name      varchar(80) NOT NULL,
    operation       varchar(10) NOT NULL,
    record_id       uuid,
    old_data        jsonb,
    new_data        jsonb,
    occurred_at     timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_audit_operation CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE'))
);
```

Esta tabla se alimenta mediante triggers en tablas sensibles como `payment`, `ticket`, `reservation` y `user_account`.

## Justificación técnica

- El modelo ya incluye la infraestructura RBAC. Esta decisión la activa con datos reales y permisos de PostgreSQL.
- Separar roles funcionales (filas en tablas) de roles de base de datos (roles PostgreSQL) permite que la capa de aplicación use los permisos de capa de negocio y la capa de base de datos use los de PostgreSQL, funcionando como defensa en profundidad.
- La tabla `audit_log` con `jsonb` evita crear una tabla de auditoría por cada tabla del modelo. Es flexible y extensible.
- Los triggers de auditoría deben aplicarse solo a tablas de alta sensibilidad para no impactar el rendimiento general.

## Consecuencias e impacto esperado

| Aspecto | Impacto |
|---|---|
| Seguridad | Principio de mínimo privilegio aplicado a nivel de base de datos |
| Trazabilidad | Registro inmutable de operaciones sobre datos sensibles |
| Liquibase | Requiere changeset separado para DDL de `audit_log` y otro para los `GRANT` |
| Backlog | Genera HU-006 (implementar roles y permisos) y tarea de creación de triggers de auditoría |
| Mantenimiento | Los roles de PostgreSQL deben sincronizarse con los datos en `security_role` cuando se agreguen nuevos actores |
