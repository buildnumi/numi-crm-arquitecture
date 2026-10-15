# ADR-001: Modelo de conexión de WhatsApp

- **Estado:** Aprobada
- **Fecha:** 2026-10-14 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-PLT-WA-01 a RF-PLT-WA-08, RF-PLT-SEG-02, RF-CNV-08, RF-TRV-02
- **Depende de:** —

## Contexto

Numi CRM atiende conversaciones de WhatsApp de varios negocios desde una sola plataforma. Hay que decidir **cómo se conecta el número de WhatsApp de cada negocio** a la Cloud API de Meta, porque esa elección determina el modelo de permisos, los límites de mensajería, qué puede hacer la consola y cuánto cuesta cambiar de modelo más adelante.

### Restricciones que aplican

- Dos personas desarrollando con tiempo parcial; MVP el 30 de noviembre de 2026 y piloto el 1 de diciembre (`CLAUDE.md` §8).
- Volumen de referencia: 500 conversaciones al mes por negocio (`CLAUDE.md` §8).
- Solo WhatsApp, por la API oficial de Meta y sus webhooks (`CLAUDE.md` §8).
- Segmento inicial: inmobiliarias, clínicas estéticas y odontologías. **Mismo producto para los tres**; solo cambia a quién se le vende. Los nichos específicos se analizan después de validar la idea.
- Las conversaciones pueden contener datos sensibles de salud; Ley 1581 de 2012, con el negocio como responsable y Numi como encargado (`CLAUDE.md` §8).

### Estado del trámite con Meta al 2026-10-14

**Hechos:**

- La app de Numi AI **ya está publicada y aprobada** en Meta.
- La **verificación de negocio (Business Verification) no está completa**. Meta responde, con el caso de uso "La app requiere acceso a los permisos en Meta for Developers": *"No pudimos verificar tu organización con la información proporcionada. Se necesita más información"*. No es un rechazo: Meta no logró cruzar los datos enviados contra sus fuentes y pide información adicional.
- Falta además **la habilitación como Tech Provider**, que exige el negocio verificado.

Queda entonces una secuencia de dos pasos —verificar el negocio, luego habilitar Tech Provider— y no de tres, porque la aprobación de la app ya está resuelta.

Esto no es un detalle administrativo: condiciona todas las opciones de este ADR y es la ruta crítica del proyecto. Los tiempos de revisión los controla Meta, no el equipo.

Pendiente de confirmar: qué permisos tiene aprobados la app (en particular `whatsapp_business_management` y `whatsapp_business_messaging`) y si la sección de Tech Provider está visible o bloqueada hasta verificar el negocio. La documentación de Meta plantea la verificación del negocio como requisito previo al proceso de revisión, de modo que la aprobación ya obtenida y la habilitación como Tech Provider parecen ser trámites distintos.

### Datos verificados de Meta

Todos verificados el 2026-10-14 contra la documentación oficial.

