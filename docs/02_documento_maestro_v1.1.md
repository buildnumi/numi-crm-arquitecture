# Numi CRM — Documento maestro del MVP

*Requisitos funcionales y vistas del sistema · Base para el design system · Franco y Anthony · Versión 1.1 · 14 de octubre de 2026*

## 1. Propósito y uso del documento

Este documento reúne **todo lo que el MVP de Numi CRM debe contener**: usuarios y permisos, requisitos funcionales, inventario completo de vistas, flujos, reglas de interfaz, textos del sistema y los componentes que el design system debe cubrir. Es la entrada para la reunión de arranque del design system y para el desarrollo posterior.

- **Fuente:** el brief de la reunión y el documento de definición del MVP v2.2.
- **Anexo:** la consola de plataforma, desde la que el equipo de Numi administra las organizaciones, se describe en el documento *Consola de plataforma v1.1*.
- **Meta del producto:** MVP terminado el 30 de noviembre de 2026.
- **Equipo:** Franco y Anthony.
- **Cómo leerlo:** las secciones 2 a 4 explican qué es el sistema y quién lo usa; la sección 5 lista los requisitos; la sección 6 describe cada vista; las secciones 7 a 11 entregan las reglas y los componentes que el diseño necesita.

Todos los requisitos de este documento son de prioridad P0, es decir, necesarios para el MVP. Cada requisito tiene un identificador (por ejemplo, RF-CNV-03) que se usa en las vistas, en las pruebas y en el backlog.

## 2. Visión del producto

Numi CRM es un CRM conversacional multiempresa, vendido como servicio (SaaS) en Colombia. Permite que un negocio atienda, organice y mida sus conversaciones de WhatsApp desde un solo lugar.

| Aspecto | Definición |
|---|---|
| Canal | WhatsApp, mediante la API oficial de Meta (WhatsApp Business Platform, Cloud API) |
| Segmento inicial | Clínicas estéticas; después inmobiliarias, odontologías y veterinarias |
| Volumen de referencia | 500 conversaciones al mes por negocio |
| Capacidades del MVP | Conversaciones, contactos, estado de los contactos, métricas básicas, usuarios y permisos |
| Plataforma | Aplicación web para computador. La versión móvil queda fuera del MVP |
| Idioma | Español (Colombia) |
| Modelo | Multiempresa: cada negocio ve únicamente sus propios datos |

## 3. Usuarios y roles

| Rol | Quién es | Qué necesita hacer |
|---|---|---|
| **Asesor** | Persona que atiende a los clientes todos los días | Ver y responder conversaciones, tomar conversaciones sin asignar, cambiar el estado de los contactos, consultar y editar contactos |
| **Administrador** | Persona que supervisa la operación del negocio | Todo lo del asesor, más: ver todas las conversaciones, reasignar, ver métricas, gestionar usuarios y configurar el negocio |
| **Dueño o gerente (comprador)** | Decide la compra | No es un rol aparte en el MVP: usa el rol Administrador |

### 3.1 Matriz de permisos

| Acción | Asesor | Administrador |
|---|---|---|
| Ver sus conversaciones y la cola sin asignar | Sí | Sí |
| Ver todas las conversaciones del negocio | No | Sí |
| Responder una conversación | Las suyas y las sin asignar (al responder una sin asignar, queda como responsable) | Todas |
| Tomar una conversación sin asignar | Sí | Sí |
| Reasignar una conversación a otra persona | No | Sí |
| Cerrar y reabrir conversaciones | Las suyas | Todas |
| Ver y editar contactos | Sí | Sí |
| Crear contactos | Sí | Sí |
| Cambiar el estado de un contacto | Sí | Sí |
| Ver métricas | No | Sí |
| Invitar, editar y desactivar usuarios | No | Sí |
| Ver y cambiar la configuración del negocio | No | Sí |

Los permisos dependen solo del rol y son fijos en el MVP. Un negocio nunca ve datos de otro negocio, sea cual sea el rol.

Además existe el rol **Operador de plataforma** (equipo de Numi). No pertenece a ninguna organización, no tiene acceso al contenido de las conversaciones y trabaja desde la consola de plataforma, descrita en su anexo.

## 4. Conceptos del dominio

| Concepto | Definición |
|---|---|
| Negocio (organización) | Cliente de Numi CRM. Es el límite de aislamiento de los datos |
| Estado de la organización | Activa o suspendida. Lo controla el equipo de Numi desde la consola de plataforma. Con la organización suspendida nadie entra ni envía mensajes, pero los mensajes entrantes se siguen guardando |
| Usuario | Persona con acceso al CRM dentro de un negocio, con un rol |
| Contacto | Persona que escribe al negocio. Se reconoce por su identidad en cada canal; en el MVP el único canal es WhatsApp y la identidad es el número de teléfono, único dentro del negocio. El modelo admite varias identidades por contacto para sumar otros canales después |
| Conversación | Hilo continuo de mensajes con un contacto. Hay una conversación por contacto |
| Mensaje | Texto entrante (del contacto) o saliente (del negocio) dentro de una conversación |
| Estado del contacto | Etapa comercial: Nuevo, En contacto, Calificado, Cliente, En pausa, Perdido |
| Estado de la conversación | Abierta o cerrada. Es independiente del estado del contacto |
| Responsable | Usuario al que está asignada una conversación. El responsable que se muestra en un contacto es el de su conversación; no se asigna por separado |
| Cola sin asignar | Conversaciones abiertas que todavía no tienen responsable |
| Ventana de 24 horas | Regla de Meta: solo se puede enviar texto libre dentro de las 24 horas siguientes al último mensaje entrante del contacto |
| Lead | En el MVP, todo contacto nuevo es un lead |

