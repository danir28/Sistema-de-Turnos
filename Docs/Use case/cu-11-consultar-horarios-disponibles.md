# Caso de Uso: Consultar Horarios Disponibles

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Sin reglas de negocio
> específicas asociadas. Endpoint **público**, sin autenticación (aplicación web de la
> clienta). Este caso de uso invoca internamente a CU-14 (Calcular Duración del Turno).
> Además de los horarios disponibles, la respuesta incluye la duración total y el precio
> total estimado de los servicios seleccionados (**RF-17**), para que la clienta los vea
> antes de confirmar la reserva (CU-12).

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-11 |
| **Nombre** | Consultar Horarios Disponibles |
| **Actor Principal** | Clienta |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Clienta → encontrar un horario disponible para los servicios que desea; Dueña/Profesional → recibir reservas solo en horarios realmente libres |
| **Disparador (Trigger)** | La clienta solicita ver los horarios disponibles para reservar |
| **Prioridad / Frecuencia** | Alta; uso frecuente (previo a cada reserva web) |
| **Reglas de negocio relacionadas** | No aplica |

---

### 1. BREVE DESCRIPCIÓN
Permite a la clienta, sin necesidad de autenticarse, consultar los horarios disponibles
para reservar un turno según los servicios que desea realizarse.

### 2. PRECONDICIONES
- Existe al menos un servicio dado de alta en el sistema (CU-02).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200)
1. La clienta accede a la aplicación de reservas y selecciona uno o más servicios de
   interés; el Actor envía una petición
   `GET /api/disponibilidad?servicios=<id1,id2,...>&fecha=<YYYY-MM-DD>`.
2. La **Negocio** (`disponibilidad_service.calcular_horarios_disponibles()`) invoca a
   `calcular_duracion_turno()` (**incluye CU-14**) para determinar la duración total
   necesaria según los servicios seleccionados.
3. La **Negocio** consulta la Persistencia (turnos existentes en la fecha solicitada) y
   calcula los huecos libres del día que puedan contener esa duración. En paralelo, suma
   el `precio` de cada `Servicio` seleccionado para obtener el precio total estimado
   (**RF-17**).
4. El Sistema devuelve un código **200 OK** con la lista de horarios disponibles que
   contienen la duración calculada, junto con la `duracion_total_minutos` y el
   `precio_total` de los servicios seleccionados.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **4a. No hay horarios disponibles en la fecha consultada (HTTP 200 OK):**
  1. Si en el paso 3 no existe ningún hueco libre que contenga la duración total
     calculada.
  2. La **Negocio** retorna una lista vacía.
  3. El Sistema devuelve un código **200 OK** con la lista vacía; el frontend sugiere
     consultar otra fecha. El flujo retorna al paso 1 con una nueva `fecha`.

* **1a. Parámetros inválidos (HTTP 400 Bad Request):**
  1. Si `servicios` está vacío, contiene un `id` no numérico, o `fecha` no respeta el
     formato `YYYY-MM-DD`.
  2. La **Presentación** rechaza la petición por error de validación de esquema.
  3. El Sistema devuelve un código **400 Bad Request**. Fin del caso de uso.

* **1b. Servicio inexistente (HTTP 404 Not Found):**
  1. Si alguno de los `id` en `servicios` no existe en la Persistencia.
  2. `calcular_duracion_turno()` (CU-14) no puede resolver la duración de un servicio
     inexistente.
  3. El Sistema devuelve un código **404 Not Found** indicando qué servicio no existe.
     Fin del caso de uso.

### 6. POSTCONDICIONES
- La clienta visualiza los horarios disponibles, junto con la duración y el precio total
  estimado de los servicios seleccionados (o la ausencia de horarios, si no hay
  disponibilidad ese día).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Consulta exitosa de disponibilidad (con o sin horarios libres). |
| `400` | Bad Request | Parámetros `servicios` o `fecha` con formato inválido. |
| `404` | Not Found | Alguno de los servicios referenciados no existe. |

### Matriz de trazabilidad CU-11 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_disponibilidad_service.py`) | Test integración (`tests/integration/test_disponibilidad_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `200 OK` | `test_calcular_horarios_disponibles_con_huecos_libres_retorna_lista` | `test_get_disponibilidad_devuelve_200_con_horarios` |
| 4a. Sin horarios disponibles | `200 OK` | `test_calcular_horarios_disponibles_sin_huecos_retorna_lista_vacia` | `test_get_disponibilidad_sin_horarios_devuelve_200_vacio` |
| 1a. Parámetros inválidos | `400 Bad Request` | — (validación de esquema en Presentación) | `test_get_disponibilidad_fecha_invalida_devuelve_400` |
| 1b. Servicio inexistente | `404 Not Found` | `test_calcular_horarios_disponibles_servicio_inexistente_lanza_excepcion` | `test_get_disponibilidad_servicio_inexistente_devuelve_404` |
