# Numi CRM — Consola de plataforma

*Gestión multiempresa para el equipo de Numi · Requisitos funcionales y vistas · Anexo al documento maestro · Versión 1.1 · 14 de octubre de 2026*

## 1. Propósito

La consola de plataforma es el área interna desde la que el equipo de Numi administra todas las organizaciones (negocios clientes) de Numi CRM. Resuelve tres necesidades:

1. **Dar de alta un negocio** y generar el primer acceso para que su administrador entre al CRM.
2. **Gestionar el ciclo de vida** de cada organización: datos, configuración, conexión de WhatsApp, usuarios y estado (activa o suspendida).
3. **Dar soporte y seguir el uso** de cada organización sin depender de scripts ni de la base de datos.

Este documento es un anexo del *Documento maestro del MVP v1.0*. Usa la misma identidad visual y el mismo design system. Los requisitos usan el prefijo **RF-PLT** y las vistas el prefijo **P**.

## 2. Decisiones que dan forma a la consola

| # | Decisión | Por qué |
|---|---|---|
| 1 | **Misma aplicación, área separada** (ruta `/plataforma`), con su propio menú y un indicador visual permanente de "Consola de plataforma" | Un solo código y un solo despliegue; el indicador evita confundir la consola con el CRM de un cliente |
| 2 | **Rol de plataforma independiente** de las organizaciones: el operador de Numi no es miembro de cada negocio | Separa el poder de administrar la plataforma del acceso a los datos de un negocio |
| 3 | **El primer acceso es una invitación, no una contraseña.** La consola crea al administrador del negocio y le envía un enlace para que defina su propia contraseña. Si el correo no llega, el operador genera un enlace nuevo, lo copia y lo envía por otro medio; el enlace anterior deja de funcionar | Numi nunca conoce ni transmite contraseñas de los clientes, y ningún enlace de acceso queda guardado para volver a mostrarse |
| 4 | **Sin acceso al contenido de las conversaciones.** La consola muestra datos de uso y de salud (cantidades, fechas, errores), nunca el texto de los mensajes ni datos de los contactos | Los chats de clínicas contienen datos sensibles; el soporte no necesita leerlos |
| 5 | **Dos estados controlados por Numi: Activa y Suspendida.** La suspensión bloquea el acceso, pero los mensajes de WhatsApp se siguen recibiendo y guardando | Suspender por un tema comercial no debe hacer perder mensajes de los clientes del negocio |
| 6 | **Preparación visible como lista de verificación** (administrador invitado, invitación aceptada, WhatsApp conectado, primer mensaje recibido) | Muestra de un vistazo qué falta para que un negocio pueda operar |
| 7 | **Toda acción del operador queda registrada** (quién, qué, cuándo, sobre qué organización) | Trazabilidad de las acciones con más poder del sistema |
| 8 | **Acceso de operadores con doble factor** (código de una aplicación autenticadora). **Se resuelve con el TOTP de Supabase Auth —confirmado disponible y gratuito (ADR-007)—, no con un flujo propio de generación de secretos y códigos QR.** La vista P02 conserva su diseño; detrás llama al proveedor | La cuenta de un operador da control sobre todos los negocios |
| 9 | **Los operadores se crean por fuera de la interfaz** (son dos personas: Franco y Anthony) | Evita construir una gestión de operadores que hoy no se necesita |
| 10 | **Dos métodos de conexión de WhatsApp:** registro manual por el operador (P0) y registro integrado de Meta, en el que el negocio conecta su propio número (P1). La vista P24 se diseña para ambos. Si se adopta la coexistencia (el negocio sigue usando la app de WhatsApp Business en el mismo número), el registro integrado de Meta pasa a P0 | El método principal depende del modelo de conexión que se adopte; el diseño no debe bloquearse por esa decisión |

## 3. Usuarios de la consola

| Rol | Quién | Qué puede hacer |
|---|---|---|
| **Operador de plataforma** | Franco y Anthony | Todo lo de este documento sobre todas las organizaciones |

Los roles del CRM (Asesor y Administrador) no tienen acceso a la consola. Un operador puede, además, ser usuario normal de una organización de prueba (por ejemplo, una organización de demostración de Numi); en ese caso entra al CRM como cualquier otro usuario.

## 4. Conceptos