## 5. Requisitos funcionales

Cada requisito indica en qué vista se materializa (columna "Vista"; las vistas se describen en la sección 6).

### 5.1 Autenticación y acceso (AUT)

| ID | Requisito | Vista |
|---|---|---|
| RF-AUT-01 | El usuario inicia sesión con correo y contraseña | V01 |
| RF-AUT-02 | El usuario cierra sesión desde el menú de usuario | L01 |
| RF-AUT-03 | El usuario invitado recibe un correo con un enlace, crea su contraseña y entra al sistema | V02 |
| RF-AUT-04 | El usuario recupera su contraseña con un enlace enviado por correo | V03, V04 |
| RF-AUT-05 | Un usuario desactivado no puede entrar y ve un mensaje claro | V01, V05 |
| RF-AUT-06 | Si la sesión expira, el sistema pide iniciar sesión de nuevo y devuelve al usuario a la pantalla donde estaba | V01 |
| RF-AUT-07 | Si la organización está suspendida, ningún usuario puede entrar y todos ven un mensaje que indica contactar a Numi. Las sesiones abiertas se cierran en la siguiente acción | V01, V05 |

### 5.2 Conversaciones (CNV)

| ID | Requisito | Vista |
|---|---|---|
| RF-CNV-01 | El sistema recibe los mensajes de texto de WhatsApp y los muestra en la bandeja sin recargar la página | V10, V11 |
| RF-CNV-02 | La lista muestra por conversación: contacto (nombre o teléfono), último mensaje, hora, responsable, estado del contacto e indicador de "sin responder" cuando el último mensaje es del contacto y nadie ha contestado | V10 |
| RF-CNV-03 | La lista se ordena por actividad reciente, primero la más reciente | V10 |
| RF-CNV-04 | Filtros de la lista. Asesor: Mías, Sin asignar. Administrador: Todas, Mías, Sin asignar y por persona. Para ambos: estado de la conversación (abierta o cerrada) y estado del contacto | V10 |
| RF-CNV-05 | Búsqueda de conversaciones por nombre o teléfono del contacto | V10 |
| RF-CNV-06 | La conversación muestra el historial completo, con mensajes entrantes y salientes diferenciados, hora de cada mensaje y separadores por día. Los mensajes se ordenan por la hora en que se enviaron, aunque lleguen desordenados | V11 |
| RF-CNV-07 | El usuario escribe y envía mensajes de texto desde el CRM. Enter envía y Mayús+Enter agrega una línea | V11 |
| RF-CNV-08 | El sistema muestra el estado de la ventana de 24 horas. Si venció, el envío se bloquea y se explica el motivo | V11 |
| RF-CNV-09 | Cada mensaje enviado muestra su estado: enviando, enviado, entregado, leído o error, según lo informe Meta. Un mensaje puede pasar a error después de figurar como enviado. Un mensaje con error se puede reintentar | V11 |
| RF-CNV-10 | Si llega un mensaje de un tipo que el MVP no muestra (audio, imagen, documento u otro), la conversación muestra un marcador con el tipo de mensaje y la hora | V11 |
| RF-CNV-11 | El asesor puede tomar una conversación sin asignar; si responde una conversación sin asignar, queda como su responsable. El administrador puede asignarla o reasignarla a cualquier persona activa | V10, V11 |
| RF-CNV-12 | Asignación automática: si está activa, una conversación nueva se asigna al asesor activo y disponible con menos conversaciones abiertas | V50 |
| RF-CNV-13 | El usuario cierra una conversación. Si el contacto vuelve a escribir, se reabre sola | V11 |
| RF-CNV-14 | Desde la conversación se cambia el estado del contacto y se abre su detalle | V11, V12 |
| RF-CNV-15 | Los cambios de asignación, estado y mensajes nuevos se reflejan en tiempo real en las pantallas de los usuarios afectados. La navegación y el título de la pestaña muestran el número de conversaciones sin responder | L01, V10 |
| RF-CNV-16 | Mientras la organización está suspendida, no se pueden enviar mensajes; los mensajes entrantes se siguen recibiendo y guardando | V11 |

### 5.3 Contactos (CON)

