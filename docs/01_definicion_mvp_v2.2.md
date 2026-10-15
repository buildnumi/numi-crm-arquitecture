# Numi CRM — Definición del MVP

*Alcance, arquitectura y plan hacia el 30 de noviembre de 2026 · Franco y Anthony · Versión 2.2 · 14 de octubre de 2026*

## 1. Resumen ejecutivo

Numi CRM es una nueva línea de negocio de Numi AI: un CRM conversacional multiempresa, vendido como servicio (SaaS) en Colombia. El primer MVP tiene un único objetivo: **validar con negocios reales que un equipo puede atender, organizar y medir sus conversaciones de WhatsApp desde un solo lugar.**

- **Meta:** MVP terminado el **30 de noviembre de 2026**.
- **Equipo:** dos socios, Franco y Anthony. Ambos desarrollan el producto.
- **Canal:** solo WhatsApp, mediante la API oficial de Meta (WhatsApp Business Platform, Cloud API).
- **Alcance:** cinco capacidades: conversaciones, contactos, estado de los contactos, métricas básicas y usuarios con permisos. Además, una **consola de plataforma** para que el equipo de Numi administre las organizaciones.
- **Validación:** piloto gratuito de una semana con clínicas estéticas (y, de forma secundaria, inmobiliarias) después de terminar el MVP.

Este documento toma como base el brief de la reunión y resuelve sus contradicciones para que pueda usarse como especificación del desarrollo. Cada vez que algo no estaba definido en el brief y fue necesario definirlo, se indica como **definición propuesta**. Lo que sigue pendiente de decisión aparece en la sección 12.

**Documentos que lo acompañan:** el *Documento maestro del MVP v1.1* (requisitos funcionales y vistas del CRM) y el anexo *Consola de plataforma v1.1* (requisitos y vistas de la consola). Si hay diferencias de detalle, mandan esos dos documentos.

## 2. Problema

Los negocios que atienden clientes por mensajería lo hacen hoy de forma desorganizada. El brief identifica estos problemas:

- Las conversaciones están dispersas en distintos canales, sin un control estándar.
- No hay control sobre los contactos ni sobre el estado de cada uno.
- El seguimiento de los leads falla.
- Hay poca atención personalizada fuera del horario laboral.
- Se repiten muchas preguntas simples que nadie automatiza.
- Faltan métricas clave: cantidad de conversaciones, leads cerrados, indicadores del negocio.

Estos problemas son hipótesis de la reunión. Se confirman o se descartan con el piloto (sección 11).

## 3. Segmento y usuarios

| Aspecto | Definición |
|---|---|
| Segmento inicial | **Clínicas estéticas.** Es el segmento que aparece tanto en la lista de prioridades como en la de validación del brief |
| Segmento secundario de validación | Inmobiliarias (*real estate*), que el brief también incluye para la prueba |
| Segmentos posteriores | Odontologías y veterinarias |
| Volumen esperado | 500 conversaciones al mes por negocio |
| Usuarios diarios | **Asesores**, que atienden a los clientes, y **administradores**, que supervisan las conversaciones |
| Quién decide la compra | El gerente o dueño del negocio |

**Por qué un solo segmento inicial:** el brief enumera cuatro segmentos en la sección de público objetivo y dos en la de validación. Diseñar para todos a la vez dispersa el MVP. Se prioriza uno y los demás se atienden si el piloto lo justifica.

## 4. Propuesta de valor

**Para el MVP:** reunir en un único lugar las conversaciones de WhatsApp del negocio, con el estado de cada contacto y métricas básicas, para que el equipo atienda más rápido y el gerente decida con datos.

El brief también menciona la integración con los sistemas que el negocio ya usa, la automatización de tareas simples y la centralización de varios canales. Esas promesas **no se comprueban en el MVP**, porque exigen funciones que el mismo brief deja fuera (otros canales, integraciones y automatizaciones). Se mantienen como dirección del producto, no como promesa del primer piloto.

Lo que el negocio debe poder hacer mejor con el MVP:

- Mejorar la atención al usuario.
- Reducir el tiempo de respuesta.
- Tomar decisiones con métricas claras.

## 5. Alcance funcional del MVP

### 5.1 Conversaciones

El brief define que todo se gestiona desde el CRM: los asesores ven y responden las conversaciones sin salir de la plataforma.

