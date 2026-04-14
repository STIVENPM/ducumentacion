# analisis de dominios funcionales

**proyecto:** sistema de aerolinea  
**archivo analizado:** modelo_postgresql.sql  
**autor:** mertinez.stiven@gmail.com  
**fecha:** 2026-04-14  

---

## introduccion

el modelo entregado representa un sistema de gestion de aerolinea construido sobre PostgreSQL.
utiliza UUIDs como claves primarias en todas las entidades, relaciones referenciadas con FOREIGN KEY,
restricciones CHECK para validaciones de negocio, indices de apoyo y comentarios tecnicos que
evidencian una madurez importante en el diseno del esquema.

el proposito de este documento es identificar los dominios funcionales presentes, describir
sus entidades principales y mapear las dependencias mas relevantes entre dominios.

---

## dominio 1 — geografia y datos de referencia

**prefijo sugerido para changelogs:** `geo`

### entidades

| tabla | descripcion |
|---|---|
| `time_zone` | zonas horarias con offset UTC en minutos |
| `continent` | continentes con codigo unico |
| `country` | paises referenciados a continente, con codigos ISO alpha2 y alpha3 |
| `state_province` | provincias o departamentos vinculados a un pais |
| `city` | ciudades con zona horaria, vinculadas a estado o provincia |
| `district` | distritos o barrios dentro de una ciudad |
| `address` | direccion fisica completa con latitud y longitud opcionales |
| `currency` | monedas con codigo ISO, simbolo y unidades menores |

### entidad raiz

`continent` es el punto de partida. todo el arbol geografico desciende desde ahi:
`continent` → `country` → `state_province` → `city` → `district` → `address`

### relaciones clave con otros dominios

- `address` es consumida por `airport` (dominio aeropuerto) y `maintenance_provider` (dominio aeronaves)
- `country` es referenciada por `airline`, `person` (nacionalidad) y `person_document` (pais emisor)
- `currency` es consumida por `loyalty_program`, `fare`, `sale`, `payment`, `invoice` y `exchange_rate`

### validaciones relevantes del modelo

- latitud entre -90 y 90, longitud entre -180 y 180 (CHECK en `address`)
- codigos de moneda entre 0 y 4 unidades menores
- unicidad garantizada en codigos ISO de pais y continente

---

## dominio 2 — aerolinea

**prefijo sugerido para changelogs:** `airline`

### entidades

| tabla | descripcion |
|---|---|
| `airline` | aerolinea con codigos IATA (2 chars) e ICAO (3 chars), vinculada a un pais |

### entidad raiz

`airline` es una entidad central del sistema. casi todos los dominios operativos la referencian directamente.

### relaciones clave con otros dominios

- referenciada por `aircraft`, `customer`, `loyalty_program`, `fare` y `flight`
- depende de `country` (dominio geografia)

### validaciones relevantes del modelo

- `iata_code` debe tener exactamente 2 caracteres si se especifica
- `icao_code` debe tener exactamente 3 caracteres si se especifica
- unicidad en `airline_code`, `airline_name`, `iata_code` e `icao_code`

---

## dominio 3 — identidad

**prefijo sugerido para changelogs:** `identity`

### entidades

| tabla | descripcion |
|---|---|
| `person_type` | tipo de persona (pasajero, empleado, etc.) |
| `document_type` | tipo de documento de identidad |
| `contact_type` | tipo de contacto (email, telefono, etc.) |
| `person` | persona natural con nombre, fecha de nacimiento, genero y nacionalidad |
| `person_document` | documentos de identidad vinculados a una persona |
| `person_contact` | datos de contacto de una persona (email, telefono, etc.) |

### entidad raiz

`person` es la entidad central. representa a cualquier actor humano del sistema,
sin importar su rol (pasajero, empleado o usuario de sistema).

### relaciones clave con otros dominios

- `person` es requerida por `user_account` (dominio seguridad) y `customer` (dominio clientes)
- `person` es requerida por `reservation_passenger` (dominio ventas)
- depende de `country` para la nacionalidad y el pais emisor del documento

### validaciones relevantes del modelo

