# Plan de Datos de Prueba — Sistema de Aerolínea

**Proyecto:** Sistema de gestión de aerolínea  
**Documento:** plan_datos_prueba.md  
**Autor:** Aprendiz SENA – ADSO  
**Fecha:** 2026-04-14  
**HU asociada:** HU-007 – Construir plan de datos de prueba con orden de carga por dependencias

---

## 1. Propósito

Este documento define la estrategia para poblar la base de datos del sistema con datos de prueba coherentes, ordenados según las dependencias del modelo y verificables mediante validaciones unitarias. El objetivo es contar con un conjunto de datos mínimo que permita probar los flujos completos del sistema: desde la creación de una reserva hasta su facturación, pasando por operaciones de vuelo, abordaje, pagos y el nuevo dominio de incidencias.

---

## 2. Principio de carga por capas

El modelo tiene dependencias transitivas entre dominios. Insertar datos sin respetar el orden genera errores de integridad referencial. La estrategia divide la carga en **seis capas**, donde cada capa solo puede ejecutarse después de que la anterior esté completa.

```
Capa 1 — Datos de referencia sin dependencias externas
Capa 2 — Entidades base que dependen solo de la Capa 1
Capa 3 — Entidades operacionales que dependen de las Capas 1 y 2
Capa 4 — Entidades transaccionales (ventas, vuelos, reservas)
Capa 5 — Flujos derivados (abordaje, pagos, facturación)
Capa 6 — Dominio de incidencias y reclamaciones (ADR-001)
```

---

## 3. Orden detallado de inserción por tabla

### Capa 1 — Referencia pura (sin FK hacia otras tablas del modelo)

| Orden | Tabla | Descripción | Registros sugeridos |
|---|---|---|---|
| 1 | `time_zone` | Zonas horarias base | 3 (UTC-5, UTC-0, UTC+1) |
| 2 | `continent` | Continentes | 3 (América, Europa, Asia) |
| 3 | `currency` | Monedas | 3 (COP, USD, EUR) |
| 4 | `person_type` | Tipos de persona | 3 (PASSENGER, EMPLOYEE, ADMIN) |
| 5 | `document_type` | Tipos de documento | 3 (PASSPORT, CC, DNI) |
| 6 | `contact_type` | Tipos de contacto | 2 (EMAIL, PHONE) |
| 7 | `user_status` | Estados de cuenta | 2 (ACTIVE, BLOCKED) |
| 8 | `security_role` | Roles del sistema | 7 (ver ADR-002) |
| 9 | `security_permission` | Permisos granulares | 10 (ver ADR-002) |
| 10 | `customer_category` | Categorías de cliente | 2 (REGULAR, VIP) |
| 11 | `benefit_type` | Tipos de beneficio | 2 (LOUNGE_ACCESS, EXTRA_BAGGAGE) |
| 12 | `flight_status` | Estados de vuelo | 4 (ON_TIME, DELAYED, CANCELLED, DEPARTED) |
| 13 | `delay_reason_type` | Razones de demora | 3 (WEATHER, TECHNICAL, ATC) |
| 14 | `reservation_status` | Estados de reserva | 3 (CONFIRMED, CANCELLED, PENDING) |
| 15 | `sale_channel` | Canales de venta | 3 (WEB, AGENCY, CALL_CENTER) |
| 16 | `ticket_status` | Estados de tiquete | 3 (ISSUED, USED, VOID) |
| 17 | `payment_status` | Estados de pago | 3 (PENDING, AUTHORIZED, FAILED) |
| 18 | `payment_method` | Métodos de pago | 3 (CREDIT_CARD, DEBIT_CARD, TRANSFER) |
| 19 | `invoice_status` | Estados de factura | 2 (DRAFT, ISSUED) |
| 20 | `check_in_status` | Estados de check-in | 2 (COMPLETED, PENDING) |
| 21 | `boarding_group` | Grupos de abordaje | 3 (PRIORITY, GROUP_A, GROUP_B) |
| 22 | `aircraft_manufacturer` | Fabricantes | 2 (Boeing, Airbus) |
| 23 | `cabin_class` | Clases de cabina | 2 (ECONOMY, BUSINESS) |
| 24 | `maintenance_type` | Tipos de mantenimiento | 2 (PREVENTIVE, CORRECTIVE) |
| 25 | `tax` | Impuestos | 1 (IVA 19%) |
| 26 | `incident_type` | Tipos de incidencia (ADR-001) | 3 (DELAY, LOST_BAGGAGE, DENIED_BOARDING) |
| 27 | `claim_status` | Estados de reclamación (ADR-001) | 3 (OPEN, IN_PROGRESS, RESOLVED) |

