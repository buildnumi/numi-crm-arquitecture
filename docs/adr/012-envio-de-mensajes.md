# ADR-012: Envío de mensajes

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-CNV-07, RF-CNV-08, RF-CNV-09, RF-CNV-11, RF-CNV-15, RF-CNV-16, RF-TRV-03, RF-AUT-07
- **Depende de:** ADR-001, ADR-003, ADR-005, ADR-006, ADR-007, ADR-008
- **Condiciona a:** —

## Contexto

Este ADR cubre un hueco detectado al auditar el tablero: **ninguna de las once decisiones anteriores cubre el camino de salida.** ADR-005 resolvió en detalle cómo entran los mensajes; cómo salen quedó implícito, citado de pasada en varios ADR pero sin decidirse en ninguno.

Es un hueco con peso, porque el envío toca **dos requisitos que no se negocian** (`CLAUDE.md` §8):

- **`RF-CNV-08`:** la ventana de 24 horas de Meta aplicada **antes** de enviar.
- **`RF-CNV-09`:** estados del mensaje —enviando, enviado, entregado, leído, error—, con la posibilidad de reintentar.

Y además:

- **`RF-CNV-07`:** el usuario escribe y envía mensajes de texto desde el CRM.
- **`RF-CNV-16`:** mientras la organización está suspendida, no se pueden enviar mensajes.
- **`RF-CNV-11`:** si un asesor responde una conversación sin asignar, queda como su responsable.

### Restricciones que aplican

- Runtime: Hono en Workers; credenciales de WhatsApp por organización, cifradas (ADR-001, ADR-011).
- El envío de archivos desde el CRM está **fuera del MVP** (ADR-004). Solo texto.
- `RF-TRV-03`: la interfaz debe sentirse inmediata.
- Volumen: menos de 1 mensaje por segundo (ADR-003).

### Datos verificados de Meta

Verificados el 2026-10-15 contra la documentación oficial.

