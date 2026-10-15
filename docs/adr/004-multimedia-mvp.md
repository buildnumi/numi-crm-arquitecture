# ADR-004: Audios, imágenes y documentos en el MVP

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-CNV-10, RF-CNV-06, RF-TRV-02, RF-TRV-07
- **Depende de:** ADR-001, ADR-003
- **Condiciona a:** D-05 (procesamiento del webhook), D-10 (archivos y R2)

## Contexto

Hay que decidir si el MVP **muestra** los audios, imágenes y documentos que llegan por WhatsApp, o si se limita a indicar que llegaron.

Lo que hoy dicen los documentos:

- `RF-CNV-10`: *"Si llega un mensaje de un tipo que el MVP no muestra (audio, imagen, documento u otro), la conversación muestra un marcador con el tipo de mensaje y la hora"*.
- `docs/02_documento_maestro_v1.1.md` §12 lista la visualización de multimedia como **elemento diferido**.
- `docs/01_definicion_mvp_v2.2.md` §13 lo marca como riesgo abierto: *"Los clientes de WhatsApp envían notas de voz e imágenes con frecuencia... con número solo en la API, la clínica no tiene otro lugar donde verlos"*.
- El texto que vería el asesor ya está escrito: *"Recibiste {un audio / una imagen / un documento}. Este tipo de mensaje todavía no se puede ver en Numi CRM."* (`docs/02_documento_maestro_v1.1.md` §9).

### Por qué la pregunta cambió desde que se escribió

La definición del MVP ata este riesgo al modelo de conexión: con número solo en la API, el negocio pierde el único lugar donde podía ver sus audios, porque deja de usar la app de WhatsApp Business en esa línea.

**ADR-001 ya resolvió esa parte.** El piloto arranca en modo transitorio con números del portafolio de Numi, que son **líneas nuevas**, no la línea que la clínica ya usa. Es decir: el contacto que escriba a ese número lo hace a un número que antes no existía, y el negocio no tiene ninguna app donde ver esos mensajes. **No hay un "fuera del CRM" al que mandar al asesor.**

Esto endurece el problema en vez de resolverlo. Si llega una nota de voz y el CRM solo muestra un marcador, esa nota de voz **no se puede escuchar en ningún lado**.

### Restricciones que aplican

- Dos personas desarrollando con tiempo parcial; MVP el 30 de noviembre de 2026 (`CLAUDE.md` §8).
- Segmento inicial: inmobiliarias, clínicas estéticas y odontologías (ADR-001). En los tres, el contacto manda fotos: una lesión, un inmueble, una radiografía.
- Las conversaciones pueden contener datos sensibles de salud; Ley 1581 de 2012, negocio responsable y Numi encargado (`CLAUDE.md` §8).
- El contenido de las conversaciones no puede llegar a la consola, los registros técnicos ni las métricas (`RF-TRV-07`, `RF-PLT-USO-04`).
- Runtime: Hono en Cloudflare Workers (ADR-003), con R2 disponible como primitiva del mismo proveedor.

### Datos verificados de Meta