### Capa 2 — Entidades geográficas e institucionales

| Orden | Tabla | Depende de | Registros sugeridos |
|---|---|---|---|
| 28 | `country` | `continent` | 3 (Colombia, España, EEUU) |
| 29 | `state_province` | `country` | 2 (Cundinamarca, Antioquia) |
| 30 | `city` | `state_province`, `time_zone` | 3 (Bogotá, Medellín, Madrid) |
| 31 | `district` | `city` | 2 (Chapinero, El Poblado) |
| 32 | `address` | `district` | 3 (aeropuertos + proveedor mantenimiento) |
| 33 | `airline` | `country` | 1 (Aerolínea Colombia S.A.) |
| 34 | `aircraft_model` | `aircraft_manufacturer` | 2 (Boeing 737, Airbus A320) |

### Capa 3 — Infraestructura y personal

| Orden | Tabla | Depende de | Registros sugeridos |
|---|---|---|---|
| 35 | `airport` | `address` | 2 (BOG El Dorado, MDE José M. Córdova) |
| 36 | `terminal` | `airport` | 2 (Terminal 1 y Terminal 2 de BOG) |
| 37 | `boarding_gate` | `terminal` | 3 (puertas por terminal) |
| 38 | `runway` | `airport` | 2 (pistas de BOG) |
| 39 | `aircraft` | `airline`, `aircraft_model` | 2 aeronaves activas |
| 40 | `aircraft_cabin` | `aircraft`, `cabin_class` | 2 cabinas por aeronave |
| 41 | `aircraft_seat` | `aircraft_cabin` | 10 asientos por cabina |
| 42 | `maintenance_provider` | `address` | 1 proveedor |
| 43 | `person` | `person_type`, `country` | 5 (3 pasajeros, 1 empleado, 1 admin) |
| 44 | `person_document` | `person`, `document_type`, `country` | 5 (un documento por persona) |
| 45 | `person_contact` | `person`, `contact_type` | 5 (email por persona) |
| 46 | `user_account` | `person`, `user_status` | 3 (agente ventas, ops vuelo, admin) |
| 47 | `user_role` | `user_account`, `security_role` | 3 asignaciones |
| 48 | `role_permission` | `security_role`, `security_permission` | 10 asignaciones |

### Capa 4 — Operaciones y ventas

| Orden | Tabla | Depende de | Registros sugeridos |
|---|---|---|---|
| 49 | `loyalty_program` | `airline`, `currency` | 1 programa |
| 50 | `loyalty_tier` | `loyalty_program` | 2 niveles (Silver, Gold) |
| 51 | `customer` | `airline`, `person`, `customer_category` | 3 clientes |
| 52 | `loyalty_account` | `customer`, `loyalty_program` | 2 cuentas |
| 53 | `fare_class` | `cabin_class` | 2 clases tarifarias |
| 54 | `fare` | `airline`, `airport`, `fare_class`, `currency` | 3 tarifas (BOG–MDE) |
| 55 | `flight` | `airline`, `aircraft`, `flight_status` | 2 vuelos |
| 56 | `flight_segment` | `flight`, `airport` | 2 segmentos (uno con delay) |
| 57 | `flight_delay` | `flight_segment`, `delay_reason_type` | 1 delay de 95 minutos |
| 58 | `reservation` | `customer`, `reservation_status`, `sale_channel` | 2 reservas |
| 59 | `reservation_passenger` | `reservation`, `person` | 3 pasajeros |
| 60 | `sale` | `reservation`, `currency` | 2 ventas |
| 61 | `ticket` | `sale`, `reservation_passenger`, `fare`, `ticket_status` | 3 tiquetes |
| 62 | `ticket_segment` | `ticket`, `flight_segment` | 3 asociaciones |
| 63 | `seat_assignment` | `ticket_segment`, `flight_segment`, `aircraft_seat` | 3 asignaciones |
| 64 | `baggage` | `ticket_segment` | 2 (1 CLAIMED, 1 LOST) |

