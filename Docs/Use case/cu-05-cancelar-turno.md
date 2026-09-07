# Caso de Uso: Cancelar Turno

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Sin reglas de negocio
> específicas asociadas.

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-05 |
| **Nombre** | Cancelar Turno |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → liberar el horario para otro cliente; Clienta → que su cancelación quede reflejada en la agenda |
| **Disparador (Trigger)** | La usuaria solicita cancelar un turno |
| **Prioridad / Frecuencia** | Media; frecuencia moderada |
| **Reglas de negocio relacionadas** | No aplica |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña cancelar un turno registrado, liberando el horario para que pueda ser
ocupado por otro cliente.

### 2. PRECONDICIONES
- El turno existe y se encuentra en estado `"reservado"`.

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200)
1. La usuaria selecciona un turno desde la agenda (CU-07) y solicita cancelarlo. La
   **Presentación** (frontend) solicita confirmación antes de disparar la petición al
   backend.
2. La usuaria confirma la cancelación y el Actor envía una petición
   `POST /api/turnos/<id>/cancelar`.
3. La **Negocio** (`turno_service.cancelar_turno()`) recupera el `Turno` de la
   Persistencia y actualiza su estado a `"cancelado"`.
4. El Sistema devuelve un código **200 OK** con el turno actualizado; el horario queda
   liberado en la agenda (CU-07).

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **1a. La usuaria desiste de la cancelación:**
  1. En el paso 1, la usuaria no confirma la acción en la interfaz.
  2. La **Presentación** (frontend) no envía ninguna petición al backend.
  3. El Sistema mantiene el turno sin cambios. Fin del caso de uso.

* **3a. Turno inexistente o ya no cancelable (HTTP 404 Not Found):**
  1. Si en el paso 3 el `id` no corresponde a ningún turno, o el turno ya no está en
     estado `"reservado"` (por ejemplo, ya fue cancelado o marcado como no asistido).
  2. La **Negocio** no encuentra un turno cancelable.
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

### 6. POSTCONDICIONES
- El turno queda en estado `"cancelado"` y el horario disponible para otro cliente en la
  agenda (CU-07).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Cancelación exitosa del turno. |
| `404` | Not Found | El turno no existe o ya no se encuentra en estado `"reservado"`. |

### Nota: Validación vs. Verificación aplicada

- **Validación (Presentación, → confirmación en frontend):** la confirmación de la
  cancelación ocurre en la interfaz antes de que exista petición HTTP alguna (1a).
- **Verificación (Negocio, → 404):** existencia y estado actual del turno.

### Matriz de trazabilidad CU-05 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_turno_service.py`) | Test integración (`tests/integration/test_turnos_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `200 OK` | `test_cancelar_turno_reservado_cambia_estado_a_cancelado` | `test_post_turno_cancelar_devuelve_200` |
| 3a. Turno inexistente o no cancelable | `404 Not Found` | `test_cancelar_turno_no_reservado_lanza_excepcion` | `test_post_turno_cancelar_id_inexistente_devuelve_404` |

> El flujo 1a (la usuaria desiste) no genera petición HTTP y por lo tanto no tiene test de
> integración asociado; se verifica con un test de UI/frontend fuera del alcance de esta
> matriz (Capa de Negocio y Persistencia).
