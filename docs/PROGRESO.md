Progreso del proyecto de arquitectura

Bitácora de sesiones. Claude agrega una entrada al cerrar cada sesión, la más reciente arriba.

Formato de cada entrada:

markdown

## AAAA-MM-DD — Título corto de la sesión

- **Se decidió o produjo:** ...
- **Quedó abierto:** ...
- **Siguiente paso:** ...

---

## Estado al 2026-10-15

**Las 13 decisiones están aprobadas.** D-01 el 15 de octubre tras ADR-001; D-02 a D-13 el mismo día. El entregable en curso es `docs/architecture/arquitectura.md`, con 2 de 12 secciones escritas.

| ADR | Decisión | Qué se eligió |
| --- | --- | --- |
| [001](adr/001-modelo-conexion-whatsapp.md) | D-01 Conexión de WhatsApp | Tech Provider como destino; piloto con números del portafolio de Numi |
| [002](adr/002-alcance-consola.md) | D-02 Alcance de la consola | Completa: 44 requisitos P0, 11 vistas |
| [003](adr/003-runtime-backend.md) | D-04 Runtime del backend | Hono en Cloudflare Workers |
| [004](adr/004-multimedia-mvp.md) | D-03 Multimedia | Se descargan, se guardan y se muestran |
| [005](adr/005-procesamiento-webhook.md) | D-05 Webhook | Webhook mínimo más Cloudflare Queues |
| [006](adr/006-aislamiento-multiempresa.md) | D-06 Aislamiento | RLS más filtrado explícito; tres roles de base de datos |
| [007](adr/007-identidad-y-roles.md) | D-08 Identidad | Supabase Auth solo como identidad; autorización en el backend |
| [008](adr/008-tiempo-real.md) | D-07 Tiempo real | Broadcast desde el servidor, con red de seguridad de 60 s |
| [009](adr/009-arquitectura-frontend.md) | D-09 Frontend | Aplicación de página única con Vite, servida desde Workers |
| [010](adr/010-archivos-r2.md) | D-10 Archivos | R2, servido por el Worker; sin URL firmadas |
| [011](adr/011-secretos-entornos-despliegue.md) | D-11 Secretos y despliegue | AES-GCM, Resend, dos entornos, Supabase Pro |
| [012](adr/012-envio-de-mensajes.md) | D-12 Envío de mensajes | Síncrono, con persistencia previa y reintento manual |
| [013](adr/013-metricas.md) | D-13 Métricas | Consultas SQL directas con `security_invoker` |

**Costo fijo mensual:** 30 USD (Workers 5 + Supabase Pro 25), más los mensajes de Meta y el dominio.

### Lo que bloquea o urge

| Qué | Por qué urge | Dónde |
| --- | --- | --- |
| **Verificación de negocio ante Meta** | Ruta crítica del proyecto y no la controla el equipo. Sin ella el piloto queda limitado a 2 clientes | ADR-001 |
| **Configurar Resend (SPF, DKIM, DMARC)** | Sin correo no hay invitación y **no hay alta de organizaciones**. La propagación de DNS no depende del equipo | ADR-007, ADR-011 |
| **Confirmar los cinco supuestos de la arquitectura** | Las secciones 3 y 11 razonan sobre ellos. El más influyente son los 20 mensajes por conversación, que multiplica todos los volúmenes | `arquitectura.md` §1.8 |
| **Revisión independiente en sesión nueva** | La pide `CLAUDE.md` §11. La auditoría del 15 de octubre la hizo la misma sesión que escribió los ADR, así que no la sustituye | — |

### Puntos abiertos

- **ADR-009:** la elección de framework de frontend no está justificada en el ADR; depende de qué conoce Anthony, que tiene asignada la interfaz.
- **Riesgo de calendario:** 31 vistas más el alcance que agregaron ADR-004 y ADR-012, en siete semanas con dos personas. Cada ADR afectado tiene su orden de repliegue escrito.
- **Concentración en Supabase:** Postgres, Auth y Realtime en un proveedor con antecedentes de caídas. Asumido conscientemente; tabla de modos de falla en ADR-011.
- **Actualizaciones pendientes en `docs/01` a `docs/03`:** cada ADR lista las suyas en su sección de consecuencias. Ninguna se ha aplicado todavía.
- **Antes del piloto:** probar una restauración de copia de seguridad, y medir V30 con 6.000 conversaciones y 60.000 mensajes (ADR-013).

**Siguiente paso:** sección 3 de `docs/architecture/arquitectura.md` (contenedores, C4 nivel 2). La revisión independiente de `CLAUDE.md` §11 queda para cuando el documento esté completo.

---

## 2026-10-15 — D-14: agente de IA al MVP, con Anthropic como proveedor

