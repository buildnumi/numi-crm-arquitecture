# ADR-005: Procesamiento del webhook de WhatsApp

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-TRV-02, RF-CNV-06, RF-CNV-09, RF-CNV-16, RF-PLT-WA-05, RF-CNV-10a
- **Depende de:** ADR-001, ADR-003, ADR-004
- **Condiciona a:** D-06 (aislamiento), D-07 (tiempo real), D-11 (observabilidad)

## Contexto

Todo lo que entra a Numi CRM entra por el webhook de Meta: mensajes de los contactos y cambios de estado de los mensajes enviados. Es **el único punto del sistema con requisitos duros**, y los tres están en `CLAUDE.md` §8:

- **`RF-TRV-02`:** firma de Meta validada, **sin duplicados** ante reintentos (clave única por identificador de mensaje de Meta), **sin pérdidas**.
- **`RF-CNV-06`:** los mensajes se ordenan por la hora en que se enviaron, **aunque lleguen desordenados**.
- **`RF-CNV-16`:** mientras la organización está suspendida no se envía nada, pero **los mensajes entrantes se siguen recibiendo y guardando**.

Hay que decidir cómo se procesa ese webhook: qué pasa dentro de la petición de Meta, qué se delega, cómo se evitan duplicados, cómo se ordena y qué pasa cuando algo falla.

### Restricciones que aplican

- Runtime: Hono en Cloudflare Workers (ADR-003), con Cloudflare Queues disponible.
- Por ADR-004, cada mensaje con archivo dispara además una descarga que **debe ocurrir dentro de 7 días** o el archivo se pierde.
- Por ADR-001, las credenciales son por organización; el webhook debe resolver a qué organización pertenece cada evento antes de poder hacer casi nada.
- Volumen: menos de 1 mensaje por segundo (ADR-003). **El rendimiento no es el problema; la corrección sí.**
- Dos personas con tiempo parcial. Toda pieza debe justificar qué problema concreto resuelve hoy (`CLAUDE.md` §4, regla 9).

### Datos verificados

Verificados el 2026-10-15 contra la documentación oficial.