| ID | Requisito | Vista |
|---|---|---|
| RF-CON-01 | Cuando un número nuevo escribe, el sistema crea el contacto automáticamente con estado Nuevo | V10, V20 |
| RF-CON-02 | El usuario crea un contacto manualmente con nombre, teléfono y correo | V22 |
| RF-CON-03 | El usuario edita el nombre, el teléfono, el correo y el estado de un contacto | V22, V21 |
| RF-CON-04 | El usuario ve el detalle del contacto: datos, estado, responsable, conversación e historial de estados | V21 |
| RF-CON-05 | La lista de contactos se busca por nombre o teléfono | V20 |
| RF-CON-06 | La lista de contactos se filtra por estado y por responsable | V20 |
| RF-CON-07 | El teléfono no puede repetirse dentro del mismo negocio. Si el usuario intenta duplicarlo, el sistema lo rechaza y ofrece abrir el contacto existente | V22 |
| RF-CON-08 | El sistema valida el formato del teléfono (con indicativo de país) y del correo | V22 |
| RF-CON-09 | La lista de contactos se ordena por última actividad, con paginación | V20 |

**Datos de un contacto:** nombre, teléfono, correo, estado, responsable (el de su conversación), fecha de creación y fecha de última actividad. La ficha del contacto no tiene campos de datos sensibles, pero el contenido de las conversaciones sí puede tenerlos (RF-TRV-07).

### 5.4 Estados del contacto (EST)

| ID | Requisito | Vista |
|---|---|---|
| RF-EST-01 | Un contacto tiene uno de seis estados: Nuevo, En contacto, Calificado, Cliente, En pausa, Perdido | V10, V11, V20, V21 |
| RF-EST-02 | Asesores y administradores cambian el estado desde la conversación, el detalle del contacto o la lista de contactos | V11, V21, V20 |
| RF-EST-03 | El estado actual es visible en la lista de conversaciones, la lista de contactos, el detalle y la conversación, con un tratamiento visual propio para cada estado | Todas |
| RF-EST-04 | Cada cambio de estado queda en un historial con quién lo hizo, cuándo y de qué estado a cuál | V21 |

### 5.5 Métricas (MET)

| ID | Requisito | Vista |
|---|---|---|
| RF-MET-01 | Conversaciones por mes: cantidad de conversaciones con al menos un mensaje durante el período | V30 |
| RF-MET-02 | Tiempo de atención: minutos entre un mensaje entrante que espera respuesta y la primera respuesta de una persona, en minutos corridos. Se muestra la mediana como valor principal y el promedio como dato secundario, para que unos pocos mensajes nocturnos no distorsionen el indicador. Las respuestas automáticas no cuentan | V30 |
| RF-MET-03 | Nuevos leads por mes: contactos creados durante el período | V30 |
| RF-MET-04 | Tasa de conversión: de los contactos creados en el período, el porcentaje que está en estado Cliente al consultar | V30 |
| RF-MET-05 | Cantidad de contactos: total y desglose por estado | V30 |
| RF-MET-06 | Todas las métricas se pueden ver para todo el negocio y por asesor | V30 |
| RF-MET-07 | Selector de período: mes actual, mes anterior, últimos 30 días y rango personalizado | V30 |
| RF-MET-08 | Las métricas son visibles solo para el administrador | V30 |

### 5.6 Usuarios (USR)

| ID | Requisito | Vista |
|---|---|---|
| RF-USR-01 | El administrador ve la lista de usuarios con nombre, correo, rol y estado (activo, invitación pendiente, invitación vencida o desactivado) | V40 |
| RF-USR-02 | El administrador invita a un usuario por correo, con su nombre y su rol | V41 |
| RF-USR-03 | El administrador reenvía o cancela una invitación pendiente o vencida. Reenviar genera un enlace nuevo e invalida el anterior | V40 |
| RF-USR-04 | El administrador cambia el rol de un usuario | V42 |
| RF-USR-05 | El administrador desactiva y reactiva usuarios. Al desactivar a un asesor, sus conversaciones abiertas quedan sin asignar | V42 |
| RF-USR-06 | El administrador marca si un asesor está disponible para recibir asignación automática | V42 |
| RF-USR-07 | Siempre debe existir al menos un administrador activo; un administrador no puede desactivarse a sí mismo si es el único | V42 |
| RF-USR-08 | Los enlaces de invitación vencen a los 7 días y se pueden usar una sola vez | V40, V02 |

### 5.7 Configuración del negocio (CFG)

| ID | Requisito | Vista |
|---|---|---|
| RF-CFG-01 | El administrador ve y edita el nombre del negocio | V50 |
| RF-CFG-02 | El administrador ve el estado de la conexión de WhatsApp: número conectado y estado (conectado, desconectado o con error). Es de solo lectura: la conexión la gestiona el equipo de Numi desde la consola de plataforma | V50 |
| RF-CFG-03 | El administrador activa o desactiva la asignación automática | V50 |

### 5.8 Requisitos transversales (TRV)