- **Se decidió o produjo:** Franco incorpora al MVP un **agente de IA configurable por negocio**: responde en primera línea, escala a un asesor y cierra conversaciones. Las integraciones quedan para después. Escrito `docs/adr/014-agente-ia.md` en estado Propuesta.

  **Es un cambio de alcance, no un hueco pendiente.** `docs/01_definicion_mvp_v2.2.md` §6 excluía explícitamente los agentes de IA del MVP, diciendo que "la arquitectura se diseña para incorporarlos después". Se ofrecieron tres caminos —entrar recortando alcance, diseñar ahora y construir después, o entrar sin recortar— y **Franco eligió entrar sin recortar nada**, asumiendo el riesgo de calendario. La objeción quedó registrada en el ADR junto con un **orden de repliegue de cuatro escalones** para la revisión del 16 de noviembre, de modo que un recorte en noviembre sea una decisión preparada.

  **Decisión técnica: el agente es un consumidor propio detrás de su propia cola**, no dentro del consumidor de ADR-005. La razón es que `RF-TRV-02` —no perder mensajes entrantes— es una invariante absoluta y el proveedor de IA es la pieza más lenta e impredecible del sistema. Acoplarlos pondría el requisito más duro a merced de un tercero. Esta separación es además lo que permite apagar el agente dejando de consumir una cola, sin tocar el camino crítico.

  **Proveedor: Anthropic**, por decisión de Franco tras una primera versión escrita sobre OpenAI. Modelos y precios verificados en documentación oficial: Claude Haiku 5.5 (0,10/0,50 USD por millón de tokens), Claude Sonnet 5.5 (2/10), Claude Opus 5.5 (4/20). Recomendación del ADR: **empezar con Haiku** y medir la tasa de escaladas antes de subir de modelo.

  **Hallazgo que mejoró el plan.** Se verificó en la documentación oficial de Anthropic que, para datos de salud, el arreglo correcto no es la retención cero sino la **preparación para HIPAA**, y que esta **se habilita por autoservicio desde la consola con efecto inmediato** (descargar y aceptar el BAA). Esto elimina lo que con otro proveedor habría sido una dependencia externa de calendario con espera desconocida. Dos advertencias que el diseño ya cumple: solo se usa el endpoint de mensajes, que es elegible, y ningún dato del contacto va en esquemas JSON. Una que exige decisión: **la habilitación es permanente** y no se puede revertir desde la consola.

  Decisiones de producto que fija el ADR: configuración por negocio (información en texto libre, nombre y tono, reglas de escalada, preguntas y respuestas fijas que se consultan **antes** que el modelo); apagado por defecto; escalada por cuatro causas, incluida **el fallo del modelo dos veces** —lo que evita que una caída del proveedor deje conversaciones en silencio—; idempotencia por `(organization_id, conversation_id, inbound_message_id)`, porque a diferencia de una persona el agente reintenta solo; y un indicador nuevo `RF-MET-09`, porcentaje de conversaciones resueltas sin escalar, sin el cual el piloto no puede evaluar esta función.

- **Quedó abierto:**
  - **Aprobación de ADR-014** por Franco y Anthony.
  - **Decidir y aceptar el BAA de Anthropic.** Es permanente; lo firma un representante legal de la empresa.
  - **El riesgo de calendario**, que sube de forma material. El orden de repliegue del ADR es la mitigación.
  - Actualizaciones pendientes en los documentos fuente, listadas en el ADR: §6 y §8.1 de la definición del MVP, requisitos nuevos en el documento maestro, y §8 de CLAUDE.md.
  - Los cinco supuestos de `arquitectura.md` §1.8 siguen sin confirmar.

- **Siguiente paso:** sección 3 de `arquitectura.md` (contenedores, C4 nivel 2), que ahora debe incluir la cola y el consumidor del agente.

---

## 2026-10-15 — Las 13 ADR aprobadas; arranca el documento de arquitectura

- **Se decidió o produjo:** **Franco aprueba ADR-002 a ADR-013.** Las 13 decisiones del tablero quedan en estado Aprobada, y el tablero de `CLAUDE.md` §5 lo refleja. Con eso se cumple la condición de `CLAUDE.md` §7 y arranca el entregable principal.

  Creado `docs/architecture/arquitectura.md` con las **secciones 1 y 2 de las 12**, más un índice que marca el estado de cada una.

  **Sección 1, requisitos no funcionales.** Los números con los que se juzga el diseño, separados en tres categorías de peso distinto: exigido (está en un requisito), derivado (se calcula, y el cálculo se muestra) y supuesto (lo propone el documento porque ninguna fuente lo da). El resultado que gobierna el resto: **menos de 1 mensaje por segundo en el pico con 20 negocios**, contra un throughput de Meta de 80 mps — un margen de 84×. El sistema está sobredimensionado por decisión, no por accidente: las piezas se eligieron por corrección y costo de operación, nunca por capacidad.

  Dos hallazgos de esta sección que no estaban escritos en ningún ADR: **el primer límite de crecimiento de Numi CRM no es técnico, es el tope de 20 números por portafolio de Meta** (ADR-001) mientras no haya rol de Tech Provider; y **el riesgo de latencia real no es la cadena webhook→bandeja —estimada en 1,6 s contra un objetivo de 5— sino la consulta de respaldo de 60 s de ADR-008** si el Broadcast falla, que es doce veces el objetivo.

  **Sección 2, contexto C4 nivel 1.** Diagrama Mermaid con los tres actores, el contacto —que origina el tráfico pero no es usuario del sistema, porque su interfaz es WhatsApp— y los ocho sistemas externos, cada uno con qué pasa si falla y qué ADR lo decide. Incluye lo que queda fuera del sistema, para que el diseño no crezca por inercia.

- **Quedó abierto:**
  - **Cinco supuestos esperan confirmación de Franco y Anthony**, listados en §1.8: mensajes por conversación (20, el más influyente porque multiplica todo lo demás), usuarios por negocio (3 a 10), negocios a 12 meses (20), umbral de la vista de métricas (2 s) y techo de costo mensual (unos 100 USD). Las secciones 3 y 11 razonan sobre ellos.
  - Secciones 3 a 12 del documento de arquitectura.
  - La revisión independiente de `CLAUDE.md` §11, que debe hacerse en sesión nueva.
  - Reconfirmar los datos de proveedores antes de construir.

- **Siguiente paso:** sección 3, contenedores (C4 nivel 2): la aplicación del CRM, la consola, la API, el consumidor de webhooks, la cola, la base, el tiempo real, el almacenamiento de archivos y el correo, con cómo se comunican.

---

## 2026-10-15 — Auditoría de coherencia entre ADR; hueco de notificación cerrado

