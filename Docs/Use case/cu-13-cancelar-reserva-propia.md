# Caso de Uso: Cancelar Reserva Propia

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Regla de negocio **RN-06**
> (antelación mínima de 24 horas para cancelar). Endpoint **público**, sin autenticación de
> usuario/contraseña (la clienta se identifica con el código/teléfono del turno).

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-13 |
| **Nombre** | Cancelar Reserva Propia |
| **Actor Principal** | Clienta |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Clienta → poder cancelar su turno sin depender de la dueña; Dueña/Profesional → liberar el horario con antelación suficiente para reasignarlo |
| **Disparador (Trigger)** | La clienta solicita cancelar un turno propio |
| **Prioridad / Frecuencia** | Media; frecuencia baja-media |
| **Reglas de negocio relacionadas** | RN-06 (cancelación con antelación mínima de 24 horas) |

---

### 1. BREVE DESCRIPCIÓN
Permite a la clienta cancelar, desde la aplicación web, un turno que reservó
previamente, respetando una antelación mínima de 24 horas.

### 2. PRECONDICIONES
- La clienta cuenta con un turno reservado y puede identificarlo ante el sistema (por
  ejemplo, con el `id` del turno y el `telefono` con el que reservó).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200)
1. La clienta accede a su turno reservado y solicita cancelarlo; el Actor envía una
   petición `POST /api/reservas/<id>/cancelar` con un JSON `{"telefono": ...}` para
   identificarse.
2. La **Negocio** (`reserva_service.cancelar_reserva_propia()`) recupera el `Turno` de la
   Persistencia y verifica que el `telefono` coincida con el registrado en el turno.
3. La **Negocio** valida, aplicando **RN-06**, que falten al menos 24 horas para el
   `fecha_hora_inicio` del turno.
4. La **Persistencia** actualiza el estado del `Turno` a `"cancelado"`.
5. El Sistema devuelve un código **200 OK** con la confirmación de la cancelación; el
   horario queda liberado en la agenda de la dueña (CU-07).

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **3a. Faltan menos de 24 horas para el turno (HTTP 409 Conflict):**
  1. Si en el paso 3 la verificación determina que faltan menos de 24 horas para el
     turno, violando **RN-06**.
  2. La **Negocio** lanza la excepción de dominio `AntelacionInsuficienteException`.
  3. El Sistema devuelve un código **409 Conflict** con el mensaje: "No es posible
     cancelar el turno: se requieren al menos 24 horas de antelación". El flujo finaliza
     sin cambios. Fin del caso de uso.

* **2a. Turno inexistente o teléfono no coincide (HTTP 404 Not Found):**
  1. Si el `id` no corresponde a ningún turno, el turno no está en estado
     `"reservado"`, o el `telefono` enviado no coincide con el registrado en el turno.
  2. La **Negocio** no puede identificar de forma válida el turno de la clienta (por
     seguridad, no distingue si falló el `id` o el `telefono`).
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

### 6. POSTCONDICIONES
- El turno queda en estado `"cancelado"` y el horario disponible para otra persona en la
  agenda de la dueña (CU-07).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Cancelación exitosa dentro del plazo permitido. |
| `404` | Not Found | El turno no existe, no es cancelable, o el teléfono no coincide. |
| `409` | Conflict | Violación de **RN-06**: faltan menos de 24 horas para el turno. |

### Nota: Validación vs. Verificación aplicada

- **Verificación (Negocio, → 404/409):** identidad de la clienta vía `telefono` (404);
  **RN-06** comparando `fecha_hora_inicio` del turno contra `datetime.now() + 24h` (409).
- No hay validación de esquema compleja: el cuerpo solo requiere el `telefono`.

### Matriz de trazabilidad CU-13 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_reserva_service.py`) | Test integración (`tests/integration/test_reservas_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `200 OK` | `test_cancelar_reserva_propia_con_24hs_o_mas_cambia_estado` | `test_post_reserva_cancelar_devuelve_200` |
| 3a. Antelación insuficiente | `409 Conflict` | `test_cancelar_reserva_propia_con_menos_de_24hs_lanza_excepcion` | `test_post_reserva_cancelar_menos_24hs_devuelve_409` |
| 2a. Turno inexistente o teléfono no coincide | `404 Not Found` | `test_cancelar_reserva_propia_telefono_no_coincide_lanza_excepcion` | `test_post_reserva_cancelar_telefono_incorrecto_devuelve_404` |