| ID | Requisito |
|---|---|
| RF-TRV-01 | **Aislamiento entre negocios:** ninguna pantalla ni consulta muestra datos de otro negocio. Se verifica con pruebas automáticas |
| RF-TRV-02 | **Seguridad del canal:** cada evento de WhatsApp se valida con la firma de Meta y no se duplica si Meta lo reenvía |
| RF-TRV-03 | **Rendimiento de referencia:** un mensaje entrante aparece en la bandeja en pocos segundos (objetivo: 5 segundos o menos); la búsqueda responde en menos de 1 segundo con 5.000 contactos |
| RF-TRV-04 | **Estados de interfaz:** toda vista maneja los estados de carga, vacío y error (sección 8) |
| RF-TRV-05 | **Formato regional:** fechas y horas en zona horaria de Bogotá; textos en español (Colombia) |
| RF-TRV-06 | **Accesibilidad básica:** navegación con teclado, foco visible y contraste suficiente en todo componente |
| RF-TRV-07 | **Datos sensibles:** las conversaciones pueden contener datos de salud y otros datos sensibles. Su contenido solo lo ven los usuarios del negocio según su rol, y nunca aparece en la consola de plataforma, en los registros técnicos ni en las métricas |
| RF-TRV-08 | **Correos del sistema:** las invitaciones y los restablecimientos de contraseña salen desde un remitente propio de Numi, con entrega confiable para cualquier destinatario |

## 6. Inventario de vistas

### 6.1 Mapa general

| ID | Vista | Ruta | Roles | Propósito |
|---|---|---|---|---|
| V01 | Iniciar sesión | /entrar | Público | Acceder con correo y contraseña |
| V02 | Aceptar invitación | /invitacion | Invitado | Crear contraseña y entrar |
| V03 | Recuperar contraseña | /recuperar | Público | Pedir el enlace de recuperación |
| V04 | Nueva contraseña | /restablecer | Público | Definir una contraseña nueva |
| V05 | Sin acceso | /sin-acceso | Usuario desactivado u organización suspendida | Explicar por qué no tiene acceso |
| L01 | Estructura de la aplicación | Todas las rutas internas | Asesor, Administrador | Navegación y marco común |
| V10 | Bandeja de conversaciones | /bandeja | Asesor, Administrador | Ver, filtrar y atender conversaciones |
| V11 | Conversación | /bandeja/:id | Asesor, Administrador | Leer y responder, gestionar la conversación |
| V12 | Panel del contacto | Panel lateral de V11 | Asesor, Administrador | Resumen del contacto junto a la conversación |
| V20 | Lista de contactos | /contactos | Asesor, Administrador | Buscar, filtrar y crear contactos |
| V21 | Detalle del contacto | /contactos/:id | Asesor, Administrador | Ver y gestionar un contacto |
| V22 | Crear o editar contacto | Ventana lateral | Asesor, Administrador | Formulario de contacto |
| V30 | Métricas | /metricas | Administrador | Indicadores del negocio y por asesor |
| V40 | Lista de usuarios | /usuarios | Administrador | Gestionar el equipo |
| V41 | Invitar usuario | Ventana modal | Administrador | Enviar una invitación |
| V42 | Detalle del usuario | Ventana lateral | Administrador | Rol, estado y disponibilidad |
| V50 | Configuración del negocio | /configuracion | Administrador | Nombre, conexión de WhatsApp y asignación automática |
| V60 | Estados del sistema | Varias | Todos | Página no encontrada, sin permiso, error y sesión expirada |

Total: 18 vistas y estructura, más los estados de la sección 8.

### 6.2 L01 — Estructura de la aplicación

- **Barra lateral de navegación:** Bandeja (con el contador de conversaciones sin responder), Contactos, Métricas (solo administrador), Usuarios (solo administrador) y Configuración (solo administrador).
- **Encabezado:** nombre del negocio y menú de usuario con nombre, rol y cerrar sesión.
- **Indicador de conexión:** señal discreta cuando la conexión en tiempo real se cae, con aviso de reconexión.
- **Área de contenido:** ocupa el resto de la pantalla; la bandeja usa todo el alto disponible.
- **Navegación visible según rol:** el asesor solo ve Bandeja y Contactos.

### 6.3 V01 — Iniciar sesión

- **Contenido:** logotipo de Numi, campos de correo y contraseña, botón *Entrar*, enlace *¿Olvidaste tu contraseña?*.
- **Estados:** error de credenciales, cuenta desactivada u organización suspendida (lleva a V05), cargando, sesión expirada con aviso "Tu sesión expiró. Entra de nuevo para continuar".
- **Acción:** al entrar, el usuario llega a la Bandeja (o a la pantalla que intentaba abrir).

### 6.4 V02 — Aceptar invitación

- **Contenido:** nombre del negocio que invita, correo del invitado (solo lectura), campos de nueva contraseña y confirmación, indicador de requisitos de la contraseña, botón *Crear cuenta y entrar*.
- **Estados:** enlace vencido o ya usado (con mensaje y sugerencia de pedir una nueva invitación al administrador), error de contraseña, cargando.