- **Se decidió o produjo:** auditoría de los 13 ADR como conjunto, no uno por uno. El grafo de dependencias es sano: sin ciclos ni referencias a decisiones posteriores. Las invariantes transversales se sostienen —"la API es el único camino hacia los datos" se mantiene intacta en ADR-007, ADR-008, ADR-010 y ADR-013, y `RF-TRV-07` queda garantizado por permisos de base de datos (ADR-006) y no por disciplina—.

  **Hueco corregido:** ADR-008 describía el Broadcast como si lo publicara únicamente el consumidor de la cola, pero el envío de ADR-012 es síncrono y escribe desde la API. Un mensaje enviado por un asesor, y la autoasignación que lo acompaña (`RF-CNV-11`), no llegaban a las pantallas de los demás usuarios hasta la consulta de respaldo de 60 segundos: cuatro veces el objetivo de `RF-TRV-03` y un incumplimiento de `RF-CNV-15`, que no distingue el origen del mensaje.

  Se corrigieron los dos lados: ADR-008 gana la sección **"Quién publica el aviso"**, con los dos escritores del sistema, y la regla de que publicar vive en una única función que ambos llaman. ADR-012 gana los avisos en los pasos 6 y 8 del camino de salida, con la aclaración de que son para los demás usuarios —quien envía ya tiene su respuesta HTTP— y que un fallo al publicar no falla el envío. Las dependencias cruzadas quedaron declaradas: ADR-012 depende de ADR-008.

  **Tablero corregido** (`CLAUDE.md` §5): la columna "Depende de" estaba desactualizada respecto a los encabezados de los ADR en cinco filas —D-10 decía `D-03` cuando el ADR declara cuatro dependencias, D-11 decía `D-04, D-09` cuando declara siete—. Ahora refleja lo que declara cada ADR, con una nota de que se actualiza con ellos. En §7 se corrigió el criterio de terminado, que seguía diciendo "D-01 a D-11" y "las 11 decisiones" desde antes de agregar D-12 y D-13, y una referencia cruzada equivocada a la sección 9 en lugar de la 11.

- **Quedó abierto:**
  - **Aprobación de ADR-002 a ADR-013.** Siguen en Propuesta. Es lo que bloquea el documento de arquitectura: `CLAUDE.md` §7 lo condiciona a decisiones aprobadas, y escribirlo sobre doce propuestas significa reescribirlo si alguna cambia.
  - **Reconfirmar datos de proveedores antes de construir.** Los ADR citan la fecha de compatibilidad de Workers (2026-08-04), el plan gratuito de Resend (3.000 al mes, 100 al día) y Supabase Pro a 25 USD. Se verificaron al escribirlos, pero ADR-001 registra que Embedded Signup v2 quedó obsoleta el 15 de octubre de 2026 —hoy mismo—, lo que muestra que estos datos se mueven.
  - La revisión independiente de `CLAUDE.md` §11 sigue pendiente: esta auditoría la hizo una sesión con el contexto de los ADR a la vista, así que no la sustituye.

- **Siguiente paso:** aprobación de los 12 ADR en Propuesta. Si Franco y Anthony prefieren avanzar antes, se pueden escribir las secciones 1 y 2 de `arquitectura.md` —requisitos no funcionales y contexto C4 nivel 1—, que no dependen de ninguna decisión pendiente.

---

## 2026-10-15 — Auditoría del tablero: dos huecos cubiertos (D-12 y D-13)

- **Se decidió o produjo:** auditoría de las once decisiones a pedido de Franco. Resultado: **115 requisitos en los documentos, 60 citados en los ADR**. La mayoría de los 55 restantes son requisitos de producto sin decisión arquitectónica pendiente, pero **dos grupos sí planteaban decisiones reales y no estaban cubiertos**. Se escribieron dos ADR nuevos y se agregaron D-12 y D-13 al tablero de CLAUDE.md §5.

  **`docs/adr/012-envio-de-mensajes.md`** — ADR-005 resolvió en detalle cómo **entran** los mensajes; cómo **salen** quedó implícito, pese a tocar dos no-negociables (`RF-CNV-08` ventana de 24 h, `RF-CNV-09` estados y reintento). Decisión: **síncrono, con persistencia previa y reintento manual**. El mensaje se guarda con estado `enviando` **antes** de llamar a Meta, así no se pierde lo que el asesor escribió; la llamada ocurre dentro de la petición para que el asesor vea el resultado real; el reintento es del asesor, no automático, **porque en el camino de salida no hay identificador de Meta antes de enviar y un reintento automático duplicaría el mensaje al contacto**.

  Va en dirección contraria a ADR-005 a propósito: en el webhook espera Meta, que castiga la demora; aquí espera una persona mirando la pantalla.

  Datos verificados: error **131047** cuando la ventana venció ("More than 24 hours have passed..."), throughput **80 mensajes/s** por defecto (contra menos de 1 que necesita el piloto), 20 mps fijos en coexistencia. Hallazgo de diseño: **la ventana de 24 h no se puede verificar con certeza localmente** —el reloj lo lleva Meta—, así que la verificación local es comodidad y la respuesta de Meta es la verdad. **Las plantillas quedan fuera del MVP**, de modo que una conversación con la ventana vencida no se puede retomar: es una limitación real que hay que explicar a los negocios del piloto.

  **`docs/adr/013-metricas.md`** — `RF-MET-01` a `RF-MET-08` no aparecían en ningún ADR. Decisión: **consultas SQL directas al abrir la vista**, con vistas `security_invoker` (ADR-006) e índices pensados para ellas. Se descartó precalcular porque **la mediana de `RF-MET-02` no es combinable por día** (la mediana de las medianas no es la mediana) y `RF-MET-04` depende del estado **actual**, así que dos de los cinco indicadores consultarían las tablas igual. Además, una métrica desincronizada es peor que una lenta: nadie la nota.

  El ADR fija ocho definiciones que los requisitos dejaban ambiguas (qué cuenta como espera de respuesta, qué asesor se atribuye un contacto, que los límites del período van en hora de Bogotá y no en UTC) y pide **medir con 6.000 conversaciones y 60.000 mensajes antes del piloto**.

- **Quedó abierto:**
  - Aprobación de ADR-002 a ADR-013 (solo ADR-001 está Aprobada).
  - Otros hallazgos de la auditoría: la actualización pendiente de `RF-PLT-USR-02` en `docs/03` (P23 como vista de soporte, ADR-002) y el punto abierto de ADR-009 sobre la elección de framework.
  - **Advertencia sobre esta auditoría:** la hizo la misma sesión que escribió los trece ADR. `CLAUDE.md` §11 pide revisión en sesión nueva, que sigue pendiente y no queda cubierta por esto.

- **Siguiente paso:** revisión independiente en sesión nueva sobre los trece ADR, y después el documento de arquitectura.

## 2026-10-15 — D-11, secretos, correo, entornos, despliegue y observabilidad