- Recibir mensajes de texto de WhatsApp.
- **Enviar mensajes de texto desde el CRM.** El brief lo menciona como "si se confirma"; aquí se confirma, porque sin respuesta dentro del CRM no se cumple "todo se manejará desde el CRM". El envío respeta la regla de Meta de la ventana de 24 horas: si el cliente no ha escrito en las últimas 24 horas, el CRM bloquea el envío y explica el motivo.
- Ver la lista de conversaciones y el historial de cada una. Los mensajes nuevos aparecen sin recargar la página.
- Buscar y filtrar conversaciones: por nombre o teléfono del contacto, por responsable, por estado del contacto y por conversaciones sin asignar.
- Asignar y reasignar un responsable. Si un asesor responde una conversación sin asignar, queda como su responsable.
- Ver el estado de cada mensaje enviado: enviando, enviado, entregado, leído o error, según lo informe Meta.

**Asociación y asignación (aclaración del brief):** el brief usa una sola frase para dos cosas distintas.

- **Asociación contacto–conversación:** cada conversación pertenece a un contacto, identificado por su número de teléfono dentro del negocio. Un número nuevo crea un contacto nuevo.
- **Asignación a un asesor:** la conversación se asigna según la carga de trabajo de los asesores. Una conversación nueva se asigna al asesor activo con menos conversaciones abiertas, y un administrador puede reasignarla. El administrador puede desactivar esta asignación automática y hacerla manual.

**Estado de la conversación (definición propuesta):** el brief pide separar el estado del contacto del estado técnico de la conversación "si ambos son necesarios". Aquí se necesita uno mínimo, porque la carga de cada asesor se calcula con las conversaciones abiertas. La conversación puede estar **abierta** o **cerrada**. Un asesor o administrador la cierra; se reabre sola cuando el contacto vuelve a escribir.

### 5.2 Contactos

- **Datos mínimos:** identificador interno, nombre, teléfono, correo y estado. El responsable que se muestra en el contacto es el de su conversación.
- **Acciones:** crear, editar, ver el detalle, buscar y filtrar.
- El contacto se crea automáticamente cuando escribe un número nuevo, con estado *Nuevo*. También puede crearse a mano.
- El teléfono no puede repetirse dentro del mismo negocio.
- El contacto se reconoce por su identidad en cada canal. En el MVP la única identidad es el número de WhatsApp; el modelo admite varias para sumar canales después.
- La ficha del contacto no tiene campos de datos sensibles. **El contenido de las conversaciones sí puede tenerlos** (por ejemplo, datos de salud en clínicas), y se trata como tal (sección 8.4).

### 5.3 Estado del contacto

- **Estados iniciales:** Nuevo, En contacto, Calificado, Cliente, En pausa, Perdido (el brief lo escribe como "Perdido / No interesado").
- **Quién lo cambia:** asesores y administradores.
- **Visualización:** el estado actual se ve en la lista de conversaciones, en la lista de contactos y en el detalle del contacto.
- **Historial (definición propuesta):** cada cambio queda registrado con quién lo hizo, cuándo y de qué estado a cuál. Las métricas de conversión dependen de este registro.

### 5.4 Métricas

El brief enumera cinco métricas sin definirlas. Para que "los indicadores coincidan con los datos registrados" (criterio de la sección 7), cada una se define así:

| Métrica | Definición propuesta |
|---|---|
| Conversaciones por mes | Cantidad de conversaciones con al menos un mensaje durante el mes |
| Tiempo de atención | Minutos entre un mensaje entrante que espera respuesta y la primera respuesta de una persona, en minutos corridos. Se muestra la mediana como valor principal y el promedio como dato secundario, para que unos pocos mensajes nocturnos no distorsionen el indicador. Las respuestas automáticas no cuentan |
| Nuevos leads por mes | Contactos creados durante el mes. En el MVP todo contacto nuevo es un lead |
| Tasa de conversión | De los contactos creados en el período, el porcentaje que está en estado *Cliente* al momento de la consulta |
| Cantidad de contactos | Total de contactos, y desglose por estado |

El administrador ve estas métricas para todo el negocio y **por asesor**, como pide el brief en la sección de usuarios. El asesor no ve métricas.

### 5.5 Usuarios y permisos

Hay dos roles, con permisos fijos en el MVP:

