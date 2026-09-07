# Caso de Uso: Sincronizar Turno Web con Agenda

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** + **Python** (Negocio) + **SQLAlchemy/SQLite**
> (Persistencia). Sin reglas de negocio específicas asociadas.
>
> **Nota de alcance:** este caso de uso **no es un endpoint HTTP propio**. En este
> proyecto, la aplicación de la dueña y la aplicación web de la clienta comparten la
> **misma base de datos SQLite** a través del mismo backend Flask. Por eso "sincronizar"
> no requiere un mecanismo de replicación o mensajería: un turno reservado o cancelado
> desde CU-12/CU-13 ya está persistido en la misma tabla `turnos` que consulta CU-07. Este
> CU documenta esa garantía de consistencia por diseño, no un proceso HTTP en sí mismo.

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-16 |
| **Nombre** | Sincronizar Turno Web con Agenda |
| **Actor Principal** | Sistema |
| **Alcance / Nivel** | Sistema; subfunción (garantía de diseño, no proceso independiente) |
| **Stakeholders e intereses** | Dueña/Profesional → ver en su agenda, sin carga manual, los turnos que reservan o cancelan las clientas por web |
| **Disparador (Trigger)** | Se registra o cancela un turno desde la aplicación web (CU-12, CU-13) |
| **Prioridad / Frecuencia** | Alta; ocurre implícitamente en cada reserva o cancelación web |
| **Reglas de negocio relacionadas** | No aplica |

---

### 1. BREVE DESCRIPCIÓN
Refleja automáticamente, en la agenda que utiliza la dueña, los turnos reservados o
cancelados por las clientas a través de la aplicación web. En este proyecto esto se logra
por diseño (persistencia compartida), no mediante un proceso de sincronización activo.

### 2. PRECONDICIONES
- Se completó el registro (CU-12) o la cancelación (CU-13) de un turno desde la
  aplicación web.

### 3. FLUJO PRINCIPAL (Camino Feliz)
1. CU-12 o CU-13 completan su propia transacción de escritura sobre la tabla `turnos` de
   la base de datos SQLite compartida.
2. No existe un paso de "envío" o "sincronización" adicional: la escritura ya es visible
   para cualquier lectura posterior sobre la misma base de datos, incluida la que realiza
   `agenda_service.obtener_agenda()` (**incluye CU-07**).
3. La próxima vez que la dueña consulta la agenda (CU-07), la consulta a la Persistencia
   trae el turno reservado o cancelado desde la web sin ninguna acción manual de su
   parte.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)
No se registran flujos alternativos relevantes: al no existir un proceso de
sincronización independiente (sino persistencia compartida dentro de la misma
transacción de CU-12/CU-13), no hay un canal adicional que pueda fallar de forma
aislada. Cualquier fallo de escritura ya está cubierto por los flujos alternativos de
CU-12 y CU-13 (por ejemplo, `409 Conflict` si el horario se ocupó, o `500 Internal Server
Error` ante un fallo de persistencia).

### 6. POSTCONDICIONES
- El turno reservado o cancelado desde la web queda reflejado en la agenda de la dueña
  (CU-07) sin necesidad de carga manual.

---

## Anexo: matrices de referencia

### Códigos HTTP usados

Este caso de uso **no expone código HTTP propio**: no hay una petición HTTP dedicada a
"sincronizar". La consistencia entre ambas aplicaciones es una garantía de diseño
(persistencia compartida) verificada a través de los endpoints de CU-12, CU-13 y CU-07:

| CU relacionado | Rol en la sincronización |
| --- | --- |
| CU-12 (Reservar Turno) | Escribe el turno en la tabla compartida (`201 Created`). |
| CU-13 (Cancelar Reserva Propia) | Actualiza el estado del turno en la tabla compartida (`200 OK`). |
| CU-07 (Consultar Agenda) | Lee la tabla compartida y refleja el resultado (`200 OK`). |

### Matriz de trazabilidad CU-16 → Test

| Paso del CU | Resultado | Test unitario | Test integración (`tests/integration/test_agenda_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | Turno reservado por web visible en la agenda de la dueña | — (no aplica: no hay lógica de negocio propia que aislar) | `test_get_agenda_refleja_turno_reservado_por_clienta` (ver también CU-07) |
| Flujo principal (cancelación) | Turno cancelado por web visible como liberado en la agenda | — (no aplica) | `test_get_agenda_refleja_turno_cancelado_por_clienta` |

> Este CU no tiene tests unitarios propios porque no contiene lógica de negocio propia:
> la garantía se verifica end-to-end (reservar/cancelar por CU-12/CU-13 y luego consultar
> por CU-07 en la misma base de datos de test), no aislando una función de servicio.
