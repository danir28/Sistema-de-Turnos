# Caso de Uso: Gestionar Servicios

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Regla de negocio **RN-02**
> (un servicio no puede eliminarse si está asociado a turnos futuros).

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-02 |
| **Nombre** | Gestionar Servicios |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → mantener actualizado el catálogo de servicios, duraciones y precios; Clienta → ver servicios y precios correctos al reservar (CU-11, CU-12) |
| **Disparador (Trigger)** | La usuaria solicita acceder al catálogo de servicios |
| **Prioridad / Frecuencia** | Alta; frecuencia baja-media (altas y ajustes de precio ocasionales) |
| **Reglas de negocio relacionadas** | RN-02 (no eliminar un servicio asociado a turnos futuros) |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña administrar el catálogo de servicios de la peluquería: consultarlos,
darlos de alta, modificarlos o eliminarlos.

### 2. PRECONDICIONES
- La usuaria inició sesión correctamente (CU-01).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200/201)
1. El Actor envía una petición `GET /api/servicios` para listar el catálogo.
2. La **Presentación** (`servicios_bp`, ruta `listar_servicios()`) delega en la
   **Negocio** (`servicio_service.listar_servicios()`), que consulta la **Persistencia**
   (modelo `Servicio`) y devuelve **200 OK** con nombre, duración y precio de cada uno.
3. La usuaria solicita dar de alta un nuevo servicio: el Actor envía
   `POST /api/servicios` con un JSON `{"nombre", "duracion_minutos", "precio"}`.
4. La **Presentación** valida el formato del JSON y la **Negocio**
   (`servicio_service.crear_servicio()`) registra el nuevo `Servicio` en la Persistencia.
5. El Sistema devuelve un código **201 Created** con el servicio creado y el listado queda
   actualizado para la próxima consulta (paso 1).

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **3a. Modificar un servicio existente (HTTP 200 OK):**
  1. La usuaria selecciona un servicio del listado y envía
     `PUT /api/servicios/<id>` con los datos actualizados.
  2. La **Negocio** (`servicio_service.actualizar_servicio()`) valida y guarda los
     cambios en la Persistencia.
  3. El Sistema devuelve un código **200 OK** con el servicio actualizado. El flujo
     retorna al paso 1.

* **3b. Eliminar un servicio (HTTP 200 OK / 409 Conflict):**
  1. La usuaria selecciona un servicio del listado y envía
     `DELETE /api/servicios/<id>`.
  2. La **Negocio** (`servicio_service.eliminar_servicio()`) verifica, aplicando
     **RN-02**, que el servicio no esté asociado a turnos futuros con estado "Reservado".
  3. Si no hay turnos futuros asociados, el Sistema elimina el registro y devuelve
     **200 OK**. El flujo retorna al paso 1.

* **3c. Servicio asociado a turnos futuros (HTTP 409 Conflict):**
  1. Si en el paso 3b la verificación detecta que el servicio está asociado a al menos
     un turno futuro con estado "Reservado", violando **RN-02**.
  2. La **Negocio** frena la ejecución y lanza la excepción de dominio
     `ServicioConTurnosFuturosException`.
  3. El Sistema devuelve un código **409 Conflict** con el mensaje: "No se puede eliminar
     el servicio: tiene turnos futuros asociados". Fin del caso de uso.

* **4a. Datos inválidos (HTTP 400 Bad Request):**
  1. Si en el paso 4 (o en 3a) `duracion_minutos` o `precio` no son valores numéricos
     válidos o positivos, o `nombre` llega vacío.
  2. La **Presentación** rechaza la petición por error de validación de esquema.
  3. El Sistema devuelve un código **400 Bad Request** indicando el campo inválido. Fin
     del caso de uso.

* **3d. Servicio inexistente (HTTP 404 Not Found):**
  1. Si en 3a o 3b el `id` del servicio enviado no existe en la Persistencia.
  2. La **Negocio** no encuentra la entidad correspondiente.
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

### 6. POSTCONDICIONES
- El catálogo de servicios queda actualizado con la información dada de alta, modificada
  o eliminada.
- Los cambios quedan disponibles de inmediato para CU-03 (Registrar Turno), CU-11
  (Consultar Horarios Disponibles) y CU-12 (Reservar Turno).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Éxito al listar, modificar o eliminar un servicio. |
| `201` | Created | Confirmación de persistencia exitosa de un nuevo servicio. |
| `400` | Bad Request | Datos inválidos (nombre vacío, duración o precio no numéricos/negativos). |
| `404` | Not Found | El servicio referenciado por `id` no existe. |
| `409` | Conflict | Violación de **RN-02**: el servicio tiene turnos futuros asociados. |

### Nota: Validación vs. Verificación aplicada

- **Validación (Presentación, → 400):** formato del JSON, tipos de dato y valores
  positivos de `duracion_minutos` y `precio`.
- **Verificación (Negocio, → 404/409):** existencia del servicio (404) y **RN-02**
  (409), revisando `existen_turnos_futuros_con_servicio()` antes de eliminar.

### Matriz de trazabilidad CU-02 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_servicio_service.py`) | Test integración (`tests/integration/test_servicios_api.py`) |
| --- | --- | --- | --- |
| Flujo principal (alta) | `201 Created` | `test_crear_servicio_datos_validos_persiste_servicio` | `test_post_servicios_datos_validos_devuelve_201` |
| Flujo principal (listar) | `200 OK` | `test_listar_servicios_retorna_catalogo` | `test_get_servicios_devuelve_200_con_listado` |
| 3a. Modificar servicio | `200 OK` | `test_actualizar_servicio_datos_validos_guarda_cambios` | `test_put_servicios_datos_validos_devuelve_200` |
| 3b. Eliminar servicio | `200 OK` | `test_eliminar_servicio_sin_turnos_futuros_elimina_registro` | `test_delete_servicios_sin_turnos_futuros_devuelve_200` |
| 3c. Servicio con turnos futuros | `409 Conflict` | `test_eliminar_servicio_con_turnos_futuros_lanza_excepcion` | `test_delete_servicios_con_turnos_futuros_devuelve_409` |
| 4a. Datos inválidos | `400 Bad Request` | — (validación de esquema en Presentación) | `test_post_servicios_precio_invalido_devuelve_400` |
| 3d. Servicio inexistente | `404 Not Found` | `test_actualizar_servicio_inexistente_lanza_excepcion` | `test_put_servicios_id_inexistente_devuelve_404` |