| Acción | Asesor | Administrador |
|---|---|---|
| Ver y responder sus conversaciones y la cola sin asignar | Sí | Sí |
| Ver todas las conversaciones del negocio | No | Sí |
| Cambiar el estado de un contacto | Sí | Sí |
| Reasignar conversaciones | No | Sí |
| Ver métricas | No | Sí |
| Invitar, desactivar y gestionar usuarios | No | Sí |
| Configurar el negocio | No | Sí |

El brief dice que el asesor y el administrador actúan "según los permisos que tengan asignados", lo que sugiere permisos configurables. Eso no entra en el MVP: el permiso depende solo del rol.

**Gestión de usuarios (el brief lo dejó en blanco):** el administrador invita por correo electrónico y puede desactivar a un usuario. Un usuario desactivado no puede entrar. Los enlaces de invitación vencen a los 7 días y se usan una sola vez.

**Aislamiento entre negocios:** cada negocio solo accede a sus propios contactos, conversaciones y mensajes. Es la condición de seguridad que no se negocia (sección 8.3).

**Configuración mínima del negocio:** nombre, estado de la conexión con WhatsApp (solo lectura) e interruptor de asignación automática.

### 5.6 Consola de plataforma

Área interna desde la que el equipo de Numi (rol **Operador de plataforma**) administra todas las organizaciones. El detalle está en el anexo *Consola de plataforma v1.1*.

- Dar de alta una organización y generar el primer acceso de su administrador por invitación. Numi nunca conoce ni envía contraseñas.
- Configurar y diagnosticar la conexión de WhatsApp de cada organización.
- Gestionar los usuarios de cada organización.
- Activar o suspender una organización. Con la organización suspendida nadie entra ni envía mensajes, pero los mensajes entrantes se siguen guardando.
- Seguir el uso de cada organización (usuarios activos, mensajes, conversaciones sin responder) **sin acceso al contenido de las conversaciones**.
- Registrar cada acción del equipo de Numi en una auditoría.

El acceso a la consola exige doble factor.

## 6. Fuera del MVP

Queda fuera lo que el brief excluye y lo que no es necesario para probar el producto con clientes reales:

- Módulo de marketing y campañas masivas.
- Integración con Instagram, Messenger, correo y otras plataformas.
- Versión adaptada a móvil (*responsive*) y aplicación móvil.
- Agentes de IA y constructor de automatizaciones. La arquitectura se diseña para incorporarlos después.
- Plantillas de WhatsApp para escribir después de 24 horas, porque cada plantilla requiere aprobación de Meta.
- Reportes avanzados y paneles complejos.
- Campos personalizados, importación de contactos y permisos configurables.
- Visualización de audios, imágenes y documentos: en el MVP se muestra un marcador con el tipo de mensaje. Ver el punto abierto en la sección 12, porque puede afectar la adopción por parte de las clínicas.

Cualquier función que no esté en la sección 5 se considera fuera del alcance inicial.

## 7. Criterios para considerar el MVP listo

El MVP está listo cuando el flujo principal funciona de extremo a extremo, de forma segura, y permite probar la hipótesis con usuarios reales. Los criterios del brief, con la forma de verificarlos:

| Área | Criterio mínimo | Cómo se verifica |
|---|---|---|
| Conversaciones | Un asesor recibe, responde y gestiona conversaciones de WhatsApp dentro del CRM | Un mensaje enviado desde un celular aparece en la bandeja en pocos segundos; la respuesta llega al celular; fuera de la ventana de 24 horas el envío se bloquea con aviso |
| Contactos | Se puede crear, editar, buscar y abrir el detalle de un contacto | Un número nuevo genera un solo contacto; un duplicado dentro del negocio es rechazado |
| Estados | Se puede cambiar el estado y ver claramente el estado actual | El cambio se refleja en la lista y en el detalle, y queda en el historial |
| Métricas | Los indicadores coinciden con los datos registrados | Cada métrica se compara con un cálculo manual sobre datos de prueba |
| Usuarios | Se pueden gestionar usuarios y restringir el acceso según rol y negocio | Un asesor no ve métricas ni gestiona usuarios; un usuario desactivado no puede entrar; la invitación funciona |
| Seguridad básica | Un negocio no puede acceder a los datos de otro | Prueba automática con dos negocios de prueba: ninguna consulta cruza el límite |
| Consola de plataforma | El equipo de Numi da de alta una organización, entrega el primer acceso y la suspende o reactiva | Alta de una organización de prueba de punta a punta: invitación aceptada, WhatsApp conectado, primer mensaje recibido, suspensión y reactivación |
| Operación | El sistema está desplegado en producción con entornos separados de desarrollo y producción | Despliegue completado y probado de punta a punta con un número real de WhatsApp |

