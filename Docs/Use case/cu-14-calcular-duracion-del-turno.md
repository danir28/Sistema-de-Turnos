# Caso de Uso: Calcular Duración del Turno

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** + **Python** (Negocio) + **SQLAlchemy/SQLite**
> (Persistencia). Regla de negocio **RN-04** (duración total = suma de la duración
> estimada de cada servicio asociado).
>
> **Nota de alcance:** este caso de uso **no es un endpoint HTTP propio**. Es una función
> interna de la Capa de Negocio (`turno_service.calcular_duracion_turno()`), invocada
> desde dentro de otros casos de uso (CU-03, CU-04, CU-11, CU-12) que sí son endpoints. No
> existe una petición HTTP directa hacia esta lógica, por eso su Flujo Principal se
> describe como lógica interna y no como "el Actor envía una petición a...".

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-14 |
| **Nombre** | Calcular Duración del Turno |
| **Actor Principal** | Sistema |
| **Alcance / Nivel** | Sistema; subfunción (función interna reutilizada por otros CU) |
| **Stakeholders e intereses** | Dueña/Profesional → que la agenda refleje la duración real del turno; Clienta → ver una duración y hora de fin correctas al reservar |
| **Disparador (Trigger)** | Se selecciona uno o más servicios al registrar, modificar o reservar un turno (CU-03, CU-04, CU-11, CU-12) |
| **Prioridad / Frecuencia** | Alta; se ejecuta en cada registro, modificación o consulta de disponibilidad de turno |
| **Reglas de negocio relacionadas** | RN-04 (duración total = suma de la duración estimada de cada servicio) |

---

### 1. BREVE DESCRIPCIÓN
Determina la duración total y la hora de finalización de un turno, en función de los
servicios seleccionados. Es una función interna de la Capa de Negocio, sin endpoint HTTP
propio.

### 2. PRECONDICIONES
- Se seleccionó al menos un servicio (`servicios_ids` no vacío).

### 3. FLUJO PRINCIPAL (Camino Feliz)
1. La función `turno_service.calcular_duracion_turno(servicios_ids, hora_inicio)` recibe
   la lista de servicios seleccionados y la hora de inicio del turno, invocada desde el
   CU que la llama (CU-03, CU-04, CU-11 o CU-12).
2. La función consulta en la Persistencia la `duracion_minutos` de cada `Servicio`
   asociado y suma todas las duraciones, aplicando **RN-04**.
3. La función determina la `hora_fin` sumando la duración total a la `hora_inicio`
   recibida.
4. La función retorna `(duracion_total_minutos, hora_fin)` al CU que la invocó, que
   continúa su propio flujo (por ejemplo, CU-03 continúa validando solapamiento con
   CU-15).

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **2a. Servicio inexistente:**
  1. Si alguno de los `servicios_ids` recibidos no existe en la Persistencia.
  2. La función no puede resolver su duración y lanza la excepción de dominio
     `ServicioInexistenteException`.
  3. La excepción se propaga al CU llamador, que es quien decide y devuelve el código
     HTTP correspondiente (ver Anexo). Fin del caso de uso.

### 6. POSTCONDICIONES
- Queda determinada la duración total y la hora de finalización del turno, disponibles
  para que el CU llamador continúe su propio flujo (validación de solapamiento,
  persistencia, etc.).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

Este caso de uso **no expone código HTTP propio**: no es un endpoint, por lo que no
recibe ni responde peticiones HTTP directamente. El resultado de su ejecución (éxito o
la excepción `ServicioInexistenteException`) se propaga al CU que lo invoca, que es el
que efectivamente responde al Actor:

| CU llamador | Código que devuelve ante éxito de CU-14 | Código que devuelve ante `ServicioInexistenteException` |
| --- | --- | --- |
| CU-03 (Registrar Turno) | Continúa el flujo hacia `201 Created` | `404 Not Found` |
| CU-04 (Modificar Turno) | Continúa el flujo hacia `200 OK` | `404 Not Found` |
| CU-11 (Consultar Horarios Disponibles) | Continúa el flujo hacia `200 OK` | `404 Not Found` |
| CU-12 (Reservar Turno) | Continúa el flujo hacia `201 Created` | `404 Not Found` |

### Matriz de trazabilidad CU-14 → Test

| Paso del CU | Resultado | Test unitario (`tests/unit/test_turno_service.py`) | Test integración |
| --- | --- | --- | --- |
| Flujo principal | Duración y hora de fin correctas | `test_calcular_duracion_turno_un_servicio_suma_duracion` / `test_calcular_duracion_turno_varios_servicios_suma_duraciones` | — (cubierto indirectamente por los tests de integración de CU-03, CU-04, CU-11 y CU-12, que son los que exponen HTTP) |
| 2a. Servicio inexistente | `ServicioInexistenteException` | `test_calcular_duracion_turno_servicio_inexistente_lanza_excepcion` | — (cubierto por `test_post_turnos_servicio_inexistente_devuelve_404` en CU-03, y equivalentes en CU-04/CU-11/CU-12) |
