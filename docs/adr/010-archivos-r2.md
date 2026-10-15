# ADR-010: Archivos y almacenamiento en R2

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-CNV-10a a RF-CNV-10e, RF-TRV-01, RF-TRV-07, RF-AUT-07, RF-CNV-16
- **Depende de:** ADR-003, ADR-004, ADR-006, ADR-007
- **Condiciona a:** D-11 (despliegue y observabilidad)

## Contexto

ADR-004 decidió que los audios, imágenes y documentos recibidos por WhatsApp **se descargan, se guardan y se muestran**, y ya fijó buena parte del terreno:

- El archivo se guarda en R2, con una clave que incluye la organización.
- El acceso se sirve **siempre a través de la API**, que verifica la organización en cada petición.
- **Nunca un enlace público ni un bucket de lectura abierta.**
- Retención igual a la del mensaje; al borrar un mensaje se borra su archivo.

Lo que queda por decidir es más concreto de lo que el tablero sugiere, y se reduce a tres preguntas:

1. **¿Por dónde viaja el archivo hasta el navegador?** Es la pregunta real de esta decisión, y es un cruce entre seguridad y costo.
2. **¿Cómo se estructuran las claves y qué pasa cuando hay que borrar?**
3. **¿Hay algún otro archivo en el sistema además de los de WhatsApp?**

### Restricciones que aplican

- Los archivos pueden contener **datos sensibles de salud**: fotos clínicas, radiografías (ADR-004). Ley 1581 de 2012, negocio responsable y Numi encargado (`CLAUDE.md` §8).
- `RF-TRV-01`: ninguna consulta puede devolver un archivo de otra organización.
- `RF-TRV-07`: los archivos no llegan a la consola, los registros ni las métricas.
- ADR-007: toda autorización se verifica en el servidor en **cada petición**, contra la base, no contra el token.
- Runtime: Hono en Workers, con R2 como binding del mismo proveedor (ADR-003).
- Volumen: unas 1.500 conversaciones mensuales en el piloto (ADR-003). Suponiendo que una de cada cinco traiga un archivo de 1 MB de promedio, son unos **300 MB al mes**.

### Datos verificados de Cloudflare R2

Verificados el 2026-10-15 contra la documentación oficial.