### Capa 5 — Flujos derivados

| Orden | Tabla | Depende de | Registros sugeridos |
|---|---|---|---|
| 65 | `check_in` | `ticket_segment`, `check_in_status`, `user_account` | 2 check-ins |
| 66 | `boarding_pass` | `check_in` | 2 pases de abordar |
| 67 | `boarding_validation` | `boarding_pass`, `boarding_gate`, `user_account` | 2 (1 APPROVED, 1 REJECTED) |
| 68 | `payment` | `sale`, `payment_status`, `payment_method`, `currency` | 2 pagos |
| 69 | `payment_transaction` | `payment` | 2 transacciones CAPTURE |
| 70 | `exchange_rate` | `currency` | 1 tasa COP/USD |
| 71 | `invoice` | `sale`, `invoice_status`, `currency` | 2 facturas |
| 72 | `invoice_line` | `invoice`, `tax` | 4 líneas de detalle |

### Capa 6 — Incidencias y reclamaciones (ADR-001)

| Orden | Tabla | Depende de | Registros sugeridos |
|---|---|---|---|
| 73 | `claim` | `customer`, `incident_type`, `claim_status`, `flight_segment`, `ticket`, `baggage` | 2 reclamaciones |
| 74 | `claim_compensation` | `claim` | 1 compensación (MILES) |

---

## 4. Scripts de inserción documentados

### Capa 1 — Extracto representativo

```sql
-- ============================================================
-- CAPA 1: DATOS DE REFERENCIA
-- Script: 01-referencia.sql
-- Autor: Aprendiz SENA
-- Descripción: Inserta catálogos base sin dependencias externas
-- ============================================================

-- Zonas horarias
INSERT INTO time_zone (time_zone_name, utc_offset_minutes) VALUES
    ('America/Bogota',  -300),  -- UTC-5
    ('Europe/Madrid',     60),  -- UTC+1
    ('UTC',               0);

-- Continentes
INSERT INTO continent (continent_code, continent_name) VALUES
    ('AME', 'América'),
    ('EUR', 'Europa'),
    ('ASI', 'Asia');

-- Monedas
INSERT INTO currency (iso_currency_code, currency_name, currency_symbol, minor_units) VALUES
    ('COP', 'Peso colombiano',  '$',  0),
    ('USD', 'Dólar estadounidense', 'US$', 2),
    ('EUR', 'Euro',             '€',  2);

-- Estados de vuelo
INSERT INTO flight_status (status_code, status_name) VALUES
    ('ON_TIME',   'A tiempo'),
    ('DELAYED',   'Demorado'),
    ('CANCELLED', 'Cancelado'),
    ('DEPARTED',  'Despachado');

-- Tipos de incidencia (ADR-001)
INSERT INTO incident_type (type_code, type_name) VALUES
    ('DELAY',           'Vuelo demorado'),
    ('LOST_BAGGAGE',    'Equipaje perdido'),
    ('DENIED_BOARDING', 'Denegación de embarque');

-- Estados de reclamación (ADR-001)
INSERT INTO claim_status (status_code, status_name) VALUES
    ('OPEN',        'Abierta'),
    ('IN_PROGRESS', 'En gestión'),
    ('RESOLVED',    'Resuelta');
```