| Dato | Valor | Fuente |
| --- | --- | --- |
| Números por portafolio sin verificar | 2 | [Business phone numbers](https://developers.facebook.com/documentation/business-messaging/whatsapp/business-phone-numbers/phone-numbers) |
| Números por portafolio verificado | 20 | mismo |
| Límite de mensajería de un portafolio nuevo | 250 destinatarios únicos / 24 h | [Messaging Limits](https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits) |
| Ámbito del límite | **Por portafolio, compartido entre todos sus números** | mismo |
| Qué cuenta contra el límite | Solo mensajes iniciados por el negocio **fuera** de la ventana de atención | mismo |
| Plantillas sin verificar / verificado | 250 / hasta 6.000 | [Messaging Limits](https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits) |
| Requisito para ser Tech Provider | Negocio verificado **antes** de iniciar la revisión de la app | [Become a Tech Provider](https://developers.facebook.com/documentation/business-messaging/whatsapp/solution-providers/get-started-for-tech-providers) |
| Requisito de la coexistencia | Ser ya Solution Partner o Tech Provider | [Onboard WhatsApp Business app users](https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/onboarding-business-app-users/) |
| Throughput en coexistencia | Fijo, 20 mensajes por segundo | mismo |
| Historial sincronizado en coexistencia | 180 días, solo con aprobación del negocio; sin grupos | mismo |
| Obsolescencia de Embedded Signup v2 | 15 de octubre de 2026 | mismo |

Citas literales de los dos datos que más pesan:

> "New business portfolios are initially capped at two registered business phone numbers."

> "calculated and set at the business portfolio level and are shared by all business phone numbers within a portfolio"

### El hallazgo que define la decisión

El plan inicial era que **Numi comprara un número por cada cliente dentro de su propio portafolio**. Los datos verificados muestran que ese modelo no escala, por dos razones independientes:

1. **Tope de números.** Sin verificación, el portafolio de Numi admite 2 números. Con verificación, 20. Un SaaS con más de 20 clientes choca con un muro que no se resuelve comprando más números.
2. **Cuota compartida.** El límite de 250 destinatarios únicos cada 24 horas es del portafolio y se reparte entre todos sus números. Un cliente que haga difusión consume la cuota de los demás, sin que ellos lo sepan ni puedan evitarlo. Esto es inaceptable en un producto multiempresa.

Atenuante que vale para el piloto: el límite cuenta solo mensajes iniciados por el negocio fuera de la ventana de atención. El MVP es sobre todo reactivo —el contacto escribe y el asesor responde (`RF-CNV-*`)—, así que durante una semana de piloto el techo de 250 probablemente no apriete. Aprieta en cuanto alguien quiera hacer difusión.

**Conclusión:** el único modelo que escala es que **cada negocio tenga su propio portafolio y su propia cuenta de WhatsApp**, con Numi accediendo como Tech Provider. Cada cliente tiene entonces su cuota, sus plantillas y su historial.

## Opciones consideradas

### Opción A: Números de Numi en el portafolio de Numi, sin coexistencia

Numi compra los números y los registra en su propio portafolio. Un token, un portafolio.

- **Ventajas:** es lo más rápido de construir; no depende de la revisión de la app de Meta; el cliente no toca nada; sirve para demostrar el producto.
- **Desventajas:** tope de 2 números sin verificación y 20 con ella; cuota de 250 compartida entre todos los clientes; el número es de Numi, así que el cliente no se lo lleva y el historial queda atado a la infraestructura de Numi; Numi deja de ser solo encargado del tratamiento y pasa a controlar el canal, lo que complica el encuadre de la Ley 1581.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo; costo de las líneas; riesgo alto como modelo de operación, bajo como piloto.
- **Veredicto:** sirve para pilotear, no para operar.

### Opción B: Cada negocio con su cuenta de WhatsApp, Numi como Tech Provider vía Embedded Signup

Cada cliente conecta su propio número desde su propio portafolio; Numi accede como proveedor técnico.

- **Ventajas:** cada cliente tiene su cuota, sus plantillas, su número y su historial; sin tope global de clientes; el cliente conserva su número al irse; el encuadre legal queda limpio (el negocio es responsable, Numi encargado); habilita la coexistencia más adelante.
- **Desventajas:** exige Business Verification **y** revisión de la app, ambas pendientes y con tiempos que controla Meta; el alta del cliente es más compleja (Embedded Signup).
- **Esfuerzo, costo y riesgo:** esfuerzo medio en producto; riesgo alto de calendario, concentrado en trámites externos.
- **Veredicto:** es el modelo correcto; su fecha de disponibilidad no depende del equipo.

### Opción C: Destino B, con A como modo transitorio sobre la misma capa de canal

Se diseña para Tech Provider desde el día uno. La capa de canal toma las credenciales **por organización**. Si la aprobación de Meta no llega a tiempo, el piloto corre con números de Numi sobre esa misma capa, cambiando configuración y no código.

- **Ventajas:** el piloto no queda rehén de un trámite externo; no hay reescritura posterior; coincide con los requisitos ya escritos, que contemplan conexión manual (`RF-PLT-WA-02`, P0) y Embedded Signup (`RF-PLT-WA-08`, P1); encaja con el criterio de "WhatsApp detrás de una interfaz de canal" (`CLAUDE.md` §9).
- **Desventajas:** obliga a construir la configuración por organización desde el inicio, incluso mientras haya un solo portafolio; hay que sostener dos modos durante un tiempo.
- **Esfuerzo, costo y riesgo:** el sobrecosto frente a A es pequeño si se hace ahora —es parametrizar credenciales, no una abstracción nueva— y enorme si se hace después con clientes en producción. Riesgo bajo.

## Decisión

**Opción C.** El modelo objetivo es **Tech Provider con Embedded Signup**, con cada negocio en su propio portafolio y su propia cuenta de WhatsApp. El piloto **arranca en modo transitorio** con números del portafolio de Numi, sobre la misma interfaz de canal.

Punto de partida confirmado por Franco y Anthony el 2026-10-15: el MVP y el piloto se construyen y se corren con los números del portafolio de Numi (conexión manual, `RF-PLT-WA-02`), y Tech Provider es el destino al que se migra sin reescribir la integración.

Razón: Tech Provider es el único modelo donde cada cliente tiene cuota propia y conserva su número, y es por tanto el único compatible con un SaaS multiempresa. El modo transitorio existe solo para que una dependencia externa no bloquee la validación del producto.

### Qué fija esta decisión para el diseño

1. **Credenciales por organización.** Cada organización guarda su `waba_id`, su `phone_number_id` y su token de acceso cifrado. **Nunca** un token global en variables de entorno. Esto es lo que vuelve el cambio de modo una cuestión de configuración.
2. **La capa de canal recibe la organización en toda llamada a Meta.** No puede existir un cliente de WhatsApp que asuma un único portafolio o un único token.
3. **Tokens cifrados**, fuera del navegador, de los registros y de la auditoría (`RF-PLT-SEG-02`, `RF-PLT-WA-03`).
4. **La organización registra su modo de conexión** (manual o Embedded Signup), que la consola muestra (`RF-PLT-WA-01`).
5. **Un número no puede estar en dos organizaciones** (`RF-PLT-WA-07`): restricción de unicidad sobre el identificador del número.
6. **Una organización puede existir sin WhatsApp conectado.** Es el estado normal entre el alta y la conexión, y la consola debe reflejarlo.
7. **Embedded Signup, versión vigente.** No construir sobre la v2, obsoleta desde el 15 de octubre de 2026.

## Consecuencias

### Más fácil

- Pasar del piloto a la operación real sin reescribir la integración con Meta.
- Aislar a los clientes entre sí también en los límites de Meta, no solo en la base de datos.
- Incorporar la coexistencia después, ya que exige el mismo rol de Tech Provider.

### Más difícil

- Hay que construir la gestión de credenciales por organización desde el inicio, con cifrado, aunque al principio haya un solo portafolio.
- Hay que sostener y probar dos modos de conexión durante la transición.
- El alta de un cliente por Embedded Signup es un flujo más complejo que pegar un token.

### Qué hay que construir

- Almacenamiento cifrado de credenciales de WhatsApp por organización.
- Interfaz de canal que reciba la organización como parámetro.
- Conexión manual en la consola (`RF-PLT-WA-02`), que es la P0 y cubre el modo transitorio.
- Prueba de conexión y salud del webhook (`RF-PLT-WA-04`, `RF-PLT-WA-05`).
- Más adelante, el flujo de Embedded Signup (`RF-PLT-WA-08`).

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §3: el segmento inicial pasa a ser inmobiliarias, clínicas estéticas y odontologías, con el mismo producto para los tres. Hoy dice solo clínicas estéticas.
- `docs/01_definicion_mvp_v2.2.md` §8.1 y §12: registrar que el modelo es Tech Provider y que Business Verification es la ruta crítica.
- `docs/03_consola_plataforma_v1.1.md` §5.5: confirmar que `RF-PLT-WA-08` sube a P0 cuando se obtenga el rol de Tech Provider.

### Riesgo abierto, y es el principal del proyecto

**La aprobación de Meta es la ruta crítica y no la controla el equipo.** La app ya está publicada y aprobada, así que quedan dos pasos secuenciales: **verificar el negocio** y luego **habilitarse como Tech Provider**. Hoy el primero está frenado porque faltan datos.

Mitigación: destrabar la verificación como tarea inmediata, en paralelo al diseño; y mantener el modo transitorio como red para que el piloto del 1 de diciembre ocurra igual.

Señal de alarma: si la verificación no está resuelta a mediados de noviembre, el piloto corre en modo transitorio con un máximo de 2 números, lo que limita el piloto a dos clientes.

## Qué revisar cuando el sistema crezca

- **Al superar 20 organizaciones activas en el portafolio de Numi:** el modo transitorio ya no da más. Para entonces, Tech Provider debe estar operativo.
- **Si un cliente necesita difusión masiva:** verificar que ya esté en su propio portafolio; en el transitorio consumiría la cuota de los demás.
- **Si se adopta la coexistencia:** revisar el throughput fijo de 20 mensajes por segundo y las funciones no soportadas (llamadas, listas de difusión).
- **Si el volumen supera con holgura las 500 conversaciones mensuales por negocio:** revisar los límites de mensajería del portafolio de cada cliente.
- **Antes de construir sobre Embedded Signup:** confirmar la versión vigente en la documentación de Meta.