### 6.5 V03 y V04 — Recuperar y restablecer contraseña

- **V03:** campo de correo y botón *Enviar enlace*. Confirma siempre con el mismo mensaje, exista o no el correo.
- **V04:** nueva contraseña y confirmación, botón *Guardar contraseña*; estados de enlace vencido y de éxito con acceso a V01.

### 6.6 V05 — Sin acceso

Tiene dos variantes, cada una con su mensaje y un botón para cerrar sesión:

- **Usuario desactivado:** "Tu acceso a este negocio está desactivado. Contacta al administrador."
- **Organización suspendida:** "El acceso de tu organización está suspendido. Escríbenos a Numi para reactivarlo."

### 6.7 V10 — Bandeja de conversaciones

Vista principal del producto. Tiene tres regiones en una sola pantalla:

1. **Columna izquierda, lista de conversaciones.**
   - Barra superior con el buscador (RF-CNV-05).
   - Filtros en pestañas o chips: Mías, Sin asignar y, para el administrador, Todas y selector de persona. Filtros adicionales de estado de conversación y estado del contacto (RF-CNV-04).
   - Elemento de la lista: avatar con iniciales, nombre o teléfono, último mensaje (una línea), hora, etiqueta del estado del contacto, responsable, e indicador de "sin responder".
   - Elemento seleccionado resaltado; orden por actividad reciente.
2. **Columna central, la conversación (V11)** con la conversación seleccionada.
3. **Columna derecha, panel del contacto (V12)**, que se puede ocultar.

- **Estados:** carga con esqueletos, lista vacía ("Aún no hay conversaciones"), vacía por filtro ("No hay conversaciones con estos filtros" y botón para limpiar filtros), sin conversación seleccionada (mensaje que invita a elegir una), error de carga con reintento.
- **Tiempo real:** las conversaciones nuevas y los cambios entran a la lista sin recargar.

### 6.8 V11 — Conversación

- **Encabezado:** nombre o teléfono del contacto, etiqueta del estado del contacto (editable), responsable (con acción para tomar o reasignar según el rol) y acciones: cerrar o reabrir conversación, abrir el detalle del contacto.
- **Hilo de mensajes:** burbujas entrantes y salientes diferenciadas, hora en cada mensaje, separadores por día, desplazamiento al último mensaje, carga de mensajes anteriores al subir.
- **Variantes de mensaje:** entrante de texto; saliente de texto con su estado (enviando, enviado, entregado, leído, error con *Reintentar*); marcador de tipo no mostrado, con el tipo de mensaje (audio, imagen, documento u otro) y la hora; mensaje de sistema (por ejemplo, "Conversación asignada a María", "Estado cambiado a Calificado").
- **Aviso de ventana de 24 horas:** franja sobre el campo de escritura que indica si la ventana está abierta (con el tiempo restante cuando quedan pocas horas) o vencida.
- **Campo de escritura:** área de texto con botón *Enviar*; deshabilitado con explicación cuando la ventana venció, cuando la conversación está asignada a otro asesor y el usuario no es administrador, o cuando la organización está suspendida.
- **Estados:** conversación cerrada (franja con botón *Reabrir*), sin mensajes todavía, error de envío, sin conexión.

### 6.9 V12 — Panel del contacto

- **Contenido:** avatar, nombre, teléfono, correo, selector del estado del contacto, responsable, fechas de creación y de última actividad, y los últimos cambios de estado.
- **Acciones:** editar contacto (abre V22), ver detalle completo (abre V21).

### 6.10 V20 — Lista de contactos

- **Barra superior:** título, buscador por nombre o teléfono, botón *Nuevo contacto*.
- **Filtros:** estado del contacto y responsable.
- **Tabla:** nombre, teléfono, correo, estado (etiqueta), responsable, última actividad. Fila clicable que abre V21. Ordenable por última actividad.
- **Pie:** paginación y total de contactos.
- **Estados:** carga, sin contactos (con botón para crear el primero), sin resultados de búsqueda, error.

### 6.11 V21 — Detalle del contacto

- **Encabezado:** nombre, etiqueta del estado (editable con selector), botones *Editar* e *Ir a la conversación*.
- **Datos:** teléfono, correo, responsable, fecha de creación, última actividad.
- **Historial de estados:** línea de tiempo con cada cambio (de qué estado a cuál, quién y cuándo).
- **Conversación:** resumen de la conversación asociada con acceso a V11.

### 6.12 V22 — Crear o editar contacto

- **Formulario en ventana lateral:** nombre, teléfono (con indicativo de país), correo y, al editar, estado.
- **Validación en línea:** teléfono con formato internacional, correo válido, teléfono duplicado (mensaje con enlace al contacto existente).
- **Botones:** *Guardar* y *Cancelar*; confirmación si hay cambios sin guardar.

### 6.13 V30 — Métricas

