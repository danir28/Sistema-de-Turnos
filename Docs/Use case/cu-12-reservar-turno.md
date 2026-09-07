# Caso de Uso: Reservar Turno

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Regla de negocio **RN-03**
> (sin solapamiento de horarios). Endpoint **público**, sin autenticación. Este caso de uso
> invoca internamente a CU-11 (Consultar Horarios Disponibles) y CU-15 (Bloquear
> Solapamiento de Horarios); sus efectos se reflejan en la agenda de la dueña mediante
> CU-16 (Sincronizar Turno Web con Agenda).

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-12 |
| **Nombre** | Reservar Turno |
| **Actor Principal** | Clienta |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Clienta → reservar un turno por su cuenta sin llamar a la peluquería; Dueña/Profesional → que la reserva quede en su agenda sin carga manual |
| **Disparador (Trigger)** | La clienta solicita reservar un turno |
| **Prioridad / Frecuencia** | Alta; uso frecuente |
| **Reglas de negocio relacionadas** | RN-03 (sin solapamiento de horarios) |

---

### 1. BREVE DESCRIPCIÓN
Permite a la clienta reservar un turno por su cuenta desde la aplicación web pública, sin
necesidad de comunicarse con la dueña.

### 2. PRECONDICIONES
- Existe al menos un horario disponible para los servicios elegidos (**incluye CU-11**).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 201)
1. La clienta selecciona uno o más servicios (visualizando duración y precio estimado,
   **incluye CU-11**) y un horario disponible.
2. La clienta completa nombre, apellido y teléfono de contacto; el Actor envía una
   petición `POST /api/reservas` con un JSON `{"servicios_ids", "fecha_hora_inicio",
   "nombre", "apellido", "telefono"}`.
3. La **Presentación** (`reservas_bp`, ruta `reservar_turno()`) valida que el JSON sea
   estructuralmente correcto.
4. La **Negocio** (`reserva_service.reservar_turno_web()`) valida, mediante
   `validar_solapamiento()` (**incluye CU-15**), que el horario elegido siga libre en el
   momento de la confirmación (protección ante concurrencia), aplicando **RN-03**.
5. La **Persistencia** registra al cliente (si no existía) y el nuevo `Turno` con estado
   `"reservado"` y origen `"web"`, sobre la misma base de datos que utiliza la aplicación
   de la dueña.
6. El Sistema devuelve un código **201 Created** con la confirmación de la reserva. Como
   ambas aplicaciones comparten la misma Persistencia, el turno queda reflejado en la
   agenda de la dueña sin ninguna sincronización adicional (**incluye CU-16**), visible en
   su próxima consulta (CU-07).

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **2a. Datos de contacto inválidos (HTTP 400 Bad Request):**
  1. Si en el paso 3 `nombre`, `apellido` o `telefono` llegan vacíos o con formato
     inválido.
  2. La **Presentación** rechaza la petición por error de validación de esquema.
  3. El Sistema devuelve un código **400 Bad Request** solicitando corregir los datos de
     contacto. El flujo retorna al paso 2.

* **4a. El horario fue ocupado por otra reserva simultánea (HTTP 409 Conflict):**
  1. Si en el paso 4 `validar_solapamiento()` detecta que el horario ya no está libre
     (otra reserva se confirmó primero), violando **RN-03**.
  2. La **Negocio** lanza la excepción de dominio `TurnoSolapadoException`.
  3. El Sistema devuelve un código **409 Conflict** con el mensaje: "El horario
     seleccionado ya no se encuentra disponible". El frontend sugiere elegir otro horario
     (vuelve a CU-11). El flujo retorna al paso 1. Fin del caso de uso.

* **1a. Servicio inexistente (HTTP 404 Not Found):**
  1. Si alguno de los `servicios_ids` no existe en la Persistencia (por ejemplo, fue
     eliminado entre la consulta de disponibilidad y la confirmación).
  2. La **Negocio** no puede resolver la duración de un servicio inexistente.
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

### 6. POSTCONDICIONES
- El turno queda registrado en estado `"reservado"`, con origen `"web"`, y reflejado en
  la agenda de la dueña (CU-07) sin necesidad de carga manual (CU-16).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `201` | Created | Confirmación de persistencia exitosa de la reserva. |
| `400` | Bad Request | Datos de contacto faltantes o con formato inválido. |
| `404` | Not Found | Alguno de los servicios referenciados no existe. |
| `409` | Conflict | Violación de **RN-03**: el horario fue ocupado por otra reserva simultánea. |

### Nota: Validación vs. Verificación aplicada

- **Validación (Presentación, → 400):** formato de `nombre`, `apellido`, `telefono` y
  presencia de `servicios_ids`/`fecha_hora_inicio`.
- **Verificación (Negocio, → 404/409):** existencia de los servicios (404); **RN-03**
  vía CU-15 (409), re-verificada en el momento exacto de la confirmación para cubrir el
  caso de dos clientas reservando el mismo horario al mismo tiempo.

### Matriz de trazabilidad CU-12 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_reserva_service.py`) | Test integración (`tests/integration/test_reservas_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `201 Created` | `test_reservar_turno_web_datos_validos_persiste_turno` | `test_post_reservas_datos_validos_devuelve_201` |
| 2a. Datos de contacto inválidos | `400 Bad Request` | — (validación de esquema en Presentación) | `test_post_reservas_telefono_invalido_devuelve_400` |
| 4a. Horario ocupado por reserva simultánea | `409 Conflict` | `test_reservar_turno_web_horario_ocupado_simultaneamente_lanza_excepcion` | `test_post_reservas_horario_ocupado_devuelve_409` |
| 1a. Servicio inexistente | `404 Not Found` | `test_reservar_turno_web_servicio_inexistente_lanza_excepcion` | `test_post_reservas_servicio_inexistente_devuelve_404` |
