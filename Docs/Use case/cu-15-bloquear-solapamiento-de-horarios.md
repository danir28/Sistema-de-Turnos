# Caso de Uso: Bloquear Solapamiento de Horarios

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** + **Python** (Negocio) + **SQLAlchemy/SQLite**
> (Persistencia). Regla de negocio **RN-03** (no pueden coexistir dos turnos con horarios
> superpuestos).
>
> **Nota de alcance:** este caso de uso **no es un endpoint HTTP propio**. Es una función
> interna de la Capa de Negocio (`turno_service.validar_solapamiento()`), invocada desde
> dentro de otros casos de uso (CU-03, CU-04, CU-12) que sí son endpoints. No existe una
> petición HTTP directa hacia esta lógica.

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-15 |
| **Nombre** | Bloquear Solapamiento de Horarios |
| **Actor Principal** | Sistema |
| **Alcance / Nivel** | Sistema; subfunción (función interna reutilizada por otros CU) |
| **Stakeholders e intereses** | Dueña/Profesional → no perder tiempo por turnos superpuestos; Clienta → confianza en que el horario reservado es exclusivo suyo |
| **Disparador (Trigger)** | Se intenta registrar o modificar un turno (CU-03, CU-04, CU-12) |
| **Prioridad / Frecuencia** | Alta; se ejecuta en cada registro, modificación o reserva de turno |
| **Reglas de negocio relacionadas** | RN-03 (no pueden coexistir dos turnos con horarios superpuestos) |

---

### 1. BREVE DESCRIPCIÓN
Garantiza que no se registren dos turnos con horarios superpuestos, incluso ante
solicitudes simultáneas. Es una función interna de la Capa de Negocio, sin endpoint HTTP
propio.

### 2. PRECONDICIONES
- Se determinó el horario de inicio y fin del turno a registrar o modificar (**incluye
  CU-14**).

### 3. FLUJO PRINCIPAL (Camino Feliz)
1. La función `turno_service.validar_solapamiento(fecha, hora_inicio, hora_fin,
   turno_id_excluir=None)` recibe el rango horario calculado por CU-14, invocada desde el
   CU que la llama (CU-03, CU-04 o CU-12). En modificaciones (CU-04), recibe además el
   `turno_id_excluir` para no compararse contra sí mismo.
2. La función consulta en la Persistencia los turnos existentes en estado `"reservado"`
   para la misma `fecha`, dentro de una transacción que bloquea las filas comparadas
   (`SELECT ... FOR UPDATE` o transacción `SERIALIZABLE` en SQLAlchemy, según el motor),
   para garantizar consistencia ante solicitudes simultáneas.
3. Si ningún turno existente se superpone con el rango recibido, la función permite
   continuar con la operación y retorna sin error.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **3a. Existe superposición de horarios:**
  1. Si en el paso 3 la comparación encuentra al menos un turno existente cuyo rango
     horario se cruza con el recibido, violando **RN-03**.
  2. La función rechaza la operación lanzando la excepción de dominio
     `TurnoSolapadoException` al CU que la invocó.
  3. El CU llamador es quien decide y devuelve el código HTTP correspondiente (ver
     Anexo). Fin del caso de uso.

* **3b. Dos solicitudes simultáneas para el mismo horario:**
  1. Si dos peticiones concurrentes intentan reservar el mismo horario al mismo tiempo
     (por ejemplo, dos clientas desde CU-12).
  2. El bloqueo transaccional del paso 2 garantiza que únicamente una de las dos
     operaciones complete su commit; la segunda encuentra el turno ya persistido al
     re-consultar dentro de su propia transacción.
  3. La segunda operación recibe el mismo resultado que 3a (`TurnoSolapadoException`).
     Fin del caso de uso.

### 6. POSTCONDICIONES
- Queda garantizado que no existan dos turnos con horarios superpuestos, incluso bajo
  concurrencia.

---

## Anexo: matrices de referencia

### Códigos HTTP usados

Este caso de uso **no expone código HTTP propio**: no es un endpoint, por lo que no
recibe ni responde peticiones HTTP directamente. El resultado de su ejecución (éxito o la
excepción `TurnoSolapadoException`) se propaga al CU que lo invoca, que es el que
efectivamente responde al Actor:

| CU llamador | Código que devuelve ante éxito de CU-15 | Código que devuelve ante `TurnoSolapadoException` |
| --- | --- | --- |
| CU-03 (Registrar Turno) | Continúa el flujo hacia `201 Created` | `409 Conflict` |
| CU-04 (Modificar Turno) | Continúa el flujo hacia `200 OK` | `409 Conflict` |
| CU-12 (Reservar Turno) | Continúa el flujo hacia `201 Created` | `409 Conflict` |

### Matriz de trazabilidad CU-15 → Test

| Paso del CU | Resultado | Test unitario (`tests/unit/test_turno_service.py`) | Test integración |
| --- | --- | --- | --- |
| Flujo principal | Sin superposición, continúa la operación | `test_validar_solapamiento_horario_libre_no_lanza_excepcion` | — (cubierto indirectamente por los tests de integración de CU-03, CU-04 y CU-12) |
| 3a. Superposición detectada | `TurnoSolapadoException` | `test_validar_solapamiento_horario_ocupado_lanza_excepcion` | — (cubierto por `test_post_turnos_horario_solapado_devuelve_409` en CU-03 y equivalentes en CU-04/CU-12) |
| 3b. Solicitudes simultáneas | Solo una completa; la otra recibe `TurnoSolapadoException` | `test_validar_solapamiento_solicitudes_concurrentes_solo_una_persiste` | `test_post_turnos_reservas_concurrentes_mismo_horario_solo_una_devuelve_201` |