- **Se decidió o produjo:** `docs/adr/011-secretos-entornos-despliegue.md` en estado Propuesta. **Última decisión del tablero: las 11 están resueltas** (D-01 Aprobada, D-02 a D-11 en Propuesta).

  Resoluciones: tokens de WhatsApp cifrados con **AES-GCM por Web Crypto** —parte del runtime, sin biblioteca de npm que verificar—, con la clave como secreto del Worker, **fuera de la base**. Correo con **Resend en plan gratuito** (3.000/mes, dominio propio incluido), con SMTP propio en Supabase Auth. **Dos entornos** sin nada compartido. **Supabase Pro en producción** (25 USD), porque el plan gratuito **pausa el proyecto tras una semana de inactividad** y el piloto empieza el 1 de diciembre. Despliegue desde GitHub con **las pruebas de aislamiento como requisito**; migraciones versionadas y **solo aditivas** (la reversión de Cloudflare cubre el código, no el esquema).

  **Costo fijo total: 30 USD al mes** (Workers 5 + Supabase Pro 25), más los mensajes de Meta y el dominio.

  **Franco aportó un dato de experiencia**: en un proyecto anterior tuvo que migrar de la capa REST de Supabase (PostgREST) a SQL directo por lentitud y caídas. Coincide con lo ya decidido —ADR-003 usa node-postgres por Hyperdrive, ADR-007 usa Supabase solo como identidad—, así que el camino de datos no toca PostgREST. Pero se registró la consecuencia: **Supabase son tres piezas (Postgres, Auth, Realtime) que no fallan juntas**, y concentrarlas en un proveedor con antecedentes de caídas es el mayor riesgo de infraestructura del sistema. Se asume conscientemente y se documentó la tabla de modos de falla.

  Tres alertas mínimas definidas. La primera —**webhook caído a las 2 horas**— es la más urgente porque **tiene un reloj de 7 días**: pasado ese plazo los identificadores de archivo de Meta caducan y los archivos se pierden para siempre. **Es el único fallo del sistema con pérdida irreversible.**

  Lista de **configuración manual por entorno** documentada, incluyendo el "Allow public access" de Realtime que ADR-008 dejó anotado: no está en el código y **un canal público no da error, simplemente deja de aislar**.

- **Quedó abierto:**
  - Aprobación de ADR-002 a ADR-011 por Franco y Anthony (solo ADR-001 está Aprobada).
  - **Acción recomendada para la semana del 19 de octubre: configurar Resend con SPF, DKIM y DMARC.** Es barato, desbloquea el alta de organizaciones y su tiempo de espera (propagación de DNS) no depende del equipo.
  - Reconfirmar las cifras de planes de Supabase antes de contratar: las fuentes difieren en el nombre del tamaño base y en algunos números.
  - Probar una restauración de copia de seguridad antes del piloto. Una copia no probada no es una copia.
  - **Punto abierto de ADR-009:** la elección de React no está justificada en ese ADR; depende de qué conoce Anthony, que tiene asignada la interfaz.

- **Siguiente paso:** con el tablero completo, construir `docs/architecture/arquitectura.md` según la sección 7 de CLAUDE.md (12 secciones, diagramas en Mermaid). Antes conviene la **aprobación de los ADR pendientes** y, según la sección 11, una **revisión independiente en sesión nueva**.

## 2026-10-15 — D-10, archivos y almacenamiento en R2

- **Se decidió o produjo:** `docs/adr/010-archivos-r2.md` en estado Propuesta. Decisión: **el Worker sirve los archivos**, verificando permisos en cada petición. **No se usan URL firmadas.** R2 se usa para **una sola cosa**: los archivos recibidos por WhatsApp.

  El punto real de esta decisión era por dónde viaja el archivo hasta el navegador. Normalmente eso se decide por costo —hacer pasar archivos por un servidor cuesta ancho de banda— pero **en R2 la salida de datos es gratuita** ("does not incur data transfer (egress) charges and is free"). Sin ese argumento, quedaba solo la seguridad, y ahí una URL firmada es un **token al portador**: "should be treated as bearer tokens", funciona para cualquiera que la tenga y **no se puede revocar** hasta que expire. Con fotos clínicas en juego (ADR-004), eso es una fuga que ningún control posterior revierte.

  Es la **tercera vez** que aparece el mismo problema —token que no se puede revocar— después de los tokens de sesión (ADR-007) y el canal de tiempo real (ADR-008). La respuesta es la misma: verificar en cada petición.

  Reglas fijadas: clave `org/{organization_id}/msg/{message_id}/{media_id}`, **nunca expuesta al navegador**; acceso solo por binding, sin credenciales S3 ni dominio público; archivo devuelto **en flujo**; un usuario de A pidiendo un archivo de B recibe **el mismo resultado que si no existiera**, no un error distinto que revelaría su existencia; borrado **archivo antes que fila** (el orden inverso dejaría huérfanos invisibles) más tarea periódica de limpieza; documentos forzados a descarga, no abiertos en el navegador.

  **Costo: 0 USD** al volumen del piloto. Unos 300 MB al mes contra 10 GB gratuitos; el primer costo real aparece alrededor de 25–30 organizaciones con un año de historial.

- **Quedó abierto:**
  - Aprobación de ADR-010 por Franco y Anthony.
  - Vigilar documentos de hasta 100 MB pasando por el Worker. Si dan problema, el ADR fija que las URL firmadas son el **último** recurso, no el primero.

- **Siguiente paso:** D-11, la última: secretos y tokens cifrados, correo propio, entornos, despliegue y observabilidad. Arrastra varios pendientes acumulados, entre ellos el correo que bloquea el alta de organizaciones (ADR-007) y el plan de Supabase (ADR-008).

## 2026-10-15 — D-09, arquitectura del frontend

