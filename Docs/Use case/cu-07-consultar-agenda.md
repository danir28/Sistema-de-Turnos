# Caso de Uso: Consultar Agenda

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Sin reglas de negocio
> específicas asociadas. Este caso de uso es el punto donde se reflejan los efectos de
> CU-16 (Sincronizar Turno Web con Agenda): al consultar, ya se ven los turnos originados
> en la aplicación web de la clienta, porque ambas aplicaciones comparten la misma base de
> datos.

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-07 |
| **Nombre** | Consultar Agenda |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → visualizar su ocupación diaria/semanal para organizar el trabajo |
| **Disparador (Trigger)** | La usuaria solicita consultar la agenda |
| **Prioridad / Frecuencia** | Alta; uso diario (consulta frecuente) |
| **Reglas de negocio relacionadas** | No aplica |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña consultar los turnos registrados, organizados por día o por semana,
distinguiendo los horarios ocupados de los disponibles.

### 2. PRECONDICIONES
- La usuaria inició sesión correctamente (CU-01).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 200)
1. El Actor envía una petición `GET /api/agenda` (sin parámetros, vista de día actual por
   defecto).
2. La **Negocio** (`agenda_service.obtener_agenda()`) consulta la Persistencia (modelo
   `Turno`) y arma la lista de horarios ocupados y disponibles del día, incluyendo tanto
   los turnos registrados por la dueña (CU-03, CU-04) como los reservados por clientas
   desde la aplicación web (CU-12), ya que ambos se persisten sobre la misma tabla
   `turnos` (**incluye CU-16**).
3. El Sistema devuelve un código **200 OK** con la agenda del día.
4. La usuaria alterna entre la vista de día y de semana, o navega entre fechas, enviando
   `GET /api/agenda?vista=semana&fecha=<YYYY-MM-DD>`. El Sistema repite el paso 2-3 con
   el nuevo rango de fechas.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **1a. Parámetro de fecha inválido (HTTP 400 Bad Request):**
  1. Si en el paso 4 el parámetro `fecha` no respeta el formato `YYYY-MM-DD` o `vista` no
     es `dia` ni `semana`.
  2. La **Presentación** rechaza la petición por error de validación de esquema.
  3. El Sistema devuelve un código **400 Bad Request**. Fin del caso de uso.

### 6. POSTCONDICIONES
- La usuaria visualiza la información actualizada de la agenda para el período
  consultado, incluyendo los turnos reservados a través de la aplicación web.

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `200` | OK | Consulta exitosa de la agenda (día o semana). |
| `400` | Bad Request | Parámetro `fecha` o `vista` con formato inválido. |

### Matriz de trazabilidad CU-07 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_agenda_service.py`) | Test integración (`tests/integration/test_agenda_api.py`) |
| --- | --- | --- | --- |
| Flujo principal (día actual) | `200 OK` | `test_obtener_agenda_dia_actual_retorna_turnos_ocupados_y_libres` | `test_get_agenda_devuelve_200_con_agenda_del_dia` |
| Flujo principal (incluye turnos web) | `200 OK` | `test_obtener_agenda_incluye_turnos_reservados_desde_web` | `test_get_agenda_refleja_turno_reservado_por_clienta` |
| Alternar vista semana | `200 OK` | `test_obtener_agenda_vista_semana_retorna_rango_correcto` | `test_get_agenda_vista_semana_devuelve_200` |
| 1a. Parámetro inválido | `400 Bad Request` | — (validación de esquema en Presentación) | `test_get_agenda_fecha_invalida_devuelve_400` |