| Concepto | Definición |
|---|---|
| Organización | Negocio cliente de Numi CRM. Límite de aislamiento de los datos |
| Tipo de cuenta | **Piloto** o **Cliente**. Es informativo en el MVP: no cambia funciones ni límites |
| Estado de la organización | **Activa:** sus usuarios entran y operan. **Suspendida:** nadie de la organización puede entrar ni enviar mensajes; los mensajes entrantes se siguen guardando |
| Preparación | Lista de verificación derivada del sistema: administrador invitado, invitación aceptada, WhatsApp conectado, primer mensaje recibido. Cuando todo está completo, la organización está *Lista para operar* |
| Salud de la conexión | Estado de la integración con WhatsApp: conectada, sin configurar, con error. Incluye la fecha del último evento recibido de Meta |
| Método de conexión | Cómo se conectó el número de WhatsApp: **Manual** (el operador registra los identificadores y el token) o **Registro integrado de Meta** (el negocio conecta su número desde el flujo de Meta) |
| Uso | Indicadores sin contenido: usuarios activos, último acceso, mensajes recibidos y enviados, conversaciones sin responder |
| Usuario activo | Usuario con al menos un ingreso al CRM en el período |
| Registro de auditoría | Lista de acciones de operadores sobre la plataforma |

## 5. Requisitos funcionales

La columna "Prioridad" indica qué es indispensable para el MVP (P0) y qué puede llegar después sin afectar la operación (P1).

### 5.1 Acceso de operadores (PLT-ACC)

| ID | Requisito | Vista | Prioridad |
|---|---|---|---|
| RF-PLT-ACC-01 | El operador inicia sesión con correo y contraseña en la entrada de la consola | P01 | P0 |
| RF-PLT-ACC-02 | Después de la contraseña, el operador confirma con un código de seis dígitos de su aplicación autenticadora | P02 | P0 |
| RF-PLT-ACC-03 | El primer ingreso de un operador le pide configurar el doble factor con un código QR | P02 | P0 |
| RF-PLT-ACC-04 | Solo los usuarios con rol de plataforma pueden abrir cualquier ruta de la consola; los demás ven la pantalla de acceso denegado | P01, V60 | P0 |
| RF-PLT-ACC-05 | La consola muestra siempre un indicador de que el usuario está en la consola de plataforma, y un acceso para ir al CRM si el operador también pertenece a una organización | L02 | P0 |

### 5.2 Organizaciones (PLT-ORG)

| ID | Requisito | Vista | Prioridad |
|---|---|---|---|
| RF-PLT-ORG-01 | El operador ve la lista de organizaciones con: nombre, sector, ciudad, tipo de cuenta, estado, preparación, salud de WhatsApp, cantidad de usuarios y fecha del último mensaje recibido | P20 | P0 |
| RF-PLT-ORG-02 | La lista se busca por nombre, NIT o correo del contacto principal | P20 | P0 |
| RF-PLT-ORG-03 | La lista se filtra por estado, tipo de cuenta, sector, preparación y salud de WhatsApp | P20 | P0 |
| RF-PLT-ORG-04 | La lista se ordena por nombre, fecha de creación o último mensaje recibido | P20 | P0 |
| RF-PLT-ORG-05 | El operador crea una organización con un asistente de tres pasos: datos del negocio, primer administrador y revisión. La conexión de WhatsApp se puede hacer al final o después | P21 | P0 |
| RF-PLT-ORG-06 | Al crear la organización, el sistema crea al primer administrador y le envía la invitación por correo | P21 | P0 |
| RF-PLT-ORG-07 | El nombre del negocio es obligatorio; el sistema avisa si ya existe una organización con el mismo nombre o NIT | P21 | P0 |
| RF-PLT-ORG-08 | El operador ve el detalle de una organización: estado, preparación, datos, uso y accesos a usuarios, WhatsApp, configuración y actividad | P22 | P0 |
| RF-PLT-ORG-09 | El operador edita los datos de la organización: nombre, razón social, NIT, sector, ciudad, zona horaria y contacto principal (nombre, correo, teléfono) | P26 | P0 |
| RF-PLT-ORG-10 | El operador define el tipo de cuenta (Piloto o Cliente) y, si es piloto, las fechas de inicio y fin del piloto | P26 | P0 |
| RF-PLT-ORG-11 | El operador escribe notas internas sobre la organización, visibles solo en la consola | P26 | P0 |
| RF-PLT-ORG-12 | El operador define el valor inicial de la asignación automática de la organización. El administrador del negocio lo puede cambiar después desde su configuración | P26 | P1 |

