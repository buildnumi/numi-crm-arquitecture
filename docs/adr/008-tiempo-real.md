# ADR-008: Tiempo real en la bandeja

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-CNV-15, RF-TRV-03, RF-TRV-01, RF-TRV-07, RF-PLT-SEG-01
- **Depende de:** ADR-003, ADR-005, ADR-006, ADR-007
- **Condiciona a:** D-09 (frontend), D-12 (envío de mensajes)

## Contexto

`RF-CNV-15` exige que *"los cambios de asignación, estado y mensajes nuevos se reflejan en tiempo real en las pantallas de los usuarios afectados"*, y que la navegación y el título de la pestaña muestren el número de conversaciones sin responder. `RF-TRV-03` pone el número: **un mensaje entrante aparece en la bandeja en 5 segundos o menos**.

La definición del MVP ya marcó el problema en §8.2 punto 2:

> "Tiempo real. Definir cómo llegan los mensajes nuevos a la pantalla sin recargar. La opción más simple es que la interfaz se suscriba a Supabase Realtime, lo que exige políticas de seguridad por fila correctas desde el primer día."

Y dos decisiones anteriores dejaron el mismo punto anotado como pendiente de resolver aquí:

- **ADR-006:** *"el tiempo real de Supabase tiene políticas propias, aparte del RLS de las tablas. Hay que verificar ese aislamiento por separado"*.
- **ADR-007:** *"el tiempo real de Supabase va directo del navegador a la base y **no pasa por esta verificación**"*.

Esa es la tensión central de esta decisión. ADR-007 construyó una regla fuerte —**el token identifica, la API autoriza en cada petición**— y la ruta obvia de tiempo real la esquiva por completo.

### Restricciones que aplican

- Runtime: Hono en Cloudflare Workers (ADR-003). **No hay procesos largos**: un Worker no puede sostener conexiones WebSocket abiertas por su cuenta.
- El consumidor de la cola (ADR-005) es quien tiene el mensaje guardado y listo. Es el punto natural desde donde notificar.
- `RF-PLT-SEG-01`: el navegador nunca recibe permisos para leer datos de todas las organizaciones.
- `RF-TRV-07`: el contenido de las conversaciones solo lo ven los usuarios del negocio según su rol.
- Volumen: menos de 1 mensaje por segundo; en el piloto, unos pocos asesores conectados por organización (ADR-003).
- Dos personas con tiempo parcial.

### Datos verificados de Supabase

Verificados el 2026-10-15 contra la documentación oficial.