**Ventana de 24 horas** — [Error codes](https://developers.facebook.com/documentation/business-messaging/whatsapp/support/error-codes)

| Dato | Valor |
| --- | --- |
| Error al enviar fuera de la ventana | **131047**: "More than 24 hours have passed since the recipient last replied to the sender number" |
| Remedio que indica Meta | "Send the recipient a template message instead" |
| Qué abre la ventana | Una respuesta del contacto. En coexistencia, los mensajes enviados desde la app de WhatsApp Business **no crean ni extienden** la ventana de Cloud API |

**Throughput** — [Throughput](https://developers.facebook.com/documentation/business-messaging/whatsapp/throughput)

| Dato | Valor |
| --- | --- |
| Por defecto | **80 mensajes por segundo** por número |
| Ampliación automática | Hasta 1.000 mps, sin costo |
| Coexistencia | Fijo en **20 mps** |
| Error por exceder el límite | **130429** |
| Error durante la ampliación | **131057**, el número queda inutilizable hasta un minuto |

### Los dos hechos que definen la decisión

**Primero: el throughput no es el problema.** 80 mensajes por segundo contra menos de 1 que necesita el piloto. **La decisión no es de capacidad, es de corrección**: qué se le muestra al asesor, qué pasa si Meta falla, y cómo no se pierde ni se duplica lo que el asesor escribió.

**Segundo, y es el que decide el diseño: la ventana de 24 horas no se puede verificar con certeza antes de enviar.** El sistema calcula la ventana a partir del último mensaje entrante que registró, pero esa información puede estar desactualizada:

- El reloj lo lleva Meta, no el sistema.
- En coexistencia, los mensajes de la app no extienden la ventana de Cloud API, así que la vista del sistema puede diferir.
- Entre la verificación y el envío pasa tiempo.

**Conclusión: la verificación local es una comodidad para el asesor, no una garantía.** La única verificación real es la respuesta de Meta, y el sistema tiene que manejar el caso de que diga que no (131047) aunque localmente pareciera que sí.

Esto es exactamente el mismo patrón que apareció tres veces en los ADR anteriores —no confiar en una comprobación que no se puede garantizar— y la respuesta es la misma: **el estado de verdad está donde está, y hay que consultarlo en el momento.**

## Opciones consideradas

### Opción A: Envío síncrono dentro de la petición

El asesor envía, la API llama a Meta y espera, y responde cuando Meta contesta.

- **Ventajas:** lo más simple; el asesor ve el resultado real de inmediato, incluido el error de ventana vencida; sin cola ni estados intermedios que gestionar; no hay riesgo de duplicar.
- **Desventajas:** si Meta tarda, la petición tarda; si Meta falla temporalmente, el mensaje se pierde y el asesor tiene que reescribirlo; una indisponibilidad de Meta deja el CRM sin poder enviar, sin memoria de lo intentado.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo. Riesgo medio: lo que el asesor escribió depende de que Meta responda bien en ese instante.

### Opción B: Envío encolado

El mensaje se guarda como "enviando", se encola y un consumidor lo envía a Meta.

- **Ventajas:** nada de lo que el asesor escribió se pierde, pase lo que pase con Meta; reintentos automáticos; la interfaz responde de inmediato.
- **Desventajas:** **el asesor no ve el resultado real al enviar**, y el caso más importante —ventana vencida— llegaría como un error diferido, después de que la interfaz ya mostró el mensaje como enviándose; eso es peor experiencia que bloquear el campo, que es justo lo que `RF-CNV-08` pide; agrega un consumidor y estados intermedios; reintentar un envío que Meta sí recibió duplicaría el mensaje al contacto, que es un error visible para el cliente final.
- **Esfuerzo, costo y riesgo:** esfuerzo medio-alto. Riesgo medio, concentrado en la duplicación.

### Opción C: Síncrono con persistencia previa y reintento manual

El mensaje se guarda **antes** de llamar a Meta, con estado "enviando". La API llama a Meta dentro de la misma petición y actualiza el estado según la respuesta. Si falla, queda en "error" y el asesor decide reintentar.

- **Ventajas:** el asesor ve el resultado real de inmediato, incluido el error de ventana vencida; **nada de lo que escribió se pierde**, porque se guardó antes de intentar; el reintento es explícito, que es lo que `RF-CNV-09` pide literalmente —"un mensaje con error se puede reintentar"—; **sin reintentos automáticos, no hay riesgo de duplicar**; sin cola ni consumidor nuevo.
- **Desventajas:** si Meta tarda, la petición tarda —acotado con un tiempo límite; un fallo deja trabajo manual al asesor, aunque sea un botón; no sirve si alguna vez hiciera falta enviar muchos mensajes de golpe.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo-medio. Riesgo bajo.

## Decisión

**Opción C: síncrono, con persistencia previa y reintento manual.**

Razón, en orden de peso:

1. **`RF-CNV-08` pide bloquear antes de enviar y explicar el motivo.** Eso solo funciona bien si el asesor conoce el resultado en el momento. Un error de ventana vencida que llega medio segundo después, cuando el mensaje ya aparece en pantalla, es una experiencia confusa en el punto más delicado del producto.
2. **Guardar antes de llamar elimina la única ventaja real de la cola.** El texto del asesor no se pierde aunque Meta falle: queda en "error" y se reintenta.
3. **El reintento manual evita la duplicación.** Un reintento automático sobre un envío que Meta sí procesó —pero cuya respuesta se perdió— manda el mensaje dos veces al contacto. En el camino de entrada eso se resuelve con el identificador de Meta (ADR-005); en el de salida **no hay identificador antes de enviar**, así que no se puede deduplicar. Con `RF-CNV-09` pidiendo reintento explícito, la decisión del asesor es la protección.
4. **No hace falta capacidad.** 80 mensajes por segundo contra menos de 1.

Vale la pena notar que esta decisión va **en dirección contraria a ADR-005**, que sacó todo de la petición hacia una cola. La diferencia es quién espera: en el webhook espera Meta, que castiga la demora y reintenta sola; aquí espera una persona mirando la pantalla, que necesita saber si su mensaje salió.

### El camino de un mensaje saliente

1. **Verificar la petición** (ADR-007): usuario, membresía, estado del usuario y de la organización. Una organización suspendida **no puede enviar** (`RF-CNV-16`, `RF-AUT-07`).
2. **Verificar que el asesor puede escribir en esa conversación.** Si está asignada a otro y el usuario no es administrador, se rechaza (§6 del documento maestro, V11).
3. **Verificar la ventana de 24 horas** con el último mensaje entrante registrado. Si venció, se rechaza **sin llamar a Meta** y sin guardar nada.
4. **Guardar el mensaje** con estado `enviando`, su autor y su hora.
5. **Si la conversación estaba sin asignar, asignarla al asesor** (`RF-CNV-11`), en la misma transacción.
6. **Responder a la interfaz**, que ya puede mostrar el mensaje como enviándose. **Publicar el aviso de tiempo real** (ADR-008) para el resto de la organización: mensaje nuevo y, si el paso 5 asignó, cambio de asignación.
7. **Llamar a Meta**, con el token de la organización descifrado en memoria (ADR-011) y un tiempo límite acotado.
8. **Actualizar el estado** según la respuesta: `enviado` con el identificador de Meta, o `error` con el código. **Publicar un segundo aviso** con el cambio de estado.
9. Los estados posteriores —entregado, leído— llegan por webhook y los aplica el consumidor con la progresión de ADR-005.

**Sobre los avisos de los pasos 6 y 8.** El asesor que envía no los necesita: ya tiene la respuesta HTTP de los pasos 6 y 8, y su propia pantalla se actualiza con ella. Los avisos son para **los demás** —el administrador que mira la misma conversación, el asesor que tiene abierta la lista de conversaciones sin asignar que el paso 5 acaba de cambiar—. Sin ellos, `RF-CNV-15` quedaría cubierto solo para el camino de entrada, y esos usuarios dependerían de la consulta de 60 segundos de ADR-008, cuatro veces el objetivo de `RF-TRV-03`.

Los avisos siguen las reglas de ADR-008: se publican **después** de la escritura y fuera de la transacción, no llevan contenido del mensaje, y **un fallo al publicar no falla el envío** —se registra y se sigue, porque el mensaje ya está guardado y salió—. Usan la misma función de publicación que el consumidor de la cola.

El paso 3 y el paso 8 resuelven el mismo requisito en dos momentos: **la verificación local evita el viaje innecesario y da el mensaje claro; la respuesta de Meta es la verdad.** Si Meta devuelve 131047, se aplica el mismo tratamiento que la verificación local: el mensaje queda en error y la interfaz explica que la ventana venció y actualiza su estado.

### Tratamiento de los errores de Meta

| Código | Qué significa | Qué hace el sistema |
| --- | --- | --- |
| **131047** | Pasaron más de 24 h desde la última respuesta del contacto | Mensaje en `error`, la interfaz explica la ventana vencida y bloquea el campo. **No se ofrece reintentar**: reintentar fallaría igual |
| **130429** | Se excedió el throughput | Mensaje en `error`, reintentable. No debería ocurrir a este volumen |
| **131057** | Número en proceso de ampliación, hasta un minuto | Mensaje en `error`, reintentable |
| Error de red o tiempo límite | No se sabe si Meta recibió | Mensaje en `error`, **reintentable con advertencia**: ver abajo |
| Token inválido o expirado | La conexión de la organización dejó de servir | Mensaje en `error`; además debe reflejarse en la salud de la conexión (`RF-PLT-WA-05`) para que el operador lo vea |

**El caso del tiempo límite merece atención.** Si la llamada a Meta expira, el sistema no sabe si el mensaje salió. Reintentar puede duplicarlo; no reintentar puede perderlo. **La decisión la toma el asesor, informado**: el mensaje queda en error con un texto que dice que puede haberse enviado. Es la opción honesta, y es preferible a que el sistema adivine por él.

### La ventana de 24 horas en la interfaz

- El aviso sobre el campo de escritura muestra si la ventana está abierta, con el tiempo restante cuando quedan pocas horas (§6, V11).
- **El tiempo restante se calcula en el cliente** a partir de la hora del último mensaje entrante, para que el contador avance sin consultar al servidor.
- **La verificación que decide es del servidor.** El cliente puede estar desfasado; nunca se confía en él para permitir un envío.
- Cuando la ventana vence estando el asesor en la pantalla, el campo se bloquea solo.
- **Las plantillas quedan fuera del MVP.** Meta indica que el remedio para 131047 es enviar una plantilla, pero eso exige plantillas aprobadas, gestión de su ciclo de vida y un flujo nuevo. No hay requisito que lo pida, y el MVP es reactivo (ADR-001). **El sistema explica que la ventana venció; no ofrece una salida.** Es una limitación real del producto y conviene decirlo así a los negocios del piloto.

### Qué fija esta decisión para el diseño

1. **El mensaje se guarda antes de llamar a Meta.** No al revés. Es lo que evita perder lo que el asesor escribió.
2. **El identificador de Meta se guarda al recibir la respuesta**, y es lo que permite correlacionar los estados que lleguen después por webhook (ADR-005).
3. **Sin reintentos automáticos en el camino de salida.** El reintento es una acción del asesor (`RF-CNV-09`).
4. **El token se descifra en memoria, por llamada**, y nunca se registra (`RF-PLT-SEG-02`).
5. **El tiempo límite de la llamada a Meta es corto** —del orden de 10 segundos—. Una persona está esperando; es mejor un error claro que una pantalla colgada.
6. **El envío pasa por la misma interfaz de canal** que la recepción (`CLAUDE.md` §9). Nada específico de WhatsApp fuera de ella.
7. **Un mensaje en error no bloquea la conversación.** El asesor puede escribir otro, y el fallido queda visible con su opción de reintentar.
8. **La suspensión se verifica al enviar, nunca al recibir** (ADR-006, `RF-CNV-16`).

### Qué queda fuera

- **Envío de archivos:** fuera del MVP (ADR-004).
- **Plantillas de mensaje:** no hay requisito; ver arriba.
- **Respuestas automáticas:** `RF-MET-02` las menciona al excluirlas del cálculo, pero no hay requisito que las pida en el MVP.
- **Envío masivo o difusión:** fuera del alcance, y ADR-001 ya señaló que en modo transitorio consumiría la cuota compartida del portafolio.

## Consecuencias

### Más fácil

- El asesor sabe de inmediato si su mensaje salió, que es lo que importa en una conversación.
- Nada de lo escrito se pierde, aunque Meta falle.
- Sin duplicados hacia el contacto, porque no hay reintentos automáticos.
- Sin cola ni consumidor nuevos: menos piezas que operar (`CLAUDE.md` §4, regla 9).

### Más difícil

- Si Meta está lento, el asesor lo nota. Se acota con el tiempo límite.
- Un fallo deja una acción manual, aunque sea un botón.
- El caso del tiempo límite obliga a un mensaje de interfaz incómodo pero honesto.
- La ventana vencida no tiene salida en el MVP, y eso hay que explicarlo a los negocios del piloto.

### Qué hay que construir

- Ruta de envío con las verificaciones de los pasos 1 a 3.
- Persistencia previa con estado `enviando` y asignación automática al responder (`RF-CNV-11`).
- Llamada a Meta por la interfaz de canal, con token descifrado y tiempo límite.
- Tratamiento de los códigos de error de la tabla.
- Reintento manual desde la interfaz.
- Cálculo y presentación de la ventana en el cliente, con bloqueo automático al vencer.
- Reflejo del token inválido en la salud de la conexión (`RF-PLT-WA-05`).

### Qué documentos hay que actualizar

- `docs/02_documento_maestro_v1.1.md` §5.2: precisar en `RF-CNV-09` que el reintento es manual y que un mensaje puede quedar en error con estado de envío incierto.
- `docs/02_documento_maestro_v1.1.md` §9: agregar el texto para el caso de envío con resultado incierto tras un tiempo límite.
- `docs/01_definicion_mvp_v2.2.md` §6: registrar explícitamente que las plantillas quedan fuera del MVP, y que por tanto una conversación con la ventana vencida no se puede retomar desde el CRM.

## Qué revisar cuando el sistema crezca

- **Si los negocios piden retomar conversaciones con la ventana vencida:** es el caso de uso de las plantillas, y sería alcance nuevo con su propio ADR —aprobación de plantillas, gestión de su ciclo de vida y costo por conversación iniciada.
- **Si aparece el envío de archivos:** cambia este flujo —subida, validación y envío a Meta— y probablemente justifique encolar, porque ahí sí hay trabajo pesado.
- **Si los tiempos límite dejan de ser raros:** revisar la latencia hacia Meta antes de cambiar el modelo. Si fuera persistente, la Opción B pasa a ser razonable, con deduplicación por un identificador propio.
- **Si hiciera falta enviar muchos mensajes de golpe** (difusión, recordatorios): el modelo síncrono no sirve y haría falta una cola de salida. Hoy no hay requisito.
- **Al adoptar la coexistencia** (ADR-001): el throughput baja a 20 mps y, sobre todo, los mensajes enviados desde la app no extienden la ventana de Cloud API. El cálculo local de la ventana se vuelve menos confiable y habrá que revisarlo.