- `gender_code` acepta solo 'F', 'M' o 'X'
- fecha de vencimiento del documento debe ser mayor o igual a la de emision
- unicidad del documento por tipo, pais emisor y numero

---

## dominio 4 — seguridad

**prefijo sugerido para changelogs:** `security`

### entidades

| tabla | descripcion |
|---|---|
| `user_status` | estado de la cuenta (activo, bloqueado, etc.) |
| `security_role` | roles del sistema con codigo unico |
| `security_permission` | permisos granulares con codigo unico |
| `user_account` | cuenta de acceso vinculada a una persona |
| `user_role` | asignacion de rol a usuario con trazabilidad de quien asigno |
| `role_permission` | permisos asignados a cada rol |

### entidad raiz

`user_account` vincula una `person` con credenciales de acceso. la asignacion de roles
y permisos se gestiona mediante las tablas puente `user_role` y `role_permission`.

### relaciones clave con otros dominios

- `user_account` depende de `person` (dominio identidad)
- `user_account` es referenciada por `check_in` y `boarding_validation` (dominio abordaje)
- `user_role.assigned_by_user_id` referencia a otro `user_account`, permitiendo auditoria

### validaciones relevantes del modelo

- unicidad garantizada de `username` y de la combinacion `(user_account_id, security_role_id)`
- `password_hash` almacena el hash, no la contrasena en texto plano

---

## dominio 5 — clientes y fidelizacion

**prefijo sugerido para changelogs:** `loyalty`

### entidades

| tabla | descripcion |
|---|---|
| `customer_category` | categorias de clientes (VIP, corporativo, etc.) |
| `benefit_type` | tipos de beneficios otorgables |
| `loyalty_program` | programa de millas/puntos por aerolinea |
| `loyalty_tier` | niveles del programa (plata, oro, platino, etc.) |
| `customer` | cliente registrado en una aerolinea |
| `loyalty_account` | cuenta de fidelizacion del cliente en un programa |
| `loyalty_account_tier` | historial de niveles asignados a una cuenta |
| `miles_transaction` | movimientos de millas (ganancia, canje, ajuste) |
| `customer_benefit` | beneficios vigentes asignados a un cliente |

### entidad raiz

`customer` es la entidad principal, vincula `person` con `airline`. de ella se desprende
la cuenta de fidelizacion y el historial de niveles.

### relaciones clave con otros dominios

- `customer` depende de `person` (identidad) y `airline` (aerolinea)
- `loyalty_program` depende de `airline` y `currency`
- `reservation` puede referenciar opcionalmente a un `customer`

### validaciones relevantes del modelo

- `miles_delta` no puede ser cero (evita transacciones sin efecto)
- `transaction_type` acepta solo 'EARN', 'REDEEM' o 'ADJUST'
- fechas de vigencia de niveles y beneficios con CHECK de coherencia

---

## dominio 6 — aeropuerto

**prefijo sugerido para changelogs:** `airport`

### entidades

| tabla | descripcion |
|---|---|
| `airport` | aeropuerto con codigos IATA (3 chars) e ICAO (4 chars) |
| `terminal` | terminales dentro de un aeropuerto |
| `boarding_gate` | puertas de abordaje dentro de una terminal |
| `runway` | pistas de aterrizaje y despegue |
| `airport_regulation` | regulaciones vigentes por aeropuerto |

### entidad raiz

`airport` es el punto de entrada. depende de `address` para su ubicacion geografica.

### relaciones clave con otros dominios

- `airport` es referenciado por `flight_segment` como origen y destino
- `airport` es referenciado por `fare` como ruta origen-destino
- `boarding_gate` es referenciada por `boarding_validation` (dominio abordaje)

### validaciones relevantes del modelo

- `iata_code` debe tener exactamente 3 caracteres si se especifica
- `icao_code` debe tener exactamente 4 caracteres si se especifica
- longitud de pista debe ser mayor a 0

---

## dominio 7 — aeronaves

**prefijo sugerido para changelogs:** `aircraft`

### entidades

