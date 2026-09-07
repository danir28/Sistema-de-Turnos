# Caso de Uso: Consultar Historial de Cliente

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Sin reglas de negocio
> específicas asociadas.

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-09 |
| **Nombre** | Consultar Historial de Cliente |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → conocer los servicios previos de un cliente para personalizar la atención |
| **Disparador (Trigger)** | La usuaria solicita ver el historial de un cliente |
| **Prioridad / Frecuencia** | Baja; frecuencia baja |
| **Reglas de negocio relacionadas** | No aplica |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña consultar los servicios realizados a un cliente en turnos anteriores.

### 2. PRECONDICIONES
- El cliente consultado se encuentra registrado en el sistema (**incluye CU-08**).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200)
1. La usuaria selecciona un cliente desde el listado de clientes (**incluye CU-08**) y el
   Actor envía una petición `GET /api/clientes/<id>/historial`.
2. La **Negocio** (`cliente_service.obtener_historial_cliente()`) consulta en la
   Persistencia los `Turno` asociados a ese `cliente_id` con estado `"reservado"` (pasados)
   o `"no_asistido"`, junto con sus servicios.
3. El Sistema devuelve un código **200 OK** con el historial ordenado por fecha, indicando
   fecha y servicios realizados en cada turno anterior.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **2a. El cliente no registra turnos anteriores (HTTP 200 OK):**
  1. Si en el paso 2 no se encuentran turnos pasados asociados al cliente.
  2. La **Negocio** retorna una lista vacía.
  3. El Sistema devuelve un código **200 OK** informando que el cliente no posee
     historial previo.

* **1a. Cliente inexistente (HTTP 404 Not Found):**
  1. Si el `id` del cliente enviado no existe en la Persistencia.
  2. La **Negocio** no encuentra la entidad `Cliente` correspondiente.
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

### 6. POSTCONDICIONES
- La usuaria visualiza el historial de servicios del cliente consultado (o la ausencia de
  historial, si no tiene turnos previos).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Consulta exitosa del historial (con o sin turnos previos). |
| `404` | Not Found | El cliente referenciado por `id` no existe. |

### Matriz de trazabilidad CU-09 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_cliente_service.py`) | Test integración (`tests/integration/test_clientes_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `200 OK` | `test_obtener_historial_cliente_con_turnos_retorna_lista_ordenada` | `test_get_historial_cliente_devuelve_200_con_turnos` |
| 2a. Sin historial previo | `200 OK` | `test_obtener_historial_cliente_sin_turnos_retorna_lista_vacia` | `test_get_historial_cliente_sin_turnos_devuelve_200_vacio` |
| 1a. Cliente inexistente | `404 Not Found` | `test_obtener_historial_cliente_inexistente_lanza_excepcion` | `test_get_historial_cliente_id_inexistente_devuelve_404` |