### 5.3 Estado de la organización (PLT-EST)

| ID | Requisito | Vista | Prioridad |
|---|---|---|---|
| RF-PLT-EST-01 | Una organización nueva queda **Activa** | P21 | P0 |
| RF-PLT-EST-02 | El operador **suspende** una organización indicando un motivo (pago pendiente, fin del piloto, solicitud del cliente, otro) | P27 | P0 |
| RF-PLT-EST-03 | Una organización suspendida: sus usuarios no pueden entrar y ven un mensaje que indica contactar a Numi; las sesiones abiertas se cierran en la siguiente acción; no se pueden enviar mensajes; los mensajes entrantes se siguen recibiendo y guardando | V05, P27 | P0 |
| RF-PLT-EST-04 | El operador **reactiva** una organización suspendida y todo vuelve a funcionar como antes | P27 | P0 |
| RF-PLT-EST-05 | Suspender y reactivar piden confirmación, explican el efecto y quedan en el registro de auditoría | P27 | P0 |
| RF-PLT-EST-06 | La organización muestra cuándo y por qué fue suspendida, y por quién | P22 | P0 |

### 5.4 Usuarios de una organización (PLT-USR)

| ID | Requisito | Vista | Prioridad |
|---|---|---|---|
**P23 es una vista de soporte, no la vía normal de alta de usuarios** (ADR-002). El operador crea e invita al **primer administrador** de cada negocio (`RF-PLT-ORG-06`); a partir de ahí, los asesores los invita y gestiona el administrador desde el CRM (`RF-USR-*`). P23 existe para desatascar a un negocio cuyo administrador perdió el acceso, reenviar una invitación que no llegó o corregir un rol mal asignado. Su diseño no cambia; lo que cambia es que no es el camino por el que entran los asesores — y eso vuelve `RF-USR-*` del CRM **irrecortable**.

| RF-PLT-USR-01 | El operador ve los usuarios de una organización: nombre, correo, rol, estado (activo, invitación pendiente, invitación vencida, desactivado) y último acceso | P23 | P0 |
| RF-PLT-USR-02 | El operador invita usuarios a la organización por correo, con rol Administrador o Asesor | P23 | P0 |
| RF-PLT-USR-03 | El operador reenvía una invitación pendiente o vencida (genera un enlace nuevo e invalida el anterior) | P23 | P0 |
| RF-PLT-USR-04 | El operador genera un enlace de invitación nuevo y lo copia para enviarlo por otro medio. El enlace anterior deja de funcionar y el sistema no guarda enlaces para volver a mostrarlos | P23 | P0 |
| RF-PLT-USR-05 | El operador envía a un usuario un correo para restablecer su contraseña | P23 | P0 |
| RF-PLT-USR-06 | El operador cambia el rol de un usuario y lo desactiva o reactiva. Al desactivar a un asesor, sus conversaciones abiertas quedan sin asignar. Siempre debe quedar al menos un administrador activo en la organización | P23 | P0 |
| RF-PLT-USR-07 | Los enlaces de invitación vencen a los 7 días y se pueden usar una sola vez | P23 | P0 |

### 5.5 Conexión de WhatsApp (PLT-WA)

| ID | Requisito | Vista | Prioridad |
|---|---|---|---|
| RF-PLT-WA-01 | El operador ve la conexión de WhatsApp de la organización: método de conexión, número visible, nombre verificado, identificadores de Meta (número y cuenta de WhatsApp Business), estado y fecha de conexión | P24 | P0 |
| RF-PLT-WA-02 | El operador registra o actualiza la conexión de forma manual: identificador del número, identificador de la cuenta de WhatsApp Business y token de acceso | P24 | P0 |
| RF-PLT-WA-03 | El token se puede escribir pero nunca se vuelve a mostrar; la vista solo indica que está guardado y cuándo se actualizó | P24 | P0 |
| RF-PLT-WA-04 | El operador prueba la conexión: el sistema consulta a Meta y muestra si el número responde y si el webhook está suscrito | P24 | P0 |
| RF-PLT-WA-05 | La vista muestra la salud del webhook: último evento recibido, eventos de las últimas 24 horas y errores recientes (fecha y descripción técnica, sin contenido de mensajes ni números de teléfono de los contactos) | P24 | P0 |
| RF-PLT-WA-06 | El operador desconecta el número: el sistema borra el token y deja de procesar eventos de ese número | P24 | P0 |
| RF-PLT-WA-07 | Un mismo número no puede estar conectado a dos organizaciones | P24 | P0 |
| RF-PLT-WA-08 | La organización puede conectar su número mediante el registro integrado de Meta (Embedded Signup, **versión vigente: la v2 quedó obsoleta el 2026-10-15**). La consola muestra el resultado con la misma información que una conexión manual. **Sube a P0 al obtener el rol de Tech Provider**, que ADR-001 fijó como modelo objetivo | P24 | P1 |