**Precio** — [R2 Pricing](https://developers.cloudflare.com/r2/pricing/)

| Concepto | Precio | Incluido gratis |
| --- | --- | --- |
| Almacenamiento | 0,015 USD por GB al mes | 10 GB al mes |
| Operaciones de clase A (escritura) | 4,50 USD por millón | 1.000.000 al mes |
| Operaciones de clase B (lectura) | 0,36 USD por millón | 10.000.000 al mes |
| **Salida de datos** | **Gratis** | — |

Cita del dato que más pesa:

> "Egress directly from R2 through the Workers API, S3 API, or r2.dev domains does not incur data transfer (egress) charges and is free."

**URL firmadas** — [Presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)

| Dato | Valor |
| --- | --- |
| Vigencia | De 1 segundo a **7 días** |
| Naturaleza | "**should be treated as bearer tokens**": quien tenga la URL puede hacer la operación hasta que expire |
| Recomendación | Vigencias cortas para operaciones sensibles |
| Bucket público con dominio propio | Cualquiera que conozca la URL puede leer el contenido |

### El punto que de verdad hay que decidir

**¿El archivo pasa por el Worker, o el navegador lo baja directo de R2 con una URL firmada?**

Normalmente esta pregunta se decide por costo: hacer pasar archivos por un servidor cuesta ancho de banda, y por eso casi todos los sistemas entregan una URL firmada y se desentienden. **En R2 ese argumento no existe, porque la salida de datos es gratuita.**

Eso deja la decisión apoyada enteramente en seguridad, y ahí la diferencia es nítida:

- Una **URL firmada es un token al portador**. Quien la tenga accede al archivo, sin importar quién sea. Si se copia, se reenvía por WhatsApp o queda en el historial del navegador, sigue funcionando hasta que expire. Para una foto clínica, eso es una fuga de datos sensibles que **ningún control posterior puede revertir**.
- Un archivo **servido por el Worker** se verifica en cada petición: usuario, membresía, estado de la organización (ADR-007). Si el usuario fue desactivado hace un minuto, no lo recibe.

Hay un detalle que inclina aún más la balanza: ADR-007 construyó toda la autorización sobre la premisa de que **el estado se verifica contra la base en cada petición**, precisamente porque un token no se puede revocar. Una URL firmada reintroduce exactamente ese problema, por tercera vez —ya apareció con los tokens de sesión y con el canal de tiempo real—. La respuesta coherente es la misma que las dos veces anteriores.

## Opciones consideradas

### Opción A: URL firmada, el navegador baja directo de R2

La API verifica permisos, genera una URL firmada de vida corta y la devuelve; el navegador descarga de R2.

- **Ventajas:** el archivo no pasa por el Worker, así que no consume su tiempo de CPU ni su memoria; es el patrón habitual en la industria; soporta archivos grandes sin esfuerzo.
- **Desventajas:** **la URL es un token al portador** y funciona para cualquiera que la tenga hasta que expire; una vez emitida **no se puede revocar**, así que desactivar un usuario o suspender una organización no cancela los accesos ya entregados; la URL queda en el historial del navegador, en los registros del cliente y en cualquier lugar donde se copie; con datos sensibles de salud, el riesgo no es teórico; hay que implementar la firma con credenciales S3 y una biblioteca adicional.
- **Esfuerzo, costo y riesgo:** esfuerzo medio. Costo igual que B, porque la salida es gratis en ambos casos. **Riesgo alto** sobre datos sensibles.
- **Veredicto:** su principal ventaja —ahorro de ancho de banda— no aplica en R2.

### Opción B: El Worker sirve el archivo, verificando en cada petición

El navegador pide el archivo a la API; el Worker verifica permisos, lee de R2 por el binding y devuelve el contenido en flujo.

- **Ventajas:** **cada acceso se verifica**, con usuario, membresía y estado, igual que cualquier otra ruta (ADR-007); no existen enlaces que sigan funcionando después de revocar un acceso; nada que firmar ni credenciales S3 que custodiar —el binding de R2 ya está disponible; coherente con el resto del sistema, sin un camino de datos aparte; se puede registrar quién accedió a qué, sin registrar el contenido.
- **Desventajas:** el archivo pasa por el Worker, consumiendo su tiempo y su memoria; un documento de 100 MB (ADR-004) es un caso a vigilar, aunque leer en flujo lo hace manejable; agrega una operación de lectura de R2 por acceso —dentro del margen gratuito con enorme holgura.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo. Riesgo bajo.

### Opción C: Bucket público con nombres difíciles de adivinar

Los archivos en un bucket público, con claves aleatorias largas.

- **Desventajas:** es seguridad por oscuridad; cualquiera con la URL accede, para siempre; contradice frontalmente lo que ADR-004 ya decidió —"nunca un enlace público ni un bucket de lectura abierta"— y `RF-TRV-01`.
- **Veredicto:** descartada. Se incluye solo para dejar constancia de que se consideró y por qué no.

## Decisión

**Opción B: el Worker sirve los archivos, verificando permisos en cada petición.**

Razón:

1. **El argumento habitual a favor de las URL firmadas —el costo de salida— no existe en R2.** Se estaría aceptando el riesgo de un token al portador sin obtener la ventaja que normalmente lo justifica.
2. **Son datos sensibles de salud.** Una foto clínica filtrada no se puede recuperar. La asimetría entre el costo de equivocarse y el beneficio de ahorrar tiempo de CPU es enorme.
3. **Es coherente con el resto del sistema.** ADR-007 verifica en cada petición, ADR-008 hizo que el tiempo real no fuera una vía de acceso a datos. Esta decisión mantiene la misma propiedad: **la API es el único camino hacia los datos.**

### Estructura de las claves

Clave por organización y mensaje:

```
org/{organization_id}/msg/{message_id}/{media_id}
```

Qué resuelve cada parte:

- **La organización primero** permite listar y borrar todo lo de una organización con un prefijo, lo que importa al atender una solicitud de eliminación bajo la Ley 1581.
- **El mensaje** liga el archivo a su dueño, y hace directo el borrado en cascada que ADR-004 exige.
- **El identificador de Meta** evita colisiones y es idempotente: reprocesar el mismo evento (ADR-005) escribe la misma clave, no un duplicado.

**La clave nunca se expone al navegador.** El cliente pide `/api/messages/{id}/media`; la API resuelve la clave internamente. Así la estructura puede cambiar sin romper nada, y el cliente no puede construir claves para probar suerte.

### Verificación en cada acceso

Al pedir un archivo, la API hace lo mismo que en cualquier ruta (ADR-007), más una comprobación propia:

1. Verifica el token, resuelve usuario, membresía y estado.
2. Rechaza si el usuario está desactivado o la organización suspendida.
3. **Comprueba que el mensaje pertenece a la organización del usuario**, con filtro explícito y bajo RLS (ADR-006). Una conversación de otra organización no existe para esa consulta.
4. Lee de R2 por el binding y devuelve el contenido en flujo.

El paso 3 es el que cumple `RF-TRV-01` para los archivos, y es el que debe entrar en las pruebas automáticas de aislamiento: **un usuario de A pidiendo el archivo de un mensaje de B recibe el mismo resultado que si el mensaje no existiera.** No un error distinto, que revelaría su existencia.

### Qué se guarda y qué no

| Qué | ¿Va a R2? |
| --- | --- |
| Imágenes, audios, documentos, videos y stickers recibidos (ADR-004) | Sí |
| Archivos enviados desde el CRM | No existen: el envío de archivos está fuera del MVP (ADR-004) |
| Avatares o logotipos de organizaciones | No. No hay requisito que los pida |
| Copias de seguridad o exportaciones | No. No hay requisito en el MVP |
| Cuerpos crudos de eventos del webhook | No. Van a la tabla de eventos (ADR-005), que tiene su propia retención |

**R2 se usa para una sola cosa: los archivos recibidos por WhatsApp.** Esto responde directamente a la pregunta de D-10 —"si se usa R2 y para qué"— y respeta la regla 9 de `CLAUDE.md` §4: no se agregan usos sin un requisito que los pida.

### Borrado y retención

- **Retención igual a la del mensaje**, 12 meses sujeto a validación (`docs/01_definicion_mvp_v2.2.md` §8, ADR-004).
- **Al borrar un mensaje se borra su archivo.** Como R2 y Postgres son sistemas distintos, no hay transacción común: el borrado puede quedar a medias. El orden correcto es **borrar primero el archivo y después la fila**. Si falla entre ambos pasos, queda una fila sin archivo —que el sistema ya sabe manejar, porque es el mismo caso que un archivo no recuperado (`RF-CNV-10e`)—. El orden inverso dejaría archivos huérfanos invisibles, que es peor: nadie los encontraría para borrarlos.
- **Borrado por organización:** al eliminar una organización se borra todo su prefijo. Es la capacidad que hace viable atender una solicitud de eliminación de datos.
- **Tarea periódica de limpieza** que recorre los archivos sin fila asociada y los elimina. Es la red para los borrados a medias; sin ella, los huérfanos se acumulan en silencio.

### Costo

Con unos 300 MB mensuales de archivos:

| Concepto | Primer mes | A 12 meses de retención |
| --- | --- | --- |
| Almacenamiento | 0,3 GB | ~3,6 GB |
| Costo | 0 USD (dentro de los 10 GB gratuitos) | 0 USD |
| Operaciones de escritura | ~300 | Dentro de 1.000.000 gratuitas |
| Operaciones de lectura | pocos miles | Dentro de 10.000.000 gratuitas |
| Salida de datos | Gratis | Gratis |

**R2 no cuesta nada al volumen del piloto**, y el margen gratuito aguanta un orden de magnitud más de crecimiento. El primer costo real aparecería alrededor de las 25 a 30 organizaciones activas con 12 meses de historial acumulado.

### Qué fija esta decisión para el diseño

1. **R2 se accede solo por el binding del Worker.** Sin credenciales S3, sin URL firmadas, sin dominio público sobre el bucket.
2. **Un solo bucket**, separado por prefijo de organización. Un bucket por organización no aporta aislamiento real —el control está en la API— y multiplica la administración.
3. **Buckets distintos por entorno.** Desarrollo y producción no comparten bucket; corresponde a D-11.
4. **El archivo se devuelve en flujo**, no cargado entero en memoria. Importa con documentos de hasta 100 MB.
5. **Se envía el tipo de contenido que informó Meta**, y para documentos se fuerza la descarga en vez de la apertura en el navegador. Un archivo subido por un tercero no debe interpretarse como contenido activo en el dominio de la aplicación.
6. **Los registros anotan el acceso, nunca el contenido** (`RF-TRV-07`): quién, a qué mensaje, cuándo. Nunca el archivo ni el nombre original si puede identificar a una persona.
7. **La consola no accede a archivos.** `app_platform` no tiene permiso sobre las tablas de mensajes (ADR-006), así que no puede resolver qué archivo pedir. Es coherente por construcción.

## Consecuencias

### Más fácil

- No hay enlaces que sigan funcionando después de revocar un acceso.
- Un solo camino hacia los datos, verificado igual que todo lo demás.
- Sin credenciales S3 ni firmas que custodiar.
- Se puede saber quién accedió a cada archivo, que es exactamente lo que una auditoría de datos sensibles necesita.
- Costo cero al volumen del piloto.

### Más difícil

- Los archivos consumen tiempo y memoria del Worker. Un documento de 100 MB es el caso a vigilar.
- Hay dos sistemas de almacenamiento sin transacción común, y por eso hace falta una limpieza periódica.
- Cada acceso es una petición a la API más una lectura de R2, en lugar de una descarga directa.

### Qué hay que construir

- Escritura en R2 desde el consumidor de descarga (ADR-004, ADR-005), con clave idempotente.
- Ruta de la API que verifica permisos y devuelve el archivo en flujo.
- Resolución interna de la clave, sin exponerla al cliente.
- Borrado en cascada, archivo antes que fila.
- Borrado por prefijo de organización.
- Tarea periódica de limpieza de huérfanos.
- Prueba automática: un usuario de A no puede obtener el archivo de un mensaje de B, y recibe el mismo resultado que si no existiera.

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §8.1: la fila "Archivos" deja de ser condicional —ADR-004 ya lo anticipó—. R2 entra, con un solo uso: los archivos recibidos por WhatsApp.
- `docs/01_definicion_mvp_v2.2.md` §8: la fila "Adjuntos" de la tabla de retención deja de decir "solo si los adjuntos entran en el producto".

## Qué revisar cuando el sistema crezca

- **Si un documento grande causa problemas de memoria o tiempo:** antes de pasar a URL firmadas, evaluar un límite propio de tamaño por encima del cual solo se guarde la referencia. Las URL firmadas deberían ser el último recurso, no el primero.
- **Al superar los 10 GB almacenados:** empieza a haber costo. Son unas 25 a 30 organizaciones con un año de historial. Conviene entonces revisar si los archivos necesitan la misma retención que los mensajes: pesan mucho más y puede tener sentido borrarlos antes.
- **Si entra el envío de archivos desde el CRM:** es alcance nuevo (ADR-004) y agrega subida, validación de tipo y tamaño, y análisis de contenido malicioso —que hoy no hace falta porque los archivos vienen de Meta.
- **Si un negocio pide exportar sus conversaciones:** los archivos forman parte de la exportación, y conviene revisar entonces si las URL firmadas de vida muy corta son aceptables para ese caso acotado.
- **Si aparece un requisito de vista previa de imágenes en miniatura:** hoy se sirve el archivo completo. Generar miniaturas ahorraría tráfico y mejoraría la bandeja, pero es trabajo nuevo sin requisito que lo pida.
- **Si los archivos pasan a necesitar cifrado adicional en reposo:** R2 ya cifra en reposo. Un cifrado propio por organización solo tendría sentido si un acuerdo con un cliente lo exigiera explícitamente, y conviene revisarlo con el abogado antes de construirlo.
