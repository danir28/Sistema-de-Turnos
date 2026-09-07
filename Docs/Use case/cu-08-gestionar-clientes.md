# Caso de Uso: Gestionar Clientes

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Sin reglas de negocio
> específicas asociadas.

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-08 |
| **Nombre** | Gestionar Clientes |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → mantener actualizados los datos de contacto de sus clientes |
| **Disparador (Trigger)** | La usuaria solicita acceder a la información de clientes |
| **Prioridad / Frecuencia** | Media; frecuencia media (usado también internamente por CU-03) |
| **Reglas de negocio relacionadas** | No aplica |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña administrar los datos de los clientes de la peluquería: darlos de
alta, modificar su información o consultarlos.

### 2. PRECONDICIONES
- La usuaria inició sesión correctamente (CU-01).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200/201)
1. El Actor envía una petición `GET /api/clientes` para listar los clientes registrados.
2. La **Negocio** (`cliente_service.listar_clientes()`) consulta la Persistencia (modelo
   `Cliente`) y el Sistema devuelve **200 OK** con el listado.
3. La usuaria solicita dar de alta un nuevo cliente: el Actor envía `POST /api/clientes`
   con un JSON `{"nombre", "apellido", "telefono"}`.
4. La **Presentación** (`clientes_bp`, ruta `crear_cliente()`) valida el formato del JSON
   y la **Negocio** (`cliente_service.crear_cliente()`) registra el nuevo `Cliente`.
5. El Sistema devuelve un código **201 Created** con el cliente creado.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **3a. Modificar un cliente existente (HTTP 200 OK):**
  1. La usuaria selecciona un cliente del listado y envía `PUT /api/clientes/<id>` con
     los datos actualizados.
  2. La **Negocio** (`cliente_service.actualizar_cliente()`) guarda los cambios en la
     Persistencia.
  3. El Sistema devuelve un código **200 OK** con el cliente actualizado. El flujo
     retorna al paso 1.

* **3b. Consultar un cliente por nombre o teléfono (HTTP 200 OK):**
  1. La usuaria busca un cliente enviando `GET /api/clientes?buscar=<texto>`.
  2. La **Negocio** (`cliente_service.buscar_cliente()`) filtra por nombre, apellido o
     teléfono en la Persistencia.
  3. El Sistema devuelve un código **200 OK** con los clientes que coinciden (lista vacía
     si no hay coincidencias).

* **4a. Datos inválidos (HTTP 400 Bad Request):**
  1. Si en el paso 4 (o en 3a) `nombre`, `apellido` o `telefono` llegan vacíos, o
     `telefono` no respeta un formato numérico válido.
  2. La **Presentación** rechaza la petición por error de validación de esquema.
  3. El Sistema devuelve un código **400 Bad Request** detallando el campo inválido. Fin
     del caso de uso.

* **3c. Cliente inexistente (HTTP 404 Not Found):**
  1. Si en 3a el `id` del cliente enviado no existe en la Persistencia.
  2. La **Negocio** no encuentra la entidad correspondiente.
  3. El Sistema devuelve un código **404 Not Found**. Fin del caso de uso.

### 6. POSTCONDICIONES
- La información de los clientes queda registrada, actualizada o disponible para
  consulta.
- Los clientes registrados quedan disponibles para asociarlos a turnos (CU-03, CU-04) y
  para consultar su historial (CU-09).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Éxito al listar, buscar o modificar un cliente. |
| `201` | Created | Confirmación de persistencia exitosa de un nuevo cliente. |
| `400` | Bad Request | Datos inválidos (campos vacíos o teléfono con formato incorrecto). |
| `404` | Not Found | El cliente referenciado por `id` no existe. |

### Matriz de trazabilidad CU-08 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_cliente_service.py`) | Test integración (`tests/integration/test_clientes_api.py`) |
| --- | --- | --- | --- |
| Flujo principal (alta) | `201 Created` | `test_crear_cliente_datos_validos_persiste_cliente` | `test_post_clientes_datos_validos_devuelve_201` |
| Flujo principal (listar) | `200 OK` | `test_listar_clientes_retorna_listado` | `test_get_clientes_devuelve_200_con_listado` |
| 3a. Modificar cliente | `200 OK` | `test_actualizar_cliente_datos_validos_guarda_cambios` | `test_put_clientes_datos_validos_devuelve_200` |
| 3b. Consultar cliente | `200 OK` | `test_buscar_cliente_por_telefono_retorna_coincidencias` | `test_get_clientes_buscar_devuelve_200` |
| 4a. Datos inválidos | `400 Bad Request` | — (validación de esquema en Presentación) | `test_post_clientes_telefono_invalido_devuelve_400` |
| 3c. Cliente inexistente | `404 Not Found` | `test_actualizar_cliente_inexistente_lanza_excepcion` | `test_put_clientes_id_inexistente_devuelve_404` |
