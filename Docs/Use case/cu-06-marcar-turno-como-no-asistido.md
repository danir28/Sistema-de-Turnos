# Caso de Uso: Marcar Turno como No Asistido

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Regla de negocio **RN-05**
> (solo puede marcarse como no asistido un turno con fecha/hora ya transcurridas).

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-06 |
| **Nombre** | Marcar Turno como No Asistido |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → llevar registro de ausencias de clientes |
| **Disparador (Trigger)** | La usuaria constata que un cliente no se presentó a su turno |
| **Prioridad / Frecuencia** | Baja; frecuencia baja-media |
| **Reglas de negocio relacionadas** | RN-05 (solo aplica a turnos con fecha/hora ya transcurridas) |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña registrar que un cliente no se presentó a un turno cuyo horario ya
transcurrió.

### 2. PRECONDICIONES
- El turno existe, corresponde a una fecha y hora ya transcurridas, y se encuentra en
  estado `"reservado"`.

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200)
1. La usuaria selecciona, desde la agenda (CU-07), un turno cuyo horario ya transcurrió y
   el Actor envía una petición `POST /api/turnos/<id>/no-asistido`.
2. La **Negocio** (`turno_service.marcar_no_asistido()`) recupera el `Turno` de la
   Persistencia y verifica, aplicando **RN-05**, que su `fecha_hora_fin` sea anterior al
   momento actual.
3. La **Persistencia** actualiza el estado del `Turno` a `"no_asistido"`.
4. El Sistema devuelve un código **200 OK** con el turno actualizado.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **2a. Turno con fecha u horario futuro (HTTP 409 Conflict):**
  1. Si en el paso 2 la verificación detecta que el turno corresponde a una fecha u
     horario que todavía no ocurrió, violando **RN-05**.
  2. La **Negocio** lanza la excepción de dominio `TurnoAunNoTranscurridoException`.
  3. El Sistema devuelve un código **409 Conflict** con el mensaje: "No se puede marcar
     como no asistido un turno que todavía no ocurrió". Fin del caso de uso.

* **2b. Turno inexistente o no marcable (HTTP 404 Not Found):**
  1. Si el `id` no corresponde a ningún turno, o el turno no está en estado
     `"reservado"` (ya cancelado o ya marcado como no asistido).
  2. La **Negocio** no encuentra un turno marcable.
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

### 6. POSTCONDICIONES
- El turno queda registrado en estado `"no_asistido"`, visible en el historial del
  cliente (CU-09).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Confirmación exitosa del cambio de estado a "no asistido". |
| `404` | Not Found | El turno no existe o no se encuentra en estado `"reservado"`. |
| `409` | Conflict | Violación de **RN-05**: el turno corresponde a una fecha/hora futura. |

### Nota: Validación vs. Verificación aplicada

- **Verificación (Negocio, → 404/409):** existencia y estado del turno (404); **RN-05**
  comparando `fecha_hora_fin` del turno contra `datetime.now()` (409).
- No hay validación de esquema relevante en Presentación: el endpoint no recibe cuerpo,
  solo el `id` en la URL.

### Matriz de trazabilidad CU-06 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_turno_service.py`) | Test integración (`tests/integration/test_turnos_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `200 OK` | `test_marcar_no_asistido_turno_pasado_cambia_estado` | `test_post_turno_no_asistido_devuelve_200` |
| 2a. Turno futuro | `409 Conflict` | `test_marcar_no_asistido_turno_futuro_lanza_excepcion` | `test_post_turno_no_asistido_turno_futuro_devuelve_409` |
| 2b. Turno inexistente | `404 Not Found` | `test_marcar_no_asistido_turno_inexistente_lanza_excepcion` | `test_post_turno_no_asistido_id_inexistente_devuelve_404` |