Todos verificados el 2026-10-15 contra la documentación oficial, en [Media](https://developers.facebook.com/documentation/business-messaging/whatsapp/business-phone-numbers/media).

| Dato | Valor |
| --- | --- |
| Contenido del webhook | **Solo el identificador del archivo**, no el archivo |
| Vigencia del identificador recibido por webhook | **7 días** |
| Vigencia de la URL de descarga | **5 minutos** |
| Tamaño máximo, audio | 16 MB |
| Tamaño máximo, imagen | 5 MB |
| Tamaño máximo, documento | 100 MB |
| Tamaño máximo, video | 16 MB |
| Tipos de audio | `audio/aac`, `audio/amr`, `audio/mpeg`, `audio/mp4`, `audio/ogg` (solo códec OPUS) |
| Tipos de imagen | `image/jpeg`, `image/png` |
| Tipos de documento | `text/plain`, `application/pdf`, Word, Excel, PowerPoint y sus equivalentes Open XML |

Citas literales de los dos datos que deciden:

> "Media IDs in webhooks expire after 7 days."

> "Media URLs expire after 5 minutes."

### El hallazgo que define la decisión

**Lo que no se descarga en 7 días se pierde para siempre.**

Esto rompe la simetría que normalmente tienen las decisiones de alcance. Casi todo lo que se deja fuera del MVP se puede agregar después sin costo adicional: se construye en enero y funciona desde enero. **Aquí no.** Si el piloto corre una semana sin descargar los archivos, los audios e imágenes de esa semana no existen más. No hay forma de recuperarlos cuando la función se construya.

Dicho de otro modo: **hay dos decisiones escondidas en una**, y confundirlas es el error que hay que evitar.

1. **¿Se guardan los archivos?** Es irreversible. Decidir "no" cierra la puerta.
2. **¿Se muestran en la interfaz?** Es reversible. Decidir "no" hoy y "sí" en enero no cuesta nada más que el trabajo de enero.

La discusión de los documentos fuente trata ambas como una sola, porque se escribió antes de conocer el límite de 7 días.

## Opciones consideradas

### Opción A: Solo marcador, nada se descarga

Se mantiene `RF-CNV-10` tal como está. El mensaje queda registrado con su tipo y su hora; el archivo nunca se trae de Meta.

- **Ventajas:** cero trabajo adicional; sin almacenamiento que pagar ni que asegurar; ningún dato sensible nuevo bajo custodia de Numi, lo que simplifica el encuadre de la Ley 1581.
- **Desventajas:** **los archivos del piloto se pierden de forma irreversible**; el asesor que recibe una nota de voz no tiene ningún lugar donde escucharla, porque la línea del piloto es nueva y no está en ninguna app; en clínicas estéticas y odontologías la foto es parte sustancial de la consulta, así que el CRM queda inutilizable para el caso de uso más frecuente; el piloto mediría la adopción de un producto mutilado y daría una señal falsa.
- **Esfuerzo, costo y riesgo:** esfuerzo cero, costo cero, **riesgo alto de producto**.
- **Veredicto:** es la opción escrita hoy, y la que el límite de 7 días vuelve indefendible.

### Opción B: Descargar y guardar, mostrar imágenes y audios; documentos como enlace de descarga

Al recibir un mensaje con archivo, el sistema lo descarga de Meta dentro de la ventana válida, lo guarda en R2 y lo sirve al CRM. La conversación muestra las imágenes en línea, un reproductor para los audios, y los documentos como un enlace con su nombre y tamaño.

- **Ventajas:** el producto sirve para lo que el segmento realmente hace; nada se pierde; usa R2, que es del mismo proveedor ya elegido en ADR-003, sin sumar uno nuevo; el trabajo de descarga es el mismo que la Opción C, así que la interfaz es el único costo adicional sobre ella.
- **Desventajas:** Numi pasa a custodiar archivos que pueden contener datos sensibles de salud, lo que exige cifrado en reposo, acceso verificado por organización y una política de retención explícita; hay que construir la interfaz (visor de imágenes, reproductor de audio); agrega un paso que puede fallar al procesamiento del webhook, con su propio manejo de reintentos.
- **Esfuerzo, costo y riesgo:** esfuerzo medio. Costo de R2, bajo al volumen del piloto. Riesgo medio, concentrado en el manejo de datos sensibles.

### Opción C: Descargar y guardar, mostrar solo el marcador

Se descargan y guardan todos los archivos desde el día uno, pero la interfaz sigue mostrando el marcador de `RF-CNV-10`. La visualización se construye después.

- **Ventajas:** **cierra la puerta irreversible sin abrir la discusión de interfaz**; nada se pierde y la visualización se puede agregar en cualquier momento sobre datos que ya están; es el menor esfuerzo que preserva las opciones futuras.
- **Desventajas:** durante el piloto el asesor sigue sin poder ver nada, con el agravante de que ahora el archivo **sí existe** y está guardado, lo que es difícil de explicar; se asume la custodia de datos sensibles sin obtener a cambio el beneficio de producto; el costo de pasar de aquí a la Opción B es solo la interfaz, que es la parte barata.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo-medio. Mismo riesgo legal que la Opción B, sin su beneficio.
- **Veredicto:** es la red de seguridad. Tiene sentido como posición de repliegue si el calendario aprieta, no como destino.

## Decisión

**Opción B**, con la Opción C como repliegue explícito.

Es decir: **la descarga y el guardado de archivos entran al MVP como P0 y no se recortan.** La visualización en la interfaz entra también, pero es lo que se sacrifica primero si el calendario aprieta en la revisión del 16 de noviembre.

Razón, en orden de peso:

1. **La descarga es irreversible y la interfaz no.** Es la única parte de este ADR que no admite "después". Tratarlas con el mismo criterio de alcance sería un error de categoría.

2. **El modo transitorio de ADR-001 eliminó la alternativa.** El argumento original de §13 —"la clínica no tiene otro lugar donde verlos"— no era una advertencia teórica: con una línea nueva del portafolio de Numi, es literalmente cierto. No hay app de respaldo.

3. **El segmento manda fotos y audios.** En clínicas estéticas y odontologías la imagen es parte de la consulta, no un accesorio. Un piloto donde el asesor no puede ver la foto de la lesión no mide la adopción del producto: mide la tolerancia a su ausencia.

4. **El costo incremental de B sobre C es solo la interfaz.** Toda la parte difícil —descargar a tiempo, guardar, cifrar, servir con control de acceso— es la misma en ambas. Habiendo pagado eso, no mostrar el archivo es desperdiciar el trabajo.

### Qué fija esta decisión para el diseño

1. **La descarga ocurre fuera de la invocación del webhook.** El webhook valida la firma, registra el mensaje con su identificador de archivo, encola la descarga y responde a Meta. Descargar dentro del webhook arriesga el tiempo de respuesta, y con él `RF-TRV-02`. El mecanismo de la cola es D-05.

2. **La descarga se intenta pronto y se reintenta dentro de la ventana de 7 días.** La URL vive 5 minutos, así que cada intento debe pedir una URL nueva a partir del identificador; no sirve guardar la URL. Pasados los 7 días el intento se abandona y el mensaje queda marcado como archivo no recuperado. Ese estado debe ser visible para el asesor, no un silencio.

3. **El archivo se guarda en R2, con una clave que incluye la organización.** El acceso se sirve siempre a través de la API, que verifica la organización del usuario en cada petición. **Nunca un enlace público ni un bucket de lectura abierta.** El detalle de la política de acceso es D-10.

4. **Los archivos quedan sujetos al mismo aislamiento que los mensajes.** Ninguna consulta puede devolver un archivo de otra organización (`RF-TRV-01`). Esto entra en las pruebas automáticas de aislamiento.

5. **Los archivos no llegan a la consola, ni a los registros, ni a las métricas** (`RF-TRV-07`, `RF-PLT-USO-04`). La consola puede ver que hubo un archivo y su tamaño; nunca su contenido ni su nombre si el nombre puede identificar a una persona.

6. **Se respetan los límites de tipo y tamaño de Meta.** No hace falta validar tamaño en la recepción, porque Meta ya los aplica al emisor; sí hay que tolerar un documento de hasta 100 MB sin que eso bloquee el procesamiento de otros mensajes.

7. **Retención igual a la del mensaje.** `docs/01_definicion_mvp_v2.2.md` §8 ya lo propone: *"Adjuntos: el mismo período del mensaje"*, 12 meses sujeto a validación. Al borrar un mensaje se borra su archivo.

8. **Los videos y stickers se descargan y guardan igual, aunque el MVP no los muestre.** El criterio es el mismo: guardar es irreversible, mostrar no. Para ellos la interfaz mantiene el marcador de `RF-CNV-10`.

### Qué cambia en los requisitos

`RF-CNV-10` cambia de significado y hay que reescribirlo. Hoy dice que los tipos no mostrados se marcan. Pasa a cubrir solo los tipos que efectivamente no se muestran (video, sticker, ubicación, contacto y otros), más el caso de archivo no recuperado.

Se agregan requisitos nuevos, a confirmar con el equipo al actualizar el documento maestro:

| Requisito propuesto | Qué dice | Prioridad |
| --- | --- | --- |
| `RF-CNV-10a` | Toda imagen, audio, documento, video o sticker recibido se descarga de Meta y se guarda, dentro de la ventana de 7 días | P0 |
| `RF-CNV-10b` | La conversación muestra las imágenes recibidas, con ampliación al hacer clic | P0 |
| `RF-CNV-10c` | La conversación reproduce los audios recibidos sin salir de la pantalla | P0 |
| `RF-CNV-10d` | Los documentos recibidos se muestran con su nombre y tamaño, y se descargan al hacer clic | P0 |
| `RF-CNV-10e` | Si un archivo no se pudo recuperar, el mensaje lo indica con su tipo y hora | P0 |

**El envío de archivos desde el CRM queda fuera del MVP.** Esta decisión es sobre lo que se recibe. El asesor sigue enviando solo texto (`RF-CNV-07`). Agregar el envío es una decisión de alcance separada, y sí es reversible.

## Consecuencias

### Más fácil

- El piloto mide el producto real y no una versión mutilada de él.
- Nada se pierde: aunque la interfaz llegara tarde, los archivos del piloto estarán ahí.
- El caso de uso central del segmento —mandar una foto— funciona.

### Más difícil

- **Numi pasa a custodiar datos sensibles de salud.** Esto sube la exigencia del acuerdo de encargo con cada negocio y de la política de tratamiento que `docs/01_definicion_mvp_v2.2.md` §8 ya exigía antes de usar datos reales. Conviene que el abogado lo revise sabiendo que hay fotos clínicas en juego, no solo texto.
- El procesamiento del webhook gana un paso asíncrono con su propio manejo de fallos y su ventana de caducidad.
- Hay trabajo de interfaz nuevo: visor de imágenes y reproductor de audio.
- Aparece almacenamiento que hay que asegurar, controlar por organización y borrar según la política de retención.

### Qué hay que construir

- Descarga de archivos de Meta, encolada, con reintentos y con límite de 7 días.
- Guardado en R2 con clave por organización.
- Entrega del archivo a través de la API, con verificación de la organización en cada petición.
- Estado de archivo no recuperado, visible en la conversación.
- Visor de imágenes y reproductor de audio en la vista de conversación (V11).
- Borrado del archivo cuando se borra el mensaje.

### Qué documentos hay que actualizar

- `docs/02_documento_maestro_v1.1.md` §5.2: reescribir `RF-CNV-10` y agregar `RF-CNV-10a` a `RF-CNV-10e`.
- `docs/02_documento_maestro_v1.1.md` §6, vista V11: agregar las variantes de mensaje con imagen, audio y documento.
- `docs/02_documento_maestro_v1.1.md` §9: el texto *"Este tipo de mensaje todavía no se puede ver en Numi CRM"* queda solo para video, sticker y archivo no recuperado.
- `docs/02_documento_maestro_v1.1.md` §12: quitar la visualización de multimedia de los elementos diferidos.
- `docs/01_definicion_mvp_v2.2.md` §13: el riesgo "Audios, imágenes y documentos" queda cerrado por este ADR.
- `docs/01_definicion_mvp_v2.2.md` §8: la fila "Adjuntos" de retención deja de ser condicional.
- `docs/01_definicion_mvp_v2.2.md` §8.1: la fila "Archivos" deja de decir *"no se usa si los adjuntos quedan fuera"*; R2 entra.

### Impacto en el cronograma

Es alcance nuevo que no estaba contemplado, en un calendario que ADR-002 ya dejó ajustado. Hay que revisarlo en la revisión del 16 de noviembre.

**Orden de repliegue si aprieta**, de lo primero a lo último que se sacrifica:

1. Visor de documentos → enlace de descarga simple.
2. Reproductor de audio en línea → enlace de descarga.
3. Visor de imágenes con ampliación → imagen en línea sin ampliación.
4. **Toda la interfaz** → marcador de `RF-CNV-10`, es decir la Opción C.

**Lo que no se recorta en ningún escenario: la descarga y el guardado.** Es la parte irreversible.

## Qué revisar cuando el sistema crezca

- **Si el costo de R2 deja de ser trivial:** revisar la retención de archivos por separado de la de mensajes. Los archivos pesan mucho más y puede tener sentido borrarlos antes.
- **Si aparece la necesidad de enviar archivos desde el CRM:** es una decisión nueva, con su propio alcance (subida, validación de tipo y tamaño, envío a Meta).
- **Si un negocio pide exportar sus conversaciones:** los archivos forman parte de esa exportación, y eso cambia el diseño de la función de exportación.
- **Si entra la coexistencia** (ADR-001): el negocio vuelve a tener la app de WhatsApp Business como lugar alternativo donde ver los archivos. Eso no revierte esta decisión, pero sí baja su urgencia relativa.
- **Si el volumen de archivos por conversación resulta mucho mayor de lo supuesto:** revisar si conviene comprimir imágenes al guardarlas, cosa que hoy no se hace.
