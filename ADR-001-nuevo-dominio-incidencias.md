# ADR-001: Ampliación del modelo con dominio funcional de Incidencias y Reclamaciones

- **Estado:** Propuesto
- **Fecha:** 2025-04-14
- **Autor:** Aprendiz SENA – ADSO

---

## Contexto

El modelo base (`modelo_postgresql.sql`) cubre de forma completa los flujos de geografía, identidad, seguridad, clientes, aeropuerto, aeronaves, operaciones de vuelo, ventas, abordaje, pagos y facturación. Sin embargo, no existe ningún dominio que registre incidencias operacionales ni reclamaciones de clientes. En la operación real de una aerolínea, los pasajeros presentan quejas por vuelos cancelados, equipaje perdido, denegaciones de embarque o retrasos superiores a umbrales regulatorios, y el sistema debe poder trazarlas y gestionarlas.

## Problema

No hay entidades que permitan:
- Registrar una reclamación iniciada por un cliente o pasajero.
- Asociar esa reclamación a un vuelo, segmento, ticket o equipaje específico.
- Hacer seguimiento del estado de resolución de la reclamación.
- Generar compensaciones (vouchers, millas, reembolsos) vinculadas a la resolución.

Esto crea un hueco funcional crítico entre el dominio de operaciones de vuelo (donde ocurre el incidente) y el dominio de clientes y fidelización (donde se debe reflejar la compensación).

## Decisión

Se amplía el modelo con un nuevo dominio funcional denominado **Incidencias y Reclamaciones**, compuesto por las siguientes entidades:

```sql
-- Tipo de incidencia
CREATE TABLE incident_type (
    incident_type_id  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    type_code         varchar(20) NOT NULL,
    type_name         varchar(100) NOT NULL,
    CONSTRAINT uq_incident_type_code UNIQUE (type_code)
);

-- Estado de reclamación
CREATE TABLE claim_status (
    claim_status_id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    status_code       varchar(20) NOT NULL,
    status_name       varchar(80) NOT NULL,
    CONSTRAINT uq_claim_status_code UNIQUE (status_code)
);

-- Reclamación principal
CREATE TABLE claim (
    claim_id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id           uuid NOT NULL REFERENCES customer(customer_id),
    incident_type_id      uuid NOT NULL REFERENCES incident_type(incident_type_id),
    claim_status_id       uuid NOT NULL REFERENCES claim_status(claim_status_id),
    flight_segment_id     uuid REFERENCES flight_segment(flight_segment_id),
    ticket_id             uuid REFERENCES ticket(ticket_id),
    baggage_id            uuid REFERENCES baggage(baggage_id),
    claim_reference       varchar(30) NOT NULL,
    reported_at           timestamptz NOT NULL DEFAULT now(),
    resolved_at           timestamptz,
    description           text,
    CONSTRAINT uq_claim_reference UNIQUE (claim_reference)
);

-- Compensación asociada a la reclamación
CREATE TABLE claim_compensation (
    claim_compensation_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id              uuid NOT NULL REFERENCES claim(claim_id),
    compensation_type     varchar(30) NOT NULL,
    amount                numeric(12,2),
    miles_awarded         integer,
    applied_at            timestamptz,
    CONSTRAINT ck_compensation_type CHECK (
        compensation_type IN ('VOUCHER', 'MILES', 'REFUND', 'UPGRADE', 'OTHER')
    )
);
```

## Justificación técnica

- El dominio es cohesivo: todas las entidades giran alrededor del ciclo de vida de una reclamación.
- Las relaciones propuestas son opcionales (`REFERENCES ... NULL`) para no romper el modelo base cuando no apliquen (por ejemplo, una reclamación de servicio en tierra que no tiene ticket asociado).
- La entidad `claim_compensation` se mantiene separada de `refund` y `miles_transaction` para evitar acoplamiento entre dominios. Si una compensación implica millas, genera un registro en `miles_transaction`; si implica reembolso, genera un `refund`. `claim_compensation` es el punto de control de qué tipo de compensación fue aprobada.
- El dominio no modifica ninguna entidad existente: solo agrega tablas, sin romper la integridad referencial actual.

## Consecuencias e impacto esperado

| Aspecto | Impacto |
|---|---|
| Modelo | Se agregan 4 tablas nuevas sin modificar las existentes |
| Liquibase | Requiere un changelog propio: `changelogs/incidencias/` |
| Backlog | Genera las tareas HU-009 (diseño de tablas) y HU-010 (datos de prueba para reclamaciones) |
| Seguridad | Los roles de atención al cliente deben tener permisos de escritura sobre `claim` y solo lectura sobre `claim_compensation` |
| Datos de prueba | Requiere datos previos de vuelos con delay y bagaje con estado `LOST` para simular casos reales |
