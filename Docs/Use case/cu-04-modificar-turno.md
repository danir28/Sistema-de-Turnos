# Caso de Uso: Modificar Turno

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Regla de negocio **RN-03**
> (sin solapamiento de horarios). Este caso de uso invoca internamente a CU-14 (Calcular
> Duración del Turno) y CU-15 (Bloquear Solapamiento de Horarios).

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-04 |
| **Nombre** | Modificar Turno |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → reprogramar turnos sin generar conflictos; Clienta → conservar un turno válido tras el cambio |
| **Disparador (Trigger)** | La usuaria solicita modificar un turno existente |
| **Prioridad / Frecuencia** | Media; frecuencia moderada (reprogramaciones ocasionales) |
| **Reglas de negocio relacionadas** | RN-03 (sin solapamiento de horarios) |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña modificar la fecha, el horario o los servicios de un turno ya
registrado, recalculando su duración y validando que el nuevo horario no genere
conflictos.

### 2. PRECONDICIONES
- El turno a modificar existe y se encuentra en estado `"reservado"`.

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200)
1. La usuaria selecciona un turno desde la agenda (CU-07) y el Actor envía una petición
   `PUT /api/turnos/<id>` con los campos a modificar (`fecha_hora_inicio` y/o
   `servicios_ids`).
2. La **Presentación** (`turnos_bp`, ruta `modificar_turno()`) valida el formato del JSON.
3. La **Negocio** (`turno_service.modificar_turno()`) recupera el `Turno` de la
   Persistencia y, si cambiaron los servicios, invoca a `calcular_duracion_turno()`
   (**incluye CU-14**) para recalcular la duración total y la hora de finalización.
4. La **Negocio** invoca a `validar_solapamiento()` (**incluye CU-15**), excluyendo el
   propio turno de la comparación, para verificar que el nuevo horario no se superponga
   con otro turno existente (**RN-03**).
5. La **Persistencia** guarda los cambios sobre el `Turno` existente.
6. El Sistema devuelve un código **200 OK** con los datos actualizados del turno; la
   agenda queda reflejada en la próxima consulta (CU-07).

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **2a. Datos inválidos (HTTP 400 Bad Request):**
  1. Si en el paso 2 el JSON tiene un formato de fecha/hora inválido o una lista de
     `servicios_ids` vacía.
  2. La **Presentación** rechaza la petición por error de validación de esquema.
  3. El Sistema devuelve un código **400 Bad Request**. Fin del caso de uso.

* **3a. Turno inexistente o no modificable (HTTP 404 Not Found):**
  1. Si en el paso 3 el `id` no corresponde a ningún turno, o el turno no está en estado
     `"reservado"` (por ejemplo, ya fue cancelado).
  2. La **Negocio** no encuentra un turno modificable.
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

* **4a. Horario superpuesto (HTTP 409 Conflict):**
  1. Si en el paso 4 el nuevo horario se superpone con otro turno existente, violando
     **RN-03**.
  2. La **Negocio** lanza la excepción de dominio `TurnoSolapadoException` y no persiste
     ningún cambio (mantiene los datos anteriores).
  3. El Sistema devuelve un código **409 Conflict** con el mensaje: "El nuevo horario se
     superpone con otro turno existente". El flujo retorna al paso 1. Fin del caso de uso.

### 6. POSTCONDICIONES
- El turno queda actualizado con la nueva información (fecha, horario y/o servicios) y
  reflejado en la agenda (CU-07).
- Si no se supera la validación (4a), el turno conserva sus datos originales sin cambios.

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Modificación exitosa de un turno existente. |
| `400` | Bad Request | Formato inválido de fecha/hora o lista de servicios vacía. |
| `404` | Not Found | El turno no existe o no está en un estado modificable. |
| `409` | Conflict | Violación de **RN-03**: el nuevo horario se superpone con otro turno. |

### Nota: Validación vs. Verificación aplicada

- **Validación (Presentación, → 400):** formato de los campos recibidos.
- **Verificación (Negocio, → 404/409):** existencia y estado del turno (404); **RN-03**
  vía CU-15, excluyendo el propio turno de la comparación de solapamiento (409).

### Matriz de trazabilidad CU-04 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_turno_service.py`) | Test integración (`tests/integration/test_turnos_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `200 OK` | `test_modificar_turno_datos_validos_actualiza_turno` | `test_put_turno_datos_validos_devuelve_200` |
| 2a. Datos inválidos | `400 Bad Request` | — (validación de esquema en Presentación) | `test_put_turno_fecha_invalida_devuelve_400` |
| 3a. Turno inexistente | `404 Not Found` | `test_modificar_turno_inexistente_lanza_excepcion` | `test_put_turno_id_inexistente_devuelve_404` |
| 4a. Horario superpuesto | `409 Conflict` | `test_modificar_turno_horario_solapado_lanza_excepcion_y_no_persiste` | `test_put_turno_horario_solapado_devuelve_409` |
