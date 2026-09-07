# Caso de Uso: Registrar Turno

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Reglas de negocio **RN-03**
> (sin solapamiento de horarios) y **RN-04** (duración total = suma de duraciones de los
> servicios). Este caso de uso invoca internamente a CU-14 (Calcular Duración del Turno) y
> CU-15 (Bloquear Solapamiento de Horarios).

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-03 |
| **Nombre** | Registrar Turno |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → registrar turnos sin conflictos de horario; Clienta → tener su turno correctamente agendado |
| **Disparador (Trigger)** | La usuaria solicita registrar un nuevo turno |
| **Prioridad / Frecuencia** | Alta; uso diario (varias veces por jornada) |
| **Reglas de negocio relacionadas** | RN-03 (sin solapamiento); RN-04 (duración total = suma de servicios) |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña registrar un turno para un cliente —con reserva previa o presentado
espontáneamente— asociándolo a uno o más servicios, calculando su duración y validando
que no se superponga con otro turno existente.

### 2. PRECONDICIONES
- La usuaria inició sesión correctamente (CU-01).
- Existe al menos un servicio dado de alta en el sistema (CU-02).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 201)
1. El Actor envía una petición `POST /api/turnos` con un JSON que contiene `cliente_id`
   (o los datos de un cliente nuevo), una lista de `servicios_ids` y la `fecha_hora_inicio`.
2. La **Presentación** (`turnos_bp`, ruta `registrar_turno()`) valida que el JSON sea
   estructuralmente correcto y que los campos requeridos estén presentes.
3. La **Negocio** (`turno_service.registrar_turno()`) invoca a `calcular_duracion_turno()`
   (**incluye CU-14**) para determinar la duración total y la hora de finalización,
   aplicando **RN-04**.
4. La **Negocio** invoca a `validar_solapamiento()` (**incluye CU-15**) para verificar que
   el horario calculado no se superponga con otro turno existente, aplicando **RN-03**.
5. La **Persistencia** guarda el nuevo `Turno` (estado `"reservado"`) junto con su relación
   many-to-many con los `Servicio` seleccionados (tabla `turno_servicio`).
6. El Sistema devuelve un código **201 Created** con los datos del turno registrado y la
   agenda queda actualizada para la próxima consulta (CU-07).

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **1a. Cliente no registrado (alta en línea):**
  1. Si el JSON no incluye `cliente_id` sino los datos de un cliente nuevo (`nombre`,
     `apellido`, `telefono`).
  2. La **Negocio** invoca a `cliente_service.crear_cliente()` (**incluye CU-08**) y
     obtiene el `cliente_id` recién creado antes de continuar en el paso 3.
  3. El flujo continúa normalmente desde el paso 3.

* **2a. Datos incompletos o JSON inválido (HTTP 400 Bad Request):**
  1. Si en el paso 2 falta `cliente_id`/datos de cliente nuevo, `servicios_ids` está
     vacío, o `fecha_hora_inicio` no tiene un formato válido.
  2. La **Presentación** rechaza la petición por error de validación de esquema.
  3. El Sistema devuelve un código **400 Bad Request** detallando el campo faltante o
     inválido. Fin del caso de uso.

* **4a. Horario superpuesto (HTTP 409 Conflict):**
  1. Si en el paso 4 `validar_solapamiento()` detecta que el horario calculado se
     superpone con un turno existente, violando **RN-03**.
  2. La **Negocio** lanza la excepción de dominio `TurnoSolapadoException`.
  3. El Sistema devuelve un código **409 Conflict** con el mensaje: "El horario
     seleccionado se superpone con otro turno existente". El flujo retorna al paso 1.
     Fin del caso de uso.

* **1b. Cliente inexistente (HTTP 404 Not Found):**
  1. Si el JSON incluye un `cliente_id` que no existe en la Persistencia.
  2. La **Negocio** no encuentra la entidad `Cliente` correspondiente.
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

* **3a. Servicio inexistente (HTTP 404 Not Found):**
  1. Si alguno de los `servicios_ids` no existe en la Persistencia.
  2. `calcular_duracion_turno()` (CU-14) no puede resolver la duración de un servicio
     inexistente y la **Negocio** interrumpe el registro.
  3. El Sistema devuelve un código **404 Not Found** indicando qué servicio no existe.
     Fin del caso de uso.

### 5. SUB-VARIACIONES (opcional)
1. El registro admite dos orígenes de datos que comparten el mismo endpoint y el mismo
   resultado (**201 Created**): un cliente con **reserva previa** (fecha/hora futura) o un
   cliente que **se presenta espontáneamente** sin haber reservado (fecha/hora del día
   actual). En ambos casos, la **Negocio** aplica exactamente la misma validación de
   solapamiento del paso 4 (CU-15) antes de confirmar el registro.

### 6. POSTCONDICIONES
- El turno queda registrado en estado `"reservado"` y visible en la agenda (CU-07).
- Si se dio de alta un cliente nuevo, este queda disponible para futuras consultas
  (CU-08, CU-09).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `201` | Created | Confirmación de persistencia exitosa del nuevo turno. |
| `400` | Bad Request | Datos faltantes o con formato inválido en la petición. |
| `404` | Not Found | El cliente o alguno de los servicios referenciados no existe. |
| `409` | Conflict | Violación de **RN-03**: el horario calculado se superpone con otro turno. |

### Nota: Validación vs. Verificación aplicada

- **Validación (Presentación, → 400):** presencia y formato de `cliente_id`/datos de
  cliente, `servicios_ids` y `fecha_hora_inicio`.
- **Verificación (Negocio, → 404/409):** existencia de cliente y servicios (404);
  **RN-04** vía CU-14 (cálculo de duración); **RN-03** vía CU-15 (solapamiento, → 409).

### Matriz de trazabilidad CU-03 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_turno_service.py`) | Test integración (`tests/integration/test_turnos_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `201 Created` | `test_registrar_turno_datos_validos_persiste_turno` | `test_post_turnos_datos_validos_devuelve_201` |
| 1a. Cliente nuevo | `201 Created` | `test_registrar_turno_cliente_nuevo_crea_cliente_y_turno` | `test_post_turnos_cliente_nuevo_devuelve_201` |
| 2a. Datos incompletos | `400 Bad Request` | — (validación de esquema en Presentación) | `test_post_turnos_sin_servicios_devuelve_400` |
| 4a. Horario superpuesto | `409 Conflict` | `test_registrar_turno_horario_solapado_lanza_excepcion` | `test_post_turnos_horario_solapado_devuelve_409` |
| 1b. Cliente inexistente | `404 Not Found` | `test_registrar_turno_cliente_inexistente_lanza_excepcion` | `test_post_turnos_cliente_inexistente_devuelve_404` |
| 3a. Servicio inexistente | `404 Not Found` | `test_registrar_turno_servicio_inexistente_lanza_excepcion` | `test_post_turnos_servicio_inexistente_devuelve_404` |
| Sub-variación (presentación espontánea) | `201 Created` | `test_registrar_turno_sin_reserva_previa_aplica_misma_validacion` | `test_post_turnos_presentacion_espontanea_devuelve_201` |
