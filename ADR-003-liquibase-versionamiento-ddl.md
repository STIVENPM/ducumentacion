# ADR-003: Implementación de Liquibase para versionamiento del DDL

- **Estado:** Propuesto
- **Fecha:** 2025-04-14
- **Autor:** Aprendiz SENA – ADSO

---

## Contexto

El modelo actual existe como un único archivo SQL monolítico (`modelo_postgresql.sql`). Este enfoque funciona para la creación inicial de la base de datos, pero no permite:
- Conocer qué cambios se aplicaron y en qué orden.
- Revertir cambios de forma controlada.
- Aplicar el DDL de manera incremental en distintos entornos (desarrollo, QA, producción).
- Garantizar que la estructura de la base de datos sea reproducible a partir del repositorio.

El proyecto requiere evolucionar hacia un modelo donde el DDL sea versionado, auditado y aplicable de manera automatizada.

## Problema

Sin un sistema de versionamiento del DDL:
- No es posible saber qué versión del esquema tiene cada entorno.
- Ejecutar el SQL base dos veces puede generar errores por duplicación de objetos.
- No existe mecanismo de rollback controlado ante un cambio fallido.
- La colaboración entre varios desarrolladores genera conflictos al modificar el mismo archivo SQL.

## Decisión

Se adopta **Liquibase** como herramienta oficial de versionamiento del DDL del proyecto. Liquibase gestiona los cambios mediante **changelogs** (archivos de cambio) y registra en la tabla `databasechangelog` de PostgreSQL qué changesets ya fueron aplicados.

### Estructura de changelogs por dominio

El modelo tiene 12 dominios funcionales. Cada dominio tendrá su propio archivo de changelog. El archivo maestro los orquesta en el orden correcto según dependencias:

```
liquibase/
├── changelog-master.xml
└── changelogs/
    ├── 01-geografia/
    │   └── 01-create-tables.xml
    ├── 02-aerolinea/
    │   └── 01-create-tables.xml
    ├── 03-identidad/
    │   └── 01-create-tables.xml
    ├── 04-seguridad/
    │   └── 01-create-tables.xml
    ├── 05-clientes-fidelizacion/
    │   └── 01-create-tables.xml
    ├── 06-aeropuerto/
    │   └── 01-create-tables.xml
    ├── 07-aeronaves/
    │   └── 01-create-tables.xml
    ├── 08-operaciones-vuelo/
    │   └── 01-create-tables.xml
    ├── 09-ventas-reservas/
    │   └── 01-create-tables.xml
    ├── 10-abordaje/
    │   └── 01-create-tables.xml
    ├── 11-pagos/
    │   └── 01-create-tables.xml
    └── 12-facturacion/
        └── 01-create-tables.xml
```

### Ejemplo de changelog maestro

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <!-- El orden aquí es crítico: respeta dependencias entre dominios -->
    <include file="changelogs/01-geografia/01-create-tables.xml"/>
    <include file="changelogs/02-aerolinea/01-create-tables.xml"/>
    <include file="changelogs/03-identidad/01-create-tables.xml"/>
    <include file="changelogs/04-seguridad/01-create-tables.xml"/>
    <include file="changelogs/05-clientes-fidelizacion/01-create-tables.xml"/>
    <include file="changelogs/06-aeropuerto/01-create-tables.xml"/>
    <include file="changelogs/07-aeronaves/01-create-tables.xml"/>
    <include file="changelogs/08-operaciones-vuelo/01-create-tables.xml"/>
    <include file="changelogs/09-ventas-reservas/01-create-tables.xml"/>
    <include file="changelogs/10-abordaje/01-create-tables.xml"/>
    <include file="changelogs/11-pagos/01-create-tables.xml"/>
    <include file="changelogs/12-facturacion/01-create-tables.xml"/>

</databaseChangeLog>
```

### Ejemplo de changeset individual

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="01-create-time-zone" author="aprendiz-sena" context="geografia">
        <sql splitStatements="true">
            CREATE TABLE time_zone (
                time_zone_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
                time_zone_name varchar(64) NOT NULL,
                utc_offset_minutes integer NOT NULL,
                created_at timestamptz NOT NULL DEFAULT now(),
                updated_at timestamptz NOT NULL DEFAULT now(),
                CONSTRAINT uq_time_zone_name UNIQUE (time_zone_name)
            );
        </sql>
        <rollback>
            DROP TABLE IF EXISTS time_zone;
        </rollback>
    </changeSet>

</databaseChangeLog>
```

### Reglas de organización de changesets

1. **Un dominio = un directorio de changelogs.** No se mezclan entidades de distintos dominios en el mismo archivo.
2. **Cada changeset tiene su bloque `<rollback>`.** Permite revertir el cambio de manera controlada.
3. **Los IDs de changesets son únicos y descriptivos.** Formato: `{numero}-{accion}-{entidad}`.
4. **El atributo `context`** permite aplicar solo los changelogs de un dominio específico usando `--contexts=geografia`.
5. **No se modifica un changeset ya aplicado.** Si se necesita corregir, se agrega un nuevo changeset.

### Ejecución de Liquibase

```bash
# Aplicar todos los cambios
liquibase --changeLogFile=changelog-master.xml update

# Aplicar solo el dominio de geografía
liquibase --changeLogFile=changelog-master.xml update --contexts=geografia

# Revertir el último changeset
liquibase --changeLogFile=changelog-master.xml rollbackCount 1

# Ver qué cambios están pendientes
liquibase --changeLogFile=changelog-master.xml status
```

## Justificación técnica

- Liquibase es agnóstico al motor de base de datos pero tiene soporte nativo y maduro para PostgreSQL.
- Permite usar SQL nativo en los changesets (opción `<sql>`), lo que facilita reutilizar el SQL del modelo base sin reescribirlo en formato Liquibase XML completo.
- La tabla `databasechangelog` que Liquibase crea automáticamente actúa como registro de auditoría del DDL: quién aplicó qué cambio y cuándo.
- Al integrar Liquibase en el contenedor Docker, el proceso de levantamiento de la base de datos es completamente automático y reproducible.

## Consecuencias e impacto esperado

| Aspecto | Impacto |
|---|---|
| Reproducibilidad | Cualquier desarrollador puede levantar el esquema completo desde cero con un solo comando |
| Trazabilidad del DDL | Cada cambio al esquema queda registrado con autor, fecha y hash de verificación |
| Rollback | Posible revertir cambios de manera controlada por changeset o por cantidad |
| Colaboración | Distintos desarrolladores pueden trabajar en changelogs de distintos dominios sin conflictos |
| CI/CD | Liquibase puede ejecutarse automáticamente en pipelines de integración continua |