**Meta** — [Webhooks overview](https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/overview/), [Messenger Platform webhooks](https://developers.facebook.com/documentation/business-messaging/messenger-platform/webhooks)

| Dato | Valor |
| --- | --- |
| Reintentos ante fallo | "Meta retries delivery with decreasing frequency until the request succeeds, **for up to 7 days**" |
| Qué dispara un reintento | Una respuesta distinta de 200, o cualquier otro fallo de entrega |
| Duplicados | "These retries can **result in duplicate webhook notifications**" |
| Orden de los eventos entrantes | **No hay garantía documentada** |
| Firma | HMAC-SHA256 sobre el cuerpo, con el secreto de la app, en la cabecera `X-Hub-Signature-256` con prefijo `sha256=` |
| Detalle de la firma | Meta calcula la firma sobre el cuerpo con **unicode escapado y hexadecimal en minúsculas**; hashear los bytes crudos decodificados da un resultado distinto |
| Estados contradictorios | Un mismo mensaje puede producir un estado `delivered` **y** un `failed`, si el destinatario tiene varios dispositivos |

**Cloudflare Queues** — [Delivery guarantees](https://developers.cloudflare.com/queues/reference/delivery-guarantees/), [How Queues works](https://developers.cloudflare.com/queues/reference/how-queues-works/), [Limits](https://developers.cloudflare.com/queues/platform/limits/)

| Dato | Valor |
| --- | --- |
| Garantía de entrega | "Queues provides **at least once** delivery by default" |
| Duplicados | Posibles: un mensaje "may, in rare cases, be delivered more than once" |
| Orden | **No garantizado.** "Queues does not guarantee that messages will be delivered to a consumer in the same order in which they are published" |
| Tamaño máximo de mensaje | 128 KB |
| Retención | Configurable hasta 14 días; **no configurable, 24 h, en el plan gratuito** |
| Lote máximo del consumidor | 100 mensajes |
| Reintentos | 100 |
| Lotes | Por defecto todo o nada: si falla un mensaje del lote, se reintenta el lote entero, salvo que se confirmen o reintenten mensajes de forma individual |
| **Espera máxima de lote** | **60 s** (`max_batch_timeout`) |
| **Invocaciones concurrentes del consumidor** | **250**, máximo del proveedor |
| **Duración máxima de un consumidor** | **15 min** |

*(Las tres últimas filas se agregaron el 2026-10-16, verificadas en [Queues limits](https://developers.cloudflare.com/queues/platform/limits/). La de concurrencia importa porque la sección "Qué revisar cuando el sistema crezca" propone ajustar `max_concurrency` sin decir cuál es el techo; la de duración acota la descarga de un archivo grande de ADR-004.)*

### El hallazgo que define la decisión

**Ni Meta ni Cloudflare Queues garantizan orden, y los dos pueden duplicar.**

Eso significa que `RF-TRV-02` y `RF-CNV-06` **no se pueden resolver en el transporte**. No existe una configuración de cola que los cumpla. Hay que resolverlos donde sí hay garantías: **en la base de datos**, con una restricción de unicidad para los duplicados y con un ordenamiento por la marca de tiempo de Meta para el orden.

Esto simplifica mucho la decisión: una vez que la unicidad y el ordenamiento viven en la base, **la cola deja de ser responsable de la corrección** y pasa a ser solo un mecanismo para sacar trabajo lento fuera de la respuesta a Meta. Y eso cambia qué opciones son razonables.

Hay un segundo hallazgo, más sutil, que conviene no pasar por alto: **los reintentos de Meta duran 7 días, igual que la vigencia del identificador de archivo** (ADR-004). No es coincidencia útil: significa que si el sistema está caído varios días, los mensajes se recuperan cuando Meta reintenta, pero los archivos de los primeros días ya habrán caducado. **Una caída larga pierde archivos aunque no pierda mensajes.**

## Opciones consideradas

### Opción A: Procesar todo dentro de la petición del webhook, sin cola

El webhook valida la firma, resuelve la organización, inserta el mensaje y responde 200. Si hay archivo, lo descarga en la misma petición.

- **Ventajas:** la pieza más simple posible; nada que montar ni operar; el mensaje queda visible de inmediato, sin latencia de cola; sin plan de pago de Workers por este motivo.
- **Desventajas:** **la descarga de un documento de hasta 100 MB (ADR-004) ocurre dentro de la respuesta a Meta**, lo que arriesga el tiempo de respuesta y el límite de CPU de la invocación; si la descarga falla, o Meta no responde, o la base está lenta, el webhook devuelve un error y Meta reintenta **todo el evento**, incluido lo que ya se había guardado; no hay forma de reintentar solo la parte que falló; un pico de mensajes se traduce en un pico de trabajo pesado simultáneo.
- **Esfuerzo, costo y riesgo:** esfuerzo mínimo. Riesgo alto sobre `RF-TRV-02`: la parte lenta y falible queda en el camino crítico de la respuesta a Meta.
- **Veredicto:** aceptable si no hubiera archivos. ADR-004 los metió, y eso la descarta.

### Opción B: Webhook mínimo, cola de Cloudflare para el trabajo diferido

El webhook hace solo lo que no puede fallar ni tardar: valida la firma, resuelve la organización, **inserta el evento crudo** y responde 200. Todo lo demás —interpretar el evento, crear o actualizar la conversación, disparar la asignación, descargar archivos— ocurre en un consumidor de la cola.

- **Ventajas:** la respuesta a Meta queda rápida y predecible; lo lento y lo falible se reintenta por separado, sin reintentar lo ya guardado; la descarga de archivos tiene su propia vida y sus propios reintentos dentro de los 7 días; usa una primitiva del proveedor ya elegido, sin sumar uno nuevo; 100 reintentos y hasta 14 días de retención dan margen de sobra para un incidente.
- **Desventajas:** requiere el plan de pago de Workers para tener retención mayor a 24 horas; agrega una pieza que hay que entender y observar; el mensaje tarda un poco más en aparecer en la bandeja, aunque muy por debajo de los 5 segundos que pide `CLAUDE.md` §7; al no garantizar orden, obliga a que el ordenamiento viva en la base de todos modos.
- **Esfuerzo, costo y riesgo:** esfuerzo medio. Costo: 5 USD al mes del plan de pago, con 1.000.000 de operaciones incluidas, muy por encima del volumen del piloto. Riesgo bajo.

### Opción C: Webhook mínimo, cola en una tabla de Postgres

Igual que la Opción B, pero la cola es una tabla con un trabajador que la lee periódicamente.

- **Ventajas:** sin plan de pago de Workers por este motivo; la cola es visible en la misma base, lo que facilita inspeccionarla y depurarla; control total sobre el reintento y el orden.
- **Desventajas:** hay que construir el trabajador, el bloqueo de filas, el conteo de intentos, el retroceso exponencial y la limpieza; en Workers no hay procesos largos, así que hace falta un disparador periódico (Cron Trigger), lo que introduce latencia de hasta el intervalo configurado y complica llegar al objetivo de 5 segundos de la bandeja; carga la base de datos con trabajo de encolado además del trabajo de datos; es reconstruir, peor, algo que el proveedor ya da hecho.
- **Esfuerzo, costo y riesgo:** esfuerzo alto, para ahorrar 5 USD al mes. Riesgo medio: cada pieza construida a mano es una pieza que puede tener errores sutiles bajo concurrencia.
- **Veredicto:** el ahorro no justifica el trabajo ni el riesgo. Contradice la regla 9 de `CLAUDE.md` §4 al revés: no es una pieza injustificada, es trabajo manual injustificado.

## Decisión

**Opción B: webhook mínimo más Cloudflare Queues**, con la corrección (unicidad y orden) resuelta en la base de datos y no en la cola.

Esto implica además **confirmar el plan de pago de Workers**, que ADR-003 había dejado abierto. La retención de 24 horas no configurable del plan gratuito es inaceptable para `RF-TRV-02`: un incidente de fin de semana perdería los eventos encolados.

### Qué hace el webhook, exactamente

El webhook hace **solo esto**, en este orden, y nada más:

1. **Verifica la firma** `X-Hub-Signature-256` con HMAC-SHA256 sobre el **cuerpo crudo**, antes de interpretarlo. Si no coincide, responde 403 y no procesa nada. Atención al detalle documentado: Meta calcula la firma sobre el cuerpo con unicode escapado y hexadecimal en minúsculas; **esto hay que probarlo con un mensaje que contenga tildes o emoji antes de darlo por bueno**, porque es el error típico y silencioso.
2. **Resuelve la organización** a partir del identificador del número que trae el evento (`phone_number_id`). **Si no corresponde a ninguna organización, el evento se guarda igual, con la organización sin resolver**, y se responde 200. No se encola: no hay a quién atribuirlo todavía.
3. **Inserta el evento crudo** en una tabla de eventos recibidos, con su organización, el cuerpo completo y la hora de recepción.
4. **Encola** una referencia a ese evento.
5. **Responde 200.**

No interpreta el evento, no crea conversaciones, no descarga archivos, no envía nada. Todo eso es trabajo del consumidor.

La razón de este corte: **lo único que Meta castiga es no recibir un 200**. Si el webhook responde 200 y el evento está guardado, el mensaje ya no se puede perder, pase lo que pase después. Todo lo demás es recuperable desde la tabla de eventos.

**Por qué el evento de un número desconocido también se guarda** (corregido el 2026-10-16; antes este paso registraba el hecho y descartaba el evento). El modo transitorio de ADR-001 conecta los números **a mano desde la consola** (`RF-PLT-WA-02`). Eso abre una ventana real entre el momento en que el número ya existe en Meta y empieza a recibir mensajes, y el momento en que el operador lo conecta en la consola. Un evento descartado con 200 en esa ventana **se pierde para siempre, porque un 200 le dice a Meta que no reintente** — y `RF-TRV-02` no admite pérdidas.

De ahí la regla: **el webhook nunca descarta un evento cuya firma es válida.** La firma ya demuestra que viene de Meta; que todavía no sepamos de quién es no lo vuelve descartable.

- El evento se guarda con la organización **sin resolver** y no se encola.
- **Al conectar un número en la consola, se reprocesan los eventos pendientes de ese `phone_number_id`**: se les asigna la organización y se encolan. La unicidad `(organization_id, wamid)` hace que reprocesar sea seguro aunque el evento llegue por otro camino.
- Los eventos sin resolver que superen la retención de 30 días se borran como cualquier otro. Un número que nunca se conecta no acumula nada indefinidamente.

**Lo que esto cuesta:** la tabla de eventos admite filas sin organización, así que su política de aislamiento debe contemplar ese caso —son filas que **ningún rol de negocio puede leer**, solo el camino de reprocesado—. Es el precio de no perder mensajes durante el modo transitorio, y es barato comparado con la alternativa.

### Cómo se cumple cada requisito duro

**`RF-TRV-02`, sin duplicados.** Restricción de unicidad en la base sobre el identificador de mensaje de Meta (`wamid`), por organización. La inserción usa "insertar o no hacer nada". Esto funciona sin importar cuántas veces Meta reintente ni si Queues entrega dos veces, porque **la unicidad no depende del transporte**. La clave compuesta `(organization_id, wamid)` respeta el criterio de `CLAUDE.md` §9 sobre claves foráneas compuestas; el detalle es D-06.

**`RF-TRV-02`, sin pérdidas.** Tres capas, en orden de actuación:
- El evento se guarda crudo antes de responder 200. Si el consumidor falla, el evento sigue ahí.
- Si el guardado falla, el webhook responde distinto de 200 y **Meta reintenta hasta 7 días**.
- La cola reintenta hasta 100 veces, con cola de mensajes fallidos para lo que agote los intentos.

**`RF-CNV-06`, orden.** Los mensajes se ordenan **por la marca de tiempo que envía Meta**, no por la hora de llegada ni por el orden de inserción. Esto es obligatorio, no una optimización: **ni Meta ni Queues garantizan orden**, así que el orden de llegada no significa nada. Un mensaje que llega tarde se inserta y aparece en su lugar cronológico correcto.

Dos precisiones necesarias:
- La marca de tiempo de Meta tiene **resolución de segundos**. Dos mensajes del mismo segundo necesitan un desempate estable —el identificador de Meta sirve— para que el orden no cambie entre dos cargas de la misma conversación.
- Todas las fechas en `timestamptz`, presentadas en hora de Bogotá (`CLAUDE.md` §9).

**`RF-CNV-09`, estados del mensaje.** Los estados llegan como eventos separados y **pueden llegar desordenados o contradictorios**: Meta documenta que un mismo mensaje puede producir `delivered` y `failed`. Por eso el estado **no se sobrescribe sin más**: se aplica una progresión (`enviando` → `enviado` → `entregado` → `leído`), y un estado anterior que llega tarde no pisa a uno posterior ya registrado. `error` es la excepción: puede llegar en cualquier momento y se registra siempre, porque `RF-CNV-09` dice explícitamente que *"un mensaje puede pasar a error después de figurar como enviado"*.

**`RF-CNV-16`, organización suspendida.** La suspensión se verifica **al enviar**, nunca al recibir. El webhook y el consumidor procesan y guardan igual que siempre. Es una propiedad del camino de salida, no del de entrada.

**`RF-CNV-10a`, archivos (ADR-004).** La descarga es un trabajo encolado aparte del procesamiento del mensaje. El mensaje se guarda y se muestra aunque el archivo todavía no esté. La descarga se reintenta dentro de los 7 días; agotado ese plazo, el mensaje queda marcado como archivo no recuperado (`RF-CNV-10e`).

**Cómo se estiran los reintentos hasta los 7 días** *(agregado el 2026-10-16)*. "Se reintenta dentro de los 7 días" no sale solo de los 100 reintentos: con el retroceso por defecto, los 100 intentos se agotan en horas, y el mensaje moriría con seis días de margen sin usar. Hay que configurarlo explícitamente:

- **Retención de la cola de archivos fijada en 7 días**, dentro de los 14 que admite el plan de pago.
- **Retroceso exponencial con tope**, de modo que los últimos intentos queden espaciados en horas y no en segundos. Cien intentos con tope de unas pocas horas cubren la ventana completa.
- Lo que marca el archivo como no recuperado es **la caducidad del identificador de Meta a los 7 días**, no el conteo de intentos. Si los intentos se agotan antes, el archivo se pierde con la ventana todavía abierta — que es exactamente el fallo que esta configuración evita.

Es un parámetro de configuración, no código, y entra en la lista de puesta en marcha de ADR-011.

### Qué fija esta decisión para el diseño

1. **Dos colas separadas**, no una: una para el procesamiento de eventos y otra para la descarga de archivos. Razón: un documento de 100 MB no debe bloquear ni reintentar junto con el procesamiento de mensajes de texto. Tienen ritmos y tolerancias distintas.
2. **El consumidor confirma o reintenta mensajes de forma individual**, no el lote completo. El comportamiento por defecto de Queues es todo o nada, y eso haría reprocesar mensajes ya procesados cada vez que uno falle.
3. **El consumidor es idempotente.** Procesar dos veces el mismo evento debe dar el mismo resultado, porque Queues entrega al menos una vez. La unicidad en la base es lo que lo garantiza.
4. **La tabla de eventos crudos tiene retención propia y corta** —30 días es un punto de partida razonable—. Es una red de seguridad para reprocesar, no un archivo histórico. Contiene contenido de conversaciones, así que está sujeta a `RF-TRV-07`: nunca llega a la consola ni a las métricas.
5. **Cola de mensajes fallidos obligatoria** en ambas colas. Un evento que agota sus reintentos no puede desaparecer en silencio. Debe quedar visible para el operador.
6. **El webhook nunca registra el contenido del mensaje.** Los registros técnicos llevan el identificador del evento, la organización y el resultado; nunca el texto, el teléfono del contacto ni el archivo (`RF-TRV-07`, `RF-PLT-USO-04`).
7. **La salud del webhook se deriva de la tabla de eventos** (`RF-PLT-WA-05`): último evento recibido, cantidad en 24 horas y errores recientes salen de ahí, sin contenido.
8. **Límite de 128 KB por mensaje de cola.** Se encola una **referencia** al evento, no el evento. Un cuerpo de Meta grande podría superar el límite, y encolar referencias lo evita por completo.

### Qué queda fuera de esta decisión

- **Cómo se aplica el aislamiento por organización en cada consulta:** D-06. Aquí solo se fija que la unicidad es por organización.
- **Cómo llega el mensaje nuevo a la pantalla del asesor:** D-07. Esta decisión fija que el consumidor es quien tiene el mensaje guardado y listo, es decir el punto desde el que se notifica.
- **Qué se registra y qué alertas existen:** D-11.

## Consecuencias

### Más fácil

- La respuesta a Meta es rápida y predecible, que es la única cosa que Meta castiga.
- Los tres requisitos duros quedan garantizados por la base de datos, no por configuración del transporte. Son verificables con pruebas automáticas (`RF-TRV-01`, `RF-TRV-02`).
- Un fallo del consumidor no pierde nada: el evento está guardado y la cola reintenta.
- La descarga de archivos tiene su propio ciclo, sin arriesgar el procesamiento de mensajes.
- Reprocesar es posible: los eventos crudos están guardados.

### Más difícil

- Hay dos colas, un consumidor y dos colas de mensajes fallidos que entender, observar y operar.
- La verificación de firma tiene un detalle de codificación que **hay que probar explícitamente** con texto no ASCII, o fallará de forma silenciosa en producción.
- El estado del mensaje no es una asignación simple: es una progresión con reglas.
- Aparece una tabla de eventos crudos con contenido sensible, que necesita su propia retención y su propio control de acceso.
- **Una caída de varios días pierde archivos aunque no pierda mensajes**, porque el identificador de Meta caduca a los 7 días. Esto hay que saberlo al definir las alertas en D-11: una alerta de webhook caído no es un aviso de cortesía, tiene un reloj corriendo.

### Qué hay que construir

- Endpoint de verificación del webhook (el `GET` con `hub.challenge` que Meta exige al suscribir).
- Verificación de firma HMAC-SHA256 sobre el cuerpo crudo, con prueba de texto no ASCII.
- Resolución de organización por `phone_number_id`.
- Tabla de eventos recibidos, con retención de 30 días, **con `organization_id` nulo admitido** para los eventos de un número todavía no conectado.
- **Reprocesado de los eventos pendientes de un `phone_number_id`** cuando el operador conecta ese número en la consola.
- **Trabajo programado (Cron Trigger) común**, que atiende la limpieza por retención de esta tabla, el borrado de archivos de ADR-004 y el barrido de mensajes atascados de ADR-012. Workers no tiene procesos largos (ADR-003), así que sin esto nada de eso ocurre.
- Dos colas con sus colas de mensajes fallidos.
- Consumidor idempotente que interpreta eventos, crea o actualiza conversaciones y mensajes, y dispara la asignación automática (`RF-CNV-12`).
- Restricción de unicidad `(organization_id, wamid)`.
- Ordenamiento por marca de tiempo de Meta con desempate estable.
- Progresión de estados de mensaje con tolerancia a desorden y a estados contradictorios.
- Consumidor de descarga de archivos con reintentos acotados a 7 días.
- Derivación de la salud del webhook para `RF-PLT-WA-05`.

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §8.1: confirmar Cloudflare Queues y registrar el **plan de pago de Workers** como costo fijo.
- `docs/01_definicion_mvp_v2.2.md` §8: agregar la tabla de eventos recibidos a la tabla de retención, con 30 días.
- `docs/02_documento_maestro_v1.1.md` §5.2: precisar en `RF-CNV-09` que los estados siguen una progresión y que un estado anterior no pisa a uno posterior.

### Costo

| Concepto | Costo |
| --- | --- |
| Workers, plan de pago | 5 USD al mes |
| Queues | Incluido hasta 1.000.000 de operaciones al mes |

El volumen del piloto —unos 15.000 mensajes al mes (ADR-003)— genera del orden de 30.000 a 45.000 operaciones mensuales contando encolado, lectura y descargas. Queda holgadamente dentro de lo incluido.

## Qué revisar cuando el sistema crezca

- **Si la bandeja tarda más de 5 segundos** (`CLAUDE.md` §7): medir dónde está el tiempo antes de tocar la arquitectura. Lo más probable es que sea el tiempo real (D-07), no la cola.
- **Al acercarse a 1.000.000 de operaciones mensuales:** revisar el costo marginal, 0,40 USD por millón. A 500 conversaciones mensuales por negocio, eso ocurre alrededor de las 60–70 organizaciones.
- **Si la cola de mensajes fallidos recibe eventos con regularidad:** no es un problema de capacidad sino de corrección. Hay que leerlos, no aumentar reintentos.
- **Si un documento de 100 MB causa problemas:** evaluar un límite propio de tamaño por encima del cual solo se guarda la referencia y no el archivo. Se cruza con D-10.
- **Si aparecen varios canales además de WhatsApp:** la tabla de eventos crudos y la resolución de organización tendrán que generalizarse detrás de la interfaz de canal (`CLAUDE.md` §9).
- **Si el volumen supera unos 10 mensajes por segundo sostenidos:** revisar `max_concurrency` del consumidor y si conviene separar el consumidor de estados del de mensajes.
