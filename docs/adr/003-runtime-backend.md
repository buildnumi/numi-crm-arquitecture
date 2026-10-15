# ADR-003: Runtime del backend

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-TRV-01, RF-TRV-02, RF-CNV-06, RF-CNV-08, RF-CNV-09, RF-PLT-SEG-01, RF-PLT-SEG-02
- **Depende de:** ADR-001
- **Condiciona a:** D-05 (cola y webhook), D-06 (aislamiento multiempresa), D-07 (tiempo real), D-09 (frontend), D-11 (despliegue y observabilidad)

## Contexto

El brief propone **Fastify ejecutado en Cloudflare Workers** (`docs/01_definicion_mvp_v2.2.md` §8.1, fila "Backend"). Ese mismo documento ya registra el problema en §8.2, punto 1: *"Cloudflare indica que Express y Koa ya funcionan en Workers, pero Fastify todavía no"*.

Hay que elegir el runtime ahora porque **es la decisión que más condiciona a las demás**. De ella dependen cómo se procesa el webhook (D-05), cómo se aplica el aislamiento por organización (D-06), qué mecanismo de tiempo real está disponible (D-07), dónde vive el frontend (D-09) y cómo se despliega todo (D-11). Cambiarla después de construir significa reescribir la capa de entrada de toda la API.

### Restricciones que aplican

- Dos personas desarrollando con tiempo parcial; MVP el 30 de noviembre de 2026 (`CLAUDE.md` §8).
- Volumen de referencia: 500 conversaciones al mes por negocio (`CLAUDE.md` §8). Con 3 organizaciones en el piloto son unas 1.500 conversaciones al mes; suponiendo 10 mensajes por conversación, cerca de **15.000 mensajes al mes**, es decir menos de **0,01 mensajes por segundo** de promedio. El pico razonable está muy por debajo de 1 mensaje por segundo.
- Pocos costos fijos; verificar precios vigentes (`CLAUDE.md` §8).
- Node.js con TypeScript (`CLAUDE.md` §9).
- Por ADR-001, la capa de canal recibe la organización en toda llamada a Meta y los tokens van cifrados, nunca en variables de entorno globales.

### Datos verificados de proveedores

Todos verificados el 2026-10-15 contra la documentación oficial.

