# Caso de Uso: Generar Copia de Seguridad

> Especificación elaborada siguiendo la guía
> `GUIA-Especificacion-Casos-de-Uso.md` (sección 3), tomando como referencia
> `CU-02 Alta de Medico.md`.
> Stack real del proyecto: **Flask** (Presentación, Blueprints) + funciones de servicio en
> **Python** (Negocio) + **SQLAlchemy/SQLite** (Persistencia). Sin reglas de negocio
> específicas asociadas.

| Campo | Valor |
| --- | --- |
| **ID del Caso de Uso** | CU-10 |
| **Nombre** | Generar Copia de Seguridad |
| **Actor Principal** | Dueña / Profesional |
| **Alcance / Nivel** | Sistema; meta de usuario |
| **Stakeholders e intereses** | Dueña/Profesional → prevenir la pérdida de información del negocio (turnos, clientes, servicios) |
| **Disparador (Trigger)** | La usuaria solicita generar una copia de seguridad |
| **Prioridad / Frecuencia** | Media; frecuencia baja (uso manual ocasional) |
| **Reglas de negocio relacionadas** | No aplica |

---

### 1. BREVE DESCRIPCIÓN
Permite a la dueña generar un respaldo de la información del sistema (base de datos
SQLite completa), para prevenir la pérdida de datos.

### 2. PRECONDICIONES
- La usuaria inició sesión correctamente (CU-01).

### 3. FLUJO PRINCIPAL (Camino Feliz - HTTP 201)
1. El Actor envía una petición `POST /api/backup`.
2. La **Negocio** (`backup_service.generar_backup()`) genera una copia del archivo de
   base de datos SQLite (por ejemplo, con `shutil.copyfile` sobre el archivo `.db`) con un
   nombre de archivo con marca de tiempo.
3. El Sistema almacena el archivo de respaldo en el directorio de backups configurado y
   devuelve un código **201 Created** con la referencia del respaldo generado (nombre de
   archivo y fecha/hora).

### 4. FLUJOS ALTERNATIVOS (Caminos Tristes / Excepciones)

* **2a. Error al generar el respaldo (HTTP 500 Internal Server Error):**
  1. Si en el paso 2 la copia falla (por ejemplo, falta de espacio en disco, permisos
     insuficientes o la base de datos está bloqueada por otra operación).
  2. La **Negocio** interrumpe la operación y registra el error como no controlado.
  3. El Sistema devuelve un código **500 Internal Server Error** informando que no pudo
     generar la copia de seguridad. El flujo finaliza sin cambios. Fin del caso de uso.

### 6. POSTCONDICIONES
- Queda disponible un respaldo actualizado de la información del sistema en el
  directorio de backups.

---

## Anexo: matrices de referencia

### Códigos HTTP usados

| Código HTTP | Nombre Técnico | Contexto de Aplicación en el Caso de Uso |
| --- | --- | --- |
| `201` | Created | Confirmación de que el archivo de respaldo se generó correctamente. |
| `500` | Internal Server Error | Fallo técnico no controlado durante la generación del respaldo. |

### Matriz de trazabilidad CU-10 → Test

| Paso del CU | Código | Test unitario (`tests/unit/test_backup_service.py`) | Test integración (`tests/integration/test_backup_api.py`) |
| --- | --- | --- | --- |
| Flujo principal | `201 Created` | `test_generar_backup_condiciones_normales_crea_archivo` | `test_post_backup_devuelve_201_con_referencia` |
| 2a. Error al generar el respaldo | `500 Internal Server Error` | `test_generar_backup_error_de_escritura_lanza_excepcion` | `test_post_backup_con_fallo_de_escritura_devuelve_500` |