### 5.6 Uso y seguimiento (PLT-USO)

| ID | Requisito | Vista | Prioridad |
|---|---|---|---|
| RF-PLT-USO-01 | El detalle de la organización muestra el uso de los últimos 7 días: usuarios activos (con al menos un ingreso en el período), mensajes recibidos, mensajes enviados, conversaciones sin responder en este momento y último acceso de un usuario | P22 | P0 |
| RF-PLT-USO-02 | El uso se muestra por día en un gráfico simple de los últimos 14 días (mensajes recibidos y enviados) | P22 | P1 |
| RF-PLT-USO-03 | El panel general muestra: organizaciones por estado, organizaciones en piloto con su fecha de fin, organizaciones con WhatsApp con error, organizaciones sin mensajes en los últimos 3 días y mensajes totales de las últimas 24 horas | P10 | P1 |
| RF-PLT-USO-04 | Ningún indicador de la consola muestra texto de mensajes, nombres ni teléfonos de los contactos del negocio | Todas | P0 |

### 5.7 Auditoría (PLT-AUD)

| ID | Requisito | Vista | Prioridad |
|---|---|---|---|
| RF-PLT-AUD-01 | El sistema registra cada acción de un operador: crear organización, editar datos, suspender, reactivar, invitar, reenviar, generar enlace, restablecer contraseña, cambiar rol, desactivar usuario, conectar, probar y desconectar WhatsApp | — | P0 |
| RF-PLT-AUD-02 | Cada registro guarda: fecha y hora, operador, organización, acción, detalle (valores antes y después cuando aplique) | — | P0 |
| RF-PLT-AUD-03 | El detalle de la organización muestra su actividad: las acciones de operadores sobre ella, de la más reciente a la más antigua | P25 | P0 |
| RF-PLT-AUD-04 | La consola tiene un registro global de auditoría con filtros por operador, organización, acción y fecha | P30 | P1 |
| RF-PLT-AUD-05 | Los registros de auditoría no se pueden editar ni borrar desde la interfaz | — | P0 |

### 5.8 Seguridad (PLT-SEG)

| ID | Requisito | Prioridad |
|---|---|---|
| RF-PLT-SEG-01 | Las acciones de la consola se ejecutan en el servidor, que verifica el rol de plataforma en cada petición. El navegador nunca recibe permisos para leer datos de todas las organizaciones | P0 |
| RF-PLT-SEG-02 | Los tokens de WhatsApp se guardan cifrados y nunca aparecen en la interfaz, en los registros técnicos ni en la auditoría | P0 |
| RF-PLT-SEG-03 | La sesión del operador expira tras 12 horas, o tras 30 minutos sin actividad | P0 |
| RF-PLT-SEG-04 | Suspender una organización y desconectar su WhatsApp piden confirmación escribiendo el nombre de la organización | P0 |
| RF-PLT-SEG-05 | Los correos de invitación y de restablecimiento de contraseña salen desde un remitente propio de Numi, con entrega confiable para cualquier destinatario | P0 |

## 6. Inventario de vistas

### 6.1 Mapa general