- **Se decidió o produjo:** `docs/adr/009-arquitectura-frontend.md` en estado Propuesta. Decisión: **aplicación de página única con React y Vite, servida desde Cloudflare Workers**. CRM y consola en **una sola aplicación**, con las rutas de `/plataforma` cargadas de forma diferida.

  Se abandona la propuesta de §8.1 (Next.js con OpenNext). Dos razones, y la primera pesa más que la segunda:

  **1. Next.js resolvería un problema que este sistema ya no tiene.** ADR-007 dejó al frontend como consumidor puro de una API: el navegador nunca lee la base. El renderizado en servidor —lo único que justifica la complejidad de Next.js— no se puede usar, porque el servidor del frontend no tiene acceso a datos. Se pagaría la complejidad sin obtener el beneficio.

  **2. Las dos vías de Next.js en Workers están inmaduras.** Verificado hoy: Cloudflare **cambió su recomendación** y ahora dice "Cloudflare recommends vinext as the default way to run Next.js applications on Cloudflare Workers", pero **"vinext is in beta"**. OpenNext quedó como vía para *mantener* aplicaciones existentes ("cannot yet migrate to vinext because of a compatibility gap"), con huecos conocidos y una cola de revalidación "not fully suitable for production deployments". **Es el mismo patrón que Fastify en D-04**: el dato del brief caducó.

  La ruta de aplicación de página única en Workers está documentada y madura, con complemento oficial de Vite y `assets.not_found_handling` en modo SPA.

  Otras reglas fijadas: dos Workers separados (aplicación y API); el cliente de Supabase en el navegador se usa **solo para autenticar, nunca para consultar datos**; biblioteca de datos de servidor (TanStack Query o similar) y **sin gestor de estado global**, que no tendría trabajo; el aviso de tiempo real **invalida consultas**, nunca escribe en la caché; búsqueda de contactos en el servidor (`RF-TRV-03`); carga diferida de `/plataforma` por rendimiento, **no por seguridad** —la protección real es `RF-PLT-SEG-01` en el backend.

- **Quedó abierto:**
  - Aprobación de ADR-009 por Franco y Anthony.
  - **31 vistas (18 del CRM + 13 de la consola) en siete semanas con dos personas sigue siendo el riesgo central del proyecto.** Esta decisión lo reduce al quitar una capa de adaptación, pero no lo elimina.
  - La elección de base de componentes corresponde a la reunión de arranque del design system; el criterio fijado es usar una base ya resuelta, no construir desde cero.
  - Actualizar §8.1 y §13 de la definición del MVP, y §9 de CLAUDE.md.

- **Siguiente paso:** D-10, archivos (si se usa R2 y para qué). Depende de D-03, ya resuelta; ADR-004 dejó buena parte del terreno fijado.

## 2026-10-15 — D-08, identidad, roles y sesiones

- **Se decidió o produjo:** `docs/adr/007-identidad-y-roles.md` en estado Propuesta. Decisión: **Supabase Auth solo como identidad; toda la autorización en el backend**. En una frase: el proveedor responde "quién eres"; la API responde "qué puedes hacer", y lo consulta en la base **en cada petición**.

  **TOTP confirmado** (pendiente que dejó ADR-002): "TOTP MFA API is free to use and is enabled on all Supabase projects by default". `RF-PLT-ACC-02` y `RF-PLT-ACC-03` se cumplen llamando al proveedor, sin construir nada.

  Dos hechos verificados obligaron al diseño: (1) **cerrar sesión no invalida el token de acceso** ("the access token JWT will still be valid until it expires"), y (2) los límites de sesión se aplican **de forma perezosa**, así que la duración real es el tiempo configurado más la vigencia del token. Si la autorización se apoyara en el token, un usuario desactivado seguiría entrando hasta una hora y `RF-AUT-07` (suspensión cierra sesiones "en la siguiente acción") sería imposible. Por eso **el estado de usuario y organización se verifica contra la base en cada petición**, no contra el token.

  Consecuencia: ni el rol de organización ni el rol de plataforma viajan en el token. `RF-PLT-SEG-01` exige verificación del rol en cada petición, y un claim en un token de una hora no permite revocar a un operador. Los operadores viven en **tabla propia**, separada de los usuarios de negocio, poblada por migración. **`aal2` es obligatorio en toda ruta de plataforma de la API**, no solo en la pantalla.

  Se descartó la Opción A (RLS directo desde el navegador con el cliente de Supabase): contradice `RF-PLT-SEG-01`, dejaría la suspensión en manos de un token irrevocable y no da dónde esconder los tokens de WhatsApp.

  Configuración fijada: `jwt_expiry` 3600 s, `sessions.timebox` 12 h, `sessions.inactivity_timeout` 30 min (`RF-PLT-SEG-03`), rotación de tokens de refresco activa.

- **Quedó abierto:**
  - Aprobación de ADR-007 por Franco y Anthony.
  - **Dependencia de ruta crítica: el correo con remitente propio bloquea el alta de organizaciones.** Sin él no hay invitación (`RF-PLT-ORG-06`) y el primer administrador no entra. Formalmente es D-11, pero conviene resolverlo antes: es barato y desbloquea el flujo principal de la consola.
  - **El tiempo real de Supabase va directo del navegador a la base y no pasa por esta verificación.** D-07 debe resolverlo explícitamente.
  - Actualizar §8.1 y §8.2 punto 2 de la definición del MVP, y la decisión 8 de §2 de la consola.

- **Siguiente paso:** D-07, tiempo real en la bandeja. Depende de D-04 y D-06, ambas resueltas, y arrastra el punto abierto de ADR-007 sobre el canal de tiempo real.

## 2026-10-15 — D-07, tiempo real en la bandeja

