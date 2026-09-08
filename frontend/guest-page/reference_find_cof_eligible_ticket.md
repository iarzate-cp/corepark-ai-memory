---
name: Encontrar en dev una location y un ticket que disparen el gate de Card on File
description: Queries para hallar tickets elegibles a CoF; el gateway NO sale de payment_gateway_location sino de parking_location_payment_gateway_cfg; fixtures de dev (op 8, locations 5/12/16, ticket 3340311 con hotel_room 78)
metadata:
  type: reference
---
Para probar la pantalla "Unlock your valet pass" hacen falta dos cosas a la vez: una location elegible y un ticket que no tenga tarjeta pero sí la necesite.

## La trampa: de dónde sale el gateway

`company.payment_gateway_location` **no es la tabla**. Está vacía para locations que sí funcionan (op 8 / loc 16 lo está) y un join contra ella devuelve cero filas. La lee `ticket-info` del valet para otra cosa.

La ruta de Stripe en el front la decide `GET /backoffice/payment/guest-page/configuration`, y su query es (`PaymentsDaoImpl:100`):

```sql
SELECT cpg.payment_gateway_id
FROM company.parking_location_payment_gateway_cfg plpgc
JOIN company.payment_gateway_cfg pgc
     ON plpgc.payment_gateway_cfg_id = pgc.id AND pgc.deactivated_at IS NULL
JOIN custom.cat_payment_gateway cpg
     ON pgc.payment_gateway_id = cpg.payment_gateway_id
WHERE plpgc.operator_company_id = :op AND plpgc.parking_location_id = :loc
```

Stripe es `payment_gateway_id = 4`.

## Condiciones de elegibilidad

De `StripeDaoImpl.QRY_GET_COF_TICKET_ELIGIBILITY:731` + `evaluateCofEligibility`:

- Ticket abierto (`ps.check_out IS NULL`)
- `plr.rate_type_id = 6` (overnight)
- `plr.partner_process_payment` no true
- Credencial de Stripe configurada
- Fila viva en `company.card_on_file_consent` (`deactivated_at IS NULL`)
- Y para que el gate **se muestre**, que NO haya tarjeta viva: sin fila en `card_on_file_ticket` + `card_on_file` + `card_on_file_stripe` con `status IN ('PENDING','ACTIVE')` y sin `deactivated_at`

## Query que responde "cuáles tickets disparan el gate"

```sql
WITH abiertos AS (
    SELECT ps.ticket, ps.uuid, ps.check_in, ps.phone_number, ps.hotel_room,
           ps.expected_departure, cc.country_code AS phone_code,
           plr.rate_type_id,
           COALESCE(plr.partner_process_payment, FALSE) AS lo_cobra_el_partner,
           pl.location_uuid, pl.parking_location_tz,
           EXISTS (SELECT 1
                     FROM company.card_on_file_ticket cot
                     JOIN company.card_on_file cof        ON cof.id = cot.card_on_file_id
                     JOIN company.card_on_file_stripe cos ON cos.card_on_file_id = cof.id
                    WHERE cot.operator_company_id = ps.operator_company_id
                      AND cot.parking_location_id = ps.parking_location_id
                      AND cot.ticket = ps.ticket
                      AND cot.deactivated_at IS NULL AND cof.deactivated_at IS NULL
                      AND cos.status IN ('PENDING','ACTIVE')) AS tiene_tarjeta_viva
    FROM company.parking_service ps
    JOIN company.parking_location pl
         ON pl.operator_company_id = ps.operator_company_id
        AND pl.parking_location_id = ps.parking_location_id
    JOIN company.parking_location_rate plr
         ON plr.operator_company_id = ps.operator_company_id
        AND plr.parking_location_id = ps.parking_location_id
        AND plr.rate = ps.rate AND plr.rate_version = ps.rate_version
    LEFT JOIN custom.cat_country cc ON cc.country_id = ps.phone_code_id
    WHERE ps.operator_company_id = 8 AND ps.parking_location_id = 5
      AND ps.check_out IS NULL
)
SELECT ticket, uuid, rate_type_id, lo_cobra_el_partner, tiene_tarjeta_viva,
       (rate_type_id = 6 AND NOT lo_cobra_el_partner AND NOT tiene_tarjeta_viva) AS muestra_gate_cof,
       phone_number, phone_code, hotel_room,
       expected_departure AT TIME ZONE parking_location_tz AS departure_local,
       '/ticket/' || location_uuid || '/' || ticket AS path
FROM abiertos
ORDER BY muestra_gate_cof DESC, check_in DESC;
```

Las tres columnas antes de `muestra_gate_cof` dicen **cuál** condición falla en las que salen false.

## Fixtures de dev (2026-09-08)

Locations elegibles, todas del operador **8** y todas Stripe:

| loc | nombre | tz | location_uuid |
|---|---|---|---|
| 5 | Los Cholos | America/Mexico_City | `6f89b673-e44d-45fa-810d-5a943b69eac0` |
| 12 | Hotel Integration | America/Los_Angeles | `eaf09a4b-e003-4177-ba58-85d07813a518` |
| 16 | Evolution Test | America/Los_Angeles | `3eb906a2-9855-4aca-a2b1-d5526a5ee918` |

**Ticket de prueba: `3340311`** en loc 5 → `/ticket/6f89b673-e44d-45fa-810d-5a943b69eac0/3340311`

Es el mejor caso disponible porque trae **`hotel_room = 78`** y sin departure: es el único con cuarto capturado, así que es el único que detecta la regresión de que el eco del snapshot borre el room en RTDB. Los demás tickets de esa location son `rate_type_id = 1` y sirven de control negativo — no deben mostrar el gate.

En loc 16 el ticket **828** trae teléfono MX, útil para probar el prellenado del país; el 827 y 823 lo tienen vacío.

**Para la prueba de zona horaria usa loc 12 o 16** (Los Angeles). En Los Cholos, que está en la misma zona que la máquina, un bug de offset se vería idéntico a lo correcto.

## Verificación después de guardar

```sql
-- el valor y su hora local; hotel_room debe seguir intacto
SELECT ps.phone_number, ps.hotel_room,
       ps.expected_departure AT TIME ZONE pl.parking_location_tz AS local_de_la_location
FROM company.parking_service ps
JOIN company.parking_location pl
     ON pl.operator_company_id = ps.operator_company_id
    AND pl.parking_location_id = ps.parking_location_id
WHERE ps.operator_company_id = 8 AND ps.parking_location_id = 5
  AND ps.ticket = '3340311' AND ps.check_out IS NULL;

-- 9 = EDIT TICKET INFO (teléfono), 15 = EDIT HOTEL INFO (departure)
SELECT service_status_id, service_date_time
FROM company.parking_service_info
WHERE operator_company_id = 8 AND parking_location_id = 5 AND ticket = '3340311'
  AND service_status_id IN (9, 15)
ORDER BY service_date_time DESC LIMIT 10;
```

Y en RTDB: `expectedDeparture` actualizado y `hotelRoom` todavía en `78`.

Para **prender** los campos usa Commerce → Settings → Guest page, no un INSERT a mano en `company.field_configurations`.

Relacionado: [[feature-unlock-guest-info]], [[feedback-never-connect-to-databases]].
