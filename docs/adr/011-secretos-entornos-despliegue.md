# ADR-011: Secretos, correo, entornos, despliegue y observabilidad

- **Estado:** Aprobada
- **Fecha:** 2026-10-15 (aprobada el 2026-10-15)
- **Decisores:** Franco, Anthony
- **Requisitos relacionados:** RF-PLT-SEG-02, RF-PLT-SEG-05, RF-TRV-08, RF-TRV-02, RF-PLT-WA-05, RF-PLT-ORG-06, RF-AUT-03, RF-AUT-04
- **Depende de:** ADR-001, ADR-003, ADR-005, ADR-007, ADR-008, ADR-009, ADR-010
- **Condiciona a:** —

## Contexto

Es la última decisión del tablero y recoge lo que las diez anteriores fueron dejando pendiente:

| Pendiente | De dónde viene |
| --- | --- |
| Correo con remitente propio, que **bloquea el alta de organizaciones** | ADR-007 |
| Plan de Supabase: los proyectos gratuitos se pausan tras una semana de inactividad | ADR-008 |
| "Allow public access" de Realtime desactivado, interruptor que **no está en el código** | ADR-008 |
| Cifrado de tokens de WhatsApp con biblioteca verificada en Workers | ADR-001, ADR-003 |
| Alertas de webhook caído, que **tienen un reloj de 7 días corriendo** | ADR-005 |
| Buckets de R2 separados por entorno | ADR-010 |

### Restricciones que aplican

- Dos personas con tiempo parcial; MVP el 30 de noviembre de 2026, piloto el 1 de diciembre (`CLAUDE.md` §8).
- Pocos costos fijos; verificar precios vigentes (`CLAUDE.md` §8).
- GitHub, con entornos de desarrollo y producción (`CLAUDE.md` §9).
- Datos sensibles de salud bajo la Ley 1581 de 2012.
- `RF-PLT-SEG-02`: tokens de WhatsApp cifrados, nunca en la interfaz, los registros técnicos ni la auditoría.
- `RF-TRV-07`: el contenido de las conversaciones nunca en la consola, los registros ni las métricas.

### Una advertencia que viene de la experiencia, no de la documentación

Franco reporta que en un proyecto anterior **tuvo que migrar de la capa REST de Supabase (PostgREST) a SQL directo, porque resultaba lenta y se caía con frecuencia.**

Esto coincide con lo que ya se decidió sin conocer el dato: ADR-003 fija acceso por Hyperdrive con node-postgres sobre la cadena **Direct**, y ADR-007 usa Supabase **solo como proveedor de identidad**. El camino de datos del sistema no toca PostgREST en ningún punto.

Pero tiene una consecuencia que sí corresponde a esta decisión y conviene mirar de frente: **Supabase no es una sola pieza, son tres**, y no fallan juntas.

| Pieza | Para qué se usa | Si falla |
| --- | --- | --- |
| Postgres | Todos los datos (ADR-003) | El sistema no funciona |
| Auth | Identidad (ADR-007) | Nadie puede entrar; quien ya tiene sesión sigue hasta que expire el token |
| Realtime | Avisos (ADR-008) | La bandeja sigue funcionando por la red de seguridad de 60 s |

El diseño ya reparte bien ese riesgo —la red de seguridad de ADR-008 existe precisamente para esto—, pero conviene registrarlo: **concentrar tres funciones en un proveedor con antecedentes de caídas es el riesgo de infraestructura más grande del sistema.** Se asume conscientemente, no por descuido, y se mitiga midiendo.

### Datos verificados

Verificados el 2026-10-15 contra la documentación oficial.