### Capa 4 — Vuelo con delay (caso de prueba crítico)

```sql
-- ============================================================
-- CAPA 4: VUELO CON DEMORA
-- Script: 04-vuelo-delay.sql
-- Descripción: Crea vuelo BOG→MDE con delay de 95 minutos.
-- Este registro es prerequisito para las reclamaciones de Capa 6.
-- ============================================================

-- Vuelo base
INSERT INTO flight (airline_id, aircraft_id, flight_status_id, flight_number, service_date)
VALUES (
    (SELECT airline_id FROM airline WHERE airline_code = 'ARC'),
    (SELECT aircraft_id FROM aircraft WHERE registration_number = 'HK-1234'),
    (SELECT flight_status_id FROM flight_status WHERE status_code = 'DELAYED'),
    'RC101',
    '2026-05-15'
);

-- Segmento BOG → MDE
INSERT INTO flight_segment (
    flight_id, origin_airport_id, destination_airport_id,
    segment_number, scheduled_departure_at, scheduled_arrival_at,
    actual_departure_at
)
VALUES (
    (SELECT flight_id FROM flight WHERE flight_number = 'RC101' AND service_date = '2026-05-15'),
    (SELECT airport_id FROM airport WHERE iata_code = 'BOG'),
    (SELECT airport_id FROM airport WHERE iata_code = 'MDE'),
    1,
    '2026-05-15 08:00:00-05',
    '2026-05-15 09:00:00-05',
    '2026-05-15 09:35:00-05'  -- Sale 95 minutos tarde
);

-- Registro del delay
INSERT INTO flight_delay (flight_segment_id, delay_reason_type_id, reported_at, delay_minutes, notes)
VALUES (
    (SELECT flight_segment_id FROM flight_segment
     WHERE flight_id = (SELECT flight_id FROM flight WHERE flight_number = 'RC101')
       AND segment_number = 1),
    (SELECT delay_reason_type_id FROM delay_reason_type WHERE reason_code = 'TECHNICAL'),
    '2026-05-15 07:45:00-05',
    95,
    'Falla en sistema hidráulico. Reparación en pista completada.'
);
```

### Capa 6 — Reclamaciones (ADR-001)

```sql
-- ============================================================
-- CAPA 6: RECLAMACIONES E INCIDENCIAS
-- Script: 06-reclamaciones.sql
-- Descripción: Inserta reclamaciones vinculadas al vuelo con
-- delay y al equipaje en estado LOST.
-- PREREQUISITO: Capas 1–5 deben estar cargadas.
-- ============================================================

-- Reclamación 1: por vuelo demorado
INSERT INTO claim (
    customer_id, incident_type_id, claim_status_id,
    flight_segment_id, ticket_id,
    claim_reference, description
)
VALUES (
    (SELECT customer_id FROM customer
     WHERE person_id = (SELECT person_id FROM person WHERE last_name = 'García')),
    (SELECT incident_type_id FROM incident_type WHERE type_code = 'DELAY'),
    (SELECT claim_status_id FROM claim_status WHERE status_code = 'OPEN'),
    (SELECT flight_segment_id FROM flight_segment
     WHERE flight_id = (SELECT flight_id FROM flight WHERE flight_number = 'RC101')
       AND segment_number = 1),
    (SELECT ticket_id FROM ticket WHERE ticket_number = 'RC-2026-001'),
    'CLM-2026-0001',
    'Vuelo RC101 del 15/05/2026 llegó con 95 minutos de retraso. Solicito compensación.'
);

-- Reclamación 2: por equipaje perdido
INSERT INTO claim (
    customer_id, incident_type_id, claim_status_id,
    baggage_id, claim_reference, description
)
VALUES (
    (SELECT customer_id FROM customer
     WHERE person_id = (SELECT person_id FROM person WHERE last_name = 'López')),
    (SELECT incident_type_id FROM incident_type WHERE type_code = 'LOST_BAGGAGE'),
    (SELECT claim_status_id FROM claim_status WHERE status_code = 'IN_PROGRESS'),
    (SELECT baggage_id FROM baggage WHERE baggage_tag = 'BOG-00291'),
    'CLM-2026-0002',
    'Maleta con etiqueta BOG-00291 no fue entregada en carrusel de MDE.'
);

-- Compensación por reclamación de delay (millas)
INSERT INTO claim_compensation (claim_id, compensation_type, miles_awarded, applied_at)
VALUES (
    (SELECT claim_id FROM claim WHERE claim_reference = 'CLM-2026-0001'),
    'MILES',
    5000,
    now()
);
```