- **Se decidió o produjo:** `docs/adr/008-tiempo-real.md` en estado Propuesta. Decisión: **Broadcast desde el servidor**, con consulta periódica de 60 segundos como red de seguridad. El consumidor de la cola notifica por HTTP a un canal privado por organización; el navegador recibe el aviso y **pide los datos a la API**.

  Esto resuelve el punto que ADR-006 y ADR-007 dejaron anotado dos veces. Con Broadcast desde el servidor **el navegador solo escucha**: no consulta la base, no lee filas, no necesita permisos sobre ninguna tabla. El canal deja de ser una vía de acceso a datos y pasa a ser un aviso, así que la regla de ADR-007 —el token identifica, la API autoriza en cada petición— se mantiene sin excepciones.

  Se descartó Postgres Changes (la opción que §8.2 punto 2 de la definición llamaba "la más simple"), por cuatro datos verificados: **Supabase mismo recomienda Broadcast** ("the recommended method for scalability and security", "Postgres Changes does not scale as well"); la autorización es **por suscriptor** (100 suscriptores = 100 lecturas por `INSERT`) y se procesa en **un solo hilo**; los permisos del canal **se cachean durante toda la conexión**, así que una desactivación no surte efecto hasta que expire el token —reintroduciendo por otra puerta el problema que ADR-007 resolvió—; y un fallo de política produce **silencio**, no error ("the single biggest source of silently missing events").

  Dato que lo hace viable desde Workers: existe **endpoint HTTP de Broadcast**, así que un Worker puede notificar sin sostener un WebSocket (ADR-003 lo impide).

  Reglas fijadas: el aviso **no lleva contenido** (solo tipo de evento, identificador de conversación y hora), canales privados obligatorios con "Allow public access" desactivado, solo el servidor publica, la notificación ocurre **fuera** de la transacción de escritura, y el contador de sin responder se **recalcula**, no se incrementa a ciegas. La consola no usa tiempo real.

- **Quedó abierto:**
  - Aprobación de ADR-008 por Franco y Anthony.
  - **Argumento nuevo a favor del plan Pro de Supabase:** los proyectos gratuitos **se pausan tras una semana de inactividad**, lo que es un riesgo real en los días previos al piloto del 1 de diciembre. La decisión del plan es D-11.
  - El aislamiento del canal es un sistema de políticas **aparte** del RLS de las tablas (`realtime.messages`) y necesita su propia prueba automática, que extiende `RF-TRV-01` al tiempo real.
  - "Allow public access" es un interruptor de configuración del que depende el aislamiento y **no está en el código**: hay que recordarlo en el despliegue (D-11).
  - Actualizar §8.2 punto 2 y §8.1 de la definición del MVP, y `RF-CNV-15` del documento maestro.

- **Siguiente paso:** D-09, arquitectura del frontend (framework y hosting, CRM y consola juntas o separadas, sesión, obtención de datos, estado, integración con el tiempo real). Depende de D-04, D-07 y D-08, las tres ya resueltas.

## 2026-10-15 — D-06, aislamiento multiempresa

- **Se decidió o produjo:** `docs/adr/006-aislamiento-multiempresa.md` en estado Propuesta. Decisión: **las dos capas**, RLS en la base **más** filtrado explícito por `organization_id` en el código. El filtrado es el camino normal; RLS es la red que atrapa la consulta olvidada. Esto resuelve el punto abierto de §8.2 punto 3 de la definición del MVP: la respuesta es ambos patrones, no uno.

  Razón: `RF-TRV-01` es la condición que define si el piloto fue exitoso (§11). Un requisito de ese peso no puede depender de que dos personas con prisa no olviden nunca un `WHERE`. Una consulta olvidada no falla: devuelve datos de otro negocio en silencio.

  **Tres roles de base de datos**, ninguno con `bypassrls`: `app_user` (usuarios de un negocio, filas de su organización), `app_platform` (consola, todas las organizaciones pero **sin permiso sobre las tablas de mensajes**) y `app_worker` (consumidor de la cola, escribe aunque la organización esté suspendida). El punto clave: `app_platform` no obtiene su poder saltando RLS sino por no tener permiso, así que **`RF-TRV-07` deja de depender de disciplina y pasa a ser imposible a nivel de permisos**.

  Datos verificados que lo hacen viable: Hyperdrive opera en **modo transacción** y hace `RESET` al devolver la conexión ("the connection is `RESET` such that the `SET` commands will not take effect on subsequent queries"), así que **el contexto no se filtra entre peticiones** —el riesgo clásico del patrón no existe aquí—, pero obliga a fijar el contexto **en cada transacción** con alcance local, encapsulado en una única función. En Supabase, `service_role` y el dueño `postgres` tienen `bypassrls`; de ahí que los roles de aplicación deban ser otros.

  Siete pruebas automáticas definidas. Las más valiosas por esfuerzo son las tres de catálogo: ningún rol de aplicación tiene `bypassrls`, toda tabla con `organization_id` tiene RLS activo, y las vistas usan `security_invoker` (las vistas **saltan RLS por defecto**, y §9 de CLAUDE.md propone métricas con vistas SQL).

- **Quedó abierto:**
  - Aprobación de ADR-006 por Franco y Anthony.
  - **Las políticas y los roles deben existir desde la primera migración**, no en la última semana. Si se dejan para el 26–30 de noviembre habrá consultas escritas suponiendo que no existen.
  - El tiempo real de Supabase tiene políticas propias, aparte del RLS de las tablas; hay que verificar ese aislamiento por separado en D-07.
  - Actualizar §8.2 punto 3 y §8.3 de la definición del MVP.

- **Siguiente paso:** D-08, identidad y roles (Supabase Auth, rol de plataforma, doble factor, sesiones). Depende de D-06, ya resuelta. Nota: el orden de CLAUDE.md §5 pone D-08 antes que D-07.

## 2026-10-15 — D-05, procesamiento del webhook