## 8. Arquitectura y decisiones técnicas

### 8.1 Canal

- WhatsApp mediante la **WhatsApp Business Platform (Cloud API)** y sus webhooks. Integración directa con la API oficial, sin automatizaciones no oficiales.
- El mismo diseño debe permitir incorporar después otros canales (Instagram, Messenger, correo).
- El envío debe cumplir las reglas y restricciones de Meta, en particular la ventana de 24 horas.
- Antes de comprometer la integración hay que confirmar permisos, requisitos, políticas y costos de la API. Esto incluye la **verificación del negocio en Meta** y el número de WhatsApp de cada cliente, que no dependen del equipo y pueden tardar.
- **Modelo de conexión del número (por decidir):**
  - **Coexistencia:** el negocio sigue usando la app de WhatsApp Business en el mismo número y el CRM recibe también los mensajes que se envían desde el celular. Exige que Numi sea Tech Provider de Meta (o trabaje con un proveedor que ofrezca coexistencia) y use el registro integrado de Meta.
  - **Número solo en la API:** el número deja de usarse en la app del celular y todo se atiende desde el CRM.
  - La elección cambia el alta de cada negocio, lo que se ve en la bandeja y la facilidad para que una clínica acepte el piloto (sección 12).

### 8.2 Stack propuesto en el brief

| Capa | Tecnología | Motivo |
|---|---|---|
| Lenguaje | Node.js con TypeScript | Tipado estático para un producto que va a evolucionar |
| Backend | Fastify, ejecutado en Cloudflare Workers con runtime compatible con Node.js | API modular y ligera, sin servidores que administrar |
| Frontend | Next.js en Cloudflare Workers mediante OpenNext | Mismo proveedor que el backend |
| Base de datos | PostgreSQL administrado (por ejemplo Supabase), conectado con Cloudflare Hyperdrive | Base relacional con políticas de seguridad por fila |
| Autenticación | Supabase Auth, más validación de permisos en el backend | Autenticación probada; la autorización no depende solo del frontend |
| Archivos | Cloudflare R2, cuando haya adjuntos | Almacenamiento barato; no se usa si los adjuntos quedan fuera |
| Procesamiento asíncrono | Cloudflare Queues para webhooks y mensajes | El webhook responde rápido y el procesamiento no bloquea la API |
| Código y despliegue | GitHub, con entornos de desarrollo y producción | Control de versiones y despliegues separados |

La arquitectura es serverless, con pocos costos fijos y un backend modular, no microservicios.

**Puntos técnicos por resolver antes de programar:**

1. **Fastify en Workers.** Cloudflare indica que Express y Koa ya funcionan en Workers, pero Fastify todavía no. Los caminos son: ejecutar Fastify en un servicio Node.js común y dejar Cloudflare para la interfaz; usar Express en Workers; o usar un framework hecho para Workers, como Hono. Todos mantienen TypeScript y el resto del stack.
2. **Tiempo real.** Definir cómo llegan los mensajes nuevos a la pantalla sin recargar. La opción más simple es que la interfaz se suscriba a Supabase Realtime, lo que exige políticas de seguridad por fila correctas desde el primer día.
3. **Seguridad por fila en la API.** Si la API se conecta a la base con un usuario privilegiado, las políticas de seguridad por fila no la protegen. Hay que elegir un patrón: aplicar la identidad del usuario en cada transacción, o filtrar siempre por organización con pruebas automáticas de aislamiento.
4. **Correo propio.** El servidor de correo por defecto de Supabase no es para producción. Las invitaciones y los restablecimientos de contraseña necesitan un servidor de correo propio.

Se deciden en la primera semana, con una prueba corta que reciba un webhook de WhatsApp de punta a punta y lo muestre en tiempo real.

### 8.3 Aislamiento de datos entre negocios

Arquitectura multiempresa (*multi-tenant*) con una base de datos PostgreSQL compartida:

- Cada negocio tiene un registro único identificado por `organization_id`.
- Los usuarios pertenecen a una o varias organizaciones mediante una relación de membresía. El MVP solo necesita que un usuario pertenezca a una; el modelo de datos ya permite varias.
- Contactos, conversaciones, mensajes, asignaciones y demás entidades pertenecen a una organización.
- El backend obtiene la organización autorizada desde la sesión y la membresía del usuario. No confía en el `organization_id` que envía el frontend.
- Se valida la autorización en cada operación y se aplican políticas de seguridad por fila (*Row-Level Security*, RLS) en PostgreSQL.
- Las credenciales de integraciones como WhatsApp se asocian a la organización y se guardan cifradas, nunca en el frontend ni en los registros.
- El rol **Operador de plataforma** no pertenece a ninguna organización. Las acciones de la consola se ejecutan en el servidor, que verifica ese rol en cada petición; el navegador nunca recibe permisos para leer datos de todas las organizaciones.
- Cada petición de un webhook se valida con la firma que envía Meta, y se guarda el identificador del mensaje de Meta como clave única para que un reintento no duplique datos.

Se evita una base de datos por cliente para reducir costos. Si algún cliente requiere aislamiento dedicado, el diseño permite evaluarlo después.

### 8.4 Datos que se guardan y retención

| Tipo de dato | Datos almacenados | Retención propuesta |
|---|---|---|
| Usuarios | Nombre, correo, rol y membresías | Mientras la cuenta esté activa; eliminación o anonimización según obligaciones aplicables |
| Contactos | Nombre, teléfono, correo, estado y fechas de actividad | Mientras el negocio mantenga el contacto y exista una finalidad legítima |
| Conversaciones | Canal, contacto, responsable, estado y fechas | 12 meses como período inicial, sujeto a validación |
| Mensajes | Contenido, dirección, identificadores externos y marcas de tiempo | 12 meses como período inicial, sujeto a validación |
| Adjuntos | Archivos asociados a mensajes | El mismo período del mensaje; solo si los adjuntos entran en el producto |
| Métricas | Volúmenes, tiempos de respuesta y conversiones | 12 meses en detalle; los datos agregados pueden conservarse más tiempo |
| Auditoría | Accesos relevantes y cambios de estado o permisos | 6 a 12 meses como propuesta inicial |
| Credenciales de integraciones | Tokens y referencias de conexión | Mientras la integración esté activa, con revocación al desconectarla |

La retención definitiva se fija según la finalidad del tratamiento, los acuerdos con cada negocio y los requisitos legales. En Colombia aplica la **Ley 1581 de 2012** de protección de datos personales. Las conversaciones de clínicas pueden contener datos de salud, que la ley trata como **datos sensibles**. El negocio es el responsable del tratamiento y Numi actúa como encargado. Antes de usar datos reales de un cliente se necesitan una política de tratamiento de datos y un acuerdo de encargo con cada negocio, que conviene revisar con un abogado. El contenido de las conversaciones solo lo ven los usuarios del negocio: nunca aparece en la consola de plataforma, en los registros técnicos ni en las métricas. Se implementan políticas de eliminación, exportación y atención de solicitudes de los titulares, y las copias de seguridad tienen un período de retención definido.

## 9. Equipo y responsabilidades

Franco y Anthony desarrollan el producto. No hay otros socios ni personas en el proyecto.

| Área | Responsables | Propuesta de reparto (a confirmar) |
|---|---|---|
| Producto, especificación y decisiones | Franco y Anthony | Decisiones tomadas entre los dos; cada decisión queda escrita |
| Desarrollo | Franco y Anthony | **Franco:** integración con WhatsApp (webhook y envío), base de datos y aislamiento de datos. **Anthony:** interfaz (bandeja, contactos, estados). Métricas y usuarios se reparten según el avance |
| Pruebas | Franco y Anthony | Cada uno prueba lo que no desarrolló; las pruebas de aislamiento se automatizan |
| Verificación del negocio en Meta y número de WhatsApp | Franco y Anthony | Por asignar; se inicia en la primera semana |
| Conseguir y acompañar a los negocios del piloto | Franco y Anthony | Por asignar; el brief propone redes sociales, referidos y *cold outreach* |
| Política de datos y acuerdos con los negocios | Franco y Anthony | Por asignar; se recomienda revisión legal externa |

El reparto de desarrollo es una propuesta que depende de la experiencia y la disponibilidad de cada uno; se ajusta en la primera reunión de trabajo. Con solo dos personas desarrollando, cualquier ausencia prolongada afecta directamente la fecha.

## 10. Cronograma hacia el 30 de noviembre