| tabla | descripcion |
|---|---|
| `aircraft_manufacturer` | fabricante de aeronaves |
| `aircraft_model` | modelo de aeronave con alcance maximo en km |
| `cabin_class` | clases de cabina (economica, ejecutiva, primera clase) |
| `aircraft` | aeronave especifica con matricula y numero de serie |
| `aircraft_cabin` | cabinas definidas en una aeronave |
| `aircraft_seat` | asientos individuales por cabina |
| `maintenance_provider` | proveedor de mantenimiento |
| `maintenance_type` | tipos de mantenimiento (preventivo, correctivo, etc.) |
| `maintenance_event` | eventos de mantenimiento con estado y fechas |

### entidad raiz

`aircraft` es la entidad principal. se construye a partir del modelo y el fabricante,
y se subdivide en cabinas y asientos.

### relaciones clave con otros dominios

- `aircraft` depende de `airline`
- `aircraft` es asignada a `flight` (operaciones de vuelo)
- `aircraft_seat` es referenciada por `seat_assignment` (ventas y reservas)
- `cabin_class` es referenciada por `fare_class` (ventas)

### validaciones relevantes del modelo

- fecha de retiro debe ser mayor o igual a fecha de ingreso a servicio
- `deck_number` debe ser mayor a 0
- `max_range_km` debe ser mayor a 0 si se especifica

---

## dominio 8 — operaciones de vuelo

**prefijo sugerido para changelogs:** `flight`

### entidades

| tabla | descripcion |
|---|---|
| `flight_status` | estados posibles de un vuelo (a tiempo, demorado, cancelado, etc.) |
| `delay_reason_type` | razones tipificadas de demora |
| `flight` | vuelo identificado por aerolinea, numero y fecha de servicio |
| `flight_segment` | segmento individual del vuelo (tramo origen-destino) |
| `flight_delay` | registro de demoras por segmento con causa y minutos |

### entidad raiz

`flight` es la entidad principal. puede tener uno o varios segmentos para representar
vuelos con escalas.

### relaciones clave con otros dominios

- `flight` depende de `airline` y `aircraft`
- `flight_segment` referencia aeropuertos de origen y destino
- `ticket_segment` del dominio ventas referencia `flight_segment`
- `seat_assignment` y `check_in` dependen indirectamente de `flight_segment`

### validaciones relevantes del modelo

- origen y destino de un segmento deben ser aeropuertos diferentes
- hora de llegada programada debe ser mayor a la de salida
- minutos de demora deben ser mayores a 0

---

## dominio 9 — ventas y reservas

**prefijo sugerido para changelogs:** `sales`

### entidades

| tabla | descripcion |
|---|---|
| `reservation_status` | estados de una reserva |
| `sale_channel` | canal de venta (web, agencia, call center, etc.) |
| `fare_class` | clase tarifaria vinculada a cabina |
| `fare` | tarifa con monto, ruta, clase y vigencia |
| `ticket_status` | estados de un tiquete |
| `reservation` | reserva raiz del flujo comercial |
| `reservation_passenger` | pasajeros incluidos en una reserva |
| `sale` | venta generada a partir de una reserva |
| `ticket` | tiquete emitido por pasajero y vuelo |
| `ticket_segment` | tabla puente entre tiquete y segmento de vuelo |
| `seat_assignment` | asignacion de asiento por tiquete-segmento |
| `baggage` | equipaje registrado por tiquete-segmento |

### entidad raiz

`reservation` es la entidad raiz del flujo comercial, segun el comentario tecnico del modelo.
de ella se derivan la venta, los tiquetes y los segmentos de tiquete.

### relaciones clave con otros dominios

- `reservation` puede referenciar opcionalmente a `customer`
- `ticket_segment` vincula el dominio ventas con el dominio operaciones de vuelo
- `seat_assignment` vincula con el dominio aeronaves (asiento fisico)
- `fare` depende de `airline`, `airport` y `cabin_class`

### validaciones relevantes del modelo

- `passenger_type` acepta solo 'ADULT', 'CHILD' o 'INFANT'
- `assignment_source` acepta solo 'AUTO', 'MANUAL' o 'CUSTOMER'
- tipo de equipaje acepta 'CHECKED', 'CARRY_ON' o 'SPECIAL'
- estado de equipaje acepta 'REGISTERED', 'LOADED', 'CLAIMED' o 'LOST'

