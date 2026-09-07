# Caso de Uso: Enviar Recordatorio de Turno

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** + **Python** (Negocio, job programado con
> **APScheduler** / `Flask-APScheduler`) + **SQLAlchemy/SQLite** (Persistencia). Sin
> reglas de negocio específicas asociadas.
>
> **Nota de alcance:** este caso de uso **no aplica ningún código HTTP**, en ningún punto
> de su flujo. A diferencia de CU-14, CU-15 y CU-16 (que son invocados internamente por
> otro caso de uso con origen HTTP), CU-17 lo dispara un **reloj/scheduler** —no hay
> ninguna petición HTTP entrante involucrada en absoluto—.

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-17 |
| **Nombre** | Enviar Recordatorio de Turno |
| **Actor Principal** | Sistema (disparador de tiempo) |
| **Alcance / Nivel** | Sistema; subfunción (job programado, sin origen HTTP) |
| **Stakeholders e intereses** | Clienta → recibir un aviso a tiempo y no olvidar su turno; Dueña/Profesional → reducir el ausentismo (CU-06) |
| **Disparador (Trigger)** | Se alcanza el momento programado de antelación respecto a un turno reservado (job periódico) |
| **Prioridad / Frecuencia** | Media; se ejecuta en un job periódico (por ejemplo, cada 15-30 minutos) |
| **Reglas de negocio relacionadas** | No aplica |

---

### 1. BREVE DESCRIPCIÓN
Envía un recordatorio al cliente antes de la fecha de su turno, mediante un job
programado que se ejecuta de forma periódica e independiente de cualquier petición HTTP.

### 2. PRECONDICIONES
- Existe un turno reservado con fecha futura.
- El job periódico (`recordatorio_service.enviar_recordatorios_pendientes()`,
  registrado con `Flask-APScheduler`) está activo en el proceso del backend.

### 3. FLUJO PRINCIPAL (Camino Feliz)
1. El scheduler dispara la función `enviar_recordatorios_pendientes()` según su
   configuración periódica (no hay Actor externo ni petición HTTP en este paso).
2. La función consulta en la Persistencia los `Turno` en estado `"reservado"` cuya
   `fecha_hora_inicio` está dentro de la ventana de antelación configurada (por ejemplo,
   entre 23 y 24 horas antes del turno) y que todavía no tienen un recordatorio enviado.
3. Para cada turno encontrado, la función envía un recordatorio al cliente con los datos
   del turno (fecha, hora y servicios), a través del canal configurado (por ejemplo,
   WhatsApp o email mediante un servicio externo).
4. La función marca el turno como "recordatorio enviado" (por ejemplo, un campo
   `recordatorio_enviado_en` en el modelo `Turno`) para no reenviarlo en la próxima
   ejecución del job.

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **3a. Falla el envío del recordatorio:**
  1. Si en el paso 3 el canal de envío (WhatsApp/email) falla (por ejemplo, error de red
     o del proveedor externo).
  2. La función registra la falla en el log de la aplicación (`logging`), sin afectar la
     validez del turno ni interrumpir el procesamiento de los demás turnos pendientes de
     esa ejecución.
  3. El turno **no** se marca como "recordatorio enviado", por lo que se reintentará en
     la siguiente ejecución del job. Fin del caso de uso para ese turno puntual.

### 6. POSTCONDICIONES
- El cliente recibe un recordatorio del turno próximo a ocurrir (o el turno queda
  pendiente de reintento si falló el envío).

---

## Anexo: matrices de referencia

### Códigos HTTP usados

Este caso de uso **no aplica ningún código HTTP**: no hay petición HTTP entrante en
ningún punto del flujo (el disparador es un reloj/scheduler interno del proceso backend,
no un Actor externo). Su resultado (envío exitoso o falla registrada) se verifica por
logs y por el estado del campo `recordatorio_enviado_en` del turno, no por una respuesta
HTTP.

### Matriz de trazabilidad CU-17 → Test

| Paso del CU | Resultado | Test unitario (`tests/unit/test_recordatorio_service.py`) | Test integración |
| --- | --- | --- | --- |
| Flujo principal | Recordatorio enviado y turno marcado | `test_enviar_recordatorios_pendientes_turno_en_ventana_envia_y_marca` | `test_job_recordatorios_turno_proximo_envia_recordatorio` (test de integración del job, no de un endpoint HTTP) |
| 3a. Falla el envío | Falla registrada, turno no marcado (reintentable) | `test_enviar_recordatorios_pendientes_fallo_de_canal_no_marca_turno` | `test_job_recordatorios_fallo_de_envio_permite_reintento` |

> Al no exponer un endpoint HTTP, el "test de integración" de este CU ejecuta el job
> completo (`Flask-APScheduler` disparando la función contra una base SQLite de prueba)
> en lugar de usar el cliente de pruebas HTTP de Flask.