Las fechas son una propuesta calculada hacia atrás desde la meta del 30 de noviembre. No son una estimación medida: se ajustan una vez estimado el esfuerzo real de cada bloque.

| Fechas | Actividad | Entregable |
|---|---|---|
| 12–16 oct | Cerrar las decisiones pendientes (sección 12). Crear repositorio, entornos y proyecto de base de datos. Iniciar la verificación del negocio en Meta. Definir los negocios del piloto. Reunión de arranque del design system | Repositorio y entornos listos; trámite de Meta iniciado; inventario de vistas aprobado |
| 19–23 oct | **Diseño:** fundamentos visuales sobre los componentes de la interfaz, y bocetos de bandeja con conversación, lista de organizaciones y alta de organización. En paralelo con el inicio de la base | Fundamentos y vistas críticas diseñados |
| 19–30 oct | **Base y canal.** Autenticación, organizaciones, roles (incluido el de plataforma), políticas de acceso, alta de organizaciones desde la consola, recepción de mensajes de WhatsApp, creación de contactos, bandeja básica con tiempo real | Una organización creada desde la consola recibe un mensaje de un celular en su bandeja, sin duplicados |
| 2–13 nov | **Bandeja completa.** Envío de texto con control de ventana de 24 horas, asignación, estado de conversación, búsqueda y filtros, contactos | Un asesor atiende una conversación de principio a fin |
| **16 nov** | **Revisión de avance** | Decisión sobre recortes, si hace falta |
| 16–25 nov | **Estados, métricas, usuarios y consola.** Historial de estados, métricas, invitación y desactivación de usuarios, configuración mínima. En la consola: detalle de organización, usuarios, WhatsApp, suspensión y auditoría | El administrador ve métricas y gestiona usuarios; el equipo de Numi gestiona organizaciones |
| 26–30 nov | **Cierre.** Pruebas de aislamiento y de punta a punta, despliegue a producción, prueba con un número real | **MVP terminado el 30 de noviembre** |

**Revisión del 16 de noviembre:** si el avance no alcanza, el alcance se recorta y la fecha no se mueve. Recortes sugeridos, en este orden: vistas opcionales de la consola (panel general y auditoría global), métrica por asesor, asignación automática (pasa a manual), búsqueda avanzada. **No se recortan** el aislamiento entre negocios, la recepción de mensajes, el envío de texto ni el alta de organizaciones desde la consola.

**Dependencia externa:** el piloto necesita que el negocio de Numi y el número de WhatsApp de cada cliente estén verificados por Meta. Ese trámite no depende del equipo; por eso empieza la primera semana.

## 11. Validación en el mercado

| Aspecto | Definición |
|---|---|
| Negocios participantes | Clínicas estéticas e inmobiliarias; el brief no fija cuántos. Se propone **2 a 3 negocios**, con prioridad en clínicas |
| Cómo se consiguen | Redes sociales, referidos y *cold outreach*. Se empieza a contactarlos desde ya, para tenerlos listos al terminar el MVP |
| Duración | **Una semana (5 días hábiles)**, como define el brief, a partir de la semana del 1 de diciembre de 2026 |
| Precio | Gratis, con la condición de dar feedback o llegar a un acuerdo |
| Qué se registra | Uso real, fallos, peticiones repetidas y disposición a continuar o pagar |

**Criterio para continuar, ajustar o detener.** El brief dice que se sigue "si nos dan buen feedback". El brief mismo sugiere que la señal más valiosa no es que la idea guste, sino que el negocio la use en su operación y esté dispuesto a continuar o pagar. Se propone:

| Decisión | Condición |
|---|---|
| **Continuar** | Los negocios usaron el CRM en su operación real durante el piloto, al menos uno acepta continuar con un acuerdo o un precio, y no hubo pérdida de mensajes ni fallo de aislamiento |
| **Ajustar** | Hubo uso, pero nadie quiere continuar, o se repite una petición que cambia el producto |
| **Detener** | Los asesores siguen atendiendo desde el celular o el problema no resultó real |

**Limitación:** una semana y 2–3 negocios dan una señal inicial, no una conclusión estadística. Si una semana resulta corta para ver uso sostenido, el piloto puede extenderse; se decide al cierre de la primera semana.

## 12. Riesgos y puntos abiertos