**Recomendación del proveedor** — [Subscribing to Database Changes](https://supabase.com/docs/guides/realtime/subscribing-to-database-changes)

| Dato | Valor |
| --- | --- |
| Qué recomienda Supabase | **Broadcast**, "the recommended method for **scalability and security**" |
| Recomendación general | "We recommend using Broadcast for most use cases" |
| Postgres Changes | "does not scale as well as Broadcast"; tiene limitaciones a medida que la aplicación crece |

**Postgres Changes** — [Realtime Limits](https://supabase.com/docs/guides/realtime/limits), [Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks), [Troubleshooting](https://supabase.com/docs/guides/troubleshooting/realtime-postgres-changes-troubleshooting)

| Dato | Valor |
| --- | --- |
| Autorización | **Por suscriptor.** 100 suscriptores a una tabla significan que un solo `INSERT` dispara **100 lecturas** |
| Procesamiento | **Un solo hilo**, para mantener el orden de los cambios. Mejorar el cómputo no ayuda mucho |
| Si la autorización es lenta | "the changes will be delayed until you receive a timeout" |
| Fallo típico | RLS es "the single biggest source of **silently missing** events". `SUBSCRIBED` solo significa que el WebSocket conectó, no que el cliente tenga permiso de ver los datos |
| Informes de tiempo de RLS | Solo en planes Pro, Team y Enterprise |

**Autorización del canal** — [Realtime Authorization](https://supabase.com/docs/guides/realtime/authorization)

| Dato | Valor |
| --- | --- |
| Dónde viven las políticas | En la tabla `realtime.messages`, **que no almacena mensajes**: solo define reglas de acceso |
| Canales privados | Requieren desactivar "Allow public access" y crear el canal con `private: true` |
| Caché de permisos | **Se cachean durante toda la vida de la conexión** |
| Revocación a mitad de conexión | El cliente sigue recibiendo mensajes "until the JWT expires or a new one is sent" |
| Al expirar el token | "the client will be disconnected when the JWT expires" |

**Broadcast desde el servidor** — [Broadcast](https://supabase.com/docs/guides/realtime/broadcast)

| Dato | Valor |
| --- | --- |
| Envío por HTTP | `POST` a `/realtime/v1/api/broadcast/{topic}/events/{event}` con cabecera `apikey`. "receives an HTTP request and then sends a message via WebSocket to connected clients" |
| Canales privados | Se apunta con `?private=true` |
| Envío por lotes | Hay un endpoint que acepta varios mensajes en una petición |
| Claves | `anon` y `service_role` quedan obsoletas a favor de claves publicables y secretas a fines de 2026 |

**Límites y costo** — [Realtime Limits](https://supabase.com/docs/guides/realtime/limits), [Realtime Pricing](https://supabase.com/docs/guides/realtime/pricing)

| Dato | Plan gratuito | Plan Pro |
| --- | --- | --- |
| Conexiones concurrentes | 200 | 500 |
| Mensajes incluidos | 2.000.000, sin cargo por exceso | 5.000.000, luego 2,50 USD por millón |
| Uniones a canal | 100 por segundo | 500 por segundo |
| Proyectos gratuitos | **Se pausan tras una semana de inactividad** | — |

### Los dos hechos que deciden

**Primero: hay un endpoint HTTP de Broadcast.** Esto resuelve el problema estructural de ADR-003. Un Worker no puede sostener un WebSocket, pero **sí puede hacer una petición HTTP**. El consumidor de la cola, que ya tiene el mensaje guardado, puede notificar con una llamada y terminar.

**Segundo, y es el que importa: con Broadcast desde el servidor, el navegador solo escucha.** No consulta la base, no lee filas, no necesita permisos de lectura sobre ninguna tabla. Recibe la carga que el servidor decidió enviarle.

Eso disuelve la tensión que arrastraban ADR-006 y ADR-007. No hace falta que el canal de tiempo real replique la verificación de autorización del backend, porque **el canal ya no es una vía de acceso a datos**: es un aviso. La API sigue siendo el único camino hacia los datos, tal como exige `RF-PLT-SEG-01`.

Con Postgres Changes ocurre lo contrario: el navegador se suscribe a una tabla y recibe filas directamente desde la base. Ahí sí el canal es una vía de acceso, y ahí sí haría falta que RLS lo proteja con la misma solidez que la API —con la diferencia de que **los permisos se cachean durante toda la conexión**, así que una desactivación de usuario no surtiría efecto hasta que el token expire. Es exactamente el problema que ADR-007 resolvió verificando en cada petición, reintroducido por otra puerta.

Hay un tercer argumento, menos obvio pero práctico: con Postgres Changes, un fallo de política **no produce un error, produce silencio**. La documentación lo dice: es la mayor fuente de eventos que desaparecen sin aviso, y `SUBSCRIBED` no significa que el cliente tenga permiso. Depurar "a veces no llega el mensaje" en noviembre, con el piloto encima, es un escenario que conviene no tener.

## Opciones consideradas

### Opción A: Postgres Changes, el navegador se suscribe a las tablas

El frontend se suscribe a cambios en las tablas de mensajes y conversaciones; RLS filtra.

- **Ventajas:** no hay que escribir nada en el backend; es la opción que §8.2 punto 2 llama "la más simple"; el cambio llega sin intermediarios.
- **Desventajas:** **el navegador recibe filas de la base directamente**, lo que contradice el espíritu de `RF-PLT-SEG-01` y la regla de ADR-007; los permisos se cachean durante toda la conexión, así que una desactivación o una suspensión no tienen efecto inmediato —rompiendo `RF-AUT-07`—; la autorización es por suscriptor, de modo que cada mensaje nuevo dispara tantas lecturas como asesores conectados; se procesa en un solo hilo; un fallo de política produce **silencio**, no error; el propio Supabase dice que **no escala tan bien** y recomienda otra cosa; expone la forma de las tablas al cliente, atando el esquema a la interfaz.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo, **riesgo alto** sobre requisitos que no se negocian.
- **Veredicto:** es la opción que el brief sugería, y la que el proveedor desaconseja explícitamente.

### Opción B: Broadcast desde el servidor, el navegador solo escucha

El consumidor de la cola (ADR-005), tras guardar el mensaje, hace una petición HTTP al endpoint de Broadcast sobre un canal privado por organización. El navegador está suscrito a ese canal y, al recibir el aviso, **pide los datos a la API**.

- **Ventajas:** el navegador nunca lee la base, así que `RF-PLT-SEG-01` y la regla de ADR-007 se mantienen intactas; la API sigue siendo el único camino a los datos, con su verificación de usuario, membresía y estado en cada petición; es lo que Supabase recomienda "for scalability and security"; un mensaje nuevo produce **una** notificación, no una lectura por suscriptor; el esquema de la base no queda expuesto al cliente; funciona desde Workers, que es lo que ADR-003 permite; el aviso puede ser mínimo, sin contenido sensible.
- **Desventajas:** hay que escribir la notificación en el consumidor y el manejo del canal en el frontend; el aviso y la carga de datos son dos pasos, lo que agrega una ida y vuelta —medible, pero muy por debajo de los 5 segundos—; si la notificación falla, el cambio no se ve hasta la siguiente acción, así que conviene una red de seguridad; hay que definir las políticas de `realtime.messages` para los canales privados.
- **Esfuerzo, costo y riesgo:** esfuerzo medio. Riesgo bajo.

### Opción C: Consulta periódica, sin tiempo real

La bandeja consulta la API cada pocos segundos.

- **Ventajas:** lo más simple de todo; sin canal, sin políticas nuevas, sin dependencia adicional; se comporta igual ante fallos; fácil de depurar.
- **Desventajas:** para cumplir los 5 segundos de `RF-TRV-03` habría que consultar cada 3 o 4 segundos, lo que con varios asesores conectados significa consultas constantes a la base durante toda la jornada, la mayoría devolviendo nada; gasta recursos en proporción al tiempo conectado y no al tráfico; el contador del título de la pestaña (`RF-CNV-15`) parpadearía con el ritmo de la consulta; es una experiencia perceptiblemente peor en un producto cuya promesa es la conversación.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo, costo de base de datos desproporcionado al volumen real.
- **Veredicto:** sirve como red de seguridad, no como mecanismo principal.

## Decisión

**Opción B: Broadcast desde el servidor**, con una consulta periódica lenta como red de seguridad.

Razón, en orden de peso:

1. **Mantiene intacta la regla de ADR-007.** El navegador no lee la base; la API autoriza en cada petición. Postgres Changes reintroduciría por otra puerta el problema que ADR-007 resolvió: permisos cacheados que no se pueden revocar.
2. **Es lo que el proveedor recomienda**, por escalabilidad y por seguridad. Ir contra la recomendación explícita del proveedor necesitaría una razón, y aquí no la hay.
3. **El modo de fallo es mejor.** Postgres Changes falla en silencio; una notificación que no llega se compensa con la red de seguridad y es visible en los registros.
4. **Funciona desde Workers.** El endpoint HTTP encaja con lo que ADR-003 permite, sin sostener conexiones.

### Cómo funciona

1. **Quien escribe** guarda el mensaje, la conversación y la asignación.
2. Hace un `POST` al endpoint de Broadcast, sobre un **canal privado por organización**, con un aviso mínimo.
3. El navegador, suscrito a ese canal, recibe el aviso.
4. El navegador **pide los datos a la API**, que verifica usuario, membresía y estado (ADR-007) y responde solo lo que corresponde.

El paso 4 es el que hace que todo lo demás se sostenga: el aviso no es el dato.

### Quién publica el aviso

Hay **dos escritores** en el sistema, y los dos notifican con el mismo mecanismo:

| Escritor | Qué provoca el aviso | Decisión |
| --- | --- | --- |
| **El consumidor de la cola** | Mensaje entrante; cambio de estado que llega por webhook (entregado, leído, error) | ADR-005 |
| **La API, en el camino de salida** | Mensaje saliente de un asesor; asignación derivada de ese envío (`RF-CNV-11`); cambio de estado tras la respuesta de Meta | ADR-012 |

Esto importa porque **`RF-CNV-15` no distingue el origen**: exige que los mensajes nuevos y los cambios de asignación se reflejen en las pantallas de los usuarios afectados. Un mensaje que el asesor A envía es un mensaje nuevo para el asesor B y para el administrador que mira la misma conversación, y una conversación que A se autoasigna al responder desaparece de la lista de sin asignar que B tiene abierta. Si solo notificara el consumidor, esos dos casos dependerían de la consulta de 60 segundos, que es cuatro veces el objetivo de `RF-TRV-03`.

El envío de ADR-012 es síncrono, así que publica **dos veces**: una tras guardar el mensaje con estado `enviando` (paso 6), y otra al actualizar el estado con la respuesta de Meta (paso 8). Son dos avisos baratos —no llevan contenido— y evitan que el resto del equipo vea el mensaje colgado en `enviando`.

**La regla 2 de más abajo aplica igual a este escritor:** la notificación ocurre después de la escritura, fuera de la transacción, y un fallo al notificar no falla el envío. El asesor que envió ya tiene su respuesta por HTTP; el aviso es para los demás.

### Qué lleva el aviso, y qué no

**El aviso no transporta contenido de conversaciones.** Lleva lo mínimo para que el cliente sepa qué recargar:

- Tipo de evento (mensaje nuevo, cambio de asignación, cambio de estado).
- Identificador de la conversación.
- Marca de tiempo.

**Nunca** el texto del mensaje, el nombre o el teléfono del contacto, ni el archivo. Razón doble: `RF-TRV-07`, y que el canal es por organización, no por usuario —de modo que un aviso llega a asesores que quizá no deban ver esa conversación. Al pedir los datos a la API, cada uno recibe solo lo suyo.

### Canales y aislamiento

- **Un canal privado por organización**, nombrado con el identificador de la organización.
- **Canales privados obligatorios:** hay que desactivar "Allow public access" en la configuración de Realtime y crear el canal con `private: true`. Sin eso, los canales son públicos y el aislamiento no existe.
- **Políticas sobre `realtime.messages`** que permitan a un usuario unirse únicamente al canal de las organizaciones donde tiene membresía activa. Es un sistema de políticas **aparte** del RLS de las tablas (ADR-006), y se verifica por separado.
- **Solo el servidor publica.** Las políticas permiten a los clientes *recibir*, nunca *emitir*. Un asesor no puede inyectar avisos falsos en el canal de su organización.
- **La clave de Broadcast vive solo en el servidor.** Es una credencial con permiso de publicación; en el navegador equivaldría a dejar emitir a cualquiera.

### La red de seguridad

Los permisos del canal se cachean durante la vida de la conexión y el cliente se desconecta al expirar el token. Además, una notificación puede perderse. Por eso:

- **El frontend consulta la API cada 60 segundos**, además de escuchar el canal. Es suficientemente lento para no pesar y suficientemente rápido para que un fallo del canal no pase de un minuto.
- **Al recuperar el foco de la ventana**, el frontend recarga. Es el momento en que el usuario mira, y es gratis.
- **Al reconectar el canal**, recarga. Una reconexión implica que pudo haberse perdido algo.

Estas tres reglas convierten el tiempo real en una mejora sobre una base correcta, y no en el único camino por el que los datos llegan a la pantalla. Es lo que evita que un fallo del canal se vea como "el CRM no funciona".

### Qué fija esta decisión para el diseño

1. **El tiempo real notifica; la API entrega.** Ningún dato llega a la pantalla sin pasar por la API.
2. **La notificación ocurre después de la escritura, no dentro de la transacción.** Si la notificación falla, el dato ya está guardado y la red de seguridad lo recoge. Lo contrario —que un fallo de notificación revierta la escritura— sería grave. Vale para los dos escritores: el consumidor de la cola y la API en el camino de salida.
3. **Un fallo al notificar no reintenta el procesamiento del evento.** Se registra y se sigue. El mensaje está guardado; reintentar el lote de la cola (ADR-005) solo por la notificación duplicaría trabajo. En el camino de salida, tampoco falla el envío: el asesor ya recibió su respuesta.
4. **Publicar el aviso vive en un único lugar del código.** Una función que recibe la organización, el tipo de evento y el identificador de la conversación. Los dos escritores la llaman. Es lo que evita que el camino de salida termine con un formato de aviso distinto al del consumidor, que el frontend tendría que entender dos veces.
5. **El contador de conversaciones sin responder** (`RF-CNV-15`) se recalcula al recibir un aviso, no se incrementa a ciegas. Un contador que solo suma se desincroniza en cuanto se pierde un aviso.
6. **La consola no usa tiempo real.** `RF-PLT-USO-01` y la salud del webhook (`RF-PLT-WA-05`) se cargan al abrir la vista. Agregar tiempo real a la consola sería una pieza sin problema que resolver (`CLAUDE.md` §4, regla 9), y además el rol de plataforma no debe estar suscrito a canales de organizaciones.
7. **Una organización suspendida no recibe avisos útiles**, porque sus usuarios no pueden entrar (`RF-AUT-07`). Los mensajes entrantes se siguen guardando (`RF-CNV-16`); simplemente no hay nadie escuchando.

### Plan y costo

El **plan gratuito de Realtime alcanza con holgura** para el piloto: 200 conexiones concurrentes y 2.000.000 de mensajes, contra unos pocos asesores y menos de 1 mensaje por segundo.

Hay un detalle que no se puede pasar por alto: **los proyectos gratuitos de Supabase se pausan tras una semana de inactividad.** Para un piloto de una semana que empieza el 1 de diciembre, eso es un riesgo real durante los días previos y en cualquier pausa entre pruebas. La decisión del plan de Supabase corresponde a D-11, pero **este es un argumento concreto a favor del plan Pro** que conviene registrar ahora.

## Consecuencias

### Más fácil

- La regla de ADR-007 se mantiene sin excepciones: el navegador nunca lee la base.
- Un mensaje nuevo produce una notificación, no una lectura por cada asesor conectado.
- El esquema de la base no queda expuesto al cliente, así que se puede cambiar sin romper la interfaz.
- Los fallos son visibles en los registros, no silenciosos.
- La red de seguridad hace que el producto siga funcionando si el canal falla.

### Más difícil

- Hay que escribir la notificación en el consumidor y el manejo del canal en el frontend.
- Aparece un segundo sistema de políticas —el de `realtime.messages`— que no es el RLS de las tablas y hay que verificar aparte.
- El aviso y la carga son dos pasos; hay que medir que el total cumpla `RF-TRV-03`.
- Hay que recordar desactivar "Allow public access": es un interruptor de configuración del que depende el aislamiento del canal, y no está en el código.
- La clave de publicación es una credencial más que custodiar.

### Qué hay que construir

- Notificación por HTTP desde el consumidor de la cola, tras guardar.
- Canal privado por organización en el frontend, con suscripción y reconexión.
- Políticas sobre `realtime.messages`: unirse solo a canales de organizaciones con membresía activa, solo recibir, nunca emitir.
- Desactivación de "Allow public access" en la configuración del proyecto.
- Recarga desde la API al recibir aviso, al recuperar el foco y al reconectar.
- Consulta periódica de 60 segundos como red de seguridad.
- Recálculo del contador de conversaciones sin responder.
- Prueba automática de aislamiento del canal: un usuario de A no puede unirse al canal de B (extiende `RF-TRV-01` al tiempo real).

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §8.2 punto 2: el punto abierto queda resuelto. La respuesta **no** es que la interfaz se suscriba a las tablas, sino Broadcast desde el servidor.
- `docs/01_definicion_mvp_v2.2.md` §8.1: agregar Supabase Realtime como componente, con la nota del plan.
- `docs/02_documento_maestro_v1.1.md` §5.2, `RF-CNV-15`: precisar que el tiempo real notifica y la interfaz recarga desde la API.

## Qué revisar cuando el sistema crezca

- **Si la bandeja no cumple los 5 segundos** (`RF-TRV-03`): medir los dos tramos por separado —notificación y carga— antes de cambiar nada. Lo más probable es que el tiempo esté en la consulta de la API, no en el canal.
- **Al acercarse a 200 conexiones concurrentes:** son unos 200 asesores conectados a la vez, es decir muchas decenas de organizaciones. Para entonces el plan Pro ya será necesario por otras razones.
- **Si las notificaciones perdidas dejan de ser raras:** revisar los registros del consumidor antes de acortar el intervalo de la red de seguridad. Un intervalo más corto esconde el problema.
- **Si hace falta notificar a un usuario concreto** (por ejemplo, "te asignaron una conversación"): hoy el canal es por organización. Un canal por usuario es posible, pero multiplica las conexiones y exige políticas nuevas; evaluarlo solo cuando haya un requisito real.
- **Si se agregan canales además de WhatsApp:** el aviso ya es genérico —tipo de evento e identificador de conversación—, así que debería servir sin cambios.
- **Si la consola llegara a necesitar tiempo real:** reconsiderar el punto 5, teniendo en cuenta que el rol de plataforma no debe suscribirse a canales de organizaciones y necesitaría un canal propio sin contenido.