- **Se decidió o produjo:** `docs/adr/005-procesamiento-webhook.md` en estado Propuesta. Decisión: **webhook mínimo más Cloudflare Queues**, con la corrección resuelta en la base de datos y no en la cola. Se confirma el **plan de pago de Workers** (5 USD al mes), que ADR-003 había dejado abierto.

  Hallazgo que define la decisión, verificado hoy: **ni Meta ni Cloudflare Queues garantizan orden, y los dos pueden duplicar**. Meta: "These retries can result in duplicate webhook notifications", reintentos hasta 7 días, sin garantía de orden documentada. Queues: "at least once delivery", "does not guarantee that messages will be delivered in the same order in which they are published". Por lo tanto `RF-TRV-02` y `RF-CNV-06` **no se pueden resolver en el transporte**: van a la base con unicidad `(organization_id, wamid)` y ordenamiento por la marca de tiempo de Meta con desempate estable (su resolución es de segundos).

  El webhook hace solo cinco cosas: verifica firma, resuelve organización, **inserta el evento crudo**, encola una referencia y responde 200. No interpreta, no descarga, no envía. Razón: lo único que Meta castiga es no recibir un 200; con el evento guardado ya no se puede perder nada.

  Segundo hallazgo: los reintentos de Meta duran 7 días **igual que la vigencia del identificador de archivo** (ADR-004). **Una caída larga pierde archivos aunque no pierda mensajes.** Esto condiciona las alertas de D-11: una alerta de webhook caído tiene un reloj corriendo.

  Detalle que hay que probar explícitamente: Meta calcula la firma `X-Hub-Signature-256` sobre el cuerpo con **unicode escapado y hexadecimal en minúsculas**; hashear los bytes crudos da otro resultado. **Probar con tildes o emoji antes de darlo por bueno** — es el error típico y silencioso.

  Otras reglas fijadas: dos colas separadas (eventos y descargas), confirmación individual y no por lote, consumidor idempotente, cola de mensajes fallidos obligatoria, tabla de eventos crudos con retención de 30 días, y estados de mensaje como **progresión** que tolera desorden (Meta documenta que un mensaje puede producir `delivered` y `failed` a la vez).

- **Quedó abierto:**
  - Aprobación de ADR-005 por Franco y Anthony.
  - Plan de pago de Workers como costo fijo nuevo; actualizar §8.1 de la definición del MVP.
  - Precisar `RF-CNV-09` en el documento maestro: los estados siguen una progresión y un estado anterior no pisa a uno posterior.
  - Agregar la tabla de eventos recibidos a la tabla de retención de §8.

- **Siguiente paso:** D-06, aislamiento multiempresa (cómo se aplica la seguridad por fila desde la API). Depende de D-04, ya resuelta.

## 2026-10-15 — D-03, audios, imágenes y documentos

- **Se decidió o produjo:** `docs/adr/004-multimedia-mvp.md` en estado Propuesta. Decisión: **se descargan, se guardan y se muestran** imágenes y audios; documentos como enlace de descarga. El envío de archivos desde el CRM queda fuera.

  Hallazgo que cambió el marco, verificado hoy contra Meta: el webhook trae **solo el identificador**, no el archivo; **el identificador caduca a los 7 días** y la URL de descarga a los **5 minutos**. Lo que no se descarga en esa ventana **se pierde para siempre**. Eso parte la decisión en dos: **guardar es irreversible, mostrar no lo es**. Los documentos fuente las trataban como una sola porque se escribieron sin ese dato.

  Segundo argumento: ADR-001 eliminó la alternativa. El piloto corre con líneas **nuevas** del portafolio de Numi, así que el negocio no tiene ninguna app donde ver esos archivos. El argumento de §13 de la definición del MVP ("la clínica no tiene otro lugar donde verlos") es literalmente cierto en modo transitorio.

  Límites de Meta verificados: audio 16 MB, imagen 5 MB, documento 100 MB, video 16 MB. Videos y stickers se descargan y guardan igual aunque el MVP no los muestre, por el mismo criterio.

- **Quedó abierto:**
  - Aprobación de ADR-004 por Franco y Anthony.
  - **Numi pasa a custodiar datos sensibles de salud** (fotos clínicas, no solo texto). El acuerdo de encargo y la política de tratamiento que §8 de la definición ya exigía deben revisarse con el abogado sabiendo esto.
  - Alcance nuevo en un calendario ya ajustado por ADR-002. Orden de repliegue definido en el ADR; **la descarga y el guardado no se recortan en ningún escenario**.
  - Requisitos nuevos propuestos (`RF-CNV-10a` a `RF-CNV-10e`) y reescritura de `RF-CNV-10`, a confirmar al actualizar el documento maestro.

- **Siguiente paso:** D-05, procesamiento del webhook (cola, idempotencia, orden y reintentos). Depende de D-01 y D-04, ambas ya resueltas.

## 2026-10-15 — D-04, runtime del backend

- **Se decidió o produjo:** `docs/adr/003-runtime-backend.md` en estado Propuesta. Runtime elegido: **Hono en Cloudflare Workers**.

  Dato que cierra la opción del brief, verificado hoy contra Cloudflare: **Fastify sigue sin soporte en Workers** ("we're hoping to be able to add Fastify support later"), mientras que **Express sí funciona** y tiene tutorial oficial. Además, `nodejs_compat` está activo por defecto desde la compatibility date 2026-08-04. La propuesta de §8.1 de la definición del MVP (Fastify en Workers) queda inviable tal como está escrita.

  Razonamiento: a menos de 1 mensaje por segundo el rendimiento no es criterio. Lo que decide es cuántos proveedores hay que operar con dos personas a tiempo parcial. Salir de Workers (Fastify en un servicio Node) agrega un proveedor más que montar, monitorear y desplegar, y deja el webhook de Meta —único punto con requisito duro de disponibilidad, `RF-TRV-02`— sobre un contenedor que el equipo tiene que mantener vivo. Entre Hono y Express en Workers, Hono gana por ser nativo: mismo costo y mismas primitivas, sin la capa de compatibilidad.

  Otros datos verificados: Hyperdrive con Supabase usa la cadena **Direct**, no las agrupadas, con node-postgres ≥ 8.16.3 y un rol de base de datos dedicado de privilegios mínimos. Cloudflare Queues: gratuito 10.000 operaciones/día con retención de 24 h **no configurable**; de pago 1M/mes incluidas más 0,40 USD por millón, con retención de 4 a 14 días.

  Regla de diseño que fija el ADR: **el framework vive solo en la capa de entrada**; el código de negocio no importa nada de Hono. Sin estado en memoria entre peticiones. Toda biblioteca de npm se verifica en el runtime antes de adoptarse.

- **Quedó abierto:**
  - Aprobación de ADR-003 por Franco y Anthony.
  - **Plan de Workers:** el gratuito alcanza por volumen, pero la retención de 24 h no configurable de Queues es un riesgo para `RF-TRV-02` (un incidente de más de un día perdería mensajes encolados). Se confirma en D-05.
  - Curva de aprendizaje de Hono y del modelo de ejecución de Workers, con el calendario ajustado.
  - Actualizaciones pendientes: §8.1, §8.2 punto 1 y §13 de la definición del MVP; §9 de CLAUDE.md.