---

## 5. Validaciones por capa

Cada capa debe verificarse antes de proceder con la siguiente. Las siguientes consultas permiten confirmar que los datos se insertaron correctamente.

### Validaciones de Capa 1

```sql
-- Verificar que todos los catálogos base están poblados
SELECT 'time_zone'       AS tabla, COUNT(*) AS registros FROM time_zone
UNION ALL
SELECT 'continent',       COUNT(*) FROM continent
UNION ALL
SELECT 'currency',        COUNT(*) FROM currency
UNION ALL
SELECT 'flight_status',   COUNT(*) FROM flight_status
UNION ALL
SELECT 'incident_type',   COUNT(*) FROM incident_type
UNION ALL
SELECT 'claim_status',    COUNT(*) FROM claim_status;
-- Resultado esperado: todas las tablas con COUNT >= 1
```

### Validaciones de Capa 3

```sql
-- Verificar estructura geográfica completa (árbol de dependencias)
SELECT
    a.airport_name,
    a.iata_code,
    ci.city_name,
    co.country_name
FROM airport a
JOIN address  ad ON ad.address_id  = a.address_id
JOIN district d  ON d.district_id  = ad.district_id
JOIN city     ci ON ci.city_id     = d.city_id
JOIN state_province sp ON sp.state_province_id = ci.state_province_id
JOIN country  co ON co.country_id  = sp.country_id;
-- Resultado esperado: 2 filas con BOG y MDE
```

### Validaciones de Capa 4

```sql
-- Verificar que el vuelo con delay existe y tiene el registro de demora
SELECT
    f.flight_number,
    fs.scheduled_departure_at,
    fs.actual_departure_at,
    fd.delay_minutes,
    drt.reason_name
FROM flight f
JOIN flight_segment fs   ON fs.flight_id = f.flight_id
JOIN flight_delay   fd   ON fd.flight_segment_id = fs.flight_segment_id
JOIN delay_reason_type drt ON drt.delay_reason_type_id = fd.delay_reason_type_id
WHERE f.flight_number = 'RC101';
-- Resultado esperado: 1 fila con delay_minutes = 95

-- Verificar equipaje en estado LOST (prerequisito para Capa 6)
SELECT baggage_tag, baggage_status
FROM baggage
WHERE baggage_status = 'LOST';
-- Resultado esperado: al menos 1 fila
```

### Validaciones de Capa 5

```sql
-- Verificar flujo completo de abordaje
SELECT
    bp.boarding_pass_code,
    bv.validation_result,
    bg.gate_code
FROM boarding_pass bp
JOIN boarding_validation bv ON bv.boarding_pass_id = bp.boarding_pass_id
JOIN boarding_gate       bg ON bg.boarding_gate_id  = bv.boarding_gate_id;
-- Resultado esperado: 2 filas (1 APPROVED, 1 REJECTED)

-- Verificar que los pagos tienen transacciones asociadas
SELECT p.payment_reference, pt.transaction_type, pt.transaction_amount
FROM payment p
JOIN payment_transaction pt ON pt.payment_id = p.payment_id;
-- Resultado esperado: 2 filas de tipo CAPTURE
```

### Validaciones de Capa 6