| Dato | Valor | Fuente |
| --- | --- | --- |
| Express en Workers | **Funciona.** Hay tutorial oficial | [Deploy an Express.js application on Cloudflare Workers](https://developers.cloudflare.com/workers/tutorials/deploy-an-express-app/) |
| Koa en Workers | Funciona | [A year of improving Node.js compatibility in Cloudflare Workers](https://blog.cloudflare.com/nodejs-workers-2025/) |
| Fastify en Workers | **No soportado todavía** | mismo |
| `nodejs_compat` | Activo **por defecto** con compatibility date 2026-08-04 o posterior | [Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/) |
| Servidor HTTP de Node (`http.createServer`) | Soportado desde compatibility date 2025-09-01 con `nodejs_compat` | mismo |
| Hyperdrive con Supabase | Soportado, con guía oficial | [Hyperdrive · Supabase](https://developers.cloudflare.com/hyperdrive/examples/connect-to-postgres/postgres-database-providers/supabase/) |
| Cadena de conexión para Hyperdrive | La **Direct connection**, no las agrupadas (Hyperdrive hace su propio pooling) | mismo |
| Driver recomendado | node-postgres, versión mínima 8.16.3 | mismo |
| Hyperdrive, planes | Disponible en plan gratuito y de pago | [Hyperdrive](https://developers.cloudflare.com/hyperdrive/) |
| Cloudflare Queues, gratuito | 10.000 operaciones al día; retención de 24 h no configurable | [Queues · Pricing](https://developers.cloudflare.com/queues/platform/pricing/) |
| Cloudflare Queues, de pago | 1.000.000 de operaciones al mes incluidas, luego 0,40 USD por millón; retención de 4 días, configurable hasta 14 | mismo |

Cita literal del dato que cierra la opción del brief:

> "we're hoping to be able to add Fastify support later"

### Qué cambió desde que se escribió el brief

Dos cosas, y ambas empujan en la misma dirección:

1. **Express pasó a funcionar en Workers** y tiene tutorial oficial de Cloudflare. Cuando se escribió el brief esto era una limitación general de Node en Workers; hoy es específica de Fastify.
2. **`nodejs_compat` dejó de ser una bandera que hay que recordar activar** y viene por defecto desde el 4 de agosto de 2026. La compatibilidad con Node en Workers dejó de ser territorio experimental.

Lo que **no** cambió: Fastify sigue sin soporte, y Cloudflare lo plantea como una intención, no como una fecha. **Construir sobre esa promesa sería apostar el MVP a un calendario ajeno**, que es exactamente el riesgo que ADR-001 ya identificó como principal del proyecto con Meta. Dos apuestas al mismo tipo de riesgo es una de más.

### Qué está realmente en juego

El tráfico del piloto es insignificante para cualquiera de las tres opciones: menos de 1 mensaje por segundo. **El rendimiento no es el criterio.** Lo que decide es:

- **Cuántos proveedores hay que montar y operar** en siete semanas con dos personas.
- **Qué primitivas quedan disponibles** para D-05 (cola), D-06 (aislamiento) y D-07 (tiempo real).
- **Qué pasa con el webhook de Meta**, que es el único punto del sistema con un requisito duro de disponibilidad: si no responde, Meta reintenta, y el sistema no puede perder ni duplicar mensajes (`RF-TRV-02`).

Un punto que conviene decir claro antes de comparar: **el framework HTTP es la parte menos importante de esta decisión.** El código de negocio no depende de él si se lo mantiene en la capa de entrada. Lo que de verdad se está eligiendo es **dónde se ejecuta el backend**: en Workers o en un servicio Node común.

## Opciones consideradas

### Opción A: Hono en Cloudflare Workers

Framework hecho para el runtime de Workers, sobre la API estándar de Fetch (`Request`/`Response`).

- **Ventajas:** no necesita capa de compatibilidad con Node, así que no hereda las lagunas de `nodejs_compat`; acceso directo a las primitivas de Cloudflare que las decisiones siguientes van a necesitar (Queues para D-05, R2 para D-10, Durable Objects si D-07 lo requiere); TypeScript de primera clase, con tipado del entorno y sus bindings; un solo proveedor para API, cola y archivos, con un solo despliegue; sin servidores que administrar ni parches de sistema operativo; escala a cero, lo que encaja con "pocos costos fijos"; el webhook de Meta queda servido por la red de Cloudflare, que es lo más disponible que el equipo puede conseguir sin operar nada.
- **Desventajas:** es el framework que ninguno de los dos conoce todavía, así que hay una curva de aprendizaje, aunque pequeña —su API es deliberadamente parecida a la de Express; algunas bibliotecas de npm que asumen Node completo pueden no funcionar y hay que verificarlas una por una antes de adoptarlas; el modelo de ejecución es distinto al de un servidor (no hay estado en memoria entre peticiones, hay límites de CPU por invocación), lo que obliga a pensar diferente algunas cosas; depender de un proveedor hace más caro mudarse después.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo en montaje, medio en aprendizaje. Costo: plan de pago de Workers (5 USD al mes) si se usan Queues con retención mayor a 24 horas, que D-05 muy probablemente va a pedir. Riesgo bajo.
- **Fuentes:** [Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/), [Queues · Pricing](https://developers.cloudflare.com/queues/platform/pricing/).

### Opción B: Express en Cloudflare Workers

Express sobre la capa de compatibilidad con Node de Workers.

- **Ventajas:** es el framework que todo desarrollador de Node ya conoce, así que no hay curva; tiene tutorial oficial de Cloudflare; conserva las mismas primitivas de Cloudflare que la Opción A; el ecosistema de middleware de Express es enorme.
- **Desventajas:** funciona **a través de una capa de compatibilidad**, no de forma nativa, lo que agrega una superficie de cosas que pueden fallar de maneras poco obvias y difíciles de diagnosticar; el middleware de Express suele asumir un servidor Node completo (sistema de archivos, estado en memoria, procesos), supuestos que en Workers no se cumplen, así que cada paquete hay que verificarlo igual; se paga el costo de la capa de compatibilidad sin obtener a cambio portabilidad real, porque el resto del stack (Queues, bindings, R2) sigue atado a Cloudflare.
- **Esfuerzo, costo y riesgo:** esfuerzo bajo al empezar, con riesgo de encontrarse problemas de compatibilidad a mitad de camino. Costo igual al de la Opción A. Riesgo medio.
- **Veredicto:** da la comodidad de lo conocido, pero paga una capa de traducción para no obtener ninguna ventaja estructural sobre la Opción A.

### Opción C: Fastify en un servicio Node común

Se abandona Workers para el backend y se despliega Fastify en un servicio administrado de contenedores (Railway, Render, Fly.io o similar), dejando Cloudflare solo para el frontend.

- **Ventajas:** Node completo, sin lagunas de compatibilidad ni verificación de paquetes; es lo que dice el brief; estado en memoria y conexiones persistentes disponibles, lo que simplifica algunas cosas; el proceso es largo, así que las conexiones a la base de datos se reutilizan sin Hyperdrive; menos acoplamiento a un proveedor.
- **Desventajas:** introduce **un proveedor más** que montar, pagar, monitorear y desplegar, además de Cloudflare y Supabase, con dos personas a tiempo parcial; el webhook de Meta queda servido por un contenedor con una sola instancia, que al reiniciarse o al quedarse sin memoria deja de responder, y Meta reintenta —el requisito duro de `RF-TRV-02` queda apoyado en infraestructura que el equipo sí tiene que operar; hay que resolver la cola por fuera (otro servicio, o una tabla en Postgres), porque Cloudflare Queues no se invoca cómodamente desde afuera; con el frontend en Cloudflare y el backend en otro proveedor, el despliegue y la observabilidad quedan partidos en dos; costo fijo mensual aunque no haya tráfico.
- **Esfuerzo, costo y riesgo:** esfuerzo alto en montaje y operación. Costo: entre 5 y 20 USD al mes del servicio, más Cloudflare. Riesgo medio-alto, concentrado en la operación y en el calendario.
- **Veredicto:** es la opción que más infraestructura pone en manos de un equipo de dos personas que no tiene tiempo para operarla.

## Decisión

**Opción A: Hono en Cloudflare Workers.**

Razón, en orden de peso:

1. **Fastify queda descartado por un hecho verificado**, no por preferencia: no funciona en Workers y Cloudflare no da fecha. Eso convierte la propuesta del brief en inviable tal como está escrita, y obliga a elegir entre Workers con otro framework o salir de Workers.

2. **Salir de Workers cuesta un proveedor más.** Con dos personas a tiempo parcial y siete semanas, cada servicio que haya que montar, monitorear y desplegar se paga en tiempo que no está disponible. La Opción C es la que más trabajo de operación agrega para resolver un problema —el rendimiento— que a este volumen no existe.

3. **Entre Hono y Express en Workers, Hono gana por ser nativo.** Las dos tienen el mismo costo y las mismas primitivas. La diferencia es que Express pasa por una capa de compatibilidad cuyos fallos son oscuros, y la ventaja que compra con eso —el ecosistema de middleware de Node— no se puede usar del todo, porque ese middleware asume un servidor que en Workers no existe. La curva de aprendizaje de Hono es de días, y su API se parece deliberadamente a la de Express.

4. **El webhook de Meta queda en la red de Cloudflare.** Es el único punto del sistema con un requisito duro de disponibilidad (`RF-TRV-02`): si no responde, Meta reintenta y puede duplicar o perder mensajes. Servirlo desde una red global sin operar nada es mejor que servirlo desde un contenedor que el equipo tiene que mantener vivo.

### Qué fija esta decisión para el diseño

1. **El framework vive solo en la capa de entrada.** Rutas, validación de la petición y serialización de la respuesta usan Hono. El código de negocio —canal, conversaciones, contactos, asignación, métricas, consola, auditoría (`CLAUDE.md` §7, sección 4)— **no importa nada de Hono** y recibe sus dependencias como parámetros. Esto es lo que mantiene el costo de un cambio futuro de runtime acotado a la capa de entrada.

2. **Nada de estado en memoria entre peticiones.** No hay variables de módulo que acumulen datos, ni cachés en memoria, ni contadores. Un Worker puede morir entre dos peticiones y cada invocación puede caer en una instancia distinta. Todo estado vive en la base de datos.

3. **Toda biblioteca de npm se verifica antes de adoptarse.** Se prueba que funcione en el runtime de Workers, no solo que compile. Esto vale en especial para la biblioteca de cifrado de tokens (`RF-PLT-SEG-02`) y para el cliente de Postgres.

4. **Acceso a Postgres por Hyperdrive, con la cadena de conexión Direct** de Supabase, no las agrupadas, porque Hyperdrive hace su propio pooling. Driver: node-postgres 8.16.3 o superior.

5. **Un usuario de base de datos propio para Hyperdrive, con privilegios mínimos.** La guía de Cloudflare usa por comodidad el rol `postgres` en su ejemplo y advierte que en la práctica conviene un rol con menos privilegios. Se crea un rol dedicado. Qué privilegios exactamente lo define D-06.

6. **Compatibility date de 2026-08-04 o posterior**, con lo que `nodejs_compat` queda activo por defecto.

7. **El webhook de Meta responde rápido y delega.** La invocación tiene un límite de CPU; validar la firma, encolar y responder 200 a Meta es trabajo corto. Todo lo demás ocurre fuera de esa invocación. El mecanismo concreto es D-05.

8. **Consola y CRM comparten este backend.** ADR-002 fija que la consola es un área con su propio marco; esta decisión fija que las acciones de la consola se ejecutan en el mismo servidor, que verifica el rol de plataforma en cada petición (`RF-PLT-SEG-01`).

### Qué queda explícitamente fuera de esta decisión

- **Cómo se procesa el webhook** (cola, idempotencia, orden, reintentos): D-05. Esta decisión solo fija que la invocación del webhook debe ser corta y delegar.
- **Cómo se aplica el aislamiento por organización**: D-06. Esta decisión fija que hay un rol de base de datos dedicado, no cuáles son sus privilegios.
- **Qué mecanismo de tiempo real se usa**: D-07.
- **Si el frontend va en Workers con OpenNext o en otro lado**: D-09.

## Consecuencias

### Más fácil

- Un solo proveedor y un solo despliegue para API, cola y archivos.
- Las decisiones D-05, D-07 y D-10 tienen primitivas disponibles sin montar servicios nuevos.
- Sin servidores que administrar, parchear ni vigilar: con dos personas a tiempo parcial, el trabajo que no hay que hacer vale tanto como el que sí.
- El webhook de Meta queda sobre infraestructura global, que es lo que más ayuda a cumplir `RF-TRV-02`.
- Escala a cero: el costo acompaña al uso real, no al tamaño del plan.

### Más difícil

- **Hay que aprender Hono y el modelo de ejecución de Workers.** La curva es de días, pero existe, y el calendario está ajustado.
- **Cada biblioteca de npm hay que verificarla** en el runtime antes de adoptarla. Esto puede aparecer como sorpresa a mitad de una tarea.
- **No hay estado en memoria ni conexiones persistentes.** Hay cosas que en un servidor Node se resuelven con una variable y aquí necesitan la base de datos.
- **Acoplamiento a Cloudflare.** Salir después cuesta reescribir la capa de entrada y reemplazar Queues y R2. Se acota manteniendo el código de negocio independiente del framework (punto 1), pero no desaparece.
- **Depurar es distinto:** no hay un proceso al que conectarse. La observabilidad pasa a depender de lo que se instrumente, y eso es D-11.

### Qué hay que construir

- Proyecto de Workers con Hono, TypeScript y compatibility date 2026-08-04 o posterior.
- Capa de entrada con rutas, validación de entrada y manejo de errores uniforme, separada del código de negocio.
- Configuración de Hyperdrive apuntando a la cadena Direct de Supabase, con un rol de base de datos dedicado y de privilegios mínimos.
- Acceso a datos con node-postgres 8.16.3 o superior, creando el cliente por petición.
- Verificación de firma del webhook de Meta en la capa de entrada, antes de cualquier otro trabajo (`RF-TRV-02`).
- Cifrado y descifrado de tokens de WhatsApp con una biblioteca verificada en el runtime de Workers (`RF-PLT-SEG-02`, ADR-001).

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §8.1: la fila "Backend" pasa de *"Fastify, ejecutado en Cloudflare Workers"* a *"Hono en Cloudflare Workers"*.
- `docs/01_definicion_mvp_v2.2.md` §8.2, punto 1: el punto técnico de Fastify queda resuelto por este ADR.
- `docs/01_definicion_mvp_v2.2.md` §13 (riesgos), fila "Complejidad del stack": se mantiene el stack del brief con un solo cambio de framework; el riesgo baja.
- `CLAUDE.md` §9: la fila "Backend" de la tabla de stack de partida.

### Costo mensual estimado

| Concepto | Costo |
| --- | --- |
| Workers, plan de pago | 5 USD al mes |
| Queues | Incluido hasta 1.000.000 de operaciones al mes |
| Hyperdrive | Incluido |
| Supabase | Según D-11 |

El plan gratuito de Workers alcanzaría para el volumen del piloto, pero la retención de 24 horas no configurable de Queues en ese plan es un riesgo para `RF-TRV-02`: un incidente de más de un día haría perder mensajes encolados. La decisión sobre el plan se confirma en D-05, con los precios revalidados en ese momento.

## Qué revisar cuando el sistema crezca

- **Si Cloudflare anuncia soporte de Fastify:** no es razón para migrar. Esta decisión no se tomó contra Fastify sino a favor de un solo proveedor y de ejecución nativa.
- **Si una biblioteca indispensable no funciona en Workers:** antes de cambiar de runtime, evaluar aislarla en un servicio aparte invocado desde el Worker. Cambiar de runtime es la última opción, no la primera.
- **Si el límite de CPU por invocación empieza a apretar:** revisar qué trabajo se está haciendo dentro de la petición que debería estar en la cola. Casi siempre es un problema de diseño, no de plataforma.
- **Si aparece un requisito de conexión persistente** (por ejemplo WebSockets propios en vez del tiempo real de Supabase): evaluar Durable Objects antes de replantear el runtime. Se cruza con D-07.
- **Al superar unas 50 organizaciones activas:** revisar el dimensionamiento de Hyperdrive y los límites de conexiones de Supabase, que a ese volumen pasan a ser el primer cuello de botella esperable.
- **Si el equipo crece y hay alguien dedicado a operar infraestructura:** la Opción C deja de tener la desventaja que la descartó, y conviene reevaluarla con los números de ese momento.