- **Barra superior:** título y selector de período (mes actual, mes anterior, últimos 30 días, rango personalizado), y selector de vista: *Todo el negocio* o *Por asesor*.
- **Tarjetas de indicadores:** conversaciones, tiempo de atención (mediana, con el promedio como dato secundario), nuevos leads, tasa de conversión y total de contactos. Cada tarjeta muestra el valor y la comparación con el período anterior.
- **Contactos por estado:** gráfico de barras o embudo con los seis estados y su cantidad.
- **Tabla por asesor:** una fila por asesor con conversaciones atendidas, tiempo de atención (mediana), contactos asignados y contactos en estado Cliente.
- **Estados:** carga, sin datos en el período ("Aún no hay datos para este período"), error con reintento.
- **Ayuda:** cada indicador tiene un texto que explica cómo se calcula (sección 5.5).

### 6.14 V40 — Lista de usuarios

- **Barra superior:** título y botón *Invitar usuario*.
- **Tabla:** nombre, correo, rol, estado (activo, invitación pendiente, invitación vencida o desactivado) y disponibilidad para asignación automática.
- **Acciones por fila:** abrir detalle (V42); en invitaciones pendientes, *Reenviar* y *Cancelar invitación*.
- **Estados:** carga, error, sin otros usuarios (invita a crear el primero).

### 6.15 V41 — Invitar usuario

- **Ventana modal:** nombre y correo del invitado, selector de rol (Asesor o Administrador), botón *Enviar invitación* y una línea que indica que el enlace vence en 7 días.
- **Estados:** correo inválido, correo que ya pertenece al negocio, éxito con confirmación.

### 6.16 V42 — Detalle del usuario

- **Ventana lateral:** nombre y correo, selector de rol, interruptor *Disponible para asignación automática*, botón *Desactivar usuario* (o *Reactivar*).
- **Confirmaciones:** al desactivar, se avisa cuántas conversaciones abiertas quedarán sin asignar. No se puede desactivar ni quitar el rol al único administrador activo.

### 6.17 V50 — Configuración del negocio

- **Sección Negocio:** nombre editable y botón *Guardar*.
- **Sección WhatsApp (solo lectura):** número conectado, estado de la conexión (conectado, desconectado, con error) con indicador de color y texto, la fecha de la última verificación y el texto "La conexión de WhatsApp la gestiona el equipo de Numi. Escríbenos si necesitas cambiarla."
- **Sección Asignación:** interruptor *Asignación automática de conversaciones nuevas*, con una línea que explica la regla.

### 6.18 V60 — Estados del sistema

- **Página no encontrada,** sin permiso para ver la pantalla, error inesperado con botón para reintentar y volver al inicio, y aviso de sesión expirada. Cada uno con un mensaje breve y una acción.

## 7. Flujos principales

**F1. Recibir y responder una conversación**

1. Un cliente escribe por WhatsApp.
2. El sistema crea el contacto (si es nuevo) y la conversación, y la asigna si la asignación automática está activa.
3. La conversación aparece en la bandeja con el indicador de "sin responder".
4. El asesor la abre, lee el historial y escribe la respuesta.
5. Si la ventana de 24 horas está abierta, el mensaje se envía y muestra su estado. Si venció, el campo queda bloqueado con la explicación.
6. El asesor cambia el estado del contacto y, al terminar, cierra la conversación.

**F2. Tomar o reasignar una conversación**

1. El asesor abre la pestaña *Sin asignar*, elige una conversación y pulsa *Tomar*.
2. El administrador, en cualquier conversación, abre el selector de responsable y elige a otra persona activa.
3. La persona afectada ve el cambio en su bandeja sin recargar.

**F3. Cambiar el estado de un contacto**

1. El usuario abre el selector de estado en la conversación, el panel del contacto, el detalle o la lista.
2. Elige el nuevo estado.
3. El cambio se refleja en todas las vistas y queda en el historial.

**F4. Invitar a un usuario y primer acceso**

1. El administrador abre *Usuarios* y pulsa *Invitar usuario*; ingresa correo y rol.
2. El invitado recibe el correo, abre el enlace y crea su contraseña (V02).
3. Entra a la Bandeja con el rol asignado. El administrador lo ve como activo.

**F5. Consultar las métricas**

1. El administrador abre *Métricas*.
2. Elige el período y alterna entre todo el negocio y por asesor.
3. Revisa las tarjetas, el gráfico de estados y la tabla por asesor.

**F6. Crear un contacto manualmente**

1. El usuario pulsa *Nuevo contacto* en la lista de contactos.
2. Completa nombre, teléfono y correo; el sistema valida y rechaza duplicados.
3. Guarda y llega al detalle del nuevo contacto.

## 8. Estados de interfaz y casos límite

Todas las vistas manejan estos estados de forma consistente. El design system debe tener un patrón para cada uno.