---

## dominio 10 — abordaje

**prefijo sugerido para changelogs:** `boarding`

### entidades

| tabla | descripcion |
|---|---|
| `boarding_group` | grupos de abordaje con orden de secuencia |
| `check_in_status` | estados del proceso de check-in |
| `check_in` | registro de check-in por tiquete-segmento |
| `boarding_pass` | pase de abordar generado del check-in |
| `boarding_validation` | validacion del pase en puerta con resultado |

### entidad raiz

`check_in` es el punto de entrada. se genera a partir de un `ticket_segment` y produce
un `boarding_pass`, el cual luego es validado en puerta.

### relaciones clave con otros dominios

- `check_in` depende de `ticket_segment` (dominio ventas)
- `boarding_validation` referencia `boarding_gate` (dominio aeropuerto)
- `check_in` y `boarding_validation` registran el usuario que realizo la accion (dominio seguridad)

### validaciones relevantes del modelo

- `validation_result` acepta solo 'APPROVED', 'REJECTED' o 'MANUAL_REVIEW'
- unicidad del pase de abordar por check-in, por codigo y por codigo de barras

---

## dominio 11 — pagos y facturacion

**prefijo sugerido para changelogs:** `billing`

### entidades

| tabla | descripcion |
|---|---|
| `payment_status` | estados de un pago |
| `payment_method` | metodos de pago (tarjeta, transferencia, etc.) |
| `payment` | pago asociado a una venta |
| `payment_transaction` | transacciones individuales de un pago (auth, capture, refund, etc.) |
| `refund` | solicitudes y procesamiento de devoluciones |
| `tax` | impuestos con tasa porcentual y vigencia |
| `exchange_rate` | tasas de cambio entre monedas por fecha |
| `invoice_status` | estados de una factura |
| `invoice` | factura emitida por una venta |
| `invoice_line` | lineas de detalle de la factura sin totales derivados |

### entidad raiz

`payment` y `invoice` son las dos entidades principales. ambas dependen de `sale`
y representan respectivamente el flujo de cobro y el flujo de facturacion.

### relaciones clave con otros dominios

- `payment` e `invoice` dependen de `sale` (dominio ventas)
- `invoice_line` puede referenciar `tax`
- `exchange_rate` depende de `currency` (dominio geografia/referencia)
- el comentario tecnico del modelo indica que `invoice_line` no persiste totales derivados para preservar 3FN

### validaciones relevantes del modelo

- `transaction_type` acepta 'AUTH', 'CAPTURE', 'VOID', 'REFUND' o 'REVERSAL'
- montos de pago y transaccion deben ser mayores a 0
- tasa de impuesto debe ser mayor o igual a 0
- par de monedas en `exchange_rate` no puede referenciar la misma moneda

---

## mapa de dependencias entre dominios

```
geografia/referencia
    ├── aerolinea
    │     ├── aeronaves
    │     ├── clientes y fidelizacion
    │     ├── operaciones de vuelo
    │     └── ventas (via fare)
    └── identidad
          ├── seguridad
          ├── clientes y fidelizacion
          └── ventas (via reservation_passenger)

aeropuerto (depende de geografia)
    ├── operaciones de vuelo (flight_segment)
    └── abordaje (boarding_gate)

operaciones de vuelo
    └── ventas (ticket_segment → flight_segment)

ventas
    ├── abordaje (check_in → ticket_segment)
    └── pagos y facturacion (payment, invoice → sale)
```

---

## observaciones tecnicas del modelo

- uso consistente de `gen_random_uuid()` con la extension `pgcrypto` en todas las PKs
- campos `created_at` y `updated_at` presentes en todas las tablas para trazabilidad
- comentarios en tablas clave como `reservation`, `ticket_segment`, `seat_assignment`,
  `loyalty_account_tier` e `invoice_line` que evidencian decisiones de diseno deliberadas
- indices de apoyo en todas las FK para optimizar consultas de navegacion entre entidades
- restricciones CHECK para valores enumerados en lugar de tipos ENUM, lo que facilita
  el versionamiento con Liquibase sin necesidad de migraciones de tipo
