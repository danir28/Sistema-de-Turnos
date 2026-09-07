# Caso de Uso: Iniciar Sesión

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Regla de negocio **RN-01**
> (usuario y contraseña obligatorios, contraseña almacenada de forma encriptada).

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-01 |
| **Nombre** | Iniciar Sesión |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → acceder de forma segura a la gestión de turnos, servicios y clientes; Sistema → garantizar que solo personal autorizado modifique la agenda |
| **Disparador (Trigger)** | La usuaria solicita ingresar al sistema |
| **Prioridad / Frecuencia** | Alta; uso diario (varias veces por jornada) |
| **Reglas de negocio relacionadas** | RN-01 (usuario y contraseña obligatorios; contraseña encriptada) |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña o profesional autenticarse en el sistema mediante usuario y
contraseña para acceder a la gestión de turnos, servicios y clientes.

### 2. PRECONDICIONES
- La usuaria posee un usuario y una contraseña registrados previamente en la tabla
  `usuarios`.

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200)
1. La usuaria ingresa su nombre de usuario y contraseña y el Actor envía una petición
   `POST /api/auth/login` con un JSON `{"username": ..., "password": ...}`.
2. La **Presentación** (`auth_bp`, ruta `login()`) valida que el JSON sea estructuralmente
   correcto y que ambos campos estén presentes.
3. La **Negocio** (`auth_service.autenticar_usuario()`) busca el `Usuario` por `username`
   en la **Persistencia** (modelo `Usuario`, SQLAlchemy) y compara la contraseña ingresada
   contra el hash almacenado con `check_password_hash` (**RN-01**).
4. El Sistema inicia la sesión del usuario (`flask_login.login_user()`) y devuelve un
   código **200 OK** con los datos básicos de la usuaria autenticada.
5. El frontend, ya autenticado, consulta la agenda del día mediante `GET /api/agenda`
   (incluye CU-07) para presentarla.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **2a. Campos incompletos (HTTP 400 Bad Request):**
  1. Si en el Paso 2 el JSON no incluye `username` o `password`, o alguno llega vacío.
  2. La **Presentación** rechaza la petición por error de validación de esquema.
  3. El Sistema devuelve un código **400 Bad Request** solicitando completar los datos
     faltantes. Fin del caso de uso.

* **3a. Credenciales incorrectas (HTTP 401 Unauthorized):**
  1. Si en el Paso 3 el `username` no existe o la contraseña no coincide con el hash
     almacenado.
  2. La **Negocio** no confirma la autenticación (por seguridad, no distingue si falló el
     usuario o la contraseña).
  3. El Sistema devuelve un código **401 Unauthorized** con el mensaje: "Usuario o
     contraseña incorrectos". El flujo retorna al paso 1. Fin del caso de uso.

### 6. POSTCONDICIONES
- La usuaria queda autenticada (sesión activa vía `flask_login`) y con acceso habilitado
  a las funcionalidades del sistema.

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Autenticación exitosa; el sistema no crea un recurso nuevo, solo abre sesión. |
| `400` | Bad Request | Falta `username` o `password` en el cuerpo de la petición. |
| `401` | Unauthorized | Las credenciales no coinciden con los registros (**RN-01**). |

### Nota: Validación vs. Verificación aplicada

- **Validación (Presentación, → 400):** presencia de `username` y `password` en el JSON
  recibido.
- **Verificación (Negocio, → 401):** existencia del usuario y coincidencia de la
  contraseña con el hash almacenado (`check_password_hash`), aplicando **RN-01**.

### Matriz de trazabilidad CU-01 → Test

> Convención de nombres: tests unitarios en `snake_case` con el patrón
> `test_<funcion>_<condicion>_<resultado>`; tests de integración con el patrón
> `test_<endpoint>_<condicion>_devuelve_<codigo>`.

| Paso del CU | Código | Test unitario (`tests/unit/test_auth_service.py`) | Test integración (`tests/integration/test_auth_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `200 OK` | `test_autenticar_usuario_credenciales_validas_retorna_usuario` | `test_login_con_credenciales_validas_devuelve_200` |
| 2a. Campos incompletos | `400 Bad Request` | — (validación de esquema en Presentación) | `test_login_sin_password_devuelve_400` |
| 3a. Credenciales incorrectas | `401 Unauthorized` | `test_autenticar_usuario_password_incorrecta_retorna_none` | `test_login_con_password_incorrecta_devuelve_401` |