| Estado | Comportamiento |
|---|---|
| Carga | Esqueletos que imitan la forma del contenido; indicador en botones durante una acción |
| Vacío | Mensaje breve que explica por qué está vacío y la acción siguiente, si existe |
| Sin resultados | Mensaje de que la búsqueda o filtro no devolvió nada, con botón para limpiar |
| Error recuperable | Mensaje claro y botón *Reintentar* |
| Sin conexión en tiempo real | Aviso discreto; los datos se actualizan al reconectar |
| Sin permiso | Pantalla V60 de acceso denegado; los controles no permitidos no se muestran |
| Éxito | Confirmación breve (aviso temporal) sin interrumpir el trabajo |
| Acción destructiva | Ventana de confirmación que explica la consecuencia (desactivar usuario, cancelar invitación, descartar cambios) |

## 9. Datos que muestra la interfaz

| Entidad | Campos visibles |
|---|---|
| Negocio | Nombre, estado (activa o suspendida), número de WhatsApp conectado, estado de la conexión, asignación automática (sí o no) |
| Usuario | Nombre, correo, rol (Asesor o Administrador), estado (activo, invitación pendiente, invitación vencida, desactivado), disponible para asignación (sí o no) |
| Contacto | Nombre, teléfono (su identidad en WhatsApp), correo, estado, responsable (el de su conversación), fecha de creación, última actividad |
| Historial de estados | Estado anterior, estado nuevo, quién, cuándo |
| Conversación | Contacto, responsable, estado (abierta o cerrada), último mensaje, hora del último mensaje, hora del último mensaje entrante, indicador de sin responder |
| Mensaje | Dirección (entrante o saliente), texto, hora, estado de envío (enviando, enviado, entregado, leído o error), origen (persona o automático), tipo (texto, audio, imagen, documento u otro) |

## 10. Textos clave del sistema

Mensajes que el diseño y el desarrollo usan tal cual. El tono es directo, en español de Colombia, sin tecnicismos.

| Situación | Texto |
|---|---|
| Ventana de 24 horas vencida | "Pasaron más de 24 horas desde el último mensaje del cliente. WhatsApp no permite enviar mensajes libres hasta que vuelva a escribir." |
| Ventana por vencer | "Te quedan {horas} h para responder a este cliente." |
| Mensaje de tipo no mostrado | "Recibiste {un audio / una imagen / un documento}. Este tipo de mensaje todavía no se puede ver en Numi CRM." |
| Mensaje no entregado | "WhatsApp no pudo entregar este mensaje." con botón *Reintentar* |
| Error de envío | "No se pudo enviar el mensaje." con botón *Reintentar* |
| Teléfono duplicado | "Ya existe un contacto con este teléfono." con enlace *Ver contacto* |
| Credenciales incorrectas | "El correo o la contraseña no coinciden." |
| Cuenta desactivada | "Tu acceso a este negocio está desactivado. Contacta al administrador." |
| Organización suspendida | "El acceso de tu organización está suspendido. Escríbenos a Numi para reactivarlo." |
| WhatsApp gestionado por Numi | "La conexión de WhatsApp la gestiona el equipo de Numi. Escríbenos si necesitas cambiarla." |
| Invitación enviada | "Invitación enviada a {correo}. El enlace vence en 7 días." |
| Sesión expirada | "Tu sesión expiró. Entra de nuevo para continuar." |
| Invitación vencida | "Este enlace de invitación venció. Pide una nueva al administrador." |
| Desactivar usuario | "{nombre} quedará sin acceso y sus {n} conversaciones abiertas pasarán a Sin asignar. ¿Continuar?" |
| Último administrador | "Debe haber al menos un administrador activo." |
| WhatsApp desconectado | "WhatsApp está desconectado. No se pueden recibir ni enviar mensajes." |
| Bandeja vacía | "Aún no hay conversaciones. Cuando un cliente escriba por WhatsApp, aparecerá aquí." |
| Sin datos en métricas | "Aún no hay datos para este período." |
| Cambios sin guardar | "Tienes cambios sin guardar. ¿Quieres descartarlos?" |

## 11. Base para el design system

### 11.1 Identidad de marca

Se usa la identidad de Numi (Violeta Autoridad):

| Elemento | Valor |
|---|---|
| Violeta profundo (color principal) | #2D1B4E |
| Ciruela (texto secundario) | #4A3F5C |
| Violeta de acento | #9333EA |
| Tinta (texto principal) | #12081F |
| Lavanda (fondos suaves) | #F5F3F8 |
| Tipografía de títulos | Fraunces |
| Tipografía de interfaz | Geist Sans |

### 11.2 Fundamentos por definir en el design system

- **Color semántico:** éxito, advertencia, error e información, además de los seis colores de estado del contacto y los dos estados de la conversación. Cada estado debe distinguirse por texto o forma, no solo por color.
- **Tipografía:** escala de tamaños para títulos, texto de interfaz, texto pequeño y datos numéricos.
- **Espaciado, rejilla, radios y sombras.**
- **Iconografía:** conjunto único para navegación, acciones y estados.
- **Modo de color:** claro en el MVP.
- **Resolución de referencia:** escritorio; la versión móvil queda fuera.

