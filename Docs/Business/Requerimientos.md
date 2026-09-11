# Requerimientos - Peluquería

## 1. Alcance

### 1.1 Funcionalidades incluidas

- Gestión de servicios, turnos, clientes y agenda desde la aplicación de escritorio.
- Reserva y cancelación de turnos por parte de las clientas desde una aplicación web,
  sincronizada con la agenda del salón.
- Registro de inasistencias y de turnos sin reserva previa.
- Recordatorio de turno y respaldo de la base de datos.

> El objetivo de esta versión es contar con un sistema funcional que permita gestionar
> los turnos del salón de forma simple, y que las clientas puedan reservar por su cuenta
> sin depender de que la dueña esté disponible para atender el teléfono.

### 1.2 Limitaciones y exclusiones (fuera de alcance)

En esta entrega **NO** se incluirán las siguientes funcionalidades:

- Pagos online o cobro de seña al reservar.
- Notificaciones automáticas por WhatsApp, ya que requieren integración con WhatsApp.
- Aplicación móvil nativa (la web debe poder usarse bien desde el navegador del celular,
  pero no habrá app para descargar).
- Reportes o estadísticas avanzadas.
- Notificaciones automáticas, para la dueña.

> Estas funcionalidades podrán ser consideradas en futuras versiones del sistema.

## 2. Requerimientos Funcionales

### 2.1 Sistema de gestión de turnos — aplicación de escritorio (Local)

- **RF-01:** El sistema debe permitir dar de alta, modificar y eliminar los servicios
  que ofrece la peluquería, registrando nombre, duración estimada en minutos y precio.
- **RF-02:** El sistema debe permitir registrar un turno, asociándolo a un cliente, a
  uno o más servicios y a una fecha y hora de inicio.
- **RF-03:** El sistema debe calcular automáticamente la duración total de un turno
  sumando la duración estimada de cada servicio seleccionado, y determinar la hora de
  finalización.
- **RF-04:** El sistema debe impedir el registro de un turno cuyo horario se superponga
  con el de otro turno ya existente.
- **RF-05:** El sistema debe permitir modificar la fecha o horario de un turno ya
  registrado.
- **RF-06:** El sistema debe permitir cancelar un turno, dejando el horario disponible
  para ser ocupado por otro cliente.
- **RF-07:** El sistema debe permitir marcar un turno como "no asistido" cuando el
  cliente no se presenta.
- **RF-08:** El sistema debe permitir registrar un turno para un cliente que llega sin
  haber reservado, siempre que el horario esté disponible.
- **RF-09:** El sistema debe mostrar la agenda de turnos organizada por día y por
  semana, distinguiendo los horarios ocupados de los disponibles, incluyendo los turnos
  reservados desde la aplicación web.
- **RF-10:** El sistema debe permitir registrar, modificar y consultar los datos de cada
  cliente (nombre, apellido y teléfono).
- **RF-11:** El sistema debe mantener un historial de los servicios realizados a cada
  cliente en base a sus turnos anteriores. *(Ver con pato)*
- **RF-12:** El sistema debe permitir enviar un recordatorio al cliente antes de la
  fecha del turno.
- **RF-13:** El sistema debe requerir usuario y contraseña para acceder a sus
  funcionalidades.
- **RF-14:** El sistema debe permitir generar copias de seguridad de la base de datos.
- **RF-15:** El sistema debe permitir modificar un servicio de un turno ya registrado.
  *(Misma lógica que RF-05)*

### 2.2 Sistema de reserva de turnos — aplicación web (clienta)

- **RF-16:** El sistema debe mostrar los horarios disponibles en base a los servicios
  seleccionados y a su duración total.
- **RF-17:** El sistema debe permitir seleccionar uno o más servicios antes de reservar,
  mostrando la duración y el precio estimado.
- **RF-18:** El sistema debe permitir reservar un turno en un horario disponible,
  solicitando nombre, apellido y teléfono de contacto.
- **RF-19:** El sistema debe permitir cancelar un turno propio desde la aplicación web,
  con una antelación mínima de 24 horas.
