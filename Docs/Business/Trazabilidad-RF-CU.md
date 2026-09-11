# Matriz de Trazabilidad: Requerimientos ↔ Casos de Uso

> Cruce entre `Docs/Business/Requerimientos.md` y los 17 Casos de Uso de
> `Docs/Use case/`. Sirve para verificar que todo requerimiento esté cubierto por al
> menos un CU y que todo CU responda a un requerimiento real.

## Requerimientos Funcionales → Caso de Uso

| RF | Descripción breve | CU |
| --- | --- | --- |
| RF-01 | ABM de servicios (nombre, duración, precio) | CU-02 |
| RF-02 | Registrar turno (cliente, servicios, fecha/hora) | CU-03 |
| RF-03 | Calcular duración total del turno | CU-14 *(subfunción interna, invocada por CU-03/04/11/12)* |
| RF-04 | Impedir solapamiento de horarios | CU-15 *(subfunción interna, invocada por CU-03/04/12)* |
| RF-05 | Modificar fecha/horario de un turno | CU-04 |
| RF-06 | Cancelar turno | CU-05 |
| RF-07 | Marcar turno como "no asistido" | CU-06 |
| RF-08 | Registrar turno sin reserva previa (walk-in) | CU-03 *(sub-variación del mismo endpoint, sin CU propio)* |
| RF-09 | Mostrar agenda por día/semana (incluye turnos web) | CU-07 |
| RF-10 | ABM y consulta de clientes | CU-08 |
| RF-11 | Historial de servicios por cliente | CU-09 |
| RF-12 | Enviar recordatorio de turno | CU-17 |
| RF-13 | Login con usuario y contraseña | CU-01 |
| RF-14 | Generar copia de seguridad de la BD | CU-10 |
| RF-15 | Modificar un servicio de un turno ya registrado | CU-04 *(misma lógica que RF-05)* |
| RF-16 | Mostrar horarios disponibles según servicios/duración | CU-11 |
| RF-17 | Elegir servicios viendo duración y precio antes de reservar | CU-11 (devuelve duración y precio total), CU-12 (paso de selección) |
| RF-18 | Reservar turno en horario disponible (nombre/apellido/teléfono) | CU-12 |
| RF-19 | Cancelar reserva propia con ≥24hs de antelación | CU-13 |
| RF-20 | Turno reservado por web se refleja automáticamente en la agenda | CU-16 *(garantía de diseño; se lee en CU-07)* |

Los 17 CU quedan cubiertos: CU-14, CU-15, CU-16 y CU-17 son subfunciones internas sin
endpoint propio, pero cada una responde a un RF concreto (RF-03, RF-04, RF-20 y RF-12
respectivamente).

## Requerimientos No Funcionales → dónde se ejercitan

La mayoría son transversales (arquitectura/infraestructura) y no mapean a un CU puntual:
**RNF-01, RNF-02, RNF-03, RNF-06, RNF-07, RNF-08** aplican al sistema completo por igual.
Los que sí se pueden anclar a CUs concretos:

| RNF | Se ejercita en |
| --- | --- |
| RNF-04 (plataforma dueña, con login) | CU-01 |
| RNF-05 (plataforma clienta, sin instalación) | CU-11, CU-12, CU-13 |
| RNF-09 (usabilidad clienta / mobile) | CU-11, CU-12, CU-13 |
| RNF-10 (respuesta ≤2s) | CU-03, CU-07, CU-12 *(mencionados explícitamente en el propio RNF-10)* |
| RNF-11 (passwords hasheadas) | CU-01 (RN-01) |
| RNF-12 (validación de datos web) | CU-12, CU-13 |
| RNF-13 (concurrencia) | CU-15, CU-12 (flujo 4a) |
| RNF-14 (integridad multi-entidad) | CU-03, CU-12 (turno + servicios + alta de cliente en una misma operación) |

## Reglas de Negocio

Ver catálogo consolidado en `Requerimientos.md` §2.3 (RN-01 a RN-06), con su RF y CU
asociados.

## Hallazgos

- ✅ **Resuelto** — RF-17 pedía mostrar duración y precio al elegir servicios, pero el
  flujo de CU-11 (`GET /api/disponibilidad`) solo devolvía horarios, sin precio. Se
  actualizó `CU-11` para que la respuesta incluya `duracion_total_minutos` y
  `precio_total` de los servicios seleccionados.
- ✅ **Resuelto** — no existía un catálogo único de Reglas de Negocio (RN-01 a RN-06
  vivían dispersas, una por CU). Se agregó la sección 2.3 en `Requerimientos.md`.
- ⏳ **Pendiente (a resolver en conjunto)** — inconsistencia de terminología en
  `Requerimientos.md`: la sección 1.1 describe la vista de la dueña como "aplicación de
  escritorio", mientras que RNF-04 la define como aplicación web accesible desde el
  navegador de la tablet. Los 17 CU están todos escritos en términos de API REST/Flask
  consumida por una vista web (consistente con RNF-04), por lo que "escritorio" en 1.1
  parece un remanente a corregir — a confirmar con el equipo antes de tocarlo.