### 11.3 Componentes requeridos

| Grupo | Componentes |
|---|---|
| Acciones | Botón principal, secundario, de texto, peligroso, con indicador de carga y solo ícono |
| Formularios | Campo de texto, área de texto, campo de contraseña, selector, selector de estado, interruptor, casilla, campo de teléfono, mensaje de validación, selector de período |
| Navegación | Barra lateral, encabezado, menú de usuario, pestañas, chips de filtro, paginación |
| Datos | Tabla, fila clicable, etiqueta de estado del contacto (seis variantes), etiqueta de estado de la conversación (dos), etiqueta de rol, avatar con iniciales, indicador de sin responder, contador |
| Conversación | Elemento de lista de conversaciones, burbuja entrante, burbuja saliente (con sus cinco estados de envío), marcador de tipo de mensaje no mostrado, mensaje de sistema, separador de día, aviso de ventana de 24 horas, campo de escritura |
| Métricas | Tarjeta de indicador con comparación, gráfico de barras o embudo, tabla por asesor |
| Superposiciones | Ventana modal, ventana lateral, ventana de confirmación, aviso temporal, información emergente |
| Estados | Esqueleto de carga, estado vacío, estado sin resultados, banner de error, indicador de conexión |
| Línea de tiempo | Elemento de historial de estados |

### 11.4 Reglas de interacción comunes

- Todos los componentes interactivos tienen estados de reposo, foco, activo, deshabilitado y error cuando aplique.
- Las acciones frecuentes tienen atajos de teclado (Enter para enviar un mensaje).
- Las acciones destructivas siempre piden confirmación.
- Los cambios en tiempo real no mueven el contenido que el usuario está leyendo.
- Las fechas se muestran como "12 oct 2026" y las horas en formato de 12 horas, en zona horaria de Bogotá.

## 12. Alcance de este documento y elementos diferidos

El documento cubre exactamente el alcance del MVP definido en la v2.2. Los siguientes elementos se definen en etapas posteriores y no forman parte de las vistas de este documento:

- Visualización de audios, imágenes y documentos (en el MVP se muestra el marcador con el tipo de mensaje).
- Plantillas de WhatsApp para escribir pasadas las 24 horas.
- Otros canales, versión móvil, agentes de IA, automatizaciones y reportes avanzados.
- Notas internas, campos personalizados, importación de contactos y permisos configurables.
- Varias organizaciones por usuario y auditoría de las acciones de los usuarios del negocio. La auditoría de las acciones del equipo de Numi sí entra en el MVP (ver el anexo de la consola).

### 12.1 Elementos que dependen del modelo de conexión de WhatsApp

Si se adopta la coexistencia (el negocio sigue usando la app de WhatsApp Business en el mismo número), el MVP suma tres elementos:

- **Mensajes enviados desde el celular:** aparecen en el hilo con una burbuja saliente propia, marcada "Enviado desde el celular", y cuentan como respuesta de una persona en las métricas.
- **Historial importado:** al conectar el número, la conversación muestra los mensajes anteriores que Meta permita importar, con un separador "Historial importado".
- **Marcador de tipo no mostrado:** agrega "Puedes verlo en la app de WhatsApp Business".

## 13. Reunión de arranque del design system

**Objetivo:** salir con el inventario de vistas aprobado, los fundamentos visuales definidos y la lista de componentes priorizada.

| Bloque | Qué se decide o produce |
|---|---|
| 1. Revisión del documento | Confirmar el inventario de vistas (sección 6) y los flujos (sección 7) |
| 2. Fundamentos visuales | Colores semánticos y de estado, escala tipográfica, espaciado, radios, iconografía |
| 3. Componentes | Confirmar la lista de la sección 11.3 y el orden de construcción |
| 4. Vistas críticas | Elegir las tres primeras vistas para bocetar: bandeja con conversación, lista de contactos y métricas |
| 5. Herramienta y entrega | Dónde vive el design system y cómo pasa al código (componentes de la interfaz del frontend) |
| 6. Tareas y fechas | Quién diseña qué, y fecha de entrega de los fundamentos y de las vistas críticas |

**Entregables de la reunión:** acta con las decisiones, orden de construcción de componentes y responsables de cada vista.

## 14. Trazabilidad

| Módulo | Requisitos | Vistas |
|---|---|---|
| Autenticación | RF-AUT-01 a 07 | V01–V05 |
| Conversaciones | RF-CNV-01 a 16 | L01, V10, V11, V12, V50 |
| Contactos | RF-CON-01 a 09 | V10, V12, V20, V21, V22 |
| Estados | RF-EST-01 a 04 | V10, V11, V12, V20, V21 |
| Métricas | RF-MET-01 a 08 | V30 |
| Usuarios | RF-USR-01 a 08 | V02, V40, V41, V42 |
| Configuración | RF-CFG-01 a 03 | V50 |
| Transversales | RF-TRV-01 a 08 | Todas |

Total: 63 requisitos funcionales y 18 vistas.