- **Siguiente paso:** D-03, visualización de audios, imágenes y documentos en el MVP (siguiente en el orden de CLAUDE.md §5; depende de D-01, ya aprobada).

## 2026-10-15 — D-02, alcance de la consola de plataforma

- **Se decidió o produjo:** `docs/adr/002-alcance-consola.md` en estado Propuesta. Alcance elegido: **consola completa**, los 44 requisitos P0 y las 11 vistas P0 de `docs/03_consola_plataforma_v1.1.md`. Fuera del MVP solo lo ya marcado P1: P10, P30, `RF-PLT-ORG-12`, `RF-PLT-USO-02`, `RF-PLT-USO-03`, `RF-PLT-AUD-04`, `RF-PLT-WA-08`.

  Tres precisiones confirmadas por Franco y Anthony: (1) la consola es herramienta de administración de Numi, no autoservicio; crea e invita solo al **primer administrador**, y los asesores los invita el administrador del negocio desde el CRM (`RF-USR-*`), lo que vuelve esos requisitos P0 irrecortables; P23 queda como vista de soporte. (2) El **doble factor del operador se resuelve con el TOTP del proveedor de identidad** (Supabase Auth), no con un flujo propio; a confirmar en D-08. (3) La **auditoría se escribe desde la primera acción** aunque P25 llegue después, porque un registro con huecos no se reconstruye.

  Se descartó la opción de consola mínima más scripts: rompe `RF-PLT-AUD-01` y `RF-PLT-SEG-02`, y contradice el propósito declarado de la consola (§1 del documento).

- **Quedó abierto:**
  - Aprobación de ADR-002 por Franco y Anthony.
  - **Riesgo de calendario:** 44 requisitos de consola más 63 del CRM en siete semanas con dos personas a tiempo parcial. Corte de emergencia definido para la revisión del 16 de noviembre, en orden: P25, luego `RF-PLT-USO-01`, luego buscador y filtros de P20.
  - Correo propio con entrega confiable (`RF-PLT-SEG-05`) sin resolver; el servidor de Supabase no sirve para producción. Se decide en D-11.
  - Actualizaciones pendientes: §5.4 y §2 (decisión 8) de la consola; §12 de la definición del MVP.

- **Siguiente paso:** D-04, runtime del backend (Hono o Express en Workers, o Fastify en un servicio Node).

## 2026-10-15 — ADR-001 aprobada; arranque en modo transitorio

- **Se decidió o produjo:** Franco y Anthony **aprueban ADR-001**. Punto de partida confirmado: el MVP y el piloto corren con **números del portafolio de Numi** (conexión manual, `RF-PLT-WA-02`), y el **objetivo final es Tech Provider** con Embedded Signup, migrando sin reescribir la integración. D-01 pasa a Aprobada en el tablero de CLAUDE.md §5.

- **Quedó abierto:**
  - Límite del modo transitorio: **2 clientes** mientras el negocio no esté verificado ante Meta, **20** una vez verificado. El piloto del 1 de diciembre con tres nichos necesita la **verificación de negocio** resuelta; Tech Provider no es requisito para arrancar.
  - Actualizaciones pendientes en documentos fuente (§3, §8.1 y §12 de la definición del MVP; §5.5 de la consola).

- **Siguiente paso:** D-02, alcance de la consola (completa o versión mínima).

## 2026-10-14 — D-01, modelo de conexión de WhatsApp

- **Se decidió o produjo:** `docs/adr/001-modelo-conexion-whatsapp.md` en estado Propuesta. Modelo objetivo: **Tech Provider con Embedded Signup**, cada negocio con su propio portafolio y su cuenta de WhatsApp. El piloto puede arrancar en modo transitorio con números de Numi, sobre la misma interfaz de canal. Queda fijado que las credenciales de WhatsApp son **por organización** (`waba_id`, `phone_number_id`, token cifrado), nunca un token global.

  Hallazgo que definió la decisión: los límites de Meta son **por portafolio y compartidos entre todos sus números** (250 destinatarios únicos cada 24 h; 2 números sin verificación, 20 con ella). El plan de alojar los números de todos los clientes en el portafolio de Numi no escala: un cliente consumiría la cuota de los demás y hay un tope duro de 20 clientes.

  Se confirmó además que el segmento inicial pasa a ser inmobiliarias, clínicas estéticas y odontologías, con **el mismo producto** para los tres; los nichos específicos se analizan después de validar la idea.

- **Quedó abierto:**
  - Aprobación de ADR-001 por Franco y Anthony (hoy en estado Propuesta).
  - **Riesgo principal del proyecto:** la app de Numi AI ya está publicada y aprobada, pero la verificación de negocio ante Meta está frenada porque faltan datos, y después queda la habilitación como Tech Provider. Son dos pasos, es la ruta crítica y no la controla el equipo. Si no se resuelve hacia mediados de noviembre, el piloto queda limitado a dos clientes.
  - Por confirmar con Meta: qué permisos tiene aprobados la app (`whatsapp_business_management`, `whatsapp_business_messaging`) y si la sección de Tech Provider está visible o bloqueada hasta verificar el negocio.
  - Actualizaciones pendientes en los documentos fuente: §3 de la definición del MVP (segmento), §8.1 y §12 (modelo Tech Provider y ruta crítica), y §5.5 de la consola (`RF-PLT-WA-08` sube a P0 al obtener el rol).

- **Siguiente paso:** D-02, alcance de la consola (completa o versión mínima). En paralelo y fuera de la arquitectura: destrabar la verificación de negocio con Meta.

## 2026-10-14 — Repositorio creado

- **Se decidió o produjo:** repositorio con CLAUDE.md y los tres documentos de docs/ (definición del MVP v2.2, documento maestro v1.1, consola de plataforma v1.1).
- **Quedó abierto:** las 11 decisiones de la sección 5 de CLAUDE.md.
- **Siguiente paso:** D-01, modelo de conexión de WhatsApp.