**Supabase** — [Pricing](https://supabase.com/pricing), [Compute and Disk](https://supabase.com/docs/guides/platform/compute-and-disk), [Pooling and limits](https://supabase.com/docs/guides/database/connecting-to-postgres/pooling-and-limits)

| Dato | Plan gratuito | Plan Pro |
| --- | --- | --- |
| Precio | 0 USD | 25 USD al mes |
| Pausa por inactividad | **Tras una semana** | No |
| Conexiones directas | 60 | 60 en el tamaño base, sube con el cómputo |
| Clientes del agrupador | 200 | 200 en el tamaño base |
| Agrupador dedicado | No, solo el compartido | Sí |
| Conexiones directas por IPv6 | Sí, salvo complemento IPv4 | Igual |

Las fuentes difieren en el nombre del tamaño base (Nano frente a Micro) y en algunas cifras; **hay que reconfirmar los números antes de contratar.**

**Resend** — [Pricing](https://resend.com/pricing)

| Dato | Plan gratuito | Plan Pro |
| --- | --- | --- |
| Precio | 0 USD | 20 USD al mes |
| Correos | 3.000 al mes, **límite de 100 por día** | 50.000 al mes, sin límite diario |
| Dominios propios | **3** | 10 |
| Remitente propio | **Sí, incluido** | Sí |

**Cloudflare** — verificado en ADR-003, ADR-005 y ADR-010.

| Concepto | Costo |
| --- | --- |
| Workers, plan de pago | 5 USD al mes |
| Queues | Incluido hasta 1.000.000 de operaciones |
| R2 | 0 USD al volumen del piloto |
| Hyperdrive | Incluido |

## Decisiones

Esta decisión agrupa seis temas. Cada uno se resuelve por separado.

### 1. Secretos y tokens cifrados

**Dos clases de secreto, con tratamiento distinto:**

| Clase | Qué son | Dónde viven |
| --- | --- | --- |
| **De la plataforma** | Clave de la app de Meta, clave de publicación de Broadcast, clave de Resend, cadena de conexión, clave de cifrado | Secretos de Workers, por entorno. Nunca en el repositorio |
| **De cada organización** | Token de acceso de WhatsApp (`waba_id`, `phone_number_id`, token) | **Cifrados en la base**, ADR-001 |

Los de la primera clase no van en variables de entorno del archivo de configuración, que se versiona, sino como secretos del Worker.

**Cifrado de los tokens de WhatsApp** (`RF-PLT-SEG-02`):

- Se cifran con **AES-GCM usando Web Crypto**, que es parte del runtime de Workers. No hace falta una biblioteca de npm, lo que evita el riesgo que ADR-003 señaló sobre verificar paquetes en el runtime.
- La clave de cifrado es un secreto del Worker, **no está en la base**. Quien obtenga una copia de la base no obtiene los tokens.
- **El token se escribe pero nunca se lee hacia la interfaz** (`RF-PLT-WA-03`). La consola solo muestra que está guardado y cuándo se actualizó.
- Se descifra únicamente en el momento de llamar a Meta, en memoria, dentro de la invocación.
- **Nunca aparece en registros ni en auditoría.** La auditoría registra "token actualizado", nunca el valor ni un fragmento.
- Se guarda junto al texto cifrado la referencia de la clave usada, para poder rotarla después sin perder lo ya cifrado.

### 2. Correo

**Resend, con dominio propio de Numi.** Es el pendiente que ADR-007 marcó como ruta crítica: sin correo no hay invitación (`RF-PLT-ORG-06`), y sin invitación el primer administrador no entra.

- El **plan gratuito alcanza**: 3.000 correos al mes con dominio propio incluido. El volumen real son invitaciones y restablecimientos —decenas al mes, no miles.
- **Atención al límite de 100 por día**, que es el que podría apretar en un día de altas múltiples. Para el piloto, con 2 o 3 organizaciones, no hay riesgo.
- **Hay que configurar SPF, DKIM y DMARC** en el dominio. Sin eso los correos llegan a la carpeta de no deseados, y una invitación que no llega es un alta que no ocurre. **Esto es lo que hay que hacer primero**, porque la propagación de DNS y el calentamiento del dominio no son instantáneos.
- No se usa el servidor de correo por defecto de Supabase, que no sirve para producción (`CLAUDE.md` §9).
- Supabase Auth se configura con SMTP propio apuntando a Resend, para que las invitaciones y los restablecimientos (`RF-AUT-03`, `RF-AUT-04`) salgan del remitente correcto.

**Recomendación de calendario: resolver esto en la semana del 19 de octubre**, antes de construir la consola. Es barato, desbloquea el flujo principal y tiene un tiempo de espera que no depende del equipo.

### 3. Entornos

**Dos entornos: desarrollo y producción.** No tres. Un entorno de pruebas intermedio sería una pieza sin problema que resolver con dos personas (`CLAUDE.md` §4, regla 9).

| Recurso | Desarrollo | Producción |
| --- | --- | --- |
| Proyecto de Supabase | Propio | Propio |
| Workers (API y aplicación) | Propios | Propios |
| Colas | Propias | Propias |
| Bucket de R2 | Propio (ADR-010) | Propio |
| Dominio de correo | Subdominio de pruebas | Dominio propio |
| Número de WhatsApp | Número de pruebas de Meta | Números del portafolio de Numi (ADR-001) |

**Nada se comparte entre entornos.** Un error en desarrollo no puede tocar datos de clientes ni enviar mensajes reales por WhatsApp.

**El plan de Supabase:**

- **Producción: plan Pro, 25 USD al mes.** Razón decisiva: el plan gratuito **pausa el proyecto tras una semana de inactividad** (ADR-008). Para un piloto que empieza el 1 de diciembre, con días previos de poca actividad, eso es un fallo en el peor momento posible. Además el plan Pro da copias de seguridad y los informes de tiempo de RLS que ADR-008 mencionó.
- **Desarrollo: plan gratuito.** Que se pause es irrelevante.

**Configuración que no está en el código** y hay que verificar a mano en cada entorno —este es el punto que ADR-008 dejó anotado:

- [ ] "Allow public access" de Realtime **desactivado** (si no, los canales son públicos y el aislamiento de ADR-008 no existe)
- [ ] `sessions.timebox` en 12 h e `inactivity_timeout` en 30 min (ADR-007)
- [ ] SMTP propio apuntando a Resend
- [ ] Rol de base de datos dedicado para Hyperdrive, con privilegios mínimos (ADR-003)
- [ ] **`max_size` de Hyperdrive fijado en 40**, por debajo de las 60 conexiones directas de Supabase Micro. Sin esto, Hyperdrive puede intentar abrir hasta ~100 conexiones contra una base que acepta 60, y las que sobran fallan bajo carga. Al cambiar el tamaño de cómputo hay que revisar este número en la misma operación (`arquitectura.md` §11.2)
- [ ] Los tres roles de aplicación, **ninguno con `bypassrls`** (ADR-006)

Esta lista va en el repositorio como documento de puesta en marcha. Es el tipo de cosa que se olvida y que no falla de forma visible: **un canal público no da error, simplemente deja de aislar.**

### 4. Despliegue

- **GitHub, con despliegue automático.** Lo que llega a la rama principal se despliega a producción; las ramas de trabajo despliegan a desarrollo.
- **Antes de desplegar se ejecutan las pruebas de aislamiento** (ADR-006). Si alguna falla, no se despliega. Es el único requisito de `CLAUDE.md` §8 que se puede verificar automáticamente en cada cambio, y desperdiciarlo sería absurdo.
- **Las migraciones de base de datos son archivos versionados**, aplicados antes del despliegue del código, y **solo aditivas** durante el MVP: agregar columnas y tablas, no renombrar ni borrar. Con dos personas y siete semanas, una migración destructiva mal aplicada en producción es una pérdida de datos sin vuelta atrás.
- **Reversión:** Cloudflare permite volver a un despliegue anterior del Worker. Eso cubre el código. **No cubre las migraciones**, y por eso solo se hacen aditivas: un código anterior sigue funcionando sobre un esquema con columnas de más, pero no sobre uno al que le falta algo.
- **Las copias de seguridad de la base las provee el plan Pro.** Hay que verificar al menos una vez que una restauración funciona; una copia que nunca se probó no es una copia.

### 5. Observabilidad

El objetivo es operar un piloto de una semana con dos personas, no construir un sistema de monitoreo.

**Qué se registra:**

- Toda petición: ruta, organización, usuario, resultado, duración.
- Todo evento del webhook: identificador, organización, resultado (ADR-005).
- Todo error, con su contexto.

**Qué no se registra, nunca** (`RF-TRV-07`, `RF-PLT-USO-04`):

- Texto de mensajes, nombres o teléfonos de contactos.
- Tokens, ni completos ni parciales.
- Contenido de archivos ni nombres originales que puedan identificar a una persona.

Esto no es una recomendación: es un requisito que no se negocia, y la forma de cumplirlo es que **el registro reciba identificadores, no objetos**. Registrar un objeto de evento completo "para depurar" es exactamente como se filtra contenido sensible a los registros.

**Las tres alertas mínimas**, en orden de urgencia:

| Alerta | Cuándo | Por qué urge |
| --- | --- | --- |
| **Webhook caído** | Sin eventos recibidos durante 2 horas en horario laboral | **Tiene un reloj de 7 días**: pasado ese plazo los identificadores de archivo de Meta caducan y los archivos se pierden para siempre (ADR-004, ADR-005). No es un aviso de cortesía |
| **Cola de mensajes fallidos con elementos** | Cualquier elemento | Un evento que agotó sus reintentos es un mensaje que no llegó a la bandeja (`RF-TRV-02`) |
| **Errores de la API por encima de lo normal** | Umbral a calibrar en la primera semana | Señal temprana de cualquier otra cosa |

Las dos primeras son las que protegen requisitos que no se negocian. La tercera es higiene.

**Cómo se observa, con el mínimo esfuerzo:**

- **Registros de Workers**, que vienen con la plataforma.
- **La salud del webhook ya está construida**: `RF-PLT-WA-05` muestra último evento, eventos en 24 horas y errores recientes por organización, derivados de la tabla de eventos (ADR-005). **La consola es la primera herramienta de operación**, y eso es trabajo que no hay que repetir.
- **Una verificación periódica externa** que consulte un endpoint de salud y avise si no responde. Es lo único que detecta una caída total, porque si todo está caído no hay nada que envíe la alerta desde adentro.

### 6. Modos de falla

Dado el antecedente de caídas de Supabase, conviene dejar escrito qué pasa en cada caso. Es además insumo directo para la sección 8 del documento de arquitectura.

| Si falla | Qué sigue funcionando | Qué se pierde | Recuperación |
| --- | --- | --- | --- |
| **Postgres** | Nada | El webhook responde error y **Meta reintenta 7 días** | Al volver, los eventos se recuperan solos. **Los archivos de más de 7 días de caída se pierden** |
| **Supabase Auth** | Sesiones abiertas, hasta que expire el token | Nadie nuevo puede entrar | Al volver, normal |
| **Supabase Realtime** | Todo; la bandeja actualiza cada 60 s (ADR-008) | Solo la inmediatez | Automática al reconectar |
| **Cloudflare Queues** | El webhook sigue guardando eventos crudos | El procesamiento se atrasa | Se reprocesa desde la tabla de eventos |
| **R2** | Todo menos ver archivos | Acceso a archivos; las descargas se reintentan dentro de 7 días | Automática |
| **Resend** | Todo | **El alta de organizaciones**: no hay invitación | El operador genera un enlace y lo envía por otro medio (`RF-PLT-USR-04`) |
| **Meta** | El CRM; los mensajes guardados se leen | Enviar y recibir | Automática |

Dos observaciones que salen de esta tabla:

1. **El único fallo que causa pérdida irreversible es una caída larga de Postgres**, por los archivos. Todo lo demás se recupera. Esto justifica la alerta de webhook caído por encima de las otras.
2. **Un fallo de Resend tiene salida manual** —el enlace copiado de `RF-PLT-USR-04`—, que ya estaba en los requisitos por otra razón y aquí resulta ser la mitigación.

## Costo mensual total

| Servicio | Costo |
| --- | --- |
| Cloudflare Workers (plan de pago) | 5 USD |
| Cloudflare Queues, R2, Hyperdrive | Incluidos |
| Supabase Pro (producción) | 25 USD |
| Supabase gratuito (desarrollo) | 0 USD |
| Resend (plan gratuito) | 0 USD |
| **Total** | **30 USD al mes** |

Más el costo de los mensajes de WhatsApp a Meta, que depende del volumen y de las conversaciones iniciadas por el negocio, y del dominio.

**30 USD al mes cumple la restricción de "pocos costos fijos"** de `CLAUDE.md` §8.

## Consecuencias

### Más fácil

- Los secretos tienen un lugar claro y el cifrado usa el runtime, sin bibliotecas que verificar.
- El correo deja de bloquear el alta, que es el flujo principal de la consola.
- Dos entornos sin recursos compartidos: un error en desarrollo no toca a los clientes.
- Las pruebas de aislamiento en cada despliegue convierten `RF-TRV-01` en algo verificado continuamente.
- La consola sirve como herramienta de operación, sin construir monitoreo aparte.

### Más difícil

- Hay una lista de configuración manual por entorno que nadie va a recordar sin el documento.
- Las migraciones solo aditivas obligan a convivir con columnas en desuso hasta después del MVP.
- La restricción a registros sin contenido exige disciplina en cada punto donde se registra algo.
- **Tres funciones críticas concentradas en Supabase**, con antecedentes de caídas. Se asume y se mide.

### Qué hay que construir

- Cifrado y descifrado de tokens con AES-GCM por Web Crypto, con referencia de clave.
- Configuración de Resend: dominio, SPF, DKIM, DMARC, y SMTP propio en Supabase Auth.
- Dos entornos completos con recursos separados.
- Despliegue automático desde GitHub con pruebas de aislamiento como requisito.
- Migraciones versionadas y aditivas.
- Registro estructurado sin contenido sensible.
- Las tres alertas y la verificación periódica externa.
- Documento de puesta en marcha con la lista de configuración manual.

### Qué documentos hay que actualizar

- `docs/01_definicion_mvp_v2.2.md` §8.1: agregar Resend, y registrar los costos fijos (Workers 5 USD, Supabase Pro 25 USD).
- `docs/01_definicion_mvp_v2.2.md` §8: la fila "Credenciales de integraciones" de la tabla de retención, precisando el cifrado con clave fuera de la base.
- `docs/01_definicion_mvp_v2.2.md` §13: agregar el riesgo de concentración en Supabase, con su mitigación.

## Qué revisar cuando el sistema crezca

- **Si el límite de 100 correos diarios aprieta:** ocurrirá al dar de alta varias organizaciones el mismo día. El plan Pro de Resend lo quita por 20 USD.
- **Al acercarse a 50 organizaciones activas:** revisar las conexiones de Postgres. ADR-003 ya señaló que es el primer cuello de botella esperable, y el camino es subir el tamaño de cómputo.
- **Si Supabase vuelve a dar problemas de disponibilidad:** el diseño ya permite separar las tres funciones. Postgres es portable —el acceso es por driver estándar (ADR-003), no por la capa REST—; Auth y Realtime serían los trabajos de migración, en ese orden de dificultad.
- **Cuando haya un cliente de pago:** revisar si hace falta un entorno de pruebas intermedio. Con clientes reales, probar en producción deja de ser aceptable.
- **Antes de la primera restauración real:** probar una restauración de copia de seguridad en desarrollo. Una copia no probada no es una copia.
- **Si el equipo crece:** el despliegue automático desde la rama principal necesitará revisión de código obligatoria, que hoy con dos personas sería ceremonia.