| Punto | Detalle | Qué se necesita |
|---|---|---|
| Esfuerzo frente a la fecha | Dos personas, un stack con varias piezas y siete semanas. Con la consola, el MVP suma 110 requisitos funcionales indispensables (63 del CRM y 47 de la consola) y 29 vistas. El esfuerzo no está medido | Estimar cada bloque en la primera semana; confirmar las horas semanales de cada socio; decidir si la consola entra completa o en una versión mínima |
| Modelo de conexión de WhatsApp | Con número solo en la API, la clínica deja de usar la app en esa línea: no puede ver audios ni imágenes fuera del CRM ni escribir pasadas 24 horas, y es difícil que acepte mudar su línea principal para un piloto gratis de una semana. La coexistencia evita eso, pero exige ser Tech Provider de Meta o usar un proveedor | Decidir el modelo antes del 16 de octubre (sección 8.1) |
| Complejidad del stack | El stack del brief usa varios servicios de Cloudflare más una base externa. Un stack más simple ahorraría tiempo de montaje y operación, a costa de apartarse de lo acordado | Decidir en la primera semana si se mantiene el stack del brief; ver el punto técnico de Fastify en la sección 8.2 |
| Verificación de Meta | No depende del equipo y puede tardar semanas | Iniciar el trámite esta semana y preparar los documentos del negocio |
| Audios, imágenes y documentos | Los clientes de WhatsApp envían notas de voz e imágenes con frecuencia. En el MVP se muestra un marcador con el tipo de mensaje; con número solo en la API, la clínica no tiene otro lugar donde verlos | Decidir antes del 26 de octubre, junto con el modelo de conexión, si su visualización entra en el MVP |
| Costos | Costo de la API de WhatsApp por conversación, plan de la base de datos, hosting y números de WhatsApp | Verificar los precios vigentes antes de fijar un precio al cliente |
| Aspectos legales | Ley 1581 de 2012: las conversaciones pueden contener datos sensibles y Numi actúa como encargado del tratamiento | Redactar la política de datos y el acuerdo de encargo antes de usar datos reales; revisión legal recomendada |
| Piloto en temporada de festividades | Diciembre puede alterar el volumen de conversaciones | Tenerlo en cuenta al interpretar los resultados |
| Alternativa de validar antes de construir | Existen CRM de código abierto, como Chatwoot, que podrían probar la hipótesis con WhatsApp sin construir nada. No se verificó su compatibilidad con la Cloud API ni su costo | Decisión de los socios; no cambia este documento mientras no se tome |

## 13. Acuerdos y próximos pasos

El brief deja en blanco los acuerdos finales. Esta es la propuesta para cerrarlos:

| Acuerdo o tarea | Responsable | Fecha objetivo |
|---|---|---|
| Confirmar el segmento inicial (clínicas estéticas) | Franco y Anthony | 16 oct |
| Aprobar el alcance de la sección 5 y lo que queda fuera (sección 6) | Franco y Anthony | 16 oct |
| Decidir el modelo de conexión de WhatsApp (coexistencia o número solo en la API) | Franco y Anthony | 16 oct |
| Decidir si la consola entra completa o en versión mínima | Franco y Anthony | 16 oct |
| Resolver los puntos técnicos de la sección 8.2 y confirmar el stack | Franco y Anthony | Primera semana de desarrollo |
| Confirmar el reparto de desarrollo y las horas semanales de cada uno | Franco y Anthony | 16 oct |
| Crear repositorio, entornos y proyecto de base de datos | Franco y Anthony | 16 oct |
| Iniciar la verificación del negocio en Meta | Franco y Anthony | 16 oct |
| Definir la lista de negocios para el piloto | Franco y Anthony | 23 oct |
| Decidir si audios e imágenes entran al MVP | Franco y Anthony | 26 oct |

**Resumen del MVP aprobado:** CRM conversacional multiempresa para WhatsApp, con bandeja para recibir y responder mensajes, contactos con estado e historial, cinco métricas básicas y dos roles (asesor y administrador), con aislamiento total entre negocios, y una consola de plataforma para que el equipo de Numi dé de alta y administre las organizaciones. Se construye entre Franco y Anthony y debe estar terminado el 30 de noviembre de 2026, para un piloto gratuito de una semana con clínicas estéticas.

---

*Los esfuerzos, fechas y reparto de tareas de este documento son propuestas, no mediciones. Los costos de WhatsApp, de la base de datos y del hosting, y las condiciones de la API de Meta, deben verificarse con los precios y políticas vigentes antes de tomar decisiones.*