- **RF-20:** Todo turno reservado desde la aplicación web debe reflejarse
  automáticamente en la agenda que utiliza la dueña, sin necesidad de carga manual.

### 2.3 Reglas de Negocio

> Reglas derivadas durante la especificación de los Casos de Uso (`Docs/Use case/`),
> consolidadas acá para tener un catálogo único.

| ID | Regla | RF relacionado | CU donde se aplica |
| --- | --- | --- | --- |
| **RN-01** | Usuario y contraseña son obligatorios para iniciar sesión; la contraseña se almacena encriptada (hash). | RF-13 | CU-01 |
| **RN-02** | Un servicio no puede eliminarse si está asociado a turnos futuros en estado "reservado". | RF-01 | CU-02 |
| **RN-03** | No pueden coexistir dos turnos con horarios superpuestos. | RF-04 | CU-03, CU-04, CU-12, CU-15 |
| **RN-04** | La duración total de un turno es la suma de la duración estimada de cada servicio asociado. | RF-03 | CU-03, CU-04, CU-11, CU-14 |
| **RN-05** | Un turno solo puede marcarse como "no asistido" si su fecha/hora ya transcurrió. | RF-07 | CU-06 |
| **RN-06** | Una clienta solo puede cancelar su propia reserva con una antelación mínima de 24 horas. | RF-19 | CU-13 |

## 3. Requerimientos No Funcionales

- **RNF-01 (Arquitectura):** El sistema debe implementarse con una arquitectura en
  capas, con una API REST en Flask como backend centralizado, consumida tanto por la
  vista de la dueña como por la vista de la clienta, evitando duplicar la lógica de
  negocio entre ambas.
- **RNF-02 (Tecnología):** El backend debe desarrollarse en Python, utilizando
  SQLAlchemy como ORM para el acceso a datos, separando la lógica de negocio de las
  rutas/endpoints.
- **RNF-03 (Base de datos):** La persistencia de la información debe realizarse en
  SQLite.
- **RNF-04 (Plataforma — dueña):** La aplicación de la dueña debe implementarse como una
  aplicación web con acceso restringido (login), accesible desde el navegador de su
  tablet.
- **RNF-05 (Plataforma — clienta):** La aplicación de la clienta debe ser una aplicación
  web accesible desde cualquier navegador, sin necesidad de instalación.
- **RNF-06 (Conectividad):** Dado que ambas vistas consumen la misma API centralizada,
  tanto la vista de la dueña como la de la clienta requieren conexión a internet para
  funcionar.
- **RNF-07 (Hosting):** La API y la base de datos deben alojarse en un servidor
  accesible desde internet, para que la aplicación web pueda ser utilizada por las
  clientas fuera del local.
- **RNF-08 (Usabilidad — dueña):** La interfaz de la vista de la dueña debe ser simple e
  intuitiva, pensada para una usuaria sin conocimientos técnicos que la utiliza
  principalmente desde una tablet.
- **RNF-09 (Usabilidad — clienta):** La interfaz web debe ser simple de usar desde un
  celular, dado que es el dispositivo más probable desde el que las clientas van a
  reservar.
- **RNF-10 (Rendimiento):** Las operaciones habituales (registrar un turno, consultar la
  agenda, reservar desde la web) deben responder en no más de 2 segundos.
- **RNF-11 (Seguridad — acceso):** Las contraseñas de acceso a la vista de la dueña
  deben almacenarse encriptadas (hash), nunca en texto plano.
- **RNF-12 (Seguridad — web):** Los datos ingresados desde la aplicación web deben
  validarse antes de guardarse, para evitar registros inválidos.
- **RNF-13 (Concurrencia):** El sistema debe garantizar que dos reservas simultáneas no
  puedan ocupar el mismo horario.
- **RNF-14 (Integridad de datos):** Las operaciones que afecten a más de una entidad
  relacionada (por ejemplo, un turno con varios servicios) deben garantizarse de forma
  consistente, evitando que queden datos guardados a medias.