```sql
-- Verificar reclamaciones con su dominio completo
SELECT
    c.claim_reference,
    it.type_name      AS tipo_incidencia,
    cs.status_name    AS estado,
    cc.compensation_type,
    cc.miles_awarded
FROM claim c
JOIN incident_type   it ON it.incident_type_id  = c.incident_type_id
JOIN claim_status    cs ON cs.claim_status_id   = c.claim_status_id
LEFT JOIN claim_compensation cc ON cc.claim_id  = c.claim_id;
-- Resultado esperado: 2 reclamaciones; la del delay tiene miles_awarded = 5000
```

---

## 6. Plan de rollback de datos

Si un script de inserción genera errores o inconsistencias, se puede revertir capa por capa. Los scripts de rollback siguen el orden inverso al de inserción.

```sql
-- Rollback Capa 6 (reclamaciones)
DELETE FROM claim_compensation;
DELETE FROM claim;

-- Rollback Capa 5 (flujos derivados)
DELETE FROM invoice_line;
DELETE FROM invoice;
DELETE FROM payment_transaction;
DELETE FROM payment;
DELETE FROM boarding_validation;
DELETE FROM boarding_pass;
DELETE FROM check_in;

-- Rollback Capa 4 (ventas y operaciones)
DELETE FROM baggage;
DELETE FROM seat_assignment;
DELETE FROM ticket_segment;
DELETE FROM ticket;
DELETE FROM sale;
DELETE FROM reservation_passenger;
DELETE FROM reservation;
DELETE FROM flight_delay;
DELETE FROM flight_segment;
DELETE FROM flight;
DELETE FROM fare;
DELETE FROM fare_class;
DELETE FROM loyalty_account;
DELETE FROM loyalty_tier;
DELETE FROM loyalty_program;
DELETE FROM customer;
```

> **Nota:** En entorno de desarrollo con Docker, el rollback más eficiente es ejecutar `docker compose down -v` y volver a levantar el entorno desde cero. Los scripts de rollback son útiles para corregir errores parciales sin reiniciar el contenedor.

---

## 7. Estructura de archivos sugerida

```
liquibase/
└── changelogs/
    └── 13-datos-prueba/
        ├── 01-referencia.sql
        ├── 02-geografia-aerolinea.sql
        ├── 03-infraestructura-personal.sql
        ├── 04-operaciones-ventas.sql
        ├── 05-abordaje-pagos-facturacion.sql
        └── 06-reclamaciones.sql
```

Cada archivo tiene su changeset en Liquibase con contexto `test-data` para que solo se ejecute en entornos de desarrollo y QA, nunca en producción:

```xml
<changeSet id="test-data-capa-1" author="aprendiz-sena" context="test-data">
    <sqlFile path="changelogs/13-datos-prueba/01-referencia.sql"
             splitStatements="true" stripComments="true"/>
    <rollback>
        <sqlFile path="changelogs/13-datos-prueba/rollback/01-referencia-rollback.sql"
                 splitStatements="true"/>
    </rollback>
</changeSet>
```

Ejecución solo en QA o desarrollo:
```bash
liquibase --changeLogFile=changelog-master.xml update --contexts=test-data
```

---

## 8. Dependencias y prerequisitos técnicos

Antes de ejecutar cualquier script de inserción se debe verificar:

1. El contenedor PostgreSQL está corriendo y `pg_isready` retorna OK.
2. La extensión `pgcrypto` está habilitada (`CREATE EXTENSION IF NOT EXISTS pgcrypto`).
3. Los changelogs de DDL de las Capas 1–12 ya fueron aplicados por Liquibase.
4. Los changelogs del dominio de incidencias (ADR-001) también fueron aplicados.

```bash
# Verificar que el esquema está completo antes de insertar datos
docker exec -it airline_db psql -U airline_admin -d airline \
  -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';"
# Resultado esperado: 56 tablas (52 base + 4 de incidencias del ADR-001)
```