| ID | Vista | Ruta | Propósito | Prioridad |
|---|---|---|---|---|
| P01 | Entrada a la consola | /plataforma/entrar | Iniciar sesión como operador | P0 |
| P02 | Doble factor | /plataforma/verificar | Configurar o confirmar el código de seis dígitos | P0 |
| L02 | Estructura de la consola | /plataforma/* | Navegación y marco común de la consola | P0 |
| P10 | Panel general | /plataforma | Estado de toda la plataforma de un vistazo | P1 |
| P20 | Lista de organizaciones | /plataforma/organizaciones | Buscar, filtrar y abrir organizaciones | P0 |
| P21 | Crear organización | /plataforma/organizaciones/nueva | Alta del negocio y su primer administrador (asistente a pantalla completa) | P0 |
| P22 | Organización: resumen | /plataforma/organizaciones/:id | Estado, preparación, datos clave y uso | P0 |
| P23 | Organización: usuarios | …/:id/usuarios | Accesos de la organización | P0 |
| P24 | Organización: WhatsApp | …/:id/whatsapp | Conexión, prueba y salud del webhook | P0 |
| P25 | Organización: actividad | …/:id/actividad | Auditoría de la organización | P0 |
| P26 | Organización: configuración | …/:id/configuracion | Datos, tipo de cuenta, piloto y notas | P0 |
| P27 | Cambiar estado | Ventana de confirmación | Suspender o reactivar | P0 |
| P30 | Auditoría global | /plataforma/auditoria | Todas las acciones de operadores | P1 |

Se reutilizan del CRM: **V03 y V04** (recuperar y restablecer contraseña), **V05** (con un texto propio para organización suspendida) y **V60** (estados del sistema).

Si el tiempo aprieta, la consola funciona completa sin P10 y P30: la lista de organizaciones (P20) y la actividad de cada organización (P25) cubren lo indispensable.

### 6.2 L02 — Estructura de la consola

- **Barra lateral:** Panel general (P1), Organizaciones, Auditoría (P1). Al pie, el menú del operador con nombre, *Ir al CRM* (si pertenece a una organización) y *Cerrar sesión*.
- **Indicador de consola:** franja o etiqueta permanente "Consola de plataforma · Numi" con un tratamiento visual distinto al del CRM (por ejemplo, el violeta profundo de la marca como fondo de la barra lateral), para que nunca se confunda con el CRM de un cliente.
- **Encabezado:** ruta de navegación (por ejemplo, *Organizaciones / Clínica Aurora / WhatsApp*).
- **Página de inicio:** mientras el panel general (P10) no exista, la ruta `/plataforma` abre la lista de organizaciones (P20).

### 6.3 P01 y P02 — Entrada y doble factor

- **P01:** logotipo de Numi, etiqueta "Consola de plataforma", correo, contraseña y botón *Entrar*. Estados: credenciales incorrectas, usuario sin rol de plataforma (lleva a acceso denegado), cargando.
- **P02, primer ingreso:** explicación corta, código QR para la aplicación autenticadora, clave en texto para copiar, campo para el primer código y botón *Activar*.
- **P02, ingresos siguientes:** campo de seis dígitos con avance automático y botón *Verificar*. Estados: código incorrecto, demasiados intentos (espera indicada).

### 6.4 P10 — Panel general (P1)

- **Tarjetas:** organizaciones activas, suspendidas, en piloto y con WhatsApp con error; mensajes de las últimas 24 horas en toda la plataforma.
- **Listas de atención:**
  - *Requieren acción:* WhatsApp con error, preparación incompleta hace más de 3 días, invitaciones del primer administrador vencidas.
  - *Pilotos:* organizaciones en piloto con días restantes, ordenadas por fecha de fin.
  - *Sin actividad:* organizaciones activas sin mensajes en los últimos 3 días.
- Cada elemento abre el detalle de la organización.

### 6.5 P20 — Lista de organizaciones

- **Barra superior:** título, buscador (nombre, NIT, correo del contacto principal), botón *Nueva organización*.
- **Filtros en chips:** estado, tipo de cuenta, sector, preparación (lista para operar, incompleta), salud de WhatsApp.
- **Tabla:** nombre (con ciudad debajo), sector, tipo de cuenta (con días restantes si es piloto), estado (etiqueta), preparación (por ejemplo "3 de 4"), salud de WhatsApp (punto de color con texto), usuarios, último mensaje recibido (tiempo relativo), fecha de creación. Fila clicable que abre P22.
- **Pie:** paginación y total.
- **Estados:** carga, sin organizaciones (con botón para crear la primera), sin resultados, error.

### 6.6 P21 — Crear organización (asistente)

Asistente de tres pasos con indicador de progreso arriba y botones *Atrás*, *Siguiente* y *Cancelar*.

1. **Datos del negocio:** nombre comercial (obligatorio), razón social, NIT, sector (clínica estética, inmobiliaria, odontología, veterinaria, otro), ciudad, zona horaria (Bogotá por defecto), tipo de cuenta (Piloto o Cliente) y, si es piloto, fechas de inicio y fin. Contacto principal: nombre, correo y teléfono.
2. **Primer administrador:** nombre y correo. Casilla *Usar los datos del contacto principal*. Texto que explica que recibirá un correo para crear su contraseña.
3. **Revisión:** resumen de todo lo ingresado con enlaces para editar cada bloque, y botón *Crear organización*.

**Pantalla final:** confirmación "Organización creada", estado de la invitación (enviada a {correo}), botón *Copiar enlace de invitación*, y dos acciones: *Conectar WhatsApp ahora* (lleva a P24) e *Ir a la organización* (lleva a P22).

**Validaciones:** nombre obligatorio, correos válidos, NIT con formato válido si se ingresa, aviso de posible duplicado por nombre o NIT (permite continuar), fecha de fin del piloto posterior a la de inicio.

### 6.7 P22 — Organización: resumen

- **Encabezado de la organización** (común a P22–P26): nombre, sector y ciudad, etiquetas de estado y de tipo de cuenta, y botón de acción de estado (*Suspender* o *Reactivar*). Debajo, pestañas: Resumen, Usuarios, WhatsApp, Actividad, Configuración.
- **Banner de suspensión** (si aplica): "Suspendida desde {fecha} por {operador}. Motivo: {motivo}", con botón *Reactivar*.
- **Tarjeta de preparación:** lista de verificación con cuatro puntos, cada uno con su estado y un acceso directo para resolverlo:
  1. Administrador invitado.
  2. Invitación aceptada (con *Reenviar* o *Generar enlace* si está pendiente).
  3. WhatsApp conectado (con *Conectar*).
  4. Primer mensaje recibido.
- **Tarjeta de uso (últimos 7 días):** usuarios activos sobre total, mensajes recibidos, mensajes enviados, conversaciones sin responder ahora, último acceso de un usuario. En P1, gráfico de los últimos 14 días.
- **Tarjeta de datos clave:** contacto principal (nombre, correo, teléfono con acción de copiar), NIT, fechas del piloto con días restantes, fecha de creación.
- **Notas internas:** extracto de las notas con acceso a editarlas en P26.

### 6.8 P23 — Organización: usuarios

- **Barra superior:** total de usuarios y botón *Invitar usuario* (ventana modal con correo, nombre y rol).
- **Tabla:** nombre, correo, rol (etiqueta), estado (activo, invitación pendiente, invitación vencida, desactivado), último acceso.
- **Acciones por fila (menú):**
  - Invitación pendiente o vencida: *Reenviar invitación*, *Generar y copiar enlace*, *Cancelar invitación*.
  - Usuario activo: *Enviar restablecimiento de contraseña*, *Cambiar rol*, *Desactivar*.
  - Usuario desactivado: *Reactivar*.
- **Reglas visibles:** el menú no permite desactivar ni cambiar el rol del último administrador activo, y explica por qué.
- **Confirmaciones:** aviso temporal tras reenviar, generar enlace o enviar restablecimiento; al desactivar, la confirmación indica cuántas conversaciones abiertas quedarán sin asignar.

### 6.9 P24 — Organización: WhatsApp

- **Tarjeta de estado:** estado de la conexión (conectada, sin configurar, con error) con color y texto, método de conexión, número visible, nombre verificado, fecha de conexión, botón *Probar conexión*.
- **Formulario de conexión manual:** identificador del número, identificador de la cuenta de WhatsApp Business y token de acceso (campo de solo escritura con el texto "Guardado el {fecha}" cuando ya existe). Botones *Guardar* y, si hay conexión, *Desconectar*.
- **Resultado de la prueba:** lista de verificaciones con resultado: el token es válido, el número responde, el webhook está suscrito. Cada una con un mensaje claro si falla.
- **Salud del webhook:** último evento recibido (tiempo relativo y fecha), eventos de las últimas 24 horas, y tabla de errores recientes (fecha, tipo, descripción técnica).
- **Registro integrado de Meta (P1):** en el estado sin configurar, además del formulario manual, botón *Conectar con Meta* y opción de copiar un enlace para que el negocio haga la conexión por su cuenta. Al terminar, la tarjeta de estado muestra el resultado.
- **Estados:** sin configurar (estado vacío con explicación y formulario), número ya conectado a otra organización (error que nombra la otra organización), prueba en curso.

### 6.10 P25 — Organización: actividad

- **Línea de tiempo** de las acciones de operadores sobre la organización: ícono por tipo de acción, descripción ("Franco suspendió la organización. Motivo: fin del piloto"), operador, fecha y hora, y detalle desplegable con valores antes y después.
- **Filtro** por tipo de acción.
- **Estado vacío:** "Sin actividad registrada".

### 6.11 P26 — Organización: configuración

- **Datos del negocio:** los mismos campos del paso 1 del asistente, editables, con botón *Guardar cambios*.
- **Cuenta:** tipo de cuenta y fechas del piloto.
- **Valores iniciales (P1):** asignación automática activada o no.
- **Notas internas:** área de texto amplia, con la fecha y el autor de la última edición.
- **Comportamiento:** confirmación si se sale con cambios sin guardar.

### 6.12 P27 — Cambiar estado

- **Suspender:** ventana que explica los efectos (los usuarios no podrán entrar ni enviar mensajes; los mensajes entrantes se siguen guardando), selector de motivo, campo de comentario opcional y confirmación escribiendo el nombre de la organización. Botón *Suspender organización* en estilo de acción peligrosa.
- **Reactivar:** ventana simple que confirma que los usuarios recuperan el acceso. Botón *Reactivar organización*.

### 6.13 P30 — Auditoría global (P1)

- **Tabla:** fecha y hora, operador, organización, acción, resumen. Fila que despliega el detalle.
- **Filtros:** operador, organización, tipo de acción, rango de fechas.
- **Paginación.**

## 7. Flujos principales

**F-PLT-1. Alta de un negocio nuevo**

1. El operador pulsa *Nueva organización* en P20.
2. Completa los datos del negocio, el primer administrador y revisa.
3. El sistema crea la organización activa, crea al administrador y envía la invitación.
4. El operador conecta WhatsApp en P24 (o lo deja para después) y prueba la conexión.
5. El administrador del negocio abre el enlace, crea su contraseña y entra al CRM.
6. La tarjeta de preparación de P22 se completa cuando llega el primer mensaje.

**F-PLT-2. El primer administrador no recibió el correo**

1. El operador abre P23 de la organización.
2. Pulsa *Generar y copiar enlace* sobre la invitación pendiente o vencida y lo envía por WhatsApp o por otro medio. El enlace anterior deja de servir.
3. Si prefiere el correo, pulsa *Reenviar invitación*: llega un correo con un enlace nuevo.

**F-PLT-3. Suspender y reactivar**

1. El operador pulsa *Suspender* en el encabezado de la organización.
2. Elige el motivo, escribe el nombre de la organización y confirma.
3. Los usuarios del negocio ven el aviso de acceso suspendido; los mensajes entrantes se siguen guardando.
4. Para reactivar, el operador pulsa *Reactivar* y confirma.

**F-PLT-4. Diagnóstico de "no me llegan los mensajes"**

1. El operador abre P24 de la organización.
2. Revisa el último evento recibido y los errores recientes.
3. Pulsa *Probar conexión* y corrige el dato que falle (por ejemplo, actualiza el token).

**F-PLT-5. Seguimiento de un piloto**

1. El operador abre P22 de la organización en piloto.
2. Revisa usuarios activos, mensajes enviados y conversaciones sin responder de los últimos 7 días.
3. Al terminar el piloto, cambia el tipo de cuenta a Cliente o suspende la organización.

## 8. Datos que muestra la consola

| Entidad | Campos visibles |
|---|---|
| Organización | Nombre comercial, razón social, NIT, sector, ciudad, zona horaria, contacto principal (nombre, correo, teléfono), tipo de cuenta, fechas del piloto, estado, motivo y fecha de suspensión, notas internas, fecha de creación |
| Preparación | Administrador invitado, invitación aceptada, WhatsApp conectado, primer mensaje recibido (cada uno con fecha) |
| Conexión de WhatsApp | Método de conexión, número visible, nombre verificado, identificador del número, identificador de la cuenta de WhatsApp Business, estado, fecha de conexión, fecha de actualización del token, último evento recibido, eventos en 24 horas, errores recientes |
| Usuario de la organización | Nombre, correo, rol, estado, último acceso, fecha de invitación y de vencimiento |
| Uso | Usuarios activos (7 días), mensajes recibidos y enviados (7 días), conversaciones sin responder ahora, último acceso |
| Registro de auditoría | Fecha y hora, operador, organización, acción, detalle antes y después |

## 9. Textos clave

| Situación | Texto |
|---|---|
| Organización suspendida (usuario del negocio al entrar) | "El acceso de tu organización está suspendido. Escríbenos a Numi para reactivarlo." |
| Confirmar suspensión | "Los usuarios de {organización} no podrán entrar ni enviar mensajes. Los mensajes que lleguen por WhatsApp se seguirán guardando. Escribe el nombre de la organización para confirmar." |
| Invitación enviada | "Invitación enviada a {correo}. El enlace vence en 7 días." |
| Enlace generado | "Enlace nuevo copiado. Solo funciona una vez y vence el {fecha}. El enlace anterior ya no funciona." |
| Desactivar usuario | "{nombre} quedará sin acceso y sus {n} conversaciones abiertas pasarán a Sin asignar. ¿Continuar?" |
| Último administrador | "La organización debe tener al menos un administrador activo." |
| Número duplicado | "Este número ya está conectado a {otra organización}." |
| Token guardado | "Token guardado el {fecha}. Por seguridad no se vuelve a mostrar." |
| Prueba de conexión correcta | "Conexión verificada: el número responde y el webhook está suscrito." |
| Sin acceso a la consola | "Esta área es solo para el equipo de Numi." |
| Posible duplicado al crear | "Ya existe una organización con este nombre o NIT. Revisa antes de continuar." |

## 10. Componentes adicionales para el design system

La consola reutiliza los componentes del CRM. Agrega estos:

| Componente | Uso |
|---|---|
| Indicador de consola de plataforma | Distingue visualmente la consola del CRM |
| Asistente por pasos con indicador de progreso | Crear organización |
| Tarjeta de lista de verificación | Preparación de la organización |
| Etiqueta de estado de organización | Activa, Suspendida |
| Etiqueta de tipo de cuenta | Piloto (con días restantes), Cliente |
| Indicador de salud | Conectada, sin configurar, con error (color más texto) |
| Campo secreto de solo escritura | Token de WhatsApp |
| Resultado de verificaciones | Prueba de conexión, con un renglón por verificación |
| Línea de tiempo de auditoría | Actividad de la organización |
| Banner de suspensión | Encabezado de la organización suspendida |
| Confirmación escribiendo un nombre | Acciones peligrosas |
| Campo de código de seis dígitos y código QR | Doble factor |
| Acción de copiar con confirmación | Enlaces de invitación, datos de contacto |
| Pestañas de sección con encabezado de entidad | Detalle de la organización |

## 11. Impacto en el documento maestro del CRM

| Elemento del CRM | Cambio |
|---|---|
| V05 Cuenta desactivada | Agrega la variante "Organización suspendida" con su texto propio |
| V50 Configuración del negocio | Muestra la conexión de WhatsApp en solo lectura; la configuración la hace Numi desde la consola |
| Modelo de usuarios | Agrega el rol de plataforma, separado de las membresías de las organizaciones |
| Auditoría | Se implementa desde ahora para las acciones de operadores |
| Envío de mensajes | Bloqueado cuando la organización está suspendida |
| V40 y V41 Usuarios | Mismas reglas de invitación que la consola: estados pendiente y vencida, vencimiento a 7 días, uso único y nombre del invitado |
| Contactos y mensajes | La consola confirma que las conversaciones contienen datos sensibles; el documento maestro debe tratarlas así |

## 12. Resumen para el diseño

La consola tiene **11 vistas P0** (P01, P02, L02, P20 a P27) y **2 vistas P1** (P10 y P30). El orden recomendado para diseñar:

1. **L02 y P20:** estructura de la consola y lista de organizaciones. Definen el lenguaje visual de la consola frente al CRM.
2. **P21:** asistente de creación y su pantalla final con el enlace de invitación.
3. **P22 con su encabezado y pestañas:** el detalle de la organización con la preparación y el uso.
4. **P23 y P24:** usuarios y WhatsApp, las dos vistas de soporte más usadas.
5. **P27, P25 y P26:** estado, actividad y configuración.
6. **P01 y P02:** entrada con doble factor.
7. **P10 y P30**, si hay tiempo.

**Totales:** 52 requisitos funcionales (47 P0 y 5 P1) y 13 vistas, más las variantes reutilizadas del CRM.
